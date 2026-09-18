# sftp-ultra — Modification Guide
**Version:** 1.0.1  
**Package:** `src/sftp_ultra/`  
**Entry point:** `sftp_ultra.cli:main`

---

## Who This Document Is For

Engineers and developers who need to extend, adapt, or embed sftp-ultra beyond what the CLI flags expose. This is not a usage guide — the README covers that. This document tells you where the code lives, what each module owns, and exactly where to make each class of change.

---

## Architecture Overview

sftp-ultra is a pipeline of four stages executed in sequence per run:

```
CLI args → Config
              ↓
        Discovery        (remote_walk, discover_patterns, discover_directories)
              ↓
        Planner          (plan_transfers → list[PlanItem])
              ↓
        Engine           (run_plan → list[Result], writes Journal)
```

Each stage is a separate module with a single responsibility. They communicate through the data classes in `model.py`. Nothing crosses module boundaries except through those types.

---

## Module Map

| Module | File | Owns |
|---|---|---|
| `model.py` | `src/sftp_ultra/model.py` | All data classes and enums. Start here for any structural change. |
| `cli.py` | `src/sftp_ultra/cli.py` | Argument parser, `build_config()`, `main()`. Only entrypoint. |
| `ssh.py` | `src/sftp_ultra/ssh.py` | `SFTPConnectionFactory`, `Credentials`, `prompt_credentials()`. All connection logic. |
| `discovery.py` | `src/sftp_ultra/discovery.py` | `remote_walk()`, `discover_patterns()`, `discover_directories()`. |
| `paths.py` | `src/sftp_ultra/paths.py` | Remote path validation and root confinement. `local_for_remote()`, `resolve_under_root()`. |
| `planner.py` | `src/sftp_ultra/planner.py` | `plan_transfers()`. Overwrite policy resolution. |
| `engine.py` | `src/sftp_ultra/engine.py` | `run_plan()`, `transfer_one()`, `TokenBucket`, `copy_with_resume()`, `write_report()`. |
| `journal.py` | `src/sftp_ultra/journal.py` | `Journal`. SQLite write, schema definition. |

---

## Data Classes — `model.py`

All pipeline data flows through these types. Understand them before modifying anything else.

### `Config` (frozen dataclass)
Built once by `build_config()` in `cli.py` and passed through every stage. Frozen — no field is mutated after construction.

```python
@dataclass(frozen=True, slots=True)
class Config:
    target: str                          # Remote hostname or IP
    username: str
    port: int                            # Default: 22
    remote_root: PurePosixPath           # All remote paths confined here
    destination: Path                    # Local destination root
    workers: int                         # ThreadPoolExecutor max_workers
    retries: int                         # Retry count per file
    timeout_seconds: int                 # SSH connect/auth/banner timeout
    overwrite: OverwritePolicy           # skip | replace | rename | ask
    checksum: ChecksumMode               # none | sha256
    resume: bool                         # Resume .part files
    delete_source: bool                  # Delete remote after transfer
    trust_unknown_host: bool             # AutoAddPolicy vs RejectPolicy
    bandwidth_limit_kib: int | None      # Token bucket rate, KiB/s
    journal_path: Path
    manifest_path: Path | None
    report_path: Path | None
    dry_run: bool
    assume_yes: bool
```

**To add a new config field:** add it to `Config`, add the corresponding `--flag` in `cli.py` `parser()`, and wire it in `build_config()`. Add validation in `build_config()` if the value has constraints.

### `RemoteFile` (frozen dataclass)
Produced by `discovery.py`. Represents one file on the remote.

```python
@dataclass(frozen=True, slots=True)
class RemoteFile:
    path: PurePosixPath
    size: int
    mtime: int
```

### `PlanItem` (frozen dataclass)
Produced by `planner.py`. One entry in the transfer plan.

```python
@dataclass(frozen=True, slots=True)
class PlanItem:
    remote: RemoteFile
    local_path: Path
    action: str      # "transfer" | "skip" | "ask"
    reason: str      # Human-readable explanation, empty string if not applicable
```

