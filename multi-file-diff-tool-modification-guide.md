# multi_file_diff_tool — Modification Guide
**File:** `multi_file_diff_tool.py` (single-file, stdlib only)  
**Entry point:** `main()`  
**Dependencies:** None — stdlib only (`difflib`, `tkinter`, `argparse`, `pathlib`, `itertools`, `dataclasses`, `webbrowser`, `datetime`)

---

## Who This Document Is For

Engineers and developers who need to extend, adapt, or embed the diff and merge logic beyond what the GUI and CLI expose. This is not a usage guide. This document tells you where the code lives, what each class and function owns, and exactly where to make each class of change.

---

## Design Principle — Core / GUI Split

The docstring at the top of the file is the key architectural fact: **the diff and merge logic has zero tkinter dependency**. The file is divided into two sections:

```
Core (no GUI)          — load_file, normalize_lines, compute_all_pairs,
                         three_way_merge, build_html_report, run_cli
GUI (tkinter)          — MultiFileDiffApp, MergeRoleDialog,
                         MergeResultsWindow, DiffResultsWindow
```

`tkinter` is imported inside a `try/except ImportError` block. If it is absent (headless server, minimal Linux), the CLI still works. The GUI classes are never reached from the CLI path.

`_TKINTER_AVAILABLE` is the guard checked in `run_gui()`. If `False`, it prints CLI instructions and exits rather than crashing.

---

## Entry Point — `main()`

```python
def main() -> None:
    if len(sys.argv) > 1:
        sys.exit(run_cli(sys.argv[1:]))
    run_gui()
```

Any argument at all → CLI. No arguments → GUI. The dispatch is purely on `sys.argv` length.

---

## Data Classes

### `DiffOptions`

Comparison settings passed through every layer. Shared between CLI, GUI, and the core functions.

```python
@dataclass
class DiffOptions:
    ignore_whitespace: bool = False   # Collapse whitespace before comparing
    ignore_case: bool = False         # Lowercase before comparing
    only_differences: bool = False    # Display filter — skip identical pairs in output
```

`only_differences` is a display filter, not a comparison setting. The diff is still computed for identical pairs — they are just omitted from the output.

### `LoadedFile`

```python
@dataclass
class LoadedFile:
    path: Path
    lines: list[str]    # splitlines(keepends=True) — newlines are preserved
```

Lines include their trailing `\n`. This is important: `difflib.unified_diff` and `three_way_merge` both expect lines with their terminators present.

### `PairResult`

```python
@dataclass
class PairResult:
    name_a: str
    name_b: str
    lines_a: list[str]      # normalized lines (what was actually diffed)
    lines_b: list[str]
    diff_lines: list[str]   # unified_diff output
    added_count: int
    removed_count: int

    @property
    def is_identical(self) -> bool:
        return not self.diff_lines
```

`lines_a` and `lines_b` are the **normalized** lines — after `ignore_whitespace` and `ignore_case` are applied. The diff is computed on the normalized text and displayed from the normalized text. There is no "match on normalized, display original" behavior.

### `MergeResult`

```python
@dataclass
class MergeResult:
    base_name: str
    ours_name: str
    theirs_name: str
    merged_lines: list[str]
    conflict_count: int

    @property
    def has_conflicts(self) -> bool:
        return self.conflict_count > 0
```

---

## Core Functions

### `load_file(path: Path) -> LoadedFile`

```python
text = path.read_text(encoding="utf-8", errors="replace")
lines = text.splitlines(keepends=True)
```

Reads UTF-8. Non-UTF-8 bytes are replaced with the Unicode replacement character rather than crashing. To change encoding:

```python
text = path.read_text(encoding="latin-1", errors="replace")
```

To support encoding detection (e.g. via `chardet`):
```python
raw = path.read_bytes()
encoding = chardet.detect(raw)["encoding"] or "utf-8"
text = raw.decode(encoding, errors="replace")
```

