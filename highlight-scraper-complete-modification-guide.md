# Highlight Scraper — Complete Modification Guide

**Version:** 0.4.1  
**Covers:** Every source file in the package  
**Author:** MainbyteLabs  

---

## How to use this guide

This document covers both visual changes (GUI appearance) and program
changes (capture logic, storage, commands, export formats, and how
everything connects). Each section maps to one source file. You do not
need to understand the whole program to make a change in one file —
each section tells you exactly what to edit and what the downstream
effect is.

**After any change to a `.py` file:**
```bash
pip install -e . --break-system-packages   # only needed if you change pyproject.toml
highlight-scraper-gui                       # restart the GUI
```

For CLI changes, the updated code is picked up the next time you run
`highlight-scraper <command>` — no reinstall needed unless you added a
new console script entry.

---

## Architecture — How the Files Connect

Understanding which file calls which saves you from making a change in
the wrong place.

```
                  ┌──────────────────────────────────────────┐
                  │           Three front ends               │
                  │                                          │
                  │  cli.py      gui.py      tray.py         │
                  │  (terminal)  (window)    (system tray)   │
                  └──────────────┬───────────────────────────┘
                                 │ all three create a
                                 ▼
                        ┌─────────────────┐
                        │ session_manager  │
                        │ .CaptureSession  │
                        └────────┬────────┘
                                 │ owns and drives
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
            ┌─────────────┐  ┌──────────┐  ┌──────────────┐
            │ watcher_core│  │ storage  │  │ source_info  │
            │ .Selection  │  │ .Capture │  │ (window/app  │
            │  Watcher    │  │  Store   │  │  attribution)│
            └─────────────┘  └──────────┘  └──────────────┘
                    │                ▲
                    │ fires          │ written to by
                    │ on_change(text)│ session_manager
                    └────────────────┘

            ┌──────────┐   ┌────────────┐   ┌──────────────┐
            │ export   │   │ config     │   │ hotkeys      │
            │ .py      │   │ .py        │   │ .py          │
            │(md/csv/  │   │(loads JSON │   │(Ctrl+Alt+1/2/│
            │ json out)│   │ settings)  │   │ 3 shortcuts) │
            └──────────┘   └────────────┘   └──────────────┘

            ┌───────────────────┐
            │ paste_on_hold.py  │
            │ (click-hold paste │
            │  gesture)         │
            └───────────────────┘
```

**Rule of thumb for where to make a change:**

| What you want to change | File |
|---|---|
| How the window looks | `gui.py` |
| What appears in the GUI | `gui.py` |
| Search/filter the capture list | `gui.py` + `storage.py` |
| Delete a capture | `storage.py` + `gui.py` |
| Auto-export on a schedule | `session_manager.py` + `config.py` |
| System tray menu items | `tray.py` |
| CLI commands | `cli.py` |
| What text gets captured vs ignored | `session_manager.py` |
| When captures get merged into one row | `session_manager.py` |
| How the X11/Wayland selection is read | `watcher_core.py` |
| How fast polling runs | `watcher_core.py` or `config.py` |
| Database schema (add/remove fields) | `storage.py` |
| Database queries | `storage.py` |
| Wayland window attribution | `source_info.py` |
| Export file formats | `export.py` |
| Persistent user settings | `config.py` |
| Keyboard shortcuts for tagging | `hotkeys.py` |
| Click-and-hold gesture timing | `paste_on_hold.py` |
| Where the app detects X11 vs Wayland | `watcher_core.py` → `is_wayland()` |

---

## Part 1 — `config.py` — Settings and Defaults

**Location:** `src/highlight_scraper/config.py`  
**What it does:** Loads `~/.config/highlight_scraper/config.json` on startup.
If the file doesn't exist it creates it with defaults. Every other file
calls `load_config()` to get its settings.

### Adding a new persistent setting

1. Add your key and default value to the `DEFAULTS` dict:

```python
DEFAULTS = {
    "poll_interval": 0.3,
    "hold_seconds": 0.45,
    "preview_chars": 300,
    "min_length": 3,
    "merge_window_seconds": 2.0,
    "excluded_apps": [...],
    "db_path": "...",
    "my_new_setting": 99,   # add your key here with its default
}
```

2. Read it anywhere in the program with:
```python
config = load_config()
value = config.get("my_new_setting", 99)
```

3. Write it from the GUI Settings panel or from any code:
```python
from highlight_scraper.config import load_config, save_config
config = load_config()
config["my_new_setting"] = new_value
save_config(config)
```

### Changing the config file location

At the top of `config.py`:
```python
CONFIG_DIR  = Path.home() / ".config" / "highlight_scraper"
CONFIG_PATH = CONFIG_DIR / "config.json"
```