### `Result` (mutable dataclass)
Produced by `engine.py`. One transfer outcome, written to the journal.

```python
@dataclass(slots=True)
class Result:
    remote_path: str
    local_path: str | None
    status: Status
    bytes_transferred: int
    resumed_from: int
    attempts: int
    checksum: str | None
    message: str
```

### Enums

| Enum | Values | Used in |
|---|---|---|
| `OverwritePolicy` | `ask`, `skip`, `replace`, `rename` | `Config.overwrite`, `planner.py` |
| `ChecksumMode` | `none`, `sha256` | `Config.checksum`, `engine.py` |
| `Status` | `planned`, `skipped`, `copied`, `moved`, `failed` | `Result.status`, `journal.py`, `engine.py` |

---

## Connection Layer — `ssh.py`

### `SFTPConnectionFactory`

Maintains one SSH/SFTP connection per worker thread using `threading.local`. Connection is reused across `transfer_one()` calls on the same thread; rebuilt automatically if the transport drops.

```python
class SFTPConnectionFactory:
    def __init__(self, config: Config, credentials: Credentials) -> None: ...
    def open(self) -> Iterator[paramiko.SFTPClient]: ...        # context manager
    def close_all_for_this_thread(self) -> None: ...
```

`open()` is a context manager. On exception it drops the cached connection so the next call reconnects rather than reusing a dead socket.

### Adding SSH Key Authentication

Current authentication: password-only via `getpass`. Key-based auth is not implemented.

To add it:

1. Add `identity_file: Path | None` to `Config` in `model.py`.
2. Add `--identity-file` flag to `pull` subparser in `cli.py`.
3. Wire it in `build_config()`.
4. In `ssh.py` `_new_ssh()`, extend the `ssh.connect()` call:

```python
ssh.connect(
    hostname=self.config.target,
    port=self.config.port,
    username=self.config.username,
    password=self.credentials.password or None,
    key_filename=str(self.config.identity_file) if self.config.identity_file else None,
    timeout=self.config.timeout_seconds,
    banner_timeout=self.config.timeout_seconds,
    auth_timeout=self.config.timeout_seconds,
)
```

5. Update `prompt_credentials()` to make the password prompt conditional on `identity_file` being absent.

### Changing the Host Key Policy

Controlled by `Config.trust_unknown_host`. The mapping is in `_new_ssh()`:

```python
if self.config.trust_unknown_host:
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
else:
    ssh.set_missing_host_key_policy(paramiko.RejectPolicy())
```

To add a known-hosts file path, replace `load_system_host_keys()` with `load_host_keys(path)`.

---

## Discovery — `discovery.py`

### `remote_walk(sftp, directory) -> Iterator[RemoteFile]`

Recursive walk of a remote directory. Uses `lstat()` explicitly on every entry (not `stat()`) to prevent symlink-to-directory from escaping root confinement. Symlinks are skipped entirely — files and directories only.

### `discover_patterns(sftp, *, root, patterns) -> list[RemoteFile]`

Walks from `root`, matches filenames (not full paths) against each pattern using `fnmatch.fnmatchcase` after case-folding both the pattern and the filename. Returns deduplicated results.

**To match on full paths instead of filenames:** replace `remote_file.path.name.casefold()` with `str(remote_file.path).casefold()` in `discover_patterns()`.

### `discover_directories(sftp, *, root, directories) -> list[RemoteFile]`

Resolves each directory path through `resolve_under_root()` (enforces root confinement), then calls `remote_walk()` on each. Returns deduplicated results across all directories.

### Adding a New Discovery Mode

Example: discover files modified within the last N hours.

1. Add `max_age_hours: int | None` to `Config`.
2. Add `--max-age-hours` flag to CLI.
3. Write a new function in `discovery.py`:

