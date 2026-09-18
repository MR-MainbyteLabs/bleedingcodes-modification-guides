# microcam-benchscope — Modification Guide
**Status:** Beta  
**Active editions:** `modern/microcam_hyperlab_ffmpeg.py`, `modern/microcam_hyperlab_pyav.py`  
**Legacy versions:** `legacy/` — development history only, not for modification

---

## Who This Document Is For

Engineers and developers who want to extend, adapt, or embed microcam-benchscope beyond what the UI exposes. This is not a usage guide — the SOP and README cover that. This document tells you where the code lives, what each class owns, and exactly where to make each class of change.

---

## Edition Differences at a Glance

Both modern editions share the same class structure and nearly identical logic. The recorder is the only substantive difference.

| | FFmpeg Edition | PyAV Edition |
|---|---|---|
| File | `microcam_hyperlab_ffmpeg.py` | `microcam_hyperlab_pyav.py` |
| Recorder class | `FFmpegRecorder` | `PyAVRecorder` |
| Recording mechanism | Spawns `ffmpeg` subprocess, pipes raw BGR bytes via stdin | Encodes via PyAV bindings on a worker thread, no subprocess |
| Pause support | No | Yes (`SPACE` key) |
| Extra config fields | — | `record_codec`, `record_queue_size` |
| Extra dependency | `ffmpeg` system binary | `av` (PyAV) pip package |

Unless you are specifically modifying recording behavior, all other changes apply identically to both files. The guide calls out recorder-specific sections explicitly.

---

## Architecture Overview

Both editions are single-file applications. All logic lives in one `.py` file per edition with no imports from the other.

```
AppConfig (dataclass, global singleton)
    ↓ read/written by all classes
CameraWorker (QObject, daemon thread)
    ↓ frame_ready Signal
MainWindow.on_raw_frame()
    ↓ raw_frame stored
MainWindow.update_display() (QTimer, ~30Hz)
    ↓ FrameProcessor.process()
    ↓ draw_overlays()
    ↓ Recorder.write()    ← FFmpegRecorder or PyAVRecorder
    ↓ VideoLabel.setPixmap()
```

---

## Class Map

| Class | Lines (FFmpeg) | Lines (PyAV) | Owns |
|---|---|---|---|
| `AppConfig` | 92–267 | 98–307 | All persistent settings. Loaded from / saved to JSON. |
| `FrameProcessor` | 269–356 | 308–403 | All image processing. Stateless — reads `config`, returns `np.ndarray`. |
| `CameraWorker` | 358–443 | 405–492 | V4L2 capture loop. Emits `frame_ready` Signal. |
| `FFmpegRecorder` | 445–537 | — | Records via ffmpeg subprocess, stdin pipe. |
| `PyAVRecorder` | — | 494–648 | Records via PyAV on worker thread with frame queue. |
| `VideoLabel` | 539–661 | 649–804 | Mouse/keyboard input. Measurement drawing. Display widget. |
| `MainWindow` | 663–end | 806–end | Wires everything together. All tabs, overlays, analysis actions. |

---

## Global Configuration — `AppConfig`

`AppConfig` is a `@dataclass` (not frozen). One global instance named `config` is created at module load. All classes read and write it directly — there is no dependency injection.

### All Fields

**Camera:**
```python
device: str = "/dev/video0"         # V4L2 device path
width: int = 1920
height: int = 1080
fps: int = 30
fourcc: str = "MJPG"                # "MJPG" or "YUYV"
```

**Image processing:**
```python
brightness: int = 0                 # cv2.convertScaleAbs beta
contrast: float = 1.0               # cv2.convertScaleAbs alpha
gamma: float = 1.0
saturation: float = 1.0
threshold_value: int = 128          # Binary threshold level
```

**View:**
```python
zoom: float = 1.0                   # 1.0–30.0
pan_x: int = 0                      # Pixel offset from center
pan_y: int = 0
```

**Calibration:**
```python
pixels_per_mm: float = 100.0        # Set via calibration workflow. Never 0.
known_mm: float = 10.0              # Reference distance for calibration
```

**Overlay toggles:**
```python
show_crosshair: bool = True
show_grid: bool = False
show_ruler: bool = True
show_focus_meter: bool = True
show_histogram: bool = False
show_measurements: bool = True
show_fps: bool = True               # FFmpeg edition
show_status_text: bool = True       # PyAV edition (same field, different name)
```

