# ssh-vid-mover — Modification Guide
**File:** `ssh_vid_mover.py` (single-file script)  
**Version:** 1.0.1  
**Entry point:** `main()`  
**Dependency:** `paramiko`

---

## Who This Document Is For

Engineers and developers who need to adapt ssh-vid-mover to a different camera setup, file filtering scheme, transfer strategy, or automation workflow. This is not a usage guide — the README covers that. This document tells you where the code lives and exactly where to make each class of change.

---

## Architecture Overview

```
main()
  ├── parse_args()                    # CLI flags override defaults
  ├── connect_to_target()             # SSH connection, RejectPolicy
  ├── sftp = ssh.open_sftp()
  ├── create_transfer_list(sftp)      # remote_walk() + name/extension filter
  └── download_and_remove_files()     # .part download → size verify → rename → delete remote
```

No classes. Seven functions plus module-level constants and logging setup.

---

## Module-Level Constants

All default values and filter rules are at the top of the file. **Change these constants rather than editing function logic** for any routine reconfiguration.

```python
DEFAULT_TARGET: str = "192.168.1.105"
DEFAULT_USERNAME: str = "side"
DEFAULT_REMOTE_FOLDER: str = "/home/side/Python/Cam_System/recordings"
DEFAULT_DESTINATION: str = "/media/fight/Tb/Downloaded_Recordings"

VIDEO_NAMES: tuple[str, ...] = (
    "d-link",
    "amcrestbullet",
)

VIDEO_EXTENSIONS: tuple[str, ...] = (
    ".mp4",
    ".avi",
    ".mkv",
    ".mov",
    ".ts",
    ".m4v",
)
```

`VIDEO_NAMES` are matched as case-insensitive substrings against the remote filename. `VIDEO_EXTENSIONS` are matched against the file extension (also case-insensitive). Both conditions must be true for a file to be included.

### Common Constant Changes

**Add a new camera name:**
```python
VIDEO_NAMES: tuple[str, ...] = (
    "d-link",
    "amcrestbullet",
    "hikvision-front",    # add here
)
```

**Restrict to MP4 only:**
```python
VIDEO_EXTENSIONS: tuple[str, ...] = (".mp4",)
```

**Change the default remote path:**
```python
DEFAULT_REMOTE_FOLDER: str = "/mnt/nas/recordings"
```

---

## CLI — `parse_args()`

All five constants above have a corresponding CLI flag. Flags override constants at runtime without editing the file.

| Flag | Default | Type |
|---|---|---|
| `--target` | `DEFAULT_TARGET` | str |
| `--username` | `DEFAULT_USERNAME` | str |
| `--remote` | `DEFAULT_REMOTE_FOLDER` | str |
| `--dest` | `DEFAULT_DESTINATION` | str |
| `--port` | `22` | int |

### Adding a New CLI Flag

To expose `VIDEO_NAMES` as a CLI argument:

```python
parser.add_argument(
    "--name",
    action="append",
    dest="video_names",
    default=None,
    help="Camera name substring to match (repeatable; default: VIDEO_NAMES constant)",
)
```

Then in `main()`:
```python
names = tuple(args.video_names) if args.video_names else VIDEO_NAMES
transfer_list = create_transfer_list(sftp, remote_folder, names=names)
```

And update `create_transfer_list()` to accept `names` as a parameter instead of reading the module constant.

---

## Remote Walk — `remote_walk(sftp, remote_directory)`

Generator. Recursively walks `remote_directory` via `sftp.listdir_attr()`. Yields full remote file paths (strings) for regular files only. Directories recurse. Symlinks, devices, and other non-regular entries are silently skipped.

```python
for item in entries:
    remote_path = str(PurePosixPath(remote_directory) / item.filename)
    if stat.S_ISDIR(item.st_mode):
        yield from remote_walk(sftp, remote_path)   # recurse
    elif stat.S_ISREG(item.st_mode):
        yield remote_path                            # emit
```

`IOError` on `listdir_attr()` (e.g. permission denied on a subdirectory) is caught and logged as a warning — the walk continues into other directories.

### Changing Walk Behavior

**Non-recursive (top-level directory only):**

Replace `remote_walk()` calls in `create_transfer_list()` with a direct `sftp.listdir_attr()`:

```python
for item in sftp.listdir_attr(remote_folder):
    if stat.S_ISREG(item.st_mode):
        yield str(PurePosixPath(remote_folder) / item.filename)
```

**Skip hidden files** (names starting with `.`):

