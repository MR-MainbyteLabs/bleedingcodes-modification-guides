# pyqt-camera-dashboard — Modification Guide
**Version:** 3.0.1  
**File:** `camera_dashboard.py` (single-file application)  
**Entry point:** `main()`

---

## Who This Document Is For

Engineers and developers who need to extend, adapt, or embed the camera dashboard beyond what the UI exposes. This is not a usage guide — the README and setup guide cover that. This document tells you where the code lives, what each class and function owns, and exactly where to make each class of change.

---

## Architecture Overview

```
main()
  ├── load_or_create_key()           # Fernet key — generated once, lives in secret.key
  ├── load_camera_details(key)       # Decrypts camera_config.json → list[dict]
  └── MainWindow(camera_details, key)
        ├── CameraTile × N           # One tile per camera
        │     └── CameraWorker       # QThread — RTSP capture, recording, reconnect
        └── cleanup_timer (QTimer)   # Disk cleanup check every 60 minutes
```

Data flow per frame:
```
CameraWorker.run() → cap.read() → frame_ready.emit(frame)
    ↓
CameraTile.update_frame(frame) → QPixmap → video QLabel
    ↓ (if recording)
CameraWorker._write_frame(frame) → VideoWriter.write()
```

---

## Module-Level Constants

All tunable parameters are at the top of the file. Change these rather than hunting for magic numbers inside methods.

```python
SCRIPT_DIR             = os.path.dirname(os.path.abspath(__file__))
CONFIG_FILE            = os.path.join(SCRIPT_DIR, "camera_config.json")
KEY_FILE               = os.path.join(SCRIPT_DIR, "secret.key")
RECORDINGS_ROOT        = os.path.join(SCRIPT_DIR, "recordings")
RECORD_DURATION_SECONDS = 15 * 60       # 900 seconds — segment length
DEFAULT_FPS            = 15.0            # Fallback when cap.get(CAP_PROP_FPS) is unreliable
RECONNECT_DELAY_SECONDS = 3              # Seconds between auto-reconnect attempts
MAX_CONNECT_FAILURES   = 3              # Permanent failure threshold
MAX_DISK_USAGE         = 75             # Percent — triggers auto-cleanup
RTSP_PATH              = "/cam/realmonitor?channel=1&subtype=0"  # RTSP path appended to all URLs
```

### Common Constant Changes

**Change segment length** (e.g. 30 minutes):
```python
RECORD_DURATION_SECONDS = 30 * 60
```

**Change disk cleanup threshold** (e.g. 85%):
```python
MAX_DISK_USAGE = 85
```

**Change max reconnect attempts before going offline**:
```python
MAX_CONNECT_FAILURES = 5
```

**Change recordings location** (e.g. external drive):
```python
RECORDINGS_ROOT = "/mnt/nas/recordings"
```

**Change RTSP path for a different camera brand:**
```python
RTSP_PATH = "/Streaming/Channels/101"   # Hikvision format
# or
RTSP_PATH = "/live/ch00_0"              # Reolink format
```

---

## Data Class — `CameraConfig`

Frozen dataclass. Created once per camera, passed to `CameraWorker` and `CameraTile`. Never modified after construction.

```python
@dataclass(frozen=True)
class CameraConfig:
    name: str           # Display name shown in the tile title
    url: str            # Full RTSP URL including credentials
    file_prefix: str    # Sanitized filename prefix for recordings
```

`file_prefix` is generated from `name` via `make_file_prefix()` and guaranteed unique across all cameras by `unique_file_prefix()`. It survives any characters that would be unsafe in a filename.

**To add a field to CameraConfig** (e.g. a channel number):
1. Add the field to `CameraConfig`.
2. Add the corresponding key to the JSON schema (see Config Persistence below).
3. Update `camera_details_to_config()` to read the new key from `details`.
4. Update `prompt_for_camera_details()` to ask for it.

---

## Encryption — `load_or_create_key`, `encrypt_config`, `decrypt_config`

Uses `cryptography.fernet.Fernet` — AES-128-CBC + HMAC-SHA256. The key is a URL-safe base64-encoded 32-byte random value.

### Key lifecycle

1. `load_or_create_key()` — called once in `main()`. If `secret.key` does not exist, generates a new key, writes it with `chmod 0o600` (owner read/write only), returns the bytes. If it exists, reads and returns it.
2. Every `save_camera_details()` call writes plaintext JSON then immediately calls `encrypt_config()`, which overwrites the file with Fernet-encrypted bytes. The plaintext never persists on disk.
3. `decrypt_config()` decrypts and returns a parsed `dict`. Called by `load_camera_details()` at startup.

