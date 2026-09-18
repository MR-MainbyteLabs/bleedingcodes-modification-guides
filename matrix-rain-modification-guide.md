# matrix-rain — Modification Guide
**File:** `matrix_rain.py` (single-file script)  
**Version:** 1.0.1  
**Entry point:** `main()`  
**Dependencies:** `pygame`, `python-xlib`  
**Platform:** Linux, X11 only (not Wayland)

---

## Who This Document Is For

Engineers and developers who want to extend or adapt the visual behavior, rendering pipeline, or X11 integration of matrix-rain. This is not a usage guide — the README covers that. This document tells you where the code lives and exactly where to make each class of change.

---

## Architecture Overview

```
main()
  ├── parse_args()             # All visual parameters come from CLI
  ├── Parameter validation     # Clamps, swaps, guards
  ├── WID conversion           # hex → decimal for SDL_WINDOWID
  ├── pygame.init()
  ├── apply_window_hints()     # X11: set desktop-level window hints via python-xlib
  ├── build_char_cache()       # Pre-renders all (char, color) combinations to surfaces
  ├── Column initialization    # make_column() per column slot
  └── Main loop
        ├── X11 re-stack (every 30 frames)
        ├── Fade blit          # Semi-transparent black overlay = trail fade effect
        ├── Column rendering   # Per column: draw head + trail chars
        ├── Column advance     # y += speed * speed_jitter
        ├── Column reset       # make_column() when trail scrolls off screen
        └── pygame.display.update()
```

---

## CLI Parameters

All visual configuration is done via CLI flags. There are no config files.

| Flag | Default | What it controls |
|---|---|---|
| `wid` | required | Window ID from xwinwrap (`%WID`) |
| `--fps` | `15` | Frame rate |
| `--font-size` | `24` | Font size in pixels; also controls column spacing |
| `--density` | `0.65` | Fraction of column slots that spawn an active column |
| `--speed-min` | `4.0` | Minimum column fall speed (px/frame) |
| `--speed-max` | `10.0` | Maximum column fall speed (px/frame) |
| `--trail-min` | `7` | Minimum trail length (character count) |
| `--trail-max` | `55` | Maximum trail length (character count) |
| `--trail-spacing` | `2.0` | Vertical spacing multiplier between trail characters |
| `--spacing-variance` | `0.2` | ±variance applied to `trail-spacing` per column |
| `--trail-taper` | `0.5` | How fast spacing widens toward the tail (0 = uniform, higher = more spread) |
| `--fade-alpha` | `45` | Alpha of the black fade overlay (0 = no fade/solid trails, 255 = instant erase) |
| `--charset` | `ascii_letters + digits` | Characters drawn in the rain |

### Adding a New CLI Parameter

1. Add to `parse_args()`:
```python
parser.add_argument("--glow", type=float, default=0.0,
    help="Glow intensity (0.0 = off)")
```

2. Validate / clamp in `main()` after `parse_args()`:
```python
args.glow = clamp(args.glow, 0.0, 1.0)
```

3. Use `args.glow` in the rendering loop or wherever relevant.

---

## Window ID Conversion

xwinwrap passes `%WID` as a hex string (e.g. `0x1e00004`). SDL's `SDL_WINDOWID` environment variable must be a **decimal** integer string. `C atoi()` — which SDL uses internally — stops parsing at `'x'`, so a hex string produces WID=0, causing SDL to open a new floating window instead of embedding in the desktop.

```python
wid_decimal = str(int(args.wid, 0))   # int(x, 0) handles both 0x... and decimal
os.environ["SDL_WINDOWID"] = wid_decimal
```

`int(x, 0)` with base 0 auto-detects hex (`0x`), octal (`0o`), and decimal. Do not change this to `int(args.wid)` — that would break hex input.

---

## X11 Window Hints — `apply_window_hints()`

Called once after `pygame.init()`. Sets four `_NET_WM_STATE` atoms and `_NET_WM_WINDOW_TYPE_DESKTOP` on the pygame window via python-xlib so it stays below all other windows.

```python
hints = [
    _NET_WM_STATE_SKIP_TASKBAR,   # not in taskbar
    _NET_WM_STATE_SKIP_PAGER,     # not in pager/switcher
    _NET_WM_STATE_BELOW,          # always below other windows
    _NET_WM_STATE_STICKY,         # visible on all virtual desktops
]
```

Returns `(xdisplay, xwin)` on success, `(None, None)` on failure. The main loop checks for `None` before calling X11 methods — a failure here is non-fatal and logged as a warning. The script runs as a normal window if xlib is unavailable.

**Important:** `xdisplay.flush()` is used (not `sync()`). `flush()` is non-blocking — it sends buffered commands to the X server without waiting for a reply. `sync()` would block the render loop.

### Changing X11 Window Hints

To remove "always below" (e.g. to test rendering without wallpaper behavior):