In `remote_walk()`, add before the `if stat.S_ISDIR` check:
```python
if item.filename.startswith("."):
    continue
```

**Filter by modification time** (e.g. only files modified in the last 24 hours):

In `remote_walk()`, after checking `stat.S_ISREG`:
```python
import time
cutoff = time.time() - 86400   # 24 hours
if stat.S_ISREG(item.st_mode) and item.st_mtime >= cutoff:
    yield remote_path
```

---

## Transfer List — `create_transfer_list(sftp, remote_folder)`

Calls `remote_walk()` and applies two filters to every yielded path:

1. **Name filter:** `any(name in filename_lower for name in names_lower)` — substring match, case-insensitive.
2. **Extension filter:** `extension_lower in VIDEO_EXTENSIONS` — exact match on the suffix.

Both must pass. The double guard prevents name-matching non-video files (e.g. a `.txt` log named `amcrestbullet_errors.txt`) from being included and deleted.

### Changing the Filter Logic

**Match all files with a valid extension** (ignore `VIDEO_NAMES`):

```python
# Replace the combined condition:
if extension_matches:
    transfer_list.append(remote_file)
```

**Add a minimum file size filter** (e.g. skip files under 1 MB):

Requires an extra `sftp.stat()` call per file — adds latency on large directories:
```python
if name_matches and extension_matches:
    size = sftp.stat(remote_file).st_size
    if size >= 1024 * 1024:    # 1 MB minimum
        transfer_list.append(remote_file)
```

**Match by regex instead of substring:**

```python
import re
patterns = [re.compile(p, re.IGNORECASE) for p in VIDEO_NAMES]
name_matches = any(p.search(filename_lower) for p in patterns)
```

---

## Download and Remove — `download_and_remove_files(sftp, transfer_list, remote_folder, destination)`

This function owns all transfer safety logic. The sequence per file is strict and must not be reordered:

```
1. Capture remote_size = sftp.stat(remote_file).st_size   ← before download
2. sftp.get(remote_file, str(temporary_path))              ← download to .part
3. Verify temporary_path exists
4. Verify local_size == remote_size
5. os.replace(temporary_path, local_path)                  ← atomic rename
6. sftp.remove(remote_file)                                ← delete remote only after verify
```

The remote file is **never** deleted unless steps 1–5 all succeed.

### `.part` File Pattern

`temporary_path = Path(f"{local_path}.part")`

If a `KeyboardInterrupt` or any `Exception` occurs, `.part` is removed and the remote file is left untouched. The transfer summary counts the file as failed.

### Existing Local File Behavior

If `local_path` already exists at transfer time, a `WARNING` is logged and the download proceeds — the local copy is overwritten. Size verification still runs. This situation indicates a previous run transferred the file but failed to delete the remote original (e.g. was interrupted between step 5 and step 6).

### Changing the Verification Method

**To add SHA-256 verification** (stronger than size-only):

After step 4, before `os.replace()`:
```python
import hashlib

def sha256_path(path):
    digest = hashlib.sha256()
    with open(path, "rb") as f:
        while chunk := f.read(1024 * 1024):
            digest.update(chunk)
    return digest.hexdigest()

def remote_sha256(sftp, remote_path):
    digest = hashlib.sha256()
    with sftp.open(remote_path, "rb") as f:
        while chunk := f.read(1024 * 1024):
            digest.update(chunk)
    return digest.hexdigest()

local_hash = sha256_path(temporary_path)
remote_hash = remote_sha256(sftp, remote_file)
if local_hash != remote_hash:
    raise RuntimeError(f"SHA-256 mismatch: {remote_file}")
```

### Changing the Output Directory Structure

Currently, the remote directory structure is preserved relative to `remote_folder`:

```python
relative_path = remote_path.relative_to(PurePosixPath(remote_folder))
local_path = destination / Path(*relative_path.parts)
```

**To flatten into a single directory** (no subdirectories):
```python
local_path = destination / remote_path.name
```

**To organize by date** (if filenames contain a date):
```python
# e.g. filename: d-link_2026-07-10_14-53-05.mp4
import re
match = re.search(r"(\d{4}-\d{2}-\d{2})", remote_path.name)
date_str = match.group(1) if match else "unknown"
local_path = destination / date_str / remote_path.name
```

### Continuing vs Stopping on Error

Current behavior: on any transfer failure, log the error, clean up `.part`, increment `failed`, continue to the next file.