**Processing toggles:**
```python
grayscale: bool = False
invert: bool = False
sharpen: bool = False
denoise: bool = False
clahe: bool = False
edges: bool = False
threshold: bool = False
adaptive_threshold: bool = False
false_color: bool = False
snapshot_clean: bool = False        # Snapshot without overlays
recording_clean: bool = False       # Record without overlays
```

**Recording (FFmpeg edition):**
```python
record_crf: int = 18                # x264 CRF (0=lossless, 51=worst)
record_preset: str = "veryfast"     # x264 preset
```

**Recording (PyAV edition — additional fields):**
```python
record_codec: str = "libx264"       # Falls back to mpeg4 if unavailable
record_crf: int = 18
record_preset: str = "veryfast"
record_queue_size: int = 90         # Max frames in encode queue before drop
```

### Adding a New Config Field

1. Add the field with a default to `AppConfig`.
2. `load_config()` and `save_config()` handle it automatically — they iterate `hasattr` and `asdict()` respectively. No changes needed there.
3. Add a UI widget in the appropriate `make_*_tab()` method in `MainWindow`.
4. Wire the widget to `config.<field>` via `lambda` or a setter method.

**Example — adding a new processing toggle:**
```python
# 1. In AppConfig:
auto_rotate: bool = False

# 2. In make_image_tab() toggles list:
("Auto Rotate", "auto_rotate"),

# 3. In FrameProcessor.process() — add after the false_color block:
if config.auto_rotate:
    frame = cv2.rotate(frame, cv2.ROTATE_90_CLOCKWISE)
```

---

## Image Processing — `FrameProcessor`

Stateless class with two static methods. Reads `config` directly.

### `FrameProcessor.process(raw: np.ndarray) -> np.ndarray`

Applied to every frame before display and recording. Processing order is fixed:

1. `convertScaleAbs` — brightness + contrast
2. `apply_gamma` — gamma LUT
3. Saturation — BGR→HSV, scale S channel, HSV→BGR
4. Denoise — `fastNlMeansDenoisingColored`
5. CLAHE — BGR→LAB, CLAHE on L channel, LAB→BGR
6. Grayscale — convert then expand back to BGR
7. Invert — `bitwise_not`
8. Sharpen — 3×3 unsharp kernel via `filter2D`
9. Edges — Gaussian blur → Canny
10. Binary threshold — `cv2.threshold`
11. Adaptive threshold — `cv2.adaptiveThreshold`
12. False color — `cv2.COLORMAP_TURBO`
13. `apply_zoom_pan` — always last

### `FrameProcessor.apply_zoom_pan(frame) -> np.ndarray`

Crops a centered region (size = `1/zoom` of original) then resizes back to original dimensions. Pan offsets shift the crop center. Returns the original frame unchanged if `zoom <= 1.0`.

### Adding a New Processing Mode

Insert a new block in `process()` at the position in the pipeline where it logically belongs. Add a boolean toggle to `AppConfig`, a checkbox in `make_image_tab()`, and the processing logic in the new block.

**Example — adding a morphological dilation step:**
```python
# AppConfig:
dilate: bool = False

# In FrameProcessor.process(), after the sharpen block:
if config.dilate:
    kernel = np.ones((3, 3), np.uint8)
    frame = cv2.dilate(frame, kernel, iterations=1)
```

### Replacing a Processing Step

Each step is an independent `if` block. To replace the sharpen kernel:

```python
# Current (lines ~330–334):
if config.sharpen:
    kernel = np.array([[0, -1, 0], [-1, 5, -1], [0, -1, 0]], dtype=np.float32)
    frame = cv2.filter2D(frame, -1, kernel)

# Replace with a stronger kernel:
if config.sharpen:
    kernel = np.array([[-1, -1, -1], [-1, 9, -1], [-1, -1, -1]], dtype=np.float32)
    frame = cv2.filter2D(frame, -1, kernel)
```

### Changing the False Color Map

The `cv2.COLORMAP_*` constant in the `false_color` block is the only thing to change:

```python
# Current:
frame = cv2.applyColorMap(gray, cv2.COLORMAP_TURBO)

# Options: COLORMAP_JET, COLORMAP_HOT, COLORMAP_INFERNO, COLORMAP_PLASMA, etc.
frame = cv2.applyColorMap(gray, cv2.COLORMAP_INFERNO)
```

---

## Camera Capture — `CameraWorker`