```python
hints = [
    xdisplay.intern_atom("_NET_WM_STATE_SKIP_TASKBAR"),
    xdisplay.intern_atom("_NET_WM_STATE_SKIP_PAGER"),
    # Remove _NET_WM_STATE_BELOW
]
```

### X11 Re-stack Rate

The main loop re-applies `stack_mode=X.Below` every 30 frames to keep the window behind others after window manager resets:

```python
XSYNC_INTERVAL = 30   # re-stack every 30 frames (~2s at 15fps)
```

To change the interval (e.g. less aggressive re-stacking):
```python
XSYNC_INTERVAL = 60   # every 60 frames (~4s at 15fps)
```

---

## Column Data Structure — `make_column(x, height, args)`

Each column is a plain dict:

```python
{
    "x":            int,     # pixel x position (index * font_size)
    "y":            int,     # current head pixel y position (starts negative — offscreen above)
    "speed":        float,   # pixels per frame
    "trail":        int,     # number of characters in the trail
    "spacing":      float,   # vertical spacing multiplier for this column
    "speed_jitter": float,   # per-column random speed multiplier (0.85–1.15)
}
```

`y` starts as a random value in `[-height, 0]` so columns begin staggered rather than all at the top simultaneously.

`reset_column()` calls `make_column()` with the same `x` to randomize all other fields for a new pass.

### Adding a Column Property

Example — add per-column color shift:

```python
def make_column(x, height, args):
    col = { ... existing fields ... }
    col["hue_shift"] = random.randint(0, 30)   # degrees to shift green hue
    return col
```

Then use `col["hue_shift"]` in the rendering loop when computing `color`.

---

## Character Cache — `build_char_cache(font, charset, colors)`

Pre-renders every `(char, color)` combination to a `pygame.Surface` at startup. The main loop indexes into this dict instead of calling `font.render()` per frame — critical for maintaining frame rate at high density.

```python
return {
    (char, color): font.render(char, True, color)
    for color in colors
    for char in charset
}
```

Cache size = `len(charset) × len(colors)`. Default: 62 chars × 7 colors = 434 surfaces.

**If you add colors**, pass them in the `colors` list argument in `main()`:
```python
char_cache = build_char_cache(font, args.charset, [head] + greens)
```

**If you change the charset at runtime** (post-init), the cache must be rebuilt — call `build_char_cache()` again.

---

## Colors

Defined in `main()` as local variables:

```python
black = (0, 0, 0)
head  = (200, 255, 200)     # bright near-white green — head character
greens = [
    (0, 255, 0),            # greens[0] — brightest trail
    (0, 210, 0),
    (0, 170, 0),
    (0, 120, 0),
    (0, 80,  0),
    (0, 45,  0),            # greens[5] — dimmest tail
]
```

The head character (trail index `i == 0`) always uses `head`. Trail characters use `greens[min(i, len(greens) - 1)]` — the last green is repeated for any trail longer than 6.

### Changing to a Different Color Scheme

Replace `head` and `greens`:

```python
# Blue scheme:
head   = (200, 200, 255)
greens = [
    (0, 0, 255),
    (0, 0, 210),
    (0, 0, 170),
    (0, 0, 120),
    (0, 0, 80),
    (0, 0, 45),
]
```

Rebuild the cache after changing colors:
```python
char_cache = build_char_cache(font, args.charset, [head] + greens)
```

### Adding More Trail Color Steps

Add entries to `greens`. The `min(i, len(greens) - 1)` clamp handles any trail length automatically — no other changes needed.

```python
greens = [
    (0, 255, 0),
    (0, 230, 0),
    (0, 200, 0),
    (0, 170, 0),
    (0, 140, 0),
    (0, 110, 0),
    (0, 80,  0),
    (0, 50,  0),
    (0, 25,  0),    # 9 steps now
]
```

---

## Trail Rendering — Main Loop

The core rendering block per column:

```python
for i in range(column["trail"]):
    tapered_spacing = spacing * (1 + i * args.trail_taper)
    extra = HEAD_GAP_PX if i >= 1 else 0
    y = column["y"] - (i * args.font_size * tapered_spacing + extra)

    if 0 <= y < height:
        char = random.choice(args.charset)
        color = head if i == 0 else greens[min(i, len(greens) - 1)]
        screen.blit(char_cache[(char, color)], (x, int(y)))
```

`i == 0` is the head character (lowest on screen, moves first). `i == 1` onward is the trail going upward.

`HEAD_GAP_PX = args.font_size` — one character-height of fixed extra space between the head and the first trail character. This is additive, not multiplicative, which prevents the visual discontinuity the old `head_gap=1.5` multiplier caused (it made the i=1 gap larger than i=2).

`trail_taper` widens the gap between successive trail characters toward the tail. At `0.0`, spacing is uniform. At `0.5` (default), the spacing grows linearly — characters spread out toward the top of the trail.

### Changing Characters per Frame