### Changing the key location

```python
KEY_FILE = "/etc/camera-dashboard/secret.key"
```

Ensure the path exists and is writable. The `chmod 0o600` call in `load_or_create_key()` still applies.

### Rotating the encryption key

There is no built-in key rotation. To rotate manually:

1. Decrypt and save the config to a temp plaintext file using Python + the old key.
2. Delete `secret.key` and `camera_config.json`.
3. Restart the app — it will generate a new key and run first-time setup, or re-import from the temp file.

### Storing credentials externally

To replace Fernet with a secrets manager (e.g. `python-keyring`, HashiCorp Vault, AWS SSM):

1. Replace `load_or_create_key()`, `encrypt_config()`, and `decrypt_config()` with calls to your secrets backend.
2. `save_camera_details()` and `load_camera_details()` call these functions — no other changes needed as long as the JSON structure (`{"cameras": [...]}`) is preserved.

---

## URL Builder — `build_rtsp_url`

```python
def build_rtsp_url(ip_address: str, username: str, password: str) -> str:
    safe_username = quote(username, safe="")
    safe_password = quote(password, safe="")
    return f"rtsp://{safe_username}:{safe_password}@{ip_address}:554{RTSP_PATH}"
```

Credentials are percent-encoded via `urllib.parse.quote`. Special characters in usernames and passwords (e.g. `@`, `:`, `/`) are safely escaped.

### Changing the RTSP port

`554` is hardcoded in the f-string. To make it configurable:

1. Add `port: int = 554` to `CameraConfig`.
2. Add `--port` to `prompt_for_camera_details()` or default it.
3. Update `build_rtsp_url()` to accept `port` and use it:
```python
return f"rtsp://{safe_username}:{safe_password}@{ip_address}:{port}{RTSP_PATH}"
```

### Supporting pre-built URLs

`camera_details_to_config()` already supports a `url` key in the JSON config. If `url` is present and non-empty, `build_rtsp_url()` is skipped entirely. To add a camera with a pre-built URL, include `"url": "rtsp://..."` in the JSON directly.

---

## Config Persistence

### JSON schema

```json
{
    "cameras": [
        {
            "name": "Front Door",
            "ip_address": "192.168.1.101",
            "username": "admin",
            "password": "hunter2",
            "file_prefix": "front_door"
        }
    ]
}
```

`file_prefix` is assigned by the app on first setup and preserved on every subsequent save. Do not edit it manually unless you also rename existing recording files.

### `camera_details_to_config(details: dict) -> CameraConfig`

The bridge between raw JSON dicts and the frozen `CameraConfig`. Reads `name`, `url` (optional), `ip_address`, `username`, `password`, and `file_prefix`. Unknown keys are ignored. Missing required keys fall back to safe defaults (`"Camera"`, empty strings).

### `save_camera_details(camera_details: list[dict], key: bytes)`

1. Writes `{"cameras": camera_details}` as indented JSON to `CONFIG_FILE`.
2. Immediately calls `encrypt_config(key)` — the plaintext is overwritten before the function returns.

### `load_camera_details(key: bytes, parent) -> list[dict]`

1. If `CONFIG_FILE` does not exist: runs `prompt_for_first_time_setup()`.
2. If it exists: decrypts, parses, validates that `"cameras"` is a list, returns it.
3. Decryption failure shows a `QMessageBox.critical` and returns `[]`, preventing startup with unknown credentials.

---

## Camera Worker — `CameraWorker(QThread)`

One instance per camera. Runs entirely on its own thread. Communicates with the GUI only via signals.

### Signals

```python
frame_ready       = pyqtSignal(object)   # np.ndarray — emitted every read frame
status_changed    = pyqtSignal(str)      # Human-readable status string
recording_changed = pyqtSignal(bool)     # True when recording starts, False when it stops
failed_permanently = pyqtSignal()        # Emitted after MAX_CONNECT_FAILURES consecutive failures
```

### State

```python
self.running              # bool — controls the outer loop; set False to exit
self.recording            # bool — whether frames are being written
self.writer               # cv2.VideoWriter | None
self.record_start         # datetime | None — start time of current segment
self.writer_size          # (width, height) | None — locked at start of segment
self.fps                  # float — from cap.get(CAP_PROP_FPS) or DEFAULT_FPS
self.cap                  # cv2.VideoCapture | None
self.connect_failures     # int — reset to 0 on successful connect
self.reconnect_requested  # bool — checked in the inner loop; triggers soft reconnect
```