`QObject` subclass running a capture loop on a daemon thread. Communicates with `MainWindow` via Qt Signals.

### Signals

```python
frame_ready = Signal(object)    # Emits np.ndarray for each captured frame
error = Signal(str)             # Camera open failure or read failure
status = Signal(str)            # Informational — device opened with actual vs requested params
```

### Key Methods

**`start()`** — stops any existing capture, sets `running = True`, starts daemon thread.

**`stop()`** — sets `running = False`, joins thread with 3-second timeout (chosen to exceed the worst-case sleep at `fps=1`), releases `cv2.VideoCapture`.

**`open_capture()`** — opens the V4L2 device, sets FOURCC, resolution, FPS, and buffer size. Emits `status` with requested vs actual parameters.

**`_loop()`** — capture loop. Reads frames at the target interval. If paused (PyAV edition only) and `last_frame` is available, re-emits the last frame instead of reading a new one.

### Changing Capture Parameters

All capture parameters come from `config`. Change them by updating `config` and calling `CameraWorker.start()` (which is what `MainWindow.restart_camera()` does after reading the UI widgets).

**To force a specific buffer size:**
```python
# In open_capture(), change:
self.cap.set(cv2.CAP_PROP_BUFFERSIZE, 1)
# To:
self.cap.set(cv2.CAP_PROP_BUFFERSIZE, 2)   # Slightly more buffering
```

### Adding a Second Camera

`CameraWorker` is not hardcoded to a single device. To run a second camera:

```python
# In MainWindow.__init__():
self.camera2 = CameraWorker()
self.camera2.frame_ready.connect(self.on_raw_frame_2)
self.camera2.error.connect(self.log)

# Add a second VideoLabel and handler:
def on_raw_frame_2(self, frame: np.ndarray):
    self.raw_frame_2 = frame
```

You will need a second `config` instance or a way to parameterize `CameraWorker` with a device path rather than reading from the global `config.device`.

---

## Recording

### FFmpeg Edition — `FFmpegRecorder`

Spawns an `ffmpeg` subprocess on `start()`. Feeds raw BGR frames to `ffmpeg`'s stdin as a rawvideo stream. `ffmpeg` encodes to H.264 MP4 and writes the file directly.

```python
recorder.start(width, height, fps)  # Returns Path of output file
recorder.write(frame_bgr)           # Thread-safe, resizes if dimensions changed
recorder.stop()                     # Closes stdin, waits up to 5s, returns Path
recorder.active                     # True if ffmpeg subprocess is running
```

**To change the output codec** (FFmpeg edition):

In `FFmpegRecorder.start()`, change the `-c:v` argument:
```python
"-c:v", "libx265",      # HEVC — better compression, slower
# or
"-c:v", "libvpx-vp9",   # VP9
```

**To add audio recording** (FFmpeg edition):

Remove the `-an` flag from the ffmpeg command in `start()` and add audio capture arguments. Requires a PulseAudio or ALSA source.

### PyAV Edition — `PyAVRecorder`

Uses a `queue.Queue` (max size = `config.record_queue_size`, default 90 frames). `write()` does a non-blocking `put_nowait()` — frames are dropped silently if the queue is full, incrementing `self.dropped_frames`. A worker thread dequeues and encodes via PyAV. Encoding happens on the worker thread, keeping the GUI thread free.

```python
recorder.start(width, height, fps)  # Returns Path
recorder.write(frame_bgr)           # Non-blocking enqueue; drops frame if full
recorder.stop()                     # Sends None sentinel, waits up to 8s, returns Path
recorder.active                     # True while running flag is set
recorder.dropped_frames             # Count of dropped frames since start
recorder.error                      # Set to string if encoding failed
```

**To change the encode queue size** (PyAV edition):

```python
# In AppConfig:
record_queue_size: int = 180    # More buffer for bursty workloads
```

**To change the codec** (PyAV edition):

```python
# In AppConfig:
record_codec: str = "libx265"   # Falls back to mpeg4 if unavailable in this PyAV build
```

---

## Display and Input — `VideoLabel`

Subclass of `QLabel`. Handles all mouse and keyboard input for the viewer.

### Coordinate Mapping

`map_to_frame(x, y)` converts widget pixel coordinates to frame pixel coordinates, accounting for letterboxing when the frame aspect ratio differs from the widget size.

### Signals