```python
def discover_recent(
    sftp: paramiko.SFTPClient,
    *,
    root: PurePosixPath,
    max_age_seconds: int,
) -> list[RemoteFile]:
    cutoff = int(time.time()) - max_age_seconds
    return [
        f for f in remote_walk(sftp, root)
        if f.mtime >= cutoff
    ]
```

4. Add the dispatch branch in `cli.py` `main()` alongside the existing `if args.pattern` / `else` block.

---

## Path Confinement — `paths.py`

All remote path handling passes through here. This is the security boundary between user input and remote filesystem access.

### Key Functions

```python
def normalize_remote(path: str | PurePosixPath) -> PurePosixPath
```
Strips whitespace, normalizes with `os.path.normpath`, rejects empty strings and relative paths.

```python
def resolve_under_root(selection: str | PurePosixPath, root: PurePosixPath) -> PurePosixPath
```
Resolves a user-supplied path against `root`. Relative paths are joined to `root` first. Rejects anything that escapes `root` after normalization. Raises `RemotePathError`.

```python
def local_for_remote(remote_path, remote_root, destination_root) -> Path
```
Computes the local destination path by stripping `remote_root` prefix from `remote_path` and joining the relative remainder to `destination_root`. Preserves directory structure.

```python
def unique_destination(path: Path) -> Path
```
Used by the `rename` overwrite policy. Appends `_1`, `_2`, etc. until a non-existent path is found.

**Do not bypass `resolve_under_root()` when constructing remote paths.** It is the only place path traversal is caught.

---

## Transfer Planning — `planner.py`

### `plan_transfers(files, config) -> list[PlanItem]`

Single function. Iterates `files`, maps each to a local path via `local_for_remote()`, and assigns an action based on whether the destination exists and what `config.overwrite` is set to:

| `OverwritePolicy` | Destination exists | Action |
|---|---|---|
| `SKIP` | yes | `skip` |
| `REPLACE` | yes | `transfer` |
| `RENAME` | yes | `transfer` (with `unique_destination()` local path) |
| `ASK` | yes | `ask` (resolved interactively in `cli.py` before `run_plan`) |
| Any | no | `transfer` |

### Adding a New Overwrite Policy

1. Add the value to `OverwritePolicy` in `model.py`:
   ```python
   class OverwritePolicy(str, Enum):
       ...
       NEWER = "newer"
   ```

2. Add the branch in `plan_transfers()`:
   ```python
   elif config.overwrite is OverwritePolicy.NEWER:
       if remote.mtime > int(requested.stat().st_mtime):
           plan.append(PlanItem(remote=remote, local_path=requested, action="transfer", reason="remote is newer"))
       else:
           plan.append(PlanItem(remote=remote, local_path=requested, action="skip", reason="local is current"))
   ```

3. Add `"newer"` to the `--overwrite` choices in `cli.py`.

---

## Transfer Engine — `engine.py`

This module owns all I/O: file download, checksum verification, atomic rename, source deletion, retry logic, and bandwidth throttling.

### `TokenBucket`

Rate limiter. Constructed in `run_plan()` from `config.bandwidth_limit_kib`. If `kib_per_second` is `None`, `consume()` is a no-op. All transfer reads call `bucket.consume(len(chunk))` to account for bytes moved.

**Important:** checksum reads after transfer intentionally pass `bucket=None` to avoid double-counting bandwidth. Do not change this without understanding the comment in `transfer_one()`.

### `copy_with_resume(sftp, remote_path, temporary_path, ...) -> int`

Downloads to `<filename>.part`. On resume, checks a `.meta` sidecar file (JSON with `remote_size` and `remote_mtime`) to verify the partial bytes still belong to the same version of the remote file. If size or mtime has changed since the partial was started, the partial is discarded and the download restarts. Returns `resumed_from` byte offset (0 if not resumed).

The `.meta` sidecar is written before the download starts and cleared after `os.replace()` succeeds.

### `transfer_one(item, *, config, factory, journal, bucket) -> Result`