### Connection flow (`run()`)

Outer loop (retry on disconnect):
1. Emit `"Connecting..."` status.
2. Open `cv2.VideoCapture(url, CAP_FFMPEG)` with buffer size 2.
3. If open fails: increment `connect_failures`. If >= `MAX_CONNECT_FAILURES`: emit `failed_permanently`, set `running = False`, return.
4. On success: reset `connect_failures`, read FPS, enter inner loop.

Inner loop (read frames):
1. Check `reconnect_requested` — if set, break to outer loop (manual reconnect).
2. `cap.read()` — if frame lost, break to outer loop (auto-reconnect).
3. `frame_ready.emit(frame)`.
4. If recording: `_write_frame(frame)`.

### Recording

**`start_recording()`** — sets `self.recording = True`, emits signals. The first `_write_frame()` call after this opens a new `VideoWriter`.

**`stop_recording()`** — sets `self.recording = False`, calls `writer.release()`, clears `writer` and `writer_size`.

**`_write_frame(frame)`**:
1. If no writer: calls `_open_new_writer(frame)`.
2. Resizes frame if dimensions changed since writer was opened.
3. Writes frame.
4. If segment duration >= `RECORD_DURATION_SECONDS`: releases writer, opens a new one.

**`_open_new_writer(frame)`**:
- Output path: `recordings/YYYY-MM-DD/HH/<file_prefix>_YYYY-MM-DD_HH-MM-SS.mp4`
- Codec: `mp4v` (MPEG-4 Part 2)
- Frame size: locked from first frame of each segment

### Changing the Recording Codec

In `_open_new_writer()`:
```python
fourcc = cv2.VideoWriter_fourcc(*"mp4v")   # Current
# Change to H.264 (requires OpenCV built with x264):
fourcc = cv2.VideoWriter_fourcc(*"avc1")
# Change to XVID:
fourcc = cv2.VideoWriter_fourcc(*"XVID")
```

Note: codec availability depends on the OpenCV build. `mp4v` is the most portable option.

### Changing the Reconnect Behavior

**Increase retry delay:**
```python
RECONNECT_DELAY_SECONDS = 10
```

**Add exponential backoff** — in `run()`, replace `time.sleep(RECONNECT_DELAY_SECONDS)`:
```python
delay = min(60, RECONNECT_DELAY_SECONDS * (2 ** self.connect_failures))
time.sleep(delay)
```

**Add a notification on permanent failure:**

Connect `failed_permanently` signal in `CameraTile.__init__()` to an additional handler:
```python
self.worker.failed_permanently.connect(self.send_offline_alert)
```

### Manual Reconnect — `request_reconnect()`

Sets `connect_failures = 0` and either sets `reconnect_requested = True` (if already running, the inner loop picks it up on the next iteration) or restarts the thread fresh (if it had stopped after permanent failure).

---

## Camera Tile — `CameraTile(QWidget)`

One widget per camera. Owns the `CameraWorker` for that camera.

### Signals

```python
remove_requested = pyqtSignal(object)   # Emits self — MainWindow removes the tile
```

### Layout (top to bottom)

```
title QLabel          — camera.name, bold 16px
video QLabel          — frame display, min 420×260, dark background
status QLabel         — current status string from worker
record_button         — "Start Recording" / "Stop Recording"
reconnect_button      — "Reconnect" (enabled when offline or live)
remove_button         — "Remove" (red text — session only, does not touch config)
```

### `update_frame(frame: np.ndarray)`

Converts BGR to RGB, builds `QImage`, scales to `video` label size with `Qt.KeepAspectRatio` and `Qt.SmoothTransformation`, sets pixmap.

### Adding a Button to Every Tile

Add it in `CameraTile.__init__()` and connect it to a method:

```python
self.snapshot_button = QPushButton("Snapshot")
self.snapshot_button.clicked.connect(self.save_snapshot)
layout.addWidget(self.snapshot_button)

def save_snapshot(self) -> None:
    if hasattr(self.worker, 'cap') and self.worker.cap is not None:
        ok, frame = self.worker.cap.read()
        if ok:
            path = f"snapshot_{self.camera.file_prefix}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.png"
            cv2.imwrite(path, frame)
```

Note: `cap.read()` from a second caller while `CameraWorker.run()` is also calling it is not thread-safe with OpenCV. A safer approach is to cache the last emitted frame in `CameraTile.update_frame()` and snapshot from that:

