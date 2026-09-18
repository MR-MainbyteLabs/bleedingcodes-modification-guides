# security_scanner — Modification Guide
**File:** `security_scanner.py` (single-file, stdlib only)  
**Version:** 1.0.1  
**Entry point:** `main()`  
**Dependencies:** None — stdlib only (`argparse`, `json`, `re`, `pathlib`, `dataclasses`, `contextlib`)

---

## Who This Document Is For

Engineers who want to add detection rules, change file targeting, extend the output format, or embed the scanner in a larger pipeline. This is not a usage guide. This document tells you where the code lives and exactly where to make each class of change.

---

## Architecture Overview

```
main()
  └── run_scan(args)
        ├── find_candidate_files()     # file discovery — extension + exclusion filters
        └── inspect_file() × N         # per-file dispatch
              ├── load_json()  → scan_json_security()   # JSON path
              └── load_text()  → scan_text_security()   # text/config path
                    ↓
              print_security_findings()
              print_summary()
```

Two scan modes run on different parsers:
- **JSON files** — `scan_json_security()` walks the parsed object tree recursively
- **Text/config files** — `scan_text_security()` scans line by line

Both modes call the same rule engine: `matching_secret_rule()` (pattern match) and `key_severity()` (keyword match).

---

## Data Classes

### `SecretRule` (frozen)
```python
@dataclass(frozen=True)
class SecretRule:
    rule_id: str         # e.g. "AWS_ACCESS_KEY"
    name: str            # human-readable name
    severity: str        # "HIGH" | "MEDIUM" | "LOW"
    pattern: Pattern     # compiled re.Pattern
    description: str     # why this is a problem
    remediation: str     # what to do about it
```

### `Finding` (frozen)
```python
@dataclass(frozen=True)
class Finding:
    severity: str
    rule_id: str
    rule_name: str
    location: str        # e.g. "root.database.password" or "line 14"
    evidence: str        # redacted value or key name
    description: str
    remediation: str

    def fingerprint(self) -> str:
        return f"{self.severity}|{self.rule_id}|{self.location}|{self.evidence}"
```

Fingerprinting deduplicates findings. Two findings with the same `(severity, rule_id, location, evidence)` are the same finding and will not be added twice. The deduplication `seen: set[str]` is passed through all recursive scan calls.

### `ScanStats`
Tracks: `files_found`, `files_loaded`, `malformed_files`, `unreadable_files`, `files_with_findings`, `total_findings`. Printed at the end when security scanning is enabled.

---

## Rule Definitions

### `SECRET_RULES` — Value Pattern Rules

List of `SecretRule` objects. Each has a compiled regex. Checked against string values (in JSON) and full lines (in text). Matched in order — first match wins.

Current rules and their patterns:

| `rule_id` | Severity | Pattern |
|---|---|---|
| `PRIVATE_KEY_BLOCK` | HIGH | `-----BEGIN [A-Z ]*PRIVATE KEY-----` |
| `AWS_ACCESS_KEY` | HIGH | `AKIA[0-9A-Z]{16}` with word-boundary lookbehind |
| `OPENAI_SK_TOKEN` | HIGH | `sk-[A-Za-z0-9_-]{20,}` with lookbehind/ahead |
| `GITHUB_CLASSIC_TOKEN` | HIGH | `ghp_[A-Za-z0-9_]{20,}` |
| `GITHUB_FINE_GRAINED_TOKEN` | HIGH | `github_pat_[A-Za-z0-9_]{20,}` |
| `SLACK_TOKEN` | HIGH | `xox[baprs]-[A-Za-z0-9-]{20,}` |
| `BEARER_TOKEN` | MEDIUM | `(?i)bearer\s+[A-Za-z0-9._~+/=-]{20,}` |

### Adding a New Pattern Rule

Add a `SecretRule` to `SECRET_RULES`:

```python
SecretRule(
    rule_id="STRIPE_SECRET_KEY",
    name="Stripe secret key",
    severity="HIGH",
    pattern=re.compile(r"(?<![A-Za-z0-9_/.-])sk_live_[A-Za-z0-9]{24,}(?![A-Za-z0-9_/.-])"),
    description="A value looks like a Stripe live secret key.",
    remediation="Revoke the key in Stripe dashboard and remove it from disk and history.",
),
```