Change either path. `CONFIG_DIR` is also where the directory is created
(`CONFIG_DIR.mkdir(parents=True, exist_ok=True)` in `save_config()`).

### Changing the default database location

In `DEFAULTS`:
```python
"db_path": str(Path.home() / ".local" / "share" / "highlight_scraper" / "captures.db"),
```

Change the path. The directory is created automatically by `storage.py`
(`self.db_path.parent.mkdir(parents=True, exist_ok=True)`).

### All settings and what they control

| Key | Default | Controls |
|---|---|---|
| `poll_interval` | `0.3` | Seconds between clipboard reads in polling fallback mode |
| `hold_seconds` | `0.45` | How long to hold a click before paste-on-hold fires |
| `preview_chars` | `300` | Characters of each capture shown in GUI list |
| `min_length` | `3` | Captures shorter than this are silently ignored |
| `merge_window_seconds` | `2.0` | Drag selections within this window get merged into one row |
| `excluded_apps` | list | App names — captures from these are silently dropped |
| `db_path` | `~/.local/share/...` | Full path to the SQLite database file |
| `entries_shown` | (GUI default: 10) | GUI list entries shown — written by Settings panel |
| `auto_export_every` | `0` (off) | Export every N captures automatically — 0 disables |
| `auto_export_path` | `~/highlights_<session>.md` | File path for auto-export output |

---

## Part 2 — `storage.py` — The Database

**Location:** `src/highlight_scraper/storage.py`  
**What it does:** Owns the SQLite database. All reads and writes go
through `CaptureStore`. Nothing else touches the database file directly.

### Current schema

```sql
CREATE TABLE IF NOT EXISTS captures (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    text          TEXT    NOT NULL,
    captured_at   TEXT    NOT NULL,
    session       TEXT    NOT NULL,
    source_window TEXT,
    source_app    TEXT,
    tag           TEXT
);
```

### Current public methods

| Method | What it does |
|---|---|
| `add_capture(text, session, source_window, source_app)` | Insert a new capture row |
| `update_text_for_last(text, session)` | Overwrite the most recent row's text (merge) |
| `set_tag_for_last(tag, session)` | Tag the most recent capture |
| `delete_capture(row_id)` | **0.4.0** — Delete one row by id, returns True/False |
| `search(query, session, limit)` | **0.4.0** — Case-insensitive substring search |
| `query(session, limit)` | Fetch captures, most recent first |
| `sessions()` | List all sessions with count and timestamps |
| `close()` | Close the database connection |

### Using `delete_capture()`

```python
# Delete a specific row by its id
deleted = store.delete_capture(row_id=42)
if deleted:
    print("Deleted")
else:
    print("Row not found")
```

The method is thread-safe — safe to call while a session is actively
capturing in the background. The GUI right-click menu calls this directly;
to add it to the CLI see Part 9.

### Using `search()`

```python
# Search across all sessions
rows = store.search("neural network")

# Search within one session only
rows = store.search("neural network", session="research_2024")

# Limit results
rows = store.search("python", limit=50)
```

Returns a list of `Capture` dataclass instances, most recent first.
The search is case-insensitive and matches any substring.

### Adding a new column to the database

**Step 1** — Add the column to the `SCHEMA` string:
```python
SCHEMA = """
CREATE TABLE IF NOT EXISTS captures (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    text          TEXT NOT NULL,
    captured_at   TEXT NOT NULL,
    session       TEXT NOT NULL,
    source_window TEXT,
    source_app    TEXT,
    tag           TEXT,
    url           TEXT    -- add new columns here
);
"""
```

**Step 2** — Add the field to the `Capture` dataclass:
```python
@dataclass
class Capture:
    id: int
    text: str
    captured_at: str
    session: str
    source_window: str
    source_app: str
    tag: str
    url: str | None = None    # add with a default of None
```

**Step 3** — Update `add_capture()` to accept and write the new field:
```python
def add_capture(self, text, session, source_window=None,
                source_app=None, url=None) -> Capture:
    ts = datetime.now().isoformat(timespec="seconds")
    with self._lock:
        cur = self.conn.execute(
            "INSERT INTO captures "
            "(text, captured_at, session, source_window, source_app, url) "
            "VALUES (?, ?, ?, ?, ?, ?)",
            (text, ts, session, source_window, source_app, url),
        )
        self.conn.commit()
        row_id = cur.lastrowid
    return Capture(row_id, text, ts, session, source_window, source_app, None, url)
```

**Step 4** — Update `query()` and `search()` to include the new
column in SELECT and in the `Capture(...)` constructor call:
```python
sql = ("SELECT id, text, captured_at, session, "
       "source_window, source_app, tag, url "   # add url here
       "FROM captures")
```