### `normalize_lines(lines, options) -> list[str]`

Applied before every diff. Order: whitespace collapse first, then case fold.

```python
if options.ignore_whitespace:
    normalized = [" ".join(line.split()) for line in normalized]
if options.ignore_case:
    normalized = [line.lower() for line in normalized]
```

**Important:** `line.split()` strips leading/trailing whitespace and collapses all internal whitespace runs to single spaces. Newlines are consumed here — normalized lines lose their `\n` when whitespace is collapsed. This is intentional: the tool compares and displays the normalized text, not the original.

**To add a new normalization option:**

1. Add a field to `DiffOptions`.
2. Add the normalization block in `normalize_lines()`.
3. Add the CLI flag in `run_cli()` parser.
4. Add the checkbox to `MultiFileDiffApp._build_widgets()`.

### `compute_all_pairs(loaded_files, options) -> list[PairResult]`

Uses `itertools.combinations(loaded_files, 2)` — every unique unordered pair. For N files: N×(N-1)/2 pairs.

Display names: if two loaded files share the same filename, both fall back to their full path to keep the diff header unambiguous. This logic lives in the `_display_name` closure inside this function.

The diff uses `difflib.unified_diff` with `lineterm=""` — hunk/header lines do not get a trailing `\n` appended, but content lines still have their original terminators from `splitlines(keepends=True)`.

**To change context lines** (default: 3):

`difflib.unified_diff` is called without a `n=` argument, which defaults to 3. To change:
```python
diff_lines = list(
    difflib.unified_diff(
        lines_a, lines_b,
        fromfile=name_a, tofile=name_b,
        lineterm="",
        n=5,        # 5 lines of context
    )
)
```

**To compare in a specific order** (not all-vs-all):

Replace `itertools.combinations` with your own pair list:
```python
pairs = [(loaded_files[0], loaded_files[1])]   # Only first vs second
for file_a, file_b in pairs:
    ...
```

### `build_html_report(loaded_files, pair_results, options) -> str`

Uses `difflib.HtmlDiff` (stdlib) for word-level highlighted side-by-side tables.

```python
differ = difflib.HtmlDiff(wrapcolumn=100)
```

**To change wrap column** (line length before wrapping in HTML table):
```python
differ = difflib.HtmlDiff(wrapcolumn=80)
```

**To change context lines in HTML report** (default: 3):
```python
differ.make_table(
    pair.lines_a, pair.lines_b,
    fromdesc=pair.name_a, todesc=pair.name_b,
    context=True, numlines=5,    # 5 context lines
)
```

**To show full file content** (no context clipping):
```python
context=False
```

**To add custom CSS to the HTML report**, insert it in the `<style>` block built inside `build_html_report()`:
```python
f"<style>{_inline_styles}\n"
"body{...}\n"
"my-custom-rule { color: red; }\n"   # add here
"</style>"
```

---

## Three-Way Merge — `three_way_merge`

```python
def three_way_merge(
    base_lines: list[str],
    ours_lines: list[str],
    theirs_lines: list[str],
    ours_label: str = "OURS",
    theirs_label: str = "THEIRS",
) -> tuple[list[str], int]:
```

Returns `(merged_lines, conflict_count)`. Exit code convention follows `diff3`/`git merge-file`: 0 = clean, 1 = conflicts.

### Algorithm

1. Two `SequenceMatcher` runs: base-vs-ours and base-vs-theirs.
2. `_find_clusters()` sweeps all changed base-index ranges into maximal clusters using a merge-intervals pass. Adjacent but non-overlapping ranges stay separate (strict `<` comparison, not `<=`).
3. Per cluster: one side touched → take that side. Both same change → take it. Both different → conflict block.

Conflict markers:
```
<<<<<<< OURS
... ours content ...
=======
... theirs content ...
>>>>>>> THEIRS
```

### Changing Conflict Marker Style