Rules are checked in list order. Add high-confidence, high-specificity rules before generic ones.

### `SUSPICIOUS_KEY_RULES` — Key Name Rules

Dict keyed by severity. Used in JSON (`scan_json_security`) and text (`scan_text_security`) when `--values-only` is not set. The `pattern` field is `re.compile(r".*")` — key name matching is done by `key_severity()`, not by the pattern.

### `key_severity(key: str) -> str | None`

Checks the normalized key name against three keyword lists:

```python
HIGH_CONFIDENCE_KEYWORDS = ["password","passwd","secret","api_key","apikey","private_key","access_key"]
MEDIUM_CONFIDENCE_KEYWORDS = ["token","credential","credentials","authorization","bearer"]
LOW_CONFIDENCE_KEYWORDS = ["auth","pwd"]
```

`normalize_key()` lowercases and replaces `-` and spaces with `_` before matching.

**To add a new keyword:**

```python
HIGH_CONFIDENCE_KEYWORDS = [
    ...,
    "encryption_key",   # add here
]
```

**To add a new severity tier** (e.g. `CRITICAL`):

1. Add `"CRITICAL": 4` to `SEVERITY_ORDER`.
2. Add a new keyword list and add a check in `key_severity()`.
3. Add a `SecretRule` entry to `SUSPICIOUS_KEY_RULES` for `"CRITICAL"`.
4. Add `"CRITICAL"` to the `--min-severity` choices in `build_parser()`.

---

## `matching_secret_rule(value: str) -> SecretRule | None`

Iterates `SECRET_RULES` in order. Returns the first rule whose pattern matches `value`. Used by both `scan_json_security` and `scan_text_security`.

---

## `scan_json_security(value, path, ...)`

Recursive walk of a parsed JSON object. At each node:
- **Dict:** for each key, check `key_severity()` (unless `values_only`), then recurse into the value
- **List:** recurse into each item
- **String:** check `matching_secret_rule()`

Location strings follow JSON path notation: `root.database.credentials.password` or `root.servers[0].token`.

`redact(value)` truncates the evidence to `first4...last4` for values over 8 characters.

---

## `scan_text_security(text, min_severity, values_only)`

Line-by-line scan. Per line:

1. Skips blank lines, comment lines (`#`, `//`, `;`), and XML MIME pattern lines (to suppress false positives from `.desktop` files).
2. If not `values_only`: matches `^([A-Za-z0-9_.-]{2,80})\s*[:=]` to extract a key name, then calls `key_severity()`.
3. Always: calls `matching_secret_rule(stripped_line)` on the full line.

**To add a new comment style to skip:**

In the `if not stripped or stripped.startswith(...)` check:
```python
if not stripped or stripped.startswith(("#", "//", ";", "REM")):  # add REM
    continue
```

**To add a new false-positive suppression:**

Add another condition after the XML MIME check:
```python
if "example_token_pattern" in stripped:
    continue
```

---

## File Discovery — `find_candidate_files()`

### `DEFAULT_EXTENSIONS`

```python
DEFAULT_EXTENSIONS = [
    ".json", ".env", ".ini", ".conf", ".config", ".cfg",
    ".yaml", ".yml", ".toml", ".xml", ".properties",
    ".txt", ".log", ".service", ".desktop",
    ".sh", ".bash", ".zsh", ".profile",
]
```

**To add a new extension:**
```python
DEFAULT_EXTENSIONS = [..., ".kdbx", ".pem"]
```

Note: `extension_allowed()` also matches on full filename (e.g. `Makefile`, `.gitconfig`) since some config files have no extension. Add bare filenames to `DEFAULT_EXTENSIONS` the same way.

### `DEFAULT_EXCLUDED_DIR_NAMES`

```python
DEFAULT_EXCLUDED_DIR_NAMES = {
    ".git", ".svn", ".hg", "node_modules", "__pycache__",
    ".venv", "venv", "env", ".mypy_cache", ".pytest_cache",
}
```