**Step 5** — For existing databases that don't have the column yet,
add a migration in `__init__` after `self.conn.execute(SCHEMA)`:
```python
try:
    self.conn.execute("ALTER TABLE captures ADD COLUMN url TEXT")
    self.conn.commit()
except sqlite3.OperationalError:
    pass  # column already exists — safe to ignore
```

**Step 6** — Pass the new field from `session_manager.py` where
`add_capture()` is called (see Part 4).

### Changing WAL mode

WAL (Write-Ahead Logging) is enabled so the CLI and GUI can access the
database at the same time. If you are only ever running one process at a
time you can remove it, but there is no reason to:
```python
# In __init__ — remove to disable WAL
self.conn.execute("PRAGMA journal_mode=WAL")
```

---

## Part 3 — `watcher_core.py` — The Capture Engine

**Location:** `src/highlight_scraper/watcher_core.py`  
**What it does:** Runs in a background thread. Watches the PRIMARY mouse
selection (text you highlight with the mouse — no Ctrl+C needed) and
calls `on_change(text)` every time it changes to something new and
non-empty.

### Changing the polling interval

The polling interval is how often the fallback (non-event-driven) mode
reads the clipboard. It has no effect when running in event-driven mode
(Xfixes or `wl-paste --watch`).

Set it in `config.json` or `config.py` DEFAULTS:
```json
{ "poll_interval": 0.3 }
```

Or pass it directly when constructing `SelectionWatcher`:
```python
watcher = SelectionWatcher(on_change=fn, poll_interval=0.5)
```

### Changing the event-driven sleep interval

In event-driven X11 mode, `time.sleep(0.05)` in `_run_x11_xfixes()`
is just CPU yielding — it is not a polling delay. Reduce to `0.01`
for faster response; increase to `0.1` to reduce CPU load on slow machines:
```python
# In _run_x11_xfixes(), inside the while loop
else:
    time.sleep(0.05)   # change this
```

### Forcing a specific backend

By default the backend is selected automatically. To force a specific one,
replace the `_pick_backend()` method's return values:

```python
def _pick_backend(self):
    # Force xclip polling on X11, ignoring python-xlib even if installed
    return self._run_x11_polling, "X11 (xclip polling — forced)"
```

Available backend functions and their requirements:

| Function | Requires | Best for |
|---|---|---|
| `self._run_x11` | `python3-xlib`, `xclip` | X11 — event-driven with automatic fallback |
| `self._run_x11_xfixes` | `python3-xlib`, `xclip` | X11 — event-driven only, no fallback |
| `self._run_x11_polling` | `xclip` | X11 — polling only |
| `self._run_wayland` | `wl-clipboard` | Wayland — event-driven with fallback |
| `self._run_wayland_polling` | `wl-clipboard` | Wayland — polling only |

### Watching CLIPBOARD instead of PRIMARY

By default the program watches PRIMARY (mouse highlight). To watch
CLIPBOARD (Ctrl+C) instead, change the xclip command:

```python
@staticmethod
def _read_primary_xclip() -> str:
    result = subprocess.run(
        ["xclip", "-selection", "clipboard", "-o"],   # change "primary" to "clipboard"
        capture_output=True, text=True, timeout=1,
    )
```

And for Wayland, remove `"--primary"` from the `wl-paste` command in
`_run_wayland()`.

### Adding a new backend

1. Add a new method to `SelectionWatcher`:
```python
def _run_x11_xsel_polling(self):
    last_seen = ""
    while not self._stop_event.is_set():
        try:
            result = subprocess.run(
                ["xsel", "--primary", "--output"],
                capture_output=True, text=True, timeout=1,
            )
            text = result.stdout
        except (FileNotFoundError, subprocess.TimeoutExpired):
            text = ""
        if text and text != last_seen:
            last_seen = text
            self.on_change(text)
        time.sleep(self.poll_interval)
```

2. Add it to `_pick_backend()` with its condition:
```python
if shutil.which("xsel"):
    return self._run_x11_xsel_polling, "X11 (xsel polling)"
```

---

## Part 4 — `session_manager.py` — Capture Filtering and Auto-Export

**Location:** `src/highlight_scraper/session_manager.py`  
**What it does:** Sits between `watcher_core` and `storage`. Every raw
selection change passes through `_handle()` before anything is written.
This is where filtering, exclusions, minimum length, merge logic, and
auto-export all live.

### Changing the minimum capture length

Set in `config.json`:
```json
{ "min_length": 3 }
```

The check in `_handle()`:
```python
if len(text.strip()) < self.config.get("min_length", 0):
    return
```

### Changing the exclude list

```json
{
  "excluded_apps": ["keepassxc", "keepass", "bitwarden", "1password",
                    "gnome-keyring", "seahorse"]
}
```

The check is case-insensitive substring match against the app name
returned by `source_info.py`. `"firefox"` in the list blocks all
captures from any window whose app name contains "firefox".

### Changing the merge window