Pass custom labels:
```python
merged_lines, count = three_way_merge(
    base.lines, ours.lines, theirs.lines,
    ours_label="feature-branch",
    theirs_label="main",
)
```

To change the marker format itself (e.g. to diff3 style), edit the conflict block inside `three_way_merge()`:
```python
conflict_count += 1
merged_lines.append(f"<<<<<<< {ours_label}\n")
merged_lines.extend(ours_piece)
merged_lines.append("=======\n")
merged_lines.extend(theirs_piece)
merged_lines.append(f">>>>>>> {theirs_label}\n")
```

### Using the Merge Engine Standalone

```python
from pathlib import Path
from multi_file_diff_tool import load_file, three_way_merge

base = load_file(Path("base.txt"))
ours = load_file(Path("ours.txt"))
theirs = load_file(Path("theirs.txt"))

merged_lines, conflict_count = three_way_merge(
    base.lines, ours.lines, theirs.lines,
    ours_label="ours.txt", theirs_label="theirs.txt",
)

Path("merged.txt").write_text("".join(merged_lines), encoding="utf-8")
print(f"{conflict_count} conflict(s)")
```

---

## CLI — `run_cli(argv)`

### Flags

| Flag | Type | Effect |
|---|---|---|
| `files` | positional, 1+ | Files to compare (or 3 for `--merge`) |
| `--ignore-whitespace` | bool | Passed to `DiffOptions` |
| `--ignore-case` | bool | Passed to `DiffOptions` |
| `--only-differences` | bool | Omits identical pairs from stdout |
| `--html REPORT.html` | path | Writes HTML report |
| `--merge` | bool | 3-way merge mode: BASE OURS THEIRS |
| `--output MERGED_FILE` | path | Merge output path; default is stdout |

Exit codes: `0` = success or clean merge, `1` = I/O error or conflicts found, per `diff3`/`git` convention.

### Adding a New CLI Flag

1. Add to the `argparse.ArgumentParser` in `run_cli()`.
2. Pass through to `DiffOptions` or the relevant function.
3. Add the corresponding `DiffOptions` field if it affects comparison behavior.

---

## GUI Classes

### `MultiFileDiffApp`

Main window. Owns:
- `self.loaded_files: list[LoadedFile]` — the file list shown in the listbox
- Three `tk.BooleanVar` instances for the checkboxes (bound to `DiffOptions` fields at diff time)
- Buttons: Add File(s), Remove Selected, All-vs-All Diff, 3-Way Merge...

**File loading** — `add_files()`: uses `filedialog.askopenfilenames()`, deduplicates by resolved path. Shows an info dialog if a file is already loaded rather than silently ignoring it.

**File removal** — `remove_selected_files()`: removes from highest index downward to avoid index-shifting during deletion.

**Adding a new checkbox option:**

1. Add a `tk.BooleanVar` in `__init__`.
2. Add `DiffOptions` field.
3. Add `ttk.Checkbutton` in `_build_widgets()` `options_frame`.
4. Read the var in `show_all_vs_all_diff()` when building `DiffOptions`.

**Adding a new button:**

Add to `_build_widgets()` `button_frame`:
```python
ttk.Button(button_frame, text="My Action", command=self.my_action).pack(side="left", padx=5)
```

### `MergeRoleDialog`

Toplevel dialog for assigning base/ours/theirs roles to loaded files. Uses `ttk.Combobox` with `state="readonly"`. Validates that all three selections are distinct before calling `on_confirm`.

Display strings include a `1.`, `2.`, `3.` prefix to disambiguate files with the same name.

**To support merging without the loaded-files list** (e.g. via file picker instead):

Replace `MergeRoleDialog` with a new dialog that calls `filedialog.askopenfilename()` three times, loads each file, then calls `three_way_merge` directly.

### `MergeResultsWindow`

Shows the merged output in a `tk.Text` widget with three tag colors:

```python
TAG_COLORS = {
    "header": "#0033cc",   # conflict marker lines (<<<, ===, >>>)
    "ours":   "#0a7d00",   # our side (green)
    "theirs": "#b30000",   # their side (red)
}
```

**Changing conflict marker colors:**

Edit `TAG_COLORS` in `MergeResultsWindow`. Same dict structure applies to `DiffResultsWindow`.

**Making the merge result editable:**

Remove `text_widget.config(state="disabled")` from `build()`. You will then need to read `text_widget.get("1.0", "end")` in `save_merged_file()` instead of `"".join(self.result.merged_lines)`.

### `DiffResultsWindow`

Shows all pair sections in a scrollable canvas. Each differing pair gets:
- A header label with filename pair and line counts
- A `tk.Text` widget (height = min 15, `len(diff_lines) + 1`)
- A Show/Hide toggle button

**Changing diff colors:**

```python
TAG_COLORS = {
    "header":  "#0033cc",   # file/hunk headers (---, +++, @@)
    "added":   "#0a7d00",   # lines added (green)
    "removed": "#b30000",   # lines removed (red)
}
```

**Changing the default text widget height:**

In `_add_pair_section()`:
```python
text_widget = tk.Text(
    section, height=min(15, len(pair.diff_lines) + 1), wrap="none"
)
# Change max height:
text_widget = tk.Text(
    section, height=min(25, len(pair.diff_lines) + 1), wrap="none"
)
```

**Changing initial expand/collapse state:**

```python
is_expanded = tk.BooleanVar(value=True)   # Change to False to collapse by default
```

---

## Using the Core as a Library

All core functions are importable. No GUI or argparse is triggered on import.

```python
from pathlib import Path
from multi_file_diff_tool import (
    DiffOptions, load_file, compute_all_pairs, build_html_report
)

options = DiffOptions(ignore_whitespace=True, only_differences=True)

files = [load_file(Path(p)) for p in ["a.py", "b.py", "c.py"]]
pairs = compute_all_pairs(files, options)

for pair in pairs:
    if not pair.is_identical:
        print(f"{pair.name_a} vs {pair.name_b}: +{pair.added_count}/-{pair.removed_count}")

html = build_html_report(files, pairs, options)
Path("report.html").write_text(html, encoding="utf-8")
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Change diff context lines (unified diff) | `compute_all_pairs()` → `unified_diff(n=N)` |
| Change HTML report wrap column | `build_html_report()` → `HtmlDiff(wrapcolumn=N)` |
| Change HTML report context lines | `build_html_report()` → `make_table(numlines=N)` |
| Show full files in HTML (no context clipping) | `make_table(context=False)` |
| Change conflict marker labels | `three_way_merge()` `ours_label` / `theirs_label` params |
| Change conflict marker format | `three_way_merge()` conflict block |
| Add a new normalization option | `DiffOptions` field + `normalize_lines()` block + CLI flag + checkbox |
| Change diff/merge colors in GUI | `TAG_COLORS` dict in `DiffResultsWindow` or `MergeResultsWindow` |
| Make merge result editable in GUI | Remove `text_widget.config(state="disabled")` in `MergeResultsWindow.build()` |
| Change default pair section height | `DiffResultsWindow._add_pair_section()` `tk.Text(height=...)` |
| Collapse sections by default | `DiffResultsWindow._add_pair_section()` `is_expanded = tk.BooleanVar(value=False)` |
| Compare specific pairs instead of all-vs-all | Replace `itertools.combinations` in `compute_all_pairs()` |
| Change file encoding | `load_file()` → `path.read_text(encoding=...)` |
| Add encoding auto-detection | `load_file()` — read bytes first, detect encoding, then decode |
| Use diff engine without GUI | Import `load_file`, `compute_all_pairs`, `three_way_merge` directly |
| Add a new CLI flag | `run_cli()` parser + `DiffOptions` field if comparison-related |
| Add a new button to the main window | `MultiFileDiffApp._build_widgets()` `button_frame` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