```python
def update_frame(self, frame: np.ndarray) -> None:
    self.last_frame = frame.copy()   # Add this line
    # ... rest of method unchanged
```

### Changing the Tile Video Size

```python
self.video.setMinimumSize(640, 400)   # Current: 420, 260
```

### Changing the Grid Layout

Grid is 2 columns wide, fills rows top-to-bottom. Change in `MainWindow.rebuild_grid()`:

```python
col = index % 2    # 2 columns — change to 3 for 3-column layout
row = index // 2
```

For 3 columns:
```python
col = index % 3
row = index // 3
```

---

## Main Window — `MainWindow(QMainWindow)`

Owns the tile list, grid layout, control buttons, and the hourly cleanup timer.

### Attributes

```python
self.camera_details   # list[dict] — JSON-backed camera list, updated on add
self.key              # bytes — Fernet key, passed to save_camera_details()
self.tiles            # list[CameraTile]
self.grid             # QGridLayout
self.cleanup_timer    # QTimer — fires cleanup_by_disk_usage() every hour
```

### Adding a Camera at Runtime — `add_camera_from_dialog()`

1. Reads existing prefixes from `self.camera_details` to prevent collisions.
2. Calls `prompt_for_camera_details(self)` — sequential `QInputDialog` calls.
3. Assigns a unique `file_prefix`.
4. Appends to `self.camera_details` and calls `save_camera_details()` — config is persisted immediately.
5. Adds tile, rebuilds grid.

### Removing a Tile — `remove_tile(tile)`

Session-only. Calls `tile.shutdown()` (stops worker, waits 3s), removes from `self.tiles` and grid, calls `deleteLater()`. Does **not** modify `camera_details` or re-save config. The camera reappears on next launch.

**To make removal permanent** (also removes from config):
```python
def remove_tile(self, tile: CameraTile) -> None:
    # Add before existing shutdown logic:
    self.camera_details = [
        d for d in self.camera_details
        if d.get("file_prefix") != tile.camera.file_prefix
    ]
    save_camera_details(self.camera_details, self.key)
    # ... existing shutdown code unchanged
```

### Disk Cleanup — `cleanup_by_disk_usage()`

Called once at startup and then every 60 minutes via `cleanup_timer`. Reads disk usage for `RECORDINGS_ROOT`, sorts all `.mp4` files by `mtime` (oldest first), deletes until usage drops below `MAX_DISK_USAGE`.

**To change the cleanup interval** (e.g. every 30 minutes):
```python
self.cleanup_timer.start(30 * 60 * 1000)
```

**To clean up empty date/hour folders after deletion** — add to `cleanup_by_disk_usage()` after the file loop:
```python
for dirpath, dirnames, filenames in os.walk(RECORDINGS_ROOT, topdown=False):
    if not os.listdir(dirpath) and dirpath != RECORDINGS_ROOT:
        os.rmdir(dirpath)
```

### Shutdown — `closeEvent()`

Three-pass shutdown (order matters):
1. `stop_recording()` on all tiles — flushes and releases all `VideoWriter` objects.
2. `stop_worker()` on all tiles — sets `running = False`.
3. `worker.wait(3000)` on all tiles — waits up to 3 seconds for each thread to exit.

This order ensures `VideoWriter` is always released before the thread that owns it exits. Do not combine these into one loop.

---

## First-Time Setup — `prompt_for_first_time_setup()`

Sequential `QInputDialog` dialogs. Asks for camera count, then for each camera: IP address, display name, username, password. Writes and encrypts config after all cameras are collected. Returns `[]` if the user cancels at any point.

**To add a new field to the setup wizard:**

In `prompt_for_camera_details()`, add a new `QInputDialog.getText()` call and include the result in the returned dict:

```python
port, ok = QInputDialog.getText(parent, f"Setup{title_suffix}", "RTSP port (default 554):")
if not ok:
    return None
# ...
return {
    "ip_address": ip_address,
    "name": name,
    "username": username,
    "password": password,
    "port": port.strip() or "554",   # new field
}
```

Then read it in `camera_details_to_config()`.

---

## Qt Environment Setup

These lines at module level fix known conflicts between PyQt5 and OpenCV:

```python
os.environ.pop("QT_PLUGIN_PATH", None)
os.environ.pop("QT_QPA_PLATFORM_PLUGIN_PATH", None)
os.environ["OPENCV_FFMPEG_CAPTURE_OPTIONS"] = "rtsp_transport;tcp|stimeout;5000000"
os.environ["QT_QPA_PLATFORM_PLUGIN_PATH"] = os.path.join(
    os.path.dirname(QtCore.__file__), "Qt5", "plugins"
)
```