```json
{ "merge_window_seconds": 2.0 }
```

Set to `0` to disable merging entirely. Increase for slow systems or
very large drag selections.

### Changing what counts as a merge

The `_looks_like_extension()` function in `session_manager.py`:
```python
def _looks_like_extension(old_text: str, new_text: str) -> bool:
    if not old_text or not new_text:
        return False
    return old_text in new_text or new_text in old_text
```

Alternatives:
```python
# Only merge if extending forward (no backtracking)
return new_text.startswith(old_text) or old_text.startswith(new_text)

# Disable merging entirely
return False

# Fuzzy merge — same content at 80%+
short, long = sorted([old_text, new_text], key=len)
return len(short) / max(len(long), 1) > 0.8
```

### Configuring auto-export

Auto-export is built into `_maybe_auto_export()` in 0.4.0. Configure it
via `config.json` or the GUI Settings panel:

```json
{
  "auto_export_every": 25,
  "auto_export_path": "/home/yourname/research/highlights.md"
}
```

- `auto_export_every`: number of captures between exports. `0` = off.
- `auto_export_path`: output file. Supports `~` expansion. Default is
  `~/highlights_<session_name>.md`.

The export runs synchronously in the capture thread but all exceptions
are caught and silently discarded — a failed export never interrupts
capture.

### Changing the auto-export format

`_maybe_auto_export()` always exports Markdown. To use a different format:

```python
def _maybe_auto_export(self) -> None:
    every = self.config.get("auto_export_every", 0)
    if not every or self._capture_count % every != 0:
        return
    from highlight_scraper import export as exporters
    raw_path = self.config.get("auto_export_path", "~/highlights.csv")
    out_path = str(Path(raw_path).expanduser())
    try:
        # force=True because auto-export writes to the same path repeatedly
        # by design — each export replaces the previous one.
        exporters.export_csv(self.store, self.session_name, out_path,
                             force=True)  # change format here
    except Exception:
        pass
```

### Adding a new field to each capture

1. Add the column to `storage.py` (see Part 2).

2. In `_handle()`, gather the new data and pass it:
```python
def _handle(self, text: str):
    ...
    window, app = get_active_window_info()
    url = get_active_browser_url()   # your new function in source_info.py
    ...
    row = self.store.add_capture(
        text=text, session=self.session_name,
        source_window=window, source_app=app,
        url=url,
    )
```

### Adding a capture listener

`self.listeners` is a list of callables. Each receives a `Capture`
dataclass on every new row. Add yours after creating the session:

```python
session = CaptureSession(db_path, session_name, config)

def notify(row):
    subprocess.Popen(["notify-send", "Captured", row.text[:80]])

session.listeners.append(notify)
session.start()
```

---

## Part 5 — `source_info.py` — Window and App Attribution

**Location:** `src/highlight_scraper/source_info.py`  
**What it does:** Returns `(window_title, app_name)` of the active window
at the moment of capture. Never raises — returns `(None, None)` on any
failure so capture is never interrupted.

### 0.4.0 — Wayland attribution is now implemented

`get_active_window_info()` now tries three compositor helpers on Wayland
before falling back to `(None, None)`:

| Compositor | Tool | Method |
|---|---|---|
| Hyprland | `hyprctl` | `hyprctl activewindow -j` — JSON with `title` and `class` |
| Sway / i3-on-Wayland | `swaymsg` | `swaymsg -t get_tree` — recursive tree walk to focused node |
| KDE Plasma Wayland | `kdotool` | Same interface as xdotool, Wayland-native |

No install is needed — these are built into their respective compositors.
If none are detected the function returns `(None, None)` exactly as
it did in 0.3.x.

### Adding attribution for a different Wayland compositor

Add a new helper function and call it from `_get_wayland_window_info()`:

```python
# At the top — add a detection flag
_have_niri = shutil.which("niri") is not None

def _get_wayland_window_info():
    if _have_hyprctl:
        result = _hyprland_active_window()
        if result != (None, None):
            return result
    if _have_swaymsg:
        result = _sway_active_window()
        if result != (None, None):
            return result
    if _have_kdotool:
        result = _kde_active_window()
        if result != (None, None):
            return result
    if _have_niri:
        result = _niri_active_window()    # add your compositor here
        if result != (None, None):
            return result
    return None, None

def _niri_active_window() -> tuple[str | None, str | None]:
    # Implement for your compositor. Return (title, app_name).
    # Always wrap in try/except and return (None, None) on any failure.
    try:
        result = subprocess.run(
            ["niri", "msg", "--json", "focused-window"],
            capture_output=True, text=True, timeout=1,
        )
        data = json.loads(result.stdout)
        return data.get("title"), data.get("app_id")
    except Exception:
        return None, None
```

### Adding browser URL capture