Handles one `PlanItem`. Retry loop: up to `config.retries + 1` attempts with exponential backoff and jitter (`min(30.0, 2^(attempt-1) + random())`).

Per attempt:
1. Opens SFTP connection via `factory.open()`
2. Stats remote before download
3. Downloads via `copy_with_resume()`
4. Stats remote after download
5. Checks size and mtime consistency (before == after)
6. Checks local size against remote
7. Optionally runs SHA-256 comparison
8. Atomically renames `.part` → destination via `os.replace()`
9. Optionally deletes remote source
10. Records result to journal

On exhausted retries: records `Status.FAILED` to journal and returns.

### `run_plan(plan, *, config, factory) -> list[Result]`

Instantiates `Journal` and `TokenBucket`, then dispatches all non-dry-run items to a `ThreadPoolExecutor`. Results are collected via `as_completed()`.

**Dry-run path:** journals all items as `Status.PLANNED` or `Status.SKIPPED` without doing any I/O, then returns immediately.

### Adding a New Checksum Mode

1. Add the value to `ChecksumMode` in `model.py`.
2. Add the hash computation in `transfer_one()` in the checksum block:
   ```python
   if config.checksum is ChecksumMode.MD5:
       local_hash = md5_file(temporary_path, None)
       remote_hash = remote_md5(sftp, str(remote.path), None)
       if local_hash != remote_hash:
           raise RuntimeError("MD5 checksum mismatch.")
       checksum = local_hash
   ```
3. Add helper functions `md5_file()` and `remote_md5()` following the same pattern as `sha256_file()` and `remote_sha256()`.
4. Add `"md5"` to `--checksum` choices in `cli.py`.

### Adding a Post-Transfer Hook

To run arbitrary logic after each successful file transfer, add a callable parameter to `transfer_one()`:

```python
from typing import Callable

def transfer_one(
    item: PlanItem,
    *,
    config: Config,
    factory: SFTPConnectionFactory,
    journal: Journal,
    bucket: TokenBucket,
    on_success: Callable[[Result], None] | None = None,
) -> Result:
    ...
    # After os.replace() and before return:
    if on_success:
        on_success(result)
    ...
```

Pass it through `run_plan()` the same way. This is the correct place to trigger downstream processing, notifications, or database writes without coupling the engine to application-layer concerns.

---

## Journal — `journal.py`

### Schema

Single table: `transfer_journal`

```sql
CREATE TABLE IF NOT EXISTS transfer_journal (
    remote_path TEXT PRIMARY KEY,
    local_path TEXT,
    status TEXT NOT NULL,
    remote_size INTEGER,
    remote_mtime INTEGER,
    checksum TEXT,
    message TEXT,
    updated_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

`remote_path` is the primary key. Repeated transfers of the same remote file update the existing row (`ON CONFLICT DO UPDATE`).

### `Journal.record(result, *, remote_size, remote_mtime)`

Thread-safe via `threading.Lock()`. Opens a new SQLite connection per write — this is intentional. SQLite connections are not safe to share across threads; the lock serializes writes but each call gets its own connection object.

### Adding Journal Columns

1. Add the column to the `SCHEMA` string in `journal.py`. Use `ALTER TABLE` if you need a migration path for existing journals, or rely on `CREATE TABLE IF NOT EXISTS` only creating on first run.
2. Add the value to the `INSERT` statement and the `VALUES` tuple in `record()`.
3. Add the field to `Result` in `model.py` if the new data comes from the transfer.

### Querying the Journal from Python

```python
from sftp_ultra.journal import Journal
from pathlib import Path
import sqlite3

conn = sqlite3.connect(".sftp-ultra.sqlite3")
conn.row_factory = sqlite3.Row

rows = conn.execute(
    "SELECT remote_path, status, checksum FROM transfer_journal WHERE status = 'failed'"
).fetchall()

for row in rows:
    print(dict(row))
