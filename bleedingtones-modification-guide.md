# bleedingtones — Modification Guide
**Version:** 1.0.0  
**Package:** `src/bleedingtones/`  
**Entry points:** four CLI tools via `pyproject.toml`  
**Dependency:** `pygame >= 2.5`

---

## Who This Document Is For

Engineers and developers who want to extend, adapt, or embed bleedingtones beyond what the four CLI tools expose. This is not a usage guide — the README covers that. This document tells you where the code lives, what each module owns, and exactly where to make each class of change.

---

## Architecture Overview

```
engine.py          ← shared audio backend used by all four tools
    ↑
chaos.py           ← play one random sound and exit
ambient.py         ← continuous shuffle loop
scheduler.py       ← timed trigger loop
notify.py          ← category-aware dispatcher
```

All four tools follow the same pattern: parse args → call `engine.collect()` → call `engine.pick()` → call `engine.play()`. The engine is the only file that touches pygame.

---

## Module Map

| Module | File | Owns |
|---|---|---|
| `engine` | `src/bleedingtones/engine.py` | pygame init, file collection, file selection, playback, teardown |
| `chaos` | `src/bleedingtones/chaos.py` | Single-shot random play with optional duration cap |
| `ambient` | `src/bleedingtones/ambient.py` | Continuous shuffle loop with gap and no-repeat |
| `scheduler` | `src/bleedingtones/scheduler.py` | Timed trigger — fixed interval or random window |
| `notify` | `src/bleedingtones/notify.py` | Category subfolder dispatch, `--strict` fallback control, `--list` |

---

## Engine — `engine.py`

This is the only file that imports or touches pygame.

### Constants

```python
SUPPORTED = {".mp3", ".wav", ".ogg", ".flac"}
```

**To add a format:**
```python
SUPPORTED = {".mp3", ".wav", ".ogg", ".flac", ".aiff"}
```

pygame's mixer supports whatever formats the underlying SDL_mixer library was built with. On most Linux systems that includes AIFF, but verify with your pygame build before adding it.

### `_pygame()` — Lazy Init

```python
def _pygame():
    """Lazy import + init so tools that don't need audio don't pay for it."""
```

Imports pygame, calls `pygame.init()` and `pygame.mixer.init()` if not already done, returns the module. Called at the start of `play()` only. This means importing any bleedingtones module does not initialize pygame — only calling `play()` does.

**To change mixer settings** (e.g. sample rate, buffer size):

```python
def _pygame():
    try:
        import pygame
    except ImportError:
        sys.exit("[bleedingtones] pygame is not installed.\nFix: pip install pygame")
    if not pygame.get_init():
        pygame.init()
    if not pygame.mixer.get_init():
        pygame.mixer.init(frequency=44100, size=-16, channels=2, buffer=512)
    return pygame
```

Default pygame mixer settings are usually fine. Change only if you experience audio latency or quality issues.

### `collect(path: Path) -> list[Path]`

Non-recursive. Lists all files in `path` whose suffix (lowercased) is in `SUPPORTED`. Exits with an error message if `path` is not a directory or if no supported files are found.

**To make collection recursive:**
```python
def collect(path: Path) -> list[Path]:
    if not path.is_dir():
        sys.exit(f"[bleedingtones] Sound folder not found: {path}")
    files = [
        f for f in path.rglob("*")      # rglob instead of iterdir
        if f.is_file() and f.suffix.lower() in SUPPORTED
    ]
    if not files:
        sys.exit(...)
    return files
```

Note: if you make `collect()` recursive, `notify.py`'s category system (which relies on direct subfolders) still works correctly — it calls `collect(category_subfolder)` not `collect(root)`.

### `pick(files: list[Path]) -> Path`

```python
def pick(files: list[Path]) -> Path:
    return random.choice(files)
```

Pure random selection, uniform distribution. **To change selection strategy:**

```python
# Weighted by file modification time (newer = more likely):
import os
weights = [os.path.getmtime(f) for f in files]
return random.choices(files, weights=weights, k=1)[0]

# Round-robin (requires external state — pass an index or use itertools.cycle):
# Better handled in the tool that calls pick() rather than here
```

### `play(sound_path: Path, duration: float | None = None) -> None`

```python
pygame.mixer.music.load(str(sound_path))
pygame.mixer.music.play()

if duration is not None:
    time.sleep(duration)
    pygame.mixer.music.stop()
else:
    while pygame.mixer.music.get_busy():
        time.sleep(0.1)

pygame.mixer.music.unload()
pygame.quit()
```

- `duration=None` plays the full track via busy-poll loop (0.1s polling interval).
- `duration=N` plays for N seconds then stops.
- `pygame.quit()` is called after every play — pygame is fully torn down on each invocation. This is intentional for single-shot tools (`chaos`, `notify`). For continuous tools (`ambient`, `scheduler`) this means pygame is re-initialized on every call to `play()` — acceptable since `_pygame()` handles the lazy init.