Add a function to `source_info.py`:
```python
def get_active_browser_url() -> str | None:
    """
    Best-effort URL extraction from the active window title.
    Only works with browsers that include the URL in the title bar,
    or via browser remote debugging. Heuristic — not reliable.
    For reliable URL capture, use a browser extension instead.
    """
    if is_wayland():
        title, _ = _get_wayland_window_info()
    elif _have_xdotool:
        title = _run_xdotool("getwindowname")
    else:
        return None

    if not title:
        return None
    # Some browsers append the URL: "Page Title — https://example.com"
    if " — https://" in title:
        return "https://" + title.split(" — https://", 1)[1]
    return None
```

Then pass it from `session_manager.py`'s `_handle()` and store it in
a new `url` column (see Part 4 and Part 2).

---

## Part 6 — `export.py` — Export Formats

**Location:** `src/highlight_scraper/export.py`  
**What it does:** Provides `export_markdown()`, `export_csv()`, and
`export_json()`. All three take a `CaptureStore`, a session name, an
output file path, and an optional `force` keyword argument (default
`False`). If the output file already exists and `force=False`, they
raise `FileExistsError` and leave the existing file untouched. Pass
`force=True` to allow overwriting.

### Changing the Markdown export format

`export_markdown()` groups captures by source app. To change to a
chronological flat list, replace the body of the function:

```python
def export_markdown(store, session, out_path, *, force: bool = False):
    _guard_overwrite(out_path, force)
    rows = _ordered_rows(store, session)
    title = f"Highlights — {session}" if session else "Highlights — all sessions"
    lines = [f"# {title}", ""]
    for r in rows:
        tag = f" `[{r.tag}]`" if r.tag else ""
        source = f" _({r.source_app or 'unknown'})_" if r.source_app else ""
        lines.append(f"- **{r.captured_at}**{tag}{source}")
        lines.append(f"  {r.text.strip()}")
        lines.append("")
    Path(out_path).write_text("\n".join(lines), encoding="utf-8")
```

### Adding a new export format

1. Add a new function in `export.py`:
```python
def export_html(store, session, out_path, *, force: bool = False):
    _guard_overwrite(out_path, force)
    rows = _ordered_rows(store, session)
    title = f"Highlights — {session or 'all sessions'}"
    parts = [f"<html><head><title>{title}</title></head><body>",
             f"<h1>{title}</h1><ul>"]
    for r in rows:
        tag = f" <span class='tag'>[{r.tag}]</span>" if r.tag else ""
        parts.append(
            f"<li><time>{r.captured_at}</time>{tag}"
            f"<blockquote>{r.text}</blockquote></li>"
        )
    parts.append("</ul></body></html>")
    Path(out_path).write_text("\n".join(parts), encoding="utf-8")
```

2. Register it in `cli.py`'s `EXPORTERS` dict:
```python
EXPORTERS = {
    "md":   exporters.export_markdown,
    "csv":  exporters.export_csv,
    "json": exporters.export_json,
    "html": exporters.export_html,
}
```

3. Add `"html"` to the CLI argument choices:
```python
p_export.add_argument("--format", choices=["md", "csv", "json", "html"], required=True)
```

4. Add it to the GUI export dialog in `gui.py`'s `_do_export()`:
```python
path = filedialog.asksaveasfilename(
    filetypes=[("Markdown", "*.md"), ("CSV", "*.csv"),
               ("JSON", "*.json"), ("HTML", "*.html")],
    ...
)
exporter = {
    "md":   exporters.export_markdown,
    "csv":  exporters.export_csv,
    "json": exporters.export_json,
    "html": exporters.export_html,
}.get(ext, exporters.export_markdown)
```

### Changing the CSV column order

In `export_csv()`, reorder both the header row and the data row:
```python
writer.writerow(["id", "captured_at", "session", "source_app",
                 "source_window", "tag", "text"])
```

---

## Part 7 — `hotkeys.py` — Keyboard Shortcuts

**Location:** `src/highlight_scraper/hotkeys.py`  
**What it does:** Registers global keyboard shortcuts to tag the most
recent capture without switching windows. Default: Ctrl+Alt+1/2/3.

### Changing the default hotkey bindings

```python
DEFAULT_BINDINGS = {
    "<ctrl>+<alt>+1": "important",
    "<ctrl>+<alt>+2": "question",
    "<ctrl>+<alt>+3": "followup",
}
```

Replace key combos with any combination using `pynput` modifier names:
`<ctrl>`, `<shift>`, `<alt>`, `<cmd>`, `<super>`.

```python
DEFAULT_BINDINGS = {
    "<ctrl>+<shift>+i": "important",
    "<ctrl>+<shift>+q": "question",
    "<ctrl>+<shift>+f": "followup",
    "<ctrl>+<shift>+d": "done",
}
```

### Adding more hotkey bindings