```

---

## CLI — `cli.py`

### Adding a New Subcommand

The CLI uses `argparse` subparsers. Current subcommand: `pull`.

To add a `push` subcommand:

```python
push = sub.add_parser("push", help="Copy files to a remote host.")
push.add_argument("--target", required=True)
# ... add flags
```

Then add dispatch in `main()`:

```python
if args.command == "pull":
    # existing pull logic
elif args.command == "push":
    # push logic
```

### `build_config(args) -> Config`

All validation of CLI args happens here before `Config` is constructed. Add new field validations here, not in the engine.

### Using sftp-ultra as a Library

The CLI is a thin wrapper. All logic is importable:

```python
from pathlib import Path, PurePosixPath
from sftp_ultra.model import Config, ChecksumMode, OverwritePolicy
from sftp_ultra.ssh import SFTPConnectionFactory, Credentials
from sftp_ultra.discovery import discover_patterns
from sftp_ultra.planner import plan_transfers
from sftp_ultra.engine import run_plan

config = Config(
    target="192.168.1.105",
    username="side",
    port=22,
    remote_root=PurePosixPath("/home/side"),
    destination=Path("/media/drive/downloads"),
    workers=4,
    retries=3,
    timeout_seconds=20,
    overwrite=OverwritePolicy.SKIP,
    checksum=ChecksumMode.SHA256,
    resume=True,
    delete_source=False,
    trust_unknown_host=False,
    bandwidth_limit_kib=None,
    journal_path=Path(".sftp-ultra.sqlite3"),
    manifest_path=None,
    report_path=None,
    dry_run=False,
    assume_yes=True,
)

credentials = Credentials(password="yourpassword")
factory = SFTPConnectionFactory(config, credentials)

with factory.open() as sftp:
    files = discover_patterns(sftp, root=config.remote_root, patterns=["*.mp4"])

plan = plan_transfers(files, config)
results = run_plan(plan, config=config, factory=factory)
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add SSH key authentication | `ssh.py` → `_new_ssh()`, `model.py` → `Config`, `cli.py` → parser |
| Add a new overwrite policy | `model.py` → `OverwritePolicy`, `planner.py` → `plan_transfers()`, `cli.py` → `--overwrite` choices |
| Add a new checksum algorithm | `model.py` → `ChecksumMode`, `engine.py` → `transfer_one()`, `cli.py` → `--checksum` choices |
| Add a new discovery mode (e.g. by age, size, extension) | `discovery.py` → new function, `cli.py` → new flag + dispatch |
| Run code after each successful transfer | `engine.py` → `transfer_one()` post-transfer hook |
| Add a new journal column | `journal.py` → `SCHEMA` + `record()`, `model.py` → `Result` if needed |
| Add a new CLI subcommand | `cli.py` → `parser()` + `main()` dispatch |
| Use sftp-ultra in another Python program | Import from `model`, `ssh`, `discovery`, `planner`, `engine` directly — bypass `cli.py` |
| Change retry backoff behavior | `engine.py` → `transfer_one()` retry loop, `delay` calculation |
| Change bandwidth accounting | `engine.py` → `TokenBucket`, `copy_with_resume()` `bucket.consume()` calls |
| Change where `.part` files land | `engine.py` → `transfer_one()` `temporary_path` assignment |

---

## Dependency

Single external dependency: `paramiko >= 3.4`

All other imports are Python stdlib: `hashlib`, `sqlite3`, `threading`, `concurrent.futures`, `pathlib`, `argparse`, `json`, `fnmatch`, `stat`, `os`, `time`, `random`, `getpass`.

---

## Version Bump Procedure

Per MainbyteLabs QA patch release rule:

1. `pyproject.toml` → `version = "X.Y.Z"`
2. `src/sftp_ultra/__init__.py` → `__version__ = "X.Y.Z"`
3. Bug fixes only → patch bump (e.g. `1.0.1` → `1.0.2`)
4. Push to main, create release tag on GitHub.

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