Currently `random.choice(args.charset)` is called every frame for every visible character. Characters flicker every frame — this is the classic Matrix effect.

**To freeze characters** (each column shows the same chars every frame):

Store characters in the column dict:
```python
def make_column(x, height, args):
    ...
    col["chars"] = [random.choice(args.charset) for _ in range(col["trail"])]
    return col
```

In the render loop, use `col["chars"][i]` instead of `random.choice(args.charset)`. To re-randomize periodically, update `col["chars"]` in `reset_column()`.

### Changing Column Spacing

Column x positions are assigned at initialization:

```python
col_count = max(1, width // args.font_size)
for index in range(col_count):
    if random.random() < args.density:
        columns.append(make_column(index * args.font_size, height, args))
```

`args.font_size` is the column pitch — columns are spaced exactly one font-size apart. To add gaps between columns:

```python
pitch = int(args.font_size * 1.5)   # 50% wider gaps
col_count = max(1, width // pitch)
for index in range(col_count):
    if random.random() < args.density:
        columns.append(make_column(index * pitch, height, args))
```

---

## Fade Effect

```python
fade = pygame.Surface((width, height))
fade.set_alpha(args.fade_alpha)   # default: 45
fade.fill(black)
```

Each frame: `screen.blit(fade, (0, 0))` draws a semi-transparent black rectangle over the entire screen. This gradually darkens previous frames, creating the trail fade. Higher `--fade-alpha` = faster fade = shorter visible trails. Lower = slower fade = longer glowing trails.

**`--fade-alpha 0`**: trails never fade — screen fills up permanently.  
**`--fade-alpha 255`**: complete blackout each frame — no trails at all, just the head character.

---

## Signal Handling

```python
running = True

def stop_script(signum=None, frame=None):
    global running
    running = False

signal.signal(signal.SIGINT, stop_script)
signal.signal(signal.SIGTERM, stop_script)
```

Both `SIGINT` (Ctrl+C) and `SIGTERM` (systemd stop) set `running = False`, which exits the main loop. The `finally` block in `main()` closes the X display connection and calls `pygame.quit()`.

The X display connection is explicitly closed (`xdisplay.close()`) in the `finally` block to release the X11 connection resource. Without this, the connection leaks on exit.

---

## Charset

Default: `string.ascii_letters + string.digits` — 62 characters.

**Katakana (classic Matrix look):**
```python
parser.add_argument("--charset", default="".join(chr(c) for c in range(0x30A0, 0x30FF)))
```

Or pass at runtime:
```bash
xwinwrap ... -- python3 matrix_rain.py %WID --charset "アイウエオカキクケコ"
```

**Note:** Non-ASCII characters require a font that supports them. `pygame.font.SysFont("monospace", size, bold=True)` may not render them correctly. Use a specific font:

```python
font = pygame.font.Font("/usr/share/fonts/truetype/noto/NotoSansCJK-Regular.ttc", args.font_size)
```

---

## Running Without xwinwrap (Windowed Mode for Testing)

The script requires a WID argument. To test in a regular window without xwinwrap, pass a fake WID and let `apply_window_hints()` fail gracefully:

```bash
python3 matrix_rain.py 0
```

`apply_window_hints()` will fail (WID 0 is not a real window) and return `(None, None)`. The script continues and renders in whatever window SDL opens. The X11 re-stack code in the main loop checks for `None` and skips safely.

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Change fall speed | `--speed-min` / `--speed-max` CLI args or `make_column()` `"speed"` field |
| Change trail length | `--trail-min` / `--trail-max` CLI args |
| Change trail fade | `--fade-alpha` CLI arg (0=no fade, 255=instant erase) |
| Change column density | `--density` CLI arg (0.05–1.0) |
| Change character set | `--charset` CLI arg |
| Change to non-ASCII charset | `--charset` arg + swap `SysFont` for a font that supports it |
| Change color scheme | `head` and `greens` locals in `main()` + rebuild `char_cache` |
| Add more trail color steps | Add entries to `greens` list |
| Add per-column color variation | Add field to `make_column()`, use in render loop |
| Freeze characters (no per-frame flicker) | Store chars list in column dict, use instead of `random.choice` |
| Change column pitch (horizontal spacing) | `col_count` and `make_column(index * pitch, ...)` in `main()` |
| Change X11 re-stack frequency | `XSYNC_INTERVAL` constant in `main()` |
| Change head gap between head and first trail char | `HEAD_GAP_PX` constant in `main()` |
| Test without xwinwrap | Pass `0` as WID; renders in a regular SDL window |
| Add a new CLI parameter | `parse_args()` + validation block after `parse_args()` call |
| Change font | `pygame.font.SysFont(...)` or `pygame.font.Font(path, size)` in `main()` |
| Remove always-below X11 hint | Remove `_NET_WM_STATE_BELOW` from `hints` in `apply_window_hints()` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