Add entries to `DEFAULT_BINDINGS`. No limit. Each maps one combo to
one tag string applied to the most recent capture.

### Passing custom bindings at runtime

```python
from highlight_scraper.hotkeys import TagHotkeys

custom = {
    "<ctrl>+<alt>+k": "keep",
    "<ctrl>+<alt>+x": "discard",
}
hotkeys = TagHotkeys(on_tag=session.tag_last, bindings=custom)
hotkeys.start()
```

### Adding hotkeys to the GUI

The GUI does not currently expose `TagHotkeys`. To add it:

In `gui.py`'s `__init__`:
```python
self._tag_hotkeys = None
```

In `start()`, after `self.session.start()`:
```python
from highlight_scraper.hotkeys import TagHotkeys
self._tag_hotkeys = TagHotkeys(on_tag=self.session.tag_last)
self._tag_hotkeys.start()
```

In `stop()`:
```python
if self._tag_hotkeys:
    self._tag_hotkeys.stop()
    self._tag_hotkeys = None
```

---

## Part 8 — `paste_on_hold.py` — Click-and-Hold Paste Gesture

**Location:** `src/highlight_scraper/paste_on_hold.py`  
**What it does:** Listens globally for left mouse button presses. If the
button is held still for `hold_seconds` without dragging more than
`move_tolerance` pixels, it fires a synthetic middle-click which pastes
the PRIMARY selection in X11.

### Changing the hold duration

```python
HOLD_SECONDS = 0.45
```

Or pass it when constructing:
```python
chp = ClickHoldPaste(hold_seconds=0.75)
```

Also configurable at runtime via the GUI spinbox (writes to `config.json`
as `hold_seconds`).

### Changing the movement tolerance

```python
MOVE_TOLERANCE_PX = 4
```

Increase if the gesture fires during normal clicking.
Decrease for more precise triggering.

### Adding a callback on paste

```python
def my_paste_callback(x, y):
    print(f"Pasted at ({x}, {y})")

chp = ClickHoldPaste(on_paste=my_paste_callback)
```

### Triggering Ctrl+V instead of middle-click

```python
def _paste_at(self, x, y):
    from pynput.keyboard import Controller as KeyController, Key
    time.sleep(0.05)
    kb = KeyController()
    with kb.pressed(Key.ctrl):
        kb.press('v')
        kb.release('v')
```

Note: Ctrl+V pastes CLIPBOARD not PRIMARY. The user must Ctrl+C first.

---

## Part 9 — `cli.py` — Command-Line Interface

**Location:** `src/highlight_scraper/cli.py`  
**What it does:** Terminal front end. Provides `start`, `stop`, `status`,
`tag`, `sessions`, and `export` subcommands.

### Adding a `search` command (uses 0.4.0 `storage.search()`)

1. Write the handler:
```python
def cmd_search(args):
    config = load_config()
    store = CaptureStore(config["db_path"])
    try:
        rows = store.search(args.query, session=args.session,
                            limit=args.limit)
    finally:
        store.close()
    if not rows:
        print("No captures found.")
        return
    for r in rows:
        tag = f" [{r.tag}]" if r.tag else ""
        print(f"{r.captured_at} ({r.source_app or 'unknown'}){tag}")
        print(f"  {r.text[:120]}")
        print()
```

2. Register it in `main()`:
```python
p_search = sub.add_parser("search", help="Search captures by text")
p_search.add_argument("query", help="Text to search for")
p_search.add_argument("--session", default=None)
p_search.add_argument("--limit", type=int, default=50)
p_search.set_defaults(func=cmd_search)
```

Available immediately as `highlight-scraper search "your query"`.

### Adding a `delete` command (uses 0.4.0 `storage.delete_capture()`)

```python
def cmd_delete(args):
    config = load_config()
    store = CaptureStore(config["db_path"])
    try:
        deleted = store.delete_capture(args.id)
    finally:
        store.close()
    if deleted:
        print(f"Deleted capture #{args.id}.")
    else:
        print(f"No capture found with id {args.id}.")
```

Register it:
```python
p_delete = sub.add_parser("delete", help="Delete one capture by id")
p_delete.add_argument("id", type=int, help="Capture id (see 'sessions' output)")
p_delete.set_defaults(func=cmd_delete)
```

### Changing the PID file location

```python
PID_FILE = Path.home() / ".highlight_scraper.pid"
```

Change to any path.

### Adding a flag to `start`

```python
p_start.add_argument("--no-merge", action="store_true",
                      help="Disable highlight merging")
```

In `cmd_start()`, pass it through:
```python
if args.no_merge:
    cmd.append("--no-merge")
```

In `cmd_run_foreground()`:
```python
if args.no_merge:
    config["merge_window_seconds"] = 0
```

### Changing the log file location