**To change the busy-poll interval:**
```python
while pygame.mixer.music.get_busy():
    time.sleep(0.05)   # 50ms instead of 100ms
```

**To add a volume control:**
```python
def play(sound_path: Path, duration: float | None = None, volume: float = 1.0) -> None:
    pygame = _pygame()
    pygame.mixer.music.load(str(sound_path))
    pygame.mixer.music.set_volume(max(0.0, min(1.0, volume)))
    pygame.mixer.music.play()
    ...
```

Then expose `--volume` in whichever tools need it and pass it through to `engine.play()`.

---

## chaos — `chaos.py`

Single-shot tool. Collect → pick → play → exit. The simplest tool in the package.

### Key Parameters

```python
DEFAULT_DURATION = 30.0   # seconds — used when --full is not set
```

`--full` sets `duration=None` (plays entire track). `--duration N` overrides the default. Without `--full`, chaos plays at most `DEFAULT_DURATION` seconds.

**To change default duration:**
```python
DEFAULT_DURATION = 15.0
```

**To make full-track the default** (remove the duration cap):
```python
engine.play(chosen, duration=None if args.full else None)
# or just always pass None and remove --duration entirely
```

### `--list` flag

Lists all available sounds and exits without playing. Useful for scripting.

---

## ambient — `ambient.py`

Continuous loop. Plays one sound to completion, waits `--gap` seconds, picks the next one, repeats until `SIGINT` or `SIGTERM`.

### Key Parameters

| Flag | Default | Effect |
|---|---|---|
| `--gap` | `2.0` | Seconds of silence between tracks |
| `--no-repeat` | off | When on and `len(files) > 1`, excludes the last-played file from the next pick |

### No-Repeat Logic

```python
candidates = [f for f in files if f != last] if args.no_repeat and len(files) > 1 else files
chosen = engine.pick(candidates)
last = chosen
```

The guard `len(files) > 1` prevents an empty `candidates` list when there's only one file.

**To extend no-repeat to a history window** (exclude last N, not just last 1):

```python
from collections import deque
history = deque(maxlen=3)   # don't repeat any of the last 3

# In the loop:
candidates = [f for f in files if f not in history] if args.no_repeat and len(files) > len(history) else files
chosen = engine.pick(candidates)
history.append(chosen)
```

### Signal Handling

`SIGINT` and `SIGTERM` both set `_running = False`. The main loop checks `_running` before sleeping after each track. Ctrl+C exits cleanly after the current track finishes (or immediately if in the gap sleep, since `_running` is checked before `time.sleep(args.gap)`).

---

## scheduler — `scheduler.py`

Timed trigger. Waits N seconds, plays a sound, waits again, repeats.

### Key Parameters

| Flag | Default | Mode |
|---|---|---|
| `--interval N` | — | Fixed interval — exactly N seconds between sounds |
| `--min N` | `60.0` | Random interval lower bound |
| `--max N` | `300.0` | Random interval upper bound |
| `--count N` | — | Stop after N sounds (default: run forever) |

`--interval` and `--min`/`--max` are mutually exclusive. If `--interval` is set, `--min`/`--max` are ignored. If neither is set, random mode uses 60–300s defaults.

### Sleep Granularity

The wait is not a single `time.sleep(wait)` — it sleeps in 0.25s chunks to keep `Ctrl+C` responsive:

```python
elapsed = 0.0
while elapsed < wait and _running:
    time.sleep(0.25)
    elapsed += 0.25
```

**To change responsiveness** (0.25s chunk = max latency to respond to Ctrl+C):
```python
time.sleep(0.1)
elapsed += 0.1
```

### Adding a --category Flag

To make the scheduler category-aware (draw from a subfolder on each trigger), import notify logic:

```python
# In scheduler.py, after args parse:
if hasattr(args, 'category') and args.category:
    target = args.path / args.category
    if not target.is_dir():
        sys.exit(f"[bleedingtones] Category not found: {target}")
    files = engine.collect(target)
else:
    files = engine.collect(args.path)
```

And add `--category` to the parser.

---

## notify — `notify.py`

Category-aware dispatcher. Reads from a named subfolder of the root sound directory.

### Folder Structure

```
~/Sounds_FX/          ← --path (root)
    win/              ← --category win
    fail/             ← --category fail
    alert/            ← --category alert
    done/             ← --category done
```

### Fallback Behavior

```python
if not target.is_dir():
    if args.strict:
        sys.exit(...)
    else:
        print(f"Category '{args.category}' not found — falling back to root", file=sys.stderr)
        target = root
```

Without `--strict`: missing category → logs a warning to stderr → falls back to root. With `--strict`: missing category → exits with code 1. This makes notify safe to use in scripts where a missing category should be a hard error.