```python
measurement_changed = Signal()          # Any change to active or locked measurements
request_snapshot = Signal()             # P key
request_record_toggle = Signal()        # R key
```

### Keyboard Shortcuts

| Key | Action |
|---|---|
| W / A / S / D | Pan (±50px per keypress) |
| Mouse wheel | Zoom ±0.25 |
| Middle / Right mouse drag | Pan |
| Left mouse drag | Draw measurement |
| P | Snapshot |
| R | Record toggle |
| L | Lock current measurement |
| C | Clear all measurements |
| SPACE | Pause (PyAV edition only) |

### Adding a Keyboard Shortcut

In `VideoLabel.keyPressEvent()`:

```python
elif key == Qt.Key_G:
    config.show_grid = not config.show_grid
```

### Adding a New Overlay

All overlays are drawn in `MainWindow.draw_overlays()` on the `display` frame copy before it is sent to `VideoLabel`. Add a new `if config.<toggle>:` block there. The frame is BGR, full resolution.

**Example — adding a center dot:**
```python
if config.show_center_dot:
    cx, cy = frame.shape[1] // 2, frame.shape[0] // 2
    cv2.circle(frame, (cx, cy), 8, (0, 0, 255), -1)
```

---

## Analysis Functions — `MainWindow`

All analysis functions operate on `self.processed_frame` (the frame after `FrameProcessor.process()`, without overlays). Results are saved to `self.last_analysis` dict and logged. Saved images go to `ANALYSIS_DIR`.

| Method | What it does |
|---|---|
| `run_ocr()` | 2.5× upscale → Gaussian blur → adaptive threshold → Tesseract `--psm 6` |
| `detect_solder_bridges()` | CLAHE → Otsu threshold → morphological close → contour filter by area and aspect ratio |
| `segment_pads_components()` | Canny edges → morphological close → contour filter by area and minimum size |
| `estimate_trace_width()` | Samples pixel values along the active measurement line, finds the longest bright run |
| `auto_white_balance_preview()` | Per-channel gray world correction on `self.raw_frame` |
| `build_focus_stack()` | Per-pixel argmax of Laplacian variance across captured frames |
| `build_hdr()` | Per-pixel median across captured frames → CLAHE post-process |
| `generate_report()` | Writes HTML report linking recent snapshots, analysis images, and videos |

### Tuning Solder Bridge Detection

In `detect_solder_bridges()`, the contour filter is:

```python
if not (25 <= area <= 7000):    # Area bounds in pixels²
    continue
if aspect >= 2.0 and min(w, h) <= 45:   # Elongated + narrow = bridge candidate
```

Adjust `area`, `aspect`, and `min(w, h)` for your image resolution and component density.

### Tuning Pad Segmentation

In `segment_pads_components()`:

```python
if not (80 <= area <= 80000):   # Area bounds
    continue
if w < 4 or h < 4:             # Minimum bounding box size
    continue
```

### Adding a New Analysis Function

1. Add a method to `MainWindow`:

```python
def measure_pcb_area(self):
    if self.processed_frame is None:
        return
    # ... your analysis logic ...
    result = "..."
    self.last_analysis["PCB Area"] = result
    self.log(result)
```

2. Add a button to `make_analysis_tab()` in the `buttons` list:

```python
("Measure PCB Area", self.measure_pcb_area),
```

---

## Output Paths

All output directories are module-level `Path` constants, created at startup:

```python
CAPTURE_DIR   = Path("captures")
SNAPSHOT_DIR  = CAPTURE_DIR / "snapshots"   # PNG snapshots
VIDEO_DIR     = CAPTURE_DIR / "videos"       # MP4 recordings
ANALYSIS_DIR  = CAPTURE_DIR / "analysis"     # Analysis output images
STACK_DIR     = CAPTURE_DIR / "focus_stack"  # Focus stack frames and result
HDR_DIR       = CAPTURE_DIR / "hdr"          # HDR fusion output
REPORT_DIR    = CAPTURE_DIR / "reports"      # HTML inspection reports
```

All paths are relative to the working directory at launch. To change them, edit the constants at the top of the file.

### Changing the Output Base Directory

```python
# Change CAPTURE_DIR and all subdirs follow:
CAPTURE_DIR = Path("/mnt/nas/inspections")
```

### Changing the Snapshot Filename Pattern

In `MainWindow.save_snapshot()`:

```python
file = SNAPSHOT_DIR / f"snapshot_{now_stamp()}.png"
# Change to include device name:
file = SNAPSHOT_DIR / f"{Path(config.device).name}_{now_stamp()}.png"
```

`now_stamp()` returns `YYYYMMDD_HHMMSS`.

---

## Config Persistence

Config is saved to `microcam_benchscope_config.json` in the working directory.

- **Load:** `load_config()` — called once in `main()` before the GUI starts. Iterates the JSON keys, sets any that exist as fields on `config`. Unknown keys are silently ignored. Validates `pixels_per_mm > 0` after load.
- **Save:** `save_config()` — called on window close via `closeEvent()` and manually via Ctrl+S / Save Config menu.
- **Format:** Pretty-printed JSON via `json.dumps(asdict(config), indent=4)`.

Any field added to `AppConfig` with a default value is automatically included in save/load with no additional code.

---

## Adding a New UI Tab

In `MainWindow.__init__()`, tabs are added via:

```python
self.tabs.addTab(self.make_camera_tab(), "Camera")
```

To add a new tab:

```python
# 1. Write the builder method:
def make_my_tab(self):
    tab = QWidget()
    layout = QGridLayout(tab)
    # ... add widgets ...
    return tab

# 2. Add it in __init__:
self.tabs.addTab(self.make_my_tab(), "My Tab")
```

---

## Embedding in a Larger Application

Both editions are single-file. To embed the camera view and processing pipeline in a larger PySide6 application:

1. Import `AppConfig`, `FrameProcessor`, `CameraWorker`, and `VideoLabel`.
2. Instantiate `AppConfig` and set it as the global `config` (or refactor to pass it explicitly — the global is the main coupling point).
3. Instantiate `CameraWorker`, connect `frame_ready` to your own slot.
4. In your slot, call `FrameProcessor.process(raw_frame)` and display the result.

The classes themselves have no hard dependency on `MainWindow` — only `VideoLabel.request_snapshot` and `request_record_toggle` signals connect to `MainWindow` methods, and those connections are made in `MainWindow.__init__()`, not inside the classes.

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add a new image processing mode | `AppConfig` field + `FrameProcessor.process()` new block + checkbox in `make_image_tab()` |
| Change the sharpen / CLAHE parameters | `FrameProcessor.process()` — the relevant block |
| Change the false color map | `FrameProcessor.process()` — `false_color` block, `cv2.COLORMAP_*` constant |
| Add a new overlay | `MainWindow.draw_overlays()` + `AppConfig` toggle |
| Add a keyboard shortcut | `VideoLabel.keyPressEvent()` |
| Add an analysis function | New `MainWindow` method + entry in `make_analysis_tab()` buttons list |
| Change output file paths | Module-level `Path` constants at the top of the file |
| Change recording codec | `FFmpegRecorder.start()` `-c:v` arg (FFmpeg) or `AppConfig.record_codec` (PyAV) |
| Add a new config field | `AppConfig` field with default — save/load handled automatically |
| Add a new UI tab | `make_my_tab()` method + `self.tabs.addTab()` in `__init__()` |
| Change calibration reference distance | `AppConfig.known_mm` default, or set via the Known Distance mm spin in the Overlays tab |
| Tune solder bridge detection sensitivity | `detect_solder_bridges()` — area bounds, aspect ratio, max width |
| Tune pad segmentation | `segment_pads_components()` — area bounds, min bounding box size |
| Add a second camera | Second `CameraWorker` instance + second `VideoLabel` + separate frame handler |
| Use the processing pipeline without the GUI | Import `AppConfig`, `FrameProcessor`, `CameraWorker` — bypass `MainWindow` |

---

## Dependencies

| Package | Edition | Purpose |
|---|---|---|
| `PySide6` | Both | GUI framework |
| `opencv-python` | Both | All CV operations, V4L2 capture |
| `numpy` | Both | Frame array operations |
| `pytesseract` | Both | OCR (optional — gracefully skipped if absent) |
| `av` (PyAV) | PyAV only | H.264 encoding via FFmpeg bindings |
| `ffmpeg` (system binary) | FFmpeg only | Video encoding subprocess |
| `v4l-utils` (system) | Both | `v4l2-ctl` for device info and hardware controls |
| `tesseract-ocr` (system) | Both | Tesseract backend for pytesseract |

Optional: `opencv-contrib-python` instead of `opencv-python` — adds SIFT, SURF, and other contrib algorithms.

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