In `cmd_start()`:
```python
log_path = Path.home() / f".highlight_scraper_{session_name}.log"
```

---

## Part 10 — `tray.py` — System Tray

**Location:** `src/highlight_scraper/tray.py`  
**What it does:** System tray icon. Menu provides Start, Pause, Stop,
Open data folder, Quit. The tray IS the running session — no separate
background process.

### Changing the tray icon colours

```python
ICON_STOPPED = _dot((150, 150, 150, 255))  # grey
ICON_ACTIVE  = _dot((60, 170, 90, 255))    # green
ICON_PAUSED  = _dot((210, 170, 40, 255))   # amber
```

Each tuple is `(R, G, B, A)` — values 0–255. MainbyteLabs green:
```python
ICON_ACTIVE = _dot((0, 255, 65, 255))    # #00FF41
```

### Changing the icon shape

```python
def _square(color):
    img = Image.new("RGBA", (64, 64), (0, 0, 0, 0))
    ImageDraw.Draw(img).rectangle((8, 8, 56, 56), fill=color)
    return img

ICON_ACTIVE = _square((0, 255, 65, 255))
```

### Adding a menu item

```python
def _build_menu(self):
    return pystray.Menu(
        pystray.MenuItem("Start", self._start, ...),
        pystray.MenuItem("Pause capture", self._toggle_pause, ...),
        pystray.MenuItem("Stop", self._stop, ...),
        pystray.Menu.SEPARATOR,
        pystray.MenuItem("Export last session", self._export_last),   # new
        pystray.MenuItem("Open data folder", self._open_data_folder),
        pystray.MenuItem("Quit", self._quit),
    )

def _export_last(self, icon=None, item=None):
    if not self.session:
        self.icon.notify("No active session.", "Highlight Scraper")
        return
    from highlight_scraper import export as exporters
    out = Path.home() / f"{self.session.session_name}_export.md"
    try:
        # force=True here because tray export to a fixed path is an
        # intentional overwrite — the user is refreshing the same file.
        exporters.export_markdown(self.session.store,
                                   self.session.session_name, str(out),
                                   force=True)
    except Exception as e:
        self.icon.notify(f"Export failed: {e}", "Highlight Scraper")
        return
    self.icon.notify(f"Exported to {out}", "Highlight Scraper")
```

### Adding a desktop notification on capture

In `_start()`, after `self.session.start()`:
```python
def _notify_capture(row):
    self.icon.notify(row.text[:80], "Captured")

self.session.listeners.append(_notify_capture)
```

---

## Part 11 — `gui.py` — The GUI Window

**Location:** `src/highlight_scraper/gui.py`  
Full visual modification reference (colours, fonts, layout, themes)
is in `highlight-scraper-gui-modification-guide.md`. This section
covers programmatic and feature-level modifications.

### 0.4.0 — Search bar

The search bar is built into `_build_capture_list()` in 0.4.0. It calls
`storage.search()` as you type and renders matching rows. The ✕ button
clears the query and restores the normal view.

To change what the search searches (e.g. restrict to tags only):
```python
# In _on_search_change(), replace store.search() with a custom query:
rows = store.query(session=session_filter, limit=self._entries_shown.get())
rows = [r for r in rows if r.tag and query.lower() in r.tag.lower()]
```

### 0.4.0 — Per-row delete is now wired up

`_ctx_delete_last()` in 0.4.0 calls `store.delete_capture(row.id)`
directly. The confirmation dialog shows a 60-character preview of the
capture text before deleting.

To add delete to the session browser (not just the active session):
```python
def _ctx_delete_last(self) -> None:
    # Determine which store to use
    if self.session:
        store = self.session.store
        session_name = self.session.session_name
        owns_store = False
    else:
        store = CaptureStore(self.config["db_path"])
        session_name = self._browse_var.get()
        owns_store = True
    try:
        rows = store.query(session=session_name, limit=1)
        if not rows:
            return
        row = rows[0]
        preview = row.text[:60] + ("…" if len(row.text) > 60 else "")
        if messagebox.askyesno("Delete", f"Delete:\n\"{preview}\""):
            store.delete_capture(row.id)
            self._on_browse_select()
    finally:
        if owns_store:
            store.close()
```

### 0.4.0 — Auto-export Settings panel

The Settings panel now has an Auto-Export section with:
- **Every N captures** spinbox (0 = off)
- **Export file path** text entry with a browse button

Both persist to `config.json` on Save & Close and are read by
`session_manager._maybe_auto_export()` on every capture.

### Making the window remember its size and position

In `_on_close()`:
```python
def _on_close(self):
    geo = self.root.geometry()
    self.config["window_geometry"] = geo
    save_config(self.config)
    ...
```

In `__init__`, after the default `root.geometry(...)`:
```python
saved_geo = self.config.get("window_geometry")
if saved_geo:
    root.geometry(saved_geo)
else:
    root.geometry("700x620")
```