- The first two lines remove any `QT_PLUGIN_PATH` set by OpenCV's own Qt, which conflicts with PyQt5's Qt.
- `OPENCV_FFMPEG_CAPTURE_OPTIONS` forces RTSP over TCP (more reliable than UDP) with a 5-second connection timeout.
- The last line forces PyQt5 to use its own Qt plugins instead of any system Qt.

**Do not remove these lines.** On most Linux systems they are required for the app to start at all.

**To change the RTSP timeout** (in microseconds):
```python
os.environ["OPENCV_FFMPEG_CAPTURE_OPTIONS"] = "rtsp_transport;tcp|stimeout;10000000"  # 10 seconds
```

---

## Using the Dashboard as a Library

To embed the camera feed or recording logic in another application:

```python
from camera_dashboard import CameraConfig, CameraWorker, load_or_create_key

key = load_or_create_key()

camera = CameraConfig(
    name="Front Door",
    url="rtsp://admin:pass@192.168.1.101:554/cam/realmonitor?channel=1&subtype=0",
    file_prefix="front_door",
)

worker = CameraWorker(camera)
worker.frame_ready.connect(your_frame_handler)
worker.start()

# To start recording:
worker.start_recording()

# To stop:
worker.stop_recording()
worker.stop_worker()
worker.wait(3000)
```

`CameraWorker` is a `QThread` and requires a running `QApplication` to use signals.

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Change recording segment length | `RECORD_DURATION_SECONDS` constant |
| Change disk cleanup threshold | `MAX_DISK_USAGE` constant |
| Change reconnect retry count | `MAX_CONNECT_FAILURES` constant |
| Change reconnect delay | `RECONNECT_DELAY_SECONDS` constant |
| Change recordings storage location | `RECORDINGS_ROOT` constant |
| Change RTSP path for different camera brand | `RTSP_PATH` constant |
| Change RTSP port from 554 | Add `port` to `CameraConfig`, update `build_rtsp_url()` |
| Change recording codec | `_open_new_writer()` — `fourcc` variable |
| Add exponential backoff on reconnect | `CameraWorker.run()` — sleep call in outer loop |
| Add a notification on camera going offline | Connect `failed_permanently` signal in `CameraTile` |
| Add a button to every camera tile | `CameraTile.__init__()` layout |
| Change tile video display size | `self.video.setMinimumSize()` in `CameraTile` |
| Change grid column count | `MainWindow.rebuild_grid()` — `col = index % N` |
| Make tile removal permanent (saves to config) | `MainWindow.remove_tile()` — filter `self.camera_details` + save |
| Change cleanup check frequency | `cleanup_timer.start()` interval in `MainWindow.__init__()` |
| Add a new field to camera setup wizard | `prompt_for_camera_details()` + `CameraConfig` + `camera_details_to_config()` |
| Support pre-built RTSP URLs directly | Store `"url"` key in JSON config — already handled by `camera_details_to_config()` |
| Rotate the encryption key | Decrypt manually with old key, delete both files, restart app |
| Replace Fernet with an external secrets manager | Replace `load_or_create_key`, `encrypt_config`, `decrypt_config` |
| Use only the capture/recording logic | Import `CameraConfig`, `CameraWorker` directly |
| Change RTSP connection timeout | `OPENCV_FFMPEG_CAPTURE_OPTIONS` env var — `stimeout` value in microseconds |

---

## Security Files — Never Commit

| File | Purpose | Action |
|---|---|---|
| `secret.key` | Fernet encryption key | Never commit. Losing this file means losing access to the config. |
| `camera_config.json` | Encrypted camera credentials | Never commit even though it is encrypted. |
| `recordings/` | Video files | Never commit. |

All three are listed in `.gitignore`. Do not remove them.

---

## Dependencies

| Package | Purpose |
|---|---|
| `PyQt5` | GUI framework |
| `opencv-python` | RTSP capture (`CAP_FFMPEG`), frame conversion, `VideoWriter` |
| `cryptography` | Fernet encryption for config file |
| `numpy` | Frame array type annotation and resize operations |

---

## Version Bump Procedure

1. Update `__version__` at the top of `camera_dashboard.py`.
2. Add a `CHANGELOG.md` entry.
3. Push to main and create a release tag on GitHub.

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