**To add a new excluded directory:**
```python
DEFAULT_EXCLUDED_DIR_NAMES = {..., ".tox", "dist", "build"}
```

### `load_text()` — File Size Limit

Default: `max_bytes=2_000_000` (2 MB). Files larger than this are skipped and counted as unreadable.

```python
def load_text(path: Path, show_errors: bool = False, max_bytes: int = 2_000_000):
```

To change the limit, modify the default or pass a different value from `inspect_file()`.

---

## `inspect_file()` — Per-File Dispatch

Routes by extension:
- `.json` → `load_json()` → `scan_json_security()`
- everything else → `load_text()` → `scan_text_security()`

Returns `(loaded: bool, finding_count: int, had_findings: bool, error: str | None)`.

**To add a new file type with custom parsing** (e.g. TOML):

```python
elif path.suffix.lower() == ".toml":
    try:
        import tomllib
    except ImportError:
        import tomli as tomllib  # pip install tomli for Python < 3.11
    with path.open("rb") as f:
        data = tomllib.load(f)
    findings = scan_json_security(data, min_severity=min_severity, values_only=values_only)
    # ... same output logic as the JSON branch
```

---

## Output Format

### `print_security_findings(findings, quiet_if_clean)`

Current format per finding:
```
 !!! [HIGH] AWS_ACCESS_KEY: AWS access key style value
     Location: root.credentials.access_key
     Evidence: AKIA...WXYZ
     Why: A value looks like an AWS access key ID.
     Fix: Check AWS IAM. If valid, rotate the key and remove it from local files and history.
```

**To change to JSON output:**

Replace `print_security_findings()` body:
```python
import json
def print_security_findings(findings, quiet_if_clean=False):
    if not findings:
        if not quiet_if_clean:
            print(json.dumps({"findings": []}))
        return
    print(json.dumps({"findings": [
        {
            "severity": f.severity,
            "rule_id": f.rule_id,
            "location": f.location,
            "evidence": f.evidence,
            "description": f.description,
            "remediation": f.remediation,
        }
        for f in findings
    ]}))
```

### Report File Output

`--output path` uses `contextlib.redirect_stdout` and redirects `sys.stderr` to the same file. Both stdout and stderr go to the report. To separate them, pass `output_file` explicitly to `run_scan()` and use separate file handles for stdout and stderr.

---

## Using as a Library

All scan functions are importable:

```python
from pathlib import Path
from security_scanner import (
    scan_json_security, scan_text_security,
    print_security_findings, load_json, load_text,
)

# Scan a JSON file
data, error = load_json(Path("config.json"))
if data is not None:
    findings = scan_json_security(data, min_severity="HIGH", values_only=True)
    print_security_findings(findings)

# Scan text
text, error = load_text(Path(".env"))
if text is not None:
    findings = scan_text_security(text, min_severity="MEDIUM")
    print_security_findings(findings)
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add a new secret pattern rule | `SECRET_RULES` list — add a `SecretRule` with compiled regex |
| Add a new key name keyword | `HIGH/MEDIUM/LOW_CONFIDENCE_KEYWORDS` lists |
| Add a new severity tier | `SEVERITY_ORDER` + new keyword list + `key_severity()` + `SUSPICIOUS_KEY_RULES` + CLI choices |
| Add a new file extension | `DEFAULT_EXTENSIONS` list |
| Exclude a new directory | `DEFAULT_EXCLUDED_DIR_NAMES` set |
| Change file size limit | `load_text()` `max_bytes` parameter |
| Add TOML / XML / custom parsing | `inspect_file()` — add extension branch |
| Skip a new comment syntax | `scan_text_security()` — `startswith()` guard |
| Add a false-positive suppression | `scan_text_security()` — add `if "pattern" in stripped: continue` |
| Change output format to JSON | Replace `print_security_findings()` body |
| Use scanner in another script | Import `scan_json_security`, `scan_text_security`, `load_json`, `load_text` |
| Add a new CLI flag | `build_parser()` + `run_scan()` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