### Adding a capture count badge to the session browser

In `_on_browse_select()`, after loading rows:
```python
self.status_var.set(f"Browsing: {name}  ({len(rows)} captures shown)")
```

---

## Part 12 — Adding a New Feature End-to-End

Example: **tag-based export** — export only captures with a specific tag.

**Step 1 — `storage.py`:** Add a query that filters by tag.
```python
def query_by_tag(self, tag: str, session: str = None,
                 limit: int = 200) -> list:
    sql = ("SELECT id, text, captured_at, session, "
           "source_window, source_app, tag "
           "FROM captures WHERE tag=?")
    params: list = [tag]
    if session:
        sql += " AND session=?"
        params.append(session)
    sql += " ORDER BY id DESC LIMIT ?"
    params.append(limit)
    cur = self.conn.execute(sql, params)
    return [Capture(*row) for row in cur.fetchall()]
```

**Step 2 — `export.py`:** Add a tag-filtered export function.
```python
def export_by_tag(store, session, out_path, tag: str, *, force: bool = False):
    _guard_overwrite(out_path, force)
    rows = store.query_by_tag(tag, session=session)
    rows = list(reversed(rows))
    title = f"Highlights tagged [{tag}]"
    if session:
        title += f" — {session}"
    lines = [f"# {title}", ""]
    for r in rows:
        lines.append(f"- **{r.captured_at}** ({r.source_app or 'unknown'})")
        lines.append(f"  {r.text.strip()}")
        lines.append("")
    Path(out_path).write_text("\n".join(lines), encoding="utf-8")
```

**Step 3 — `cli.py`:** Add a command.
```python
def cmd_export_tag(args):
    config = load_config()
    store = CaptureStore(config["db_path"])
    try:
        from highlight_scraper.export import export_by_tag
        export_by_tag(store, args.session, args.out, tag=args.tag)
    finally:
        store.close()
    print(f"Exported captures tagged [{args.tag}] to: {args.out}")

# In main():
p_et = sub.add_parser("export-tag", help="Export captures with a specific tag")
p_et.add_argument("tag", help="Tag to export")
p_et.add_argument("--out", required=True)
p_et.add_argument("--session", default=None)
p_et.set_defaults(func=cmd_export_tag)
```

**Step 4 — `gui.py`:** Add a button to the tag row.
```python
ttk.Button(row, text="↓ Export tagged…",
            command=self._export_by_tag).pack(side="right")

def _export_by_tag(self):
    if not self.session:
        messagebox.showinfo("Export", "No active session.")
        return
    tag = simpledialog.askstring("Export by tag", "Tag to export:")
    if not tag:
        return
    self._do_export_by_tag(self.session.store,
                            self.session.session_name, tag)

def _do_export_by_tag(self, store, session_name, tag):
    from highlight_scraper.export import export_by_tag
    path = filedialog.asksaveasfilename(
        defaultextension=".md",
        filetypes=[("Markdown", "*.md")],
        initialfile=f"{session_name}_{tag}.md",
    )
    if not path:
        return
    export_by_tag(store, session_name, path, tag=tag)
    messagebox.showinfo("Export", f"Written to:\n{path}")
```

No other files need to change.

---

## Quick Reference — File-to-Feature Map

| Feature | Primary file | Secondary files |
|---|---|---|
| Colours, fonts, window size | `gui.py` | — |
| GUI layout and sections | `gui.py` | — |
| Search/filter the capture list | `gui.py` | `storage.py` |
| Per-row delete | `storage.py` | `gui.py`, `cli.py` |
| Auto-export every N captures | `session_manager.py` | `config.py`, `gui.py` |
| System tray appearance | `tray.py` | — |
| CLI commands | `cli.py` | `storage.py` |
| What gets captured | `session_manager.py` | `watcher_core.py` |
| Capture deduplication/merging | `session_manager.py` | — |
| App/window attribution (X11) | `source_info.py` | `session_manager.py` |
| App/window attribution (Wayland) | `source_info.py` | — |
| Database schema | `storage.py` | `session_manager.py`, `export.py` |
| Database queries | `storage.py` | — |
| Export formats | `export.py` | `cli.py`, `gui.py` |
| Keyboard shortcuts | `hotkeys.py` | `cli.py`, `gui.py` |
| Click-and-hold gesture | `paste_on_hold.py` | `gui.py`, `cli.py` |
| Persistent settings | `config.py` | all files |
| X11/Wayland detection | `watcher_core.py` → `is_wayland()` | `source_info.py`, `paste_on_hold.py` |
| Polling speed | `config.py` (`poll_interval`) | `watcher_core.py` |
| Background process (CLI) | `cli.py` (`PID_FILE`, `cmd_run_foreground`) | — |