**To stop on first failure:**
```python
except Exception as exc:
    failed += 1
    log.error("  Status : transfer failed — %s", exc)
    if temporary_path.exists():
        temporary_path.unlink()
    break   # stop processing remaining files
```

---

## Connection — `connect_to_target(target, username, port)`

Uses `paramiko.RejectPolicy()` — the remote host key must already be in `~/.ssh/known_hosts`. Attempting to connect to an unknown host raises `paramiko.SSHException` immediately rather than silently accepting it.

This policy is intentional: the tool deletes source files after transfer. Silently accepting a changed or spoofed host key would be a serious security risk.

Password is collected via `getpass.getpass()` — never echoed to the terminal, never stored.

Connection timeout: 20 seconds (hardcoded in `ssh.connect()`).

### Adding SSH Key Authentication

Replace the `getpass.getpass()` + `password=` approach:

```python
import os

key_path = os.path.expanduser("~/.ssh/id_rsa")   # or make this a constant / CLI flag

ssh.connect(
    hostname=target,
    port=port,
    username=username,
    key_filename=key_path,
    timeout=20,
)
```

Remove the `password` parameter. If `key_filename` is set and the key exists, Paramiko uses it. Add `--identity-file` to `parse_args()` to make it configurable.

### Accepting Unknown Host Keys

**Only if you understand the risk:**

```python
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
```

This accepts any host key on first connect and adds it to `known_hosts`. Not recommended for a tool that deletes source files.

### Changing the Connection Timeout

In `connect_to_target()`:
```python
ssh.connect(
    hostname=target,
    port=port,
    username=username,
    password=password,
    timeout=30,     # seconds; default is 20
)
```

---

## Logging

Configured at module level:

```python
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s  %(levelname)-8s  %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger(__name__)
```

**To write logs to a file** (in addition to stdout):

```python
file_handler = logging.FileHandler("ssh_vid_mover.log")
file_handler.setFormatter(logging.Formatter(
    "%(asctime)s  %(levelname)-8s  %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
))
logging.getLogger().addHandler(file_handler)
```

**To suppress INFO output** (errors only):

```python
logging.basicConfig(level=logging.WARNING, ...)
```

---

## Running as a Scheduled Job

The script is designed to be run headlessly. The only interactive prompt is the SSH password.

**To run without a password prompt** (key auth or stored credential):

1. Use SSH key auth (see above), or
2. Wrap the password in a secrets manager and pipe it to `getpass.getpass()`:

```python
# Override getpass.getpass with a function that reads from a secure source:
import getpass as _getpass

def _read_password(prompt):
    return open("/run/secrets/ssh_password").read().strip()

_getpass.getpass = _read_password
```

**cron example** (after switching to key auth):

```
0 3 * * * /usr/bin/python3 /home/user/ssh-vid-mover/ssh_vid_mover.py >> /var/log/ssh_vid_mover.log 2>&1
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add a new camera name to filter | `VIDEO_NAMES` constant |
| Change file extension filter | `VIDEO_EXTENSIONS` constant |
| Change default remote path | `DEFAULT_REMOTE_FOLDER` constant |
| Change default local destination | `DEFAULT_DESTINATION` constant |
| Change SSH port default | `--port` default in `parse_args()` or add `DEFAULT_PORT` constant |
| Match all extensions (ignore name filter) | `create_transfer_list()` — remove `name_matches` condition |
| Filter by file size | `create_transfer_list()` — add `sftp.stat().st_size` check |
| Filter by modification time | `remote_walk()` — add `item.st_mtime` check |
| Skip hidden files | `remote_walk()` — add `item.filename.startswith(".")` guard |
| Non-recursive walk (top-level only) | Replace `remote_walk()` with direct `sftp.listdir_attr()` |
| Add SHA-256 verification | `download_and_remove_files()` — after size check, before `os.replace()` |
| Flatten output directory structure | `download_and_remove_files()` — change `local_path` to `destination / remote_path.name` |
| Organize output by date | `download_and_remove_files()` — parse date from filename, use as subdirectory |
| Stop on first failure instead of continuing | `download_and_remove_files()` — change `continue` to `break` in except block |
| Add SSH key authentication | `connect_to_target()` — add `key_filename=`, remove `password=` |
| Change connection timeout | `connect_to_target()` — `ssh.connect(timeout=N)` |
| Write logs to a file | Add `FileHandler` to root logger after `basicConfig()` |
| Run unattended (no password prompt) | Switch to key auth or override `getpass.getpass` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
