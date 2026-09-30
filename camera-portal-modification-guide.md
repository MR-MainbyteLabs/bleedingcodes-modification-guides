# camera-portal — Modification Guide

Covers **camera-portal v1.1.0** (`pyproject.toml` and `camera_core.__version__` both read 1.1.0).
Written for engineers who will change the code. It is not a user guide. For install and operation, see the [README](https://github.com/BleedingCodes/camera-portal/blob/main/README.md).

---

## 1. Before You Change Anything

1. Stop the server (Ctrl+C in the terminal running `run_server.py`). `edit_cameras.py` and `test_core.py` must never run while it is up. The server holds its own copy of the config and overwrites yours on its next save, and both programs would open the same cameras.
2. Back up `camera_config.json` **and** `secret.key` together. The config cannot be decrypted without that exact key.
3. Work in a venv that has only `opencv-python-headless`. `opencv-python` and `opencv-python-headless` installed together break `import cv2`.
4. Follow the project code standards for every change:
   - Python 3.11+
   - Complete type hints on public interfaces
   - No `eval` / `exec`
   - No home-grown crypto (use `cryptography` and `werkzeug.security`)
   - No shell string interpolation
5. There is no automated test suite. Verification is manual: see [Section 8](#8-testing-a-change).

---

## 2. Architecture Map

| File | Responsibility | Touches network? |
|---|---|---|
| `run_server.py` | Entry point. Loads config, asks for the portal password (first run), starts cameras, waits for the modem IP, serves the portal, handles port switching on Renew link | Serves HTTP |
| `camera_core.py` | Camera engine: encrypted config, `CameraWorker` (one thread per camera), `CameraManager` (owns all workers), recording, disk cleanup | RTSP (outbound TCP) |
| `web_portal.py` | Flask app: pages, JSON API, login, session, MJPEG and snapshot routes | Inbound HTTP |
| `modem_link.py` | Finds the modem-side interface and private IP, picks the port and secret key, writes `portal_url.txt` | Local sockets only |
| `onvif.py` | ONVIF WS-Discovery and stream-path lookup. Standard library only | UDP multicast, outbound HTTP |
| `edit_cameras.py` | CLI: `list`, `set-ip`, `set-path`, `check` | TCP connect test |
| `test_core.py` | Runs the camera engine without the portal | RTSP |
| `templates/dashboard.html`, `templates/login.html` | Jinja pages | — |
| `static/app.js`, `static/style.css` | Browser logic and styling | — |

### Startup order (`run_server.py: main()`)

1. `core.load_or_create_key()` then `core.CameraManager(key)` loads the config.
2. `ensure_password()`: asks in the terminal if no `password_hash` is stored.
3. `ensure_session_secret()`.
4. `manager.start()`: cameras begin connecting.
5. `modem_link.get_link()`: blocks until the modem network has a private IP.
6. `create_app()` then `make_server(link.ip_address, link.port, app, threaded=True)`.
7. Loop: wait for a Renew link event, shut the server down, restart it on the new port. The cameras keep running.

### Frame flow

```text
camera --RTSP/TCP--> CameraWorker._stream_loop --> _latest_frame (under _frame_lock)
                                                     |-> /snapshot/<id>  JPEG, max 640 px wide, polled by app.js
                                                     |-> /stream/<id>    MJPEG, max 960 px wide, STREAM_FPS
                                                     '-> VideoWriter     full resolution MP4, worker thread only
```

Only the worker thread writes `_latest_frame` and the `VideoWriter`. Other threads read through `get_latest_frame()` and set intent flags (`recording`, reconnect event, stop event).

### Locks

| Lock | Protects |
|---|---|
| `CameraManager._lock` | The `workers` dict |
| `CameraManager.config_lock` | `manager.config` changes and every `save_config()` call. Also used by `run_server.py` (`ensure_password`, `renew_link`) |
| `CameraWorker._frame_lock` | `_latest_frame` and `_frame_number` |

Lock order in the current code is `config_lock` then `_lock` (`add_camera` takes `config_lock`, then calls `get_worker` / `_start_worker`, which take `_lock`). Do not add code that takes them in the opposite order.

---

## 3. Config File Schema

`camera_config.json` is a Fernet-encrypted JSON document. Decrypted shape:

```json
{
  "cameras": [
    {
      "ip_address": "192.168.12.108",
      "name": "Front door",
      "username": "admin",
      "password": "…",
      "rtsp_port": 554,
      "rtsp_path": "/Streaming/Channels/101",
      "file_prefix": "front_door"
    }
  ],
  "portal": {
    "password_hash": "scrypt:…",
    "session_secret": "…",
    "port": 41873,
    "url_key": "…",
    "interface": "wlan0"
  }
}
```

- `file_prefix` is the camera ID everywhere: routes, `edit_cameras.py`, and recording filenames.
- `portal.interface` exists only if an interface was pinned (`--interface`).
- **Backward compatibility.** Configs from pyqt-camera-dashboard v3 and camera-portal v1.0.0 have no `rtsp_port` / `rtsp_path`. Code reads them with defaults (`RTSP_PORT`, `RTSP_PATH`). Some legacy entries store a full `url` instead of `ip_address` (`camera_details_to_config` uses `url` when present). `edit_cameras.py set-ip` and `set-path` refuse those entries.
- **Rule for new fields:** read with `.get(field, default)`. Never require a field that old configs lack.

---

## 4. Tunable Constants

Change the value in the file listed, then restart the server. Items marked **not in README** are missing from the README's "Hardcoded Values" table.

### `camera_core.py`

| Constant | Default | Effect and side effects |
|---|---|---|
| `RECORD_DURATION_SECONDS` | `900` | Length of each MP4 file |
| `DEFAULT_FPS` | `15.0` | Used when the camera reports an FPS outside 1–60. **Not in README** |
| `RECONNECT_DELAY_SECONDS` | `3` | Wait between reconnect attempts. **Not in README** |
| `MAX_CONNECT_FAILURES` | `3` | Failed opens before a camera goes Offline. Also appears in the status text (`attempt N/3`) |
| `MAX_DISK_USAGE` | `75` | Percent used on the **whole drive** holding `recordings/` |
| `CLEANUP_INTERVAL_SECONDS` | `3600` | How often disk cleanup runs. **Not in README** |
| `RTSP_PATH`, `RTSP_PORT` | Dahua path, `554` | Defaults for cameras with none saved |
| `MAX_RTSP_PATH_LENGTH` | `200` | Path length limit in `validate_rtsp_path`. **Not in README** |
| `OPENCV_FFMPEG_CAPTURE_OPTIONS` (env string, line ~27) | `rtsp_transport;tcp\|stimeout;5000000` | RTSP over TCP, 5 s timeout (value is in microseconds) |
| Camera name limit | `40` | Hardcoded in **three** places: `CameraManager.add_camera` (`len(name) > 40`), `templates/dashboard.html` (`maxlength="40"`), `onvif.MAX_NAME_LENGTH`. Change all three together |
| Capture buffer | `2` | `cv2.CAP_PROP_BUFFERSIZE` in `CameraWorker.run()`. **Not in README** |
| Shutdown wait | `6` s | Hardcoded in `CameraManager.shutdown()`. Keep it above the 5 s RTSP timeout |

### `web_portal.py`

| Constant | Default | Effect and side effects |
|---|---|---|
| `STREAM_FPS` | `10` | Live viewer frame rate. Does not affect recordings |
| `JPEG_QUALITY` | `70` | Applies to both snapshots and live video. **Not in README** |
| `STREAM_MAX_WIDTH` | `960` | Live viewer width |
| `SNAPSHOT_MAX_WIDTH` | `640` | Tile still width |
| `SESSION_DAYS` | `30` | Login lifetime per device |
| `MAX_LOGIN_FAILURES` | `5` | Wrong passwords per client IP before lockout. **Not in README** |
| `LOCKOUT_SECONDS` | `60` | Window and lockout length. The failure counter is in memory and resets on restart. **Not in README** |

### `modem_link.py`, `run_server.py`, `onvif.py`, others

| File | Constant | Default | Effect |
|---|---|---|---|
| `modem_link.py` | `PORT_RANGE` | `(20000, 60999)` | Range for new random ports. The saved port is unchanged until the next Renew link |
| `modem_link.py` | `poll_seconds` (arg of `wait_for_modem`) | `3.0` | How often to re-check for a modem IP |
| `run_server.py` | `MIN_PASSWORD_LENGTH` | `8` | Portal password minimum |
| `run_server.py` | `RENEW_SWITCH_DELAY_SECONDS` | `1.0` | Delay so the browser receives the new link before the old port closes |
| `onvif.py` | `DISCOVERY_SECONDS` | `3.0` | Find cameras listen time |
| `onvif.py` | `SOAP_TIMEOUT_SECONDS` | `5` | Per ONVIF request |
| `onvif.py` | `MAX_REPLY_BYTES` | `1_000_000` | Larger camera replies are refused. **Not in README** |
| `onvif.py` | `DEFAULT_DEVICE_PATH` | `/onvif/device_service` | Used when the camera reports no address |
| `edit_cameras.py` | `CHECK_TIMEOUT_SECONDS` | `2` | Per-camera timeout in `check` |
| `test_core.py` | `TEST_SECONDS` | `30` | Length of the hardware test |
| `static/app.js` | `POLL_MS` | `2000` | Status refresh. **Not in README** |
| `static/app.js` | `SNAPSHOT_MS` | `1000` | Tile still refresh |

---

## 5. Common Modifications

### 5.1 Add a camera brand preset

1. In `templates/dashboard.html`, inside `<select id="stream-select">`, add `<option value="/your/path">Brand name</option>`. The value is the RTSP path. Write `&` as `&amp;` in the HTML.
2. The path must start with `/`, be 200 characters or fewer, and contain no spaces or `#`. The server enforces this in `validate_rtsp_path`.
3. **Port limitation:** presets always use port 554. `static/app.js` sets `body.rtsp_port = 554` in the submit handler. A brand that needs another default port cannot be a preset as the code stands. Users must pick **Other**. Supporting a per-preset port means storing the port on the `<option>` and reading it in that handler. That change is not implemented and not tested.
4. Update the preset table in the README under "Add a camera by IP address".

### 5.2 Change the recording format or codec

Three places must agree:

1. `CameraWorker._open_new_writer()` in `camera_core.py`: the fourcc (`"mp4v"`) and the filename extension (`.mp4`).
2. `cleanup_by_disk_usage()` in `camera_core.py`: it deletes only `RECORDINGS_ROOT.rglob("*.mp4")`. A new extension means recordings are never cleaned up.
3. The README "Recordings" section and Known limitations.

Whether a codec works depends on the OpenCV build, not on this project. After any change, run `python test_core.py --record` and play the file. The `VideoWriter` opening is checked in code (`writer.isOpened()`), and a failure shows in the tile status as `Recording failed: VideoWriter did not open`.

### 5.3 Add a JSON API endpoint

1. In `web_portal.py`, inside `create_app()`, add `@app.post("/api/…")` (or `.get`).
2. The **first line** of the handler must be `require_api()`. It checks the `X-Portal-Key` header and the session.
3. Put the logic in `CameraManager` or `CameraWorker` (`camera_core.py`). The portal only reads `status`, `online`, `recording`, `get_latest_frame()` and calls public methods.
4. Return `jsonify({"ok": True, …})`. Return failures with `error("message", status)` so `app.js` can display the text.
5. In `static/app.js`, call it through `api(path, method, body)`. Do not use bare `fetch()` for API calls: it would omit the key header and be rejected.
6. Add the route to the docstring list at the top of `web_portal.py`.

### 5.4 Add a button to the camera tile

1. `templates/dashboard.html`: add `<button data-action="yourname">` inside the tile's `.buttons` div.
2. `static/app.js`: add a branch for `button.dataset.action === "yourname"` in the single click listener on `#grid`. It already handles disabling the button, error messages, and `refreshStatus()`.
3. If the button depends on camera state, update `updateTile()` too.
4. Add the API route (5.3) if the button changes server state.

### 5.5 Add a per-camera setting

1. `CameraManager.add_camera()`: add the field to the `details` dict.
2. `camera_details_to_config()`: read it with `details.get(...)` and a default.
3. If it should be editable from the CLI, add it to `edit_cameras.py` (`list_cameras` and a new subcommand).
4. If it comes from the add dialog: `dashboard.html` (field), `app.js` (submit handler body), `web_portal.py: api_add()` (read and pass it).
5. If it changes what makes two entries "the same camera", update the `same_stream` check in `add_camera()`.

### 5.6 Make Remove permanent

Current behavior: `remove_camera()` stops and hides a camera for this session. It stays in the config and returns at restart. Changing it touches:

1. `CameraManager.remove_camera()`: also delete the entry from `config["cameras"]` and call `save_config()`, both under `config_lock`.
2. `CameraManager.add_camera()`: the "removed this session, bring it back" branch becomes dead code.
3. `static/app.js`: the confirm text says the camera "comes back when the server restarts".
4. README: the tile controls table, Known limitations, and `CHANGELOG.md`.

### 5.6b Restyle the portal

Colors are CSS variables in `:root` at the top of `static/style.css` (`--bg`, `--panel`, `--button`, `--border`, `--text`, `--muted`, `--live`, `--down`, `--rec`, `--primary`). Change them there. No build step.

### 5.7 Serve the portal as a background service

Not covered in the README, and no unit file ships with the project. Constraints from the code:

- The first run needs a terminal. `ensure_password()` exits with an error if there is no stored password and `stdin` is not a TTY. `--reset-password` has the same requirement. Run it interactively once before any service.
- State files (`camera_config.json`, `secret.key`, `portal_url.txt`, `recordings/`) are resolved from the script's own folder (`SCRIPT_DIR`), not the working directory.
- Startup order is tolerant: `wait_for_modem()` polls until the modem network has an IP, so the service may start before the network is up.
- The port and key are stored, so the link survives restarts. Run the venv's Python explicitly.

### 5.8 Serve over HTTPS

Not implemented and not tested. Known touchpoints if you try:

1. `run_server.py`: `make_server(...)` accepts an `ssl_context` argument.
2. `modem_link.py`: `ModemLink.url` hardcodes `http://`. That URL is printed in the banner, saved to `portal_url.txt`, and returned by Renew link.
3. `web_portal.py`: add `SESSION_COOKIE_SECURE=True` to the `app.config.update(...)` call.
4. Browsers will warn about a self-signed certificate on a private IP.

### 5.9 IPv6

Not supported. IPv4 is assumed throughout: `AF_INET` sockets in `modem_link.py` and `onvif.py`, and `ipaddress.IPv4Address` checks in `camera_core.py` and `web_portal.py`.

---

## 6. Rules You Must Not Break

These protect the security model. The README and CHANGELOG state them as features.

1. **Every route checks the key first.** Page, asset, snapshot, and stream routes call `require_key()`. API routes call `require_api()`. A route without one answers requests that lack the secret key. Any new static file must be added to `PUBLIC_ASSETS` or `PRIVATE_ASSETS`; nothing else is served (`static_folder=None`).
2. **Read portal settings live.** `create_app()` reads `manager.config["portal"]` on every request through `portal()`. Do not copy `url_key` or `password_hash` into a variable at startup, or Renew link and password changes stop taking effect.
3. **Keep request logging quiet.** `logging.getLogger("werkzeug")` is set to `WARNING` because access logs print URLs containing the key. Do not enable access logs and do not log `request.url`.
4. **Never expose camera credentials.** `CameraConfig.url` contains the username and password. `status_list()` returns only id, name, status, online, and recording. Keep credentials out of API responses, templates, and print statements.
5. **Save config one way.** Use `core.save_config()` while holding `manager.config_lock`. It encrypts in memory and swaps the file atomically with 0o600 permissions. Do not write plaintext JSON.
6. **The `VideoWriter` belongs to the worker thread.** Other threads set `recording = True/False` and the worker opens and closes the file. This fixed a real race in pyqt-camera-dashboard v3.0.1.
7. **Bind to a private IP only.** `modem_link.detect()` rejects loopback and non-private addresses. Do not loosen it, and do not bind to `0.0.0.0`.
8. **Untrusted text stays text.** Camera names and models come from the network. Templates rely on Jinja autoescaping, and `app.js` uses `textContent`. Never use `innerHTML` with them.
9. **Rebuild camera addresses from the sender's IP.** `onvif._rebase_url()` ignores the host a camera reports. This stops a device on the network from redirecting the portal or a camera password to another host.
10. **Keep cookie flags.** `HttpOnly` and `SameSite=Strict` are set in `create_app()`.
11. **Import order.** `camera_core.py` sets `OPENCV_FFMPEG_CAPTURE_OPTIONS` before `import cv2`, and its comment says this is required. New entry points should import `camera_core` before any module that imports `cv2`. Note: `test_core.py` and `web_portal.py` import `cv2` first when run alone. In a test with OpenCV 4.13.0 against a local fake RTSP server, the transport was TCP with the variable set before the import, set after it, and not set at all. So on that version the order made no difference to the transport. The `stimeout` part was not tested, and neither was OpenCV 4.8 (the minimum in `requirements.txt`).

---

## 7. Network Behavior (for firewall and deployment changes)

| Traffic | Direction | Protocol and port |
|---|---|---|
| Portal | Inbound to this computer | TCP, random port 20000–60999, shown in the banner |
| Camera video | Outbound to each camera | RTSP over TCP, port 554 or the camera's saved port |
| Find cameras probe | Outbound, multicast | UDP to `239.255.255.250:3702` |
| Find cameras replies | Inbound | UDP, unicast, to a random local port |
| ONVIF login and stream lookup | Outbound | HTTP to the camera's ONVIF port |
| `edit_cameras.py check` | Outbound | TCP connect to each RTSP port |

The portal port changes on every Renew link, so any firewall rule tied to the port must be redone. `ufw` rule syntax used in the README was dry-run checked with `ufw --dry-run`, which validates syntax only, not traffic.

---

## 8. Testing a Change

There are no automated tests. Run what applies:

1. Syntax check: `python -m py_compile camera_core.py web_portal.py run_server.py modem_link.py onvif.py edit_cameras.py test_core.py`
2. Hardware check (server stopped): `python test_core.py`, then `python test_core.py --record`. Open the `snapshot_*.jpg` files and play one recording.
3. Network path: `python edit_cameras.py check` and `python modem_link.py`.
4. Start the server and, from a phone or second laptop on the same network:
   - Link without `?key=` returns "Not Found".
   - Wrong password 5 times locks that address out.
   - Add a camera by preset, by **Other**, and by **Detect automatically**.
   - Find cameras lists cameras and marks existing ones **Added**.
   - Start and stop recording, then confirm the file lands in `recordings/YYYY-MM-DD/HH/`.
   - Reconnect an Offline camera.
   - Renew link: old link stops working, the other device is signed out, this browser moves to the new link.
5. If you changed `edit_cameras.py`, test it with the server stopped.

---

## 9. Releasing a Change

This repo keeps its version in `pyproject.toml` and in `camera_core.py` (`__version__`). It has no `src/<package>/__init__.py`.

1. Bump `version` in `pyproject.toml`.
2. Bump `__version__` in `camera_core.py` to the same value.
3. Use semantic versioning: bug fixes only = patch (1.1.0 to 1.1.1); new features = minor.
4. Add an entry at the top of `CHANGELOG.md`.
5. Update the README if you changed a constant, a control, or a command.
6. Push to `main`, then create a new release tag on GitHub.

---

## 10. Known Limits After Modification

- Development-grade HTTP server. `run_server.py` uses Werkzeug's `make_server` with `threaded=True`, not a production WSGI server.
- Plain HTTP, Linux only, IPv4 only.
- Login lockout counters live in memory and reset when the server restarts.
- A hard kill can leave the MP4 being written unplayable. Ctrl+C shuts down cleanly.

---

*camera-portal Modification Guide — MainbyteLabs*
*https://github.com/MR-MainbyteLabs*