### Exit Codes

- `0` — played successfully
- `1` — error (category not found with `--strict`, no files found, root not found)

These are pipe-friendly. Use in Makefiles and shell scripts:

```bash
make build && bleedingtones-notify --category win \
    || bleedingtones-notify --category fail
```

### `--list` Flag

Lists all category subfolders and the sound count in each:

```python
cats = [d for d in sorted(root.iterdir()) if d.is_dir()]
for cat in cats:
    sounds = engine.collect(cat) if any(
        f.suffix.lower() in engine.SUPPORTED for f in cat.iterdir()
    ) else []
    print(f"  {cat.name}/  ({len(sounds)} sound(s))")
```

### Adding a Default Category

To make notify default to a specific category when `--category` is not given:

```python
parser.add_argument(
    "--category",
    type=str,
    default="done",   # Change None to a default category name
    help="Subfolder name to draw from (default: done)",
)
```

---

## Using the Engine as a Library

The engine is importable independently of the CLI tools:

```python
from pathlib import Path
from bleedingtones.engine import collect, pick, play

files = collect(Path("/home/user/Sounds_FX/win"))
chosen = pick(files)
play(chosen, duration=None)    # full track
# or
play(chosen, duration=5.0)     # 5 seconds only
```

This is the pattern to use when embedding bleedingtones into a larger script or build system.

**Triggering on a build event:**

```python
import subprocess
from pathlib import Path
from bleedingtones.engine import collect, pick, play

result = subprocess.run(["make", "build"])
sounds_root = Path.home() / "Sounds_FX"

if result.returncode == 0:
    play(pick(collect(sounds_root / "win")))
else:
    play(pick(collect(sounds_root / "fail")))
```

---

## CLI Entry Points — `pyproject.toml`

```toml
[project.scripts]
bleedingtones-chaos     = "bleedingtones.chaos:main"
bleedingtones-ambient   = "bleedingtones.ambient:main"
bleedingtones-scheduler = "bleedingtones.scheduler:main"
bleedingtones-notify    = "bleedingtones.notify:main"
```

After `pip install .`, all four entry points are available as commands. To add a new tool:

1. Create `src/bleedingtones/mytool.py` with a `main()` function.
2. Add to `[project.scripts]`:
   ```toml
   bleedingtones-mytool = "bleedingtones.mytool:main"
   ```
3. Reinstall: `pip install -e .`

---

## Default Sound Folder

All four tools default to `Path.home() / "Sounds_FX"` (`~/Sounds_FX`). To change the default system-wide:

In each tool's `parse_args()`, change the `default=` for `--path`:
```python
parser.add_argument(
    "--path",
    type=Path,
    default=Path("/opt/sounds"),   # change here
    ...
)
```

Or set `BLEEDINGTONES_PATH` as an environment variable and read it:
```python
import os
DEFAULT_PATH = Path(os.environ.get("BLEEDINGTONES_PATH", Path.home() / "Sounds_FX"))
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add a new audio format | `engine.py` → `SUPPORTED` set |
| Change default sound folder | All four tools → `--path` default in `parse_args()` |
| Make file collection recursive | `engine.py` → `collect()` — change `iterdir()` to `rglob("*")` |
| Change file selection strategy | `engine.py` → `pick()` |
| Add volume control | `engine.py` → `play()` add `volume` param + `set_volume()` |
| Change mixer settings (sample rate, buffer) | `engine.py` → `_pygame()` → `pygame.mixer.init(...)` |
| Change default chaos duration | `chaos.py` → `DEFAULT_DURATION` constant |
| Make chaos play full track by default | `chaos.py` → `engine.play(chosen, duration=None)` always |
| Change ambient gap between tracks | `--gap` default in `ambient.py` `parse_args()` |
| Extend no-repeat to a history window | `ambient.py` main loop — replace `last` with `deque(maxlen=N)` |
| Change scheduler responsiveness to Ctrl+C | `scheduler.py` — sleep chunk size in the wait loop |
| Add category awareness to scheduler | `scheduler.py` — add `--category` flag, set `target` before `collect()` |
| Make notify default to a category | `notify.py` → `--category` `default=` in `parse_args()` |
| Add a new CLI tool | New `src/bleedingtones/mytool.py` + entry point in `pyproject.toml` |
| Use engine in a script without CLI | `from bleedingtones.engine import collect, pick, play` |
| Trigger on build events | Library usage pattern — call `play()` based on subprocess return code |

---

## Version Bump Procedure

Per MainbyteLabs QA patch release rule:

1. `pyproject.toml` → `version = "X.Y.Z"`
2. `src/bleedingtones/__init__.py` → `__version__ = "X.Y.Z"`
3. Bug fixes only → patch bump (e.g. `1.0.0` → `1.0.1`)
4. Push to main, create release tag on GitHub.

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
