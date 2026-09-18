# rd-idea-engine — Modification Guide
**Version:** v2  
**Three options:** Option A (OpenAI browser), Option B (Anthropic CLI), Option C (Ollama local browser)  
**Common logic:** Three-phase pipeline — Research → Generate → Stress Test

---

## Who This Document Is For

Engineers who want to change the focus areas, modify the prompts, add or swap models, change output format, or adapt the pipeline for a different domain. This is not a usage guide. This document covers all three options with callouts where they differ.

---

## Option Comparison

| | Option A | Option B | Option C |
|---|---|---|---|
| **Files** | `app.html`, `proxy.py` | `cli.py` | `app.html`, `proxy.py` |
| **Model** | OpenAI (GPT-4o) | Anthropic (claude-sonnet-4-6) | Ollama (llama3 or any local model) |
| **API Key** | `OPENAI_API_KEY` env var | `ANTHROPIC_API_KEY` env var | None — fully local |
| **Research Mode** | Browser sends search prompt to OpenAI | Anthropic web_search tool | Ollama (no external search) |
| **UI** | Browser app via Flask proxy | Terminal interactive | Browser app via Flask proxy |
| **Output** | Browser display | Markdown file (`rd_output_*.md`) | Browser display |

---

## Three-Phase Pipeline

All three options implement the same logical pipeline:

```
Phase 1 — Research
  Research Mode: query model with web search tools
  Paste Mode:    user provides own research text

Phase 2 — Idea Generation
  Input: target name + research text + focus areas + idea count
  Output: list of idea dicts (JSON array)

Phase 3 — Stress Test
  Input: selected ideas (user picks from generated list)
  Output: evaluation dict with verdicts, scores, execution plans
```

---

## Option B (CLI) — `option-b-cli/cli.py`

The CLI is the most readable option for understanding the pipeline. All logic is in one file.

### Constants

```python
FOCUS_OPTIONS = [
    ("performance",   "Performance"),
    ("efficiency",    "Efficiency"),
    ("features",      "Features"),
    ("cost",          "Cost Reduction"),
    ("reliability",   "Reliability"),
    ("software",      "Software / Firmware"),
    ("integration",   "Integration"),
    ("manufacturing", "Manufacturing"),
    ("safety",        "Safety"),
    ("ux",            "UX / Interfaces"),
]
```

**To add a focus area:**
```python
("sustainability", "Sustainability"),
```

The tag (`"sustainability"`) is passed into the idea generation system prompt. The label (`"Sustainability"`) is shown in the terminal menu.

### `call_claude(client, system, user) -> str`

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4000,
    system=system,
    messages=[{"role": "user", "content": user}],
)
```

**To change the model:**
```python
model="claude-opus-5",
```

**To change max tokens:**
```python
max_tokens=8000,
```

### `research_target(client, target) -> str`

Uses the Anthropic `web_search_20250305` tool:

```python
tools=[{"type": "web_search_20250305", "name": "web_search"}]
```

Collects only `text` content blocks from the response (ignores tool use blocks):
```python
return "".join(b.text for b in response.content if getattr(b, "type", "") == "text")
```

**To change the research prompt**, edit the `content` string in the `messages` list inside `research_target()`.

**To disable web search** and use pure model knowledge:
```python
def research_target(client, target):
    return call_claude(client,
        "You are an expert product researcher.",
        f'Summarize publicly known information about "{target}": specifications, known limitations, competitor products, market positioning.'
    )
```

### `generate_ideas(client, target, research, focus, count) -> list[dict]`

System prompt (abbreviated):
```python
system = f"""You are a ruthless R&D strategist. Generate exactly {count} distinct, actionable R&D improvement ideas.
Focus areas: {', '.join(focus)}.
CRITICAL: Return ONLY a valid JSON array. No markdown, no preamble. Start with [ end with ]."""
```

Each idea JSON schema:
```json
{
  "title": "",
  "domain": "",
  "description": "",
  "difficulty": 1,
  "dependencies": "",
  "estimated_time": "",
  "visible_result": "",
  "smallest_version": "",
  "why_interesting": "",
  "scope_risk": "low|medium|high",
  "connects_to": ""
}
```

**To add a new field to every idea**, add it to the schema string in the system prompt and it will be populated by the model.

JSON parsing strips markdown fences before parsing:
```python
cleaned = raw.replace("```json", "").replace("```", "").strip()
return json.loads(cleaned[cleaned.index("["):cleaned.rindex("]") + 1])
```

### `stress_test(client, target, ideas) -> dict`

System prompt instructs the model to return a single JSON object:

```json
{
  "summary": "",
  "evaluations": [{
    "title": "",
    "verdict": "BUILD NOW|DELAY|DROP",
    "reality_check": "",
    "motivation_collapse": "",
    "hidden_complexity": "",
    "value_analysis": "",
    "scope_classification": "TOO BIG|TOO SMALL|MISCOPED|VALID",
    "simplified_versions": {
      "two_to_five_hours": "",
      "one_day": "",
      "mvp": ""
    },
    "execution_risks": [],
    "scores": {
      "execution_likelihood": 1, "learning_value": 1,
      "reusability": 1, "clarity": 1,
      "visible_progress": 1, "scope_control": 1,
      "setup_friction": 1, "debugging_friction": 1,
      "maintenance_friction": 1
    },
    "final_score": 0,
    "execution_plan": {
      "hour_1": "", "first_file": "", "done_means": "",
      "do_not_add": "", "stall_risk": ""
    }
  }],
  "ranking": [{"rank": 1, "title": "", "verdict": "", "main_risk": ""}]
}
```

**To add a new evaluation field**, add it to the schema string in the `stress_test()` system prompt and handle it in `save_output()` and `display_results()`.

**To change verdict labels** (e.g. add `"SPIKE"`):

Edit the system prompt schema to allow the new label, then add it to `display_results()`:
```python
verdict_sym = {"BUILD NOW": "✓", "DELAY": "◆", "DROP": "✗", "SPIKE": "⚡"}.get(ev.get("verdict", ""), "◆")
```

### `save_output(target, ideas, selected, results) -> str`

Writes a Markdown file named `rd_output_{safe_target}_{timestamp}.md` in the current directory. All fields from both `ideas` and `results` are formatted into sections.

**To change the output directory:**
```python
output_dir = Path.home() / "rd-outputs"
output_dir.mkdir(exist_ok=True)
filename = str(output_dir / f"rd_output_{safe_target}_{ts}.md")
```

**To add a new section to the report**, append to `lines` in `save_output()` after the existing sections.

### Idea Count Options

```python
def pick_idea_count() -> int:
    raw = input("  Number of ideas to generate [8/12/15, default 12]: ").strip()
    if raw in ("8", "12", "15"):
        return int(raw)
    return 12
```

**To allow any count:**
```python
def pick_idea_count() -> int:
    raw = input("  Number of ideas to generate [default 12]: ").strip()
    try:
        n = int(raw)
        return max(1, min(30, n))   # clamp to 1–30
    except ValueError:
        return 12
```

---

## Option A (OpenAI Browser) — `option-a-openai/`

### `proxy.py`

Flask server. Two routes:
- `GET /` → serves `app.html`
- `POST /v1/chat/completions` → proxies to `https://api.openai.com/v1/chat/completions` with `OPENAI_API_KEY` injected server-side
- `GET /health` → JSON status check

**To change the OpenAI model**, the model is sent by `app.html` in the request body. Change the model string in the HTML file's JavaScript.

**To change the proxy port** (default 5050):
```bash
PORT=8080 python proxy.py
```

**To add a new proxy route** (e.g. for image generation):
```python
@app.route("/v1/images/generations", methods=["POST"])
def proxy_images():
    ...
```

---

## Option C (Ollama Local) — `option-c-ollama/`

### `proxy.py`

Flask server. Routes:
- `GET /` → serves `app.html`
- `POST /v1/chat` → translates app payload to Ollama format and forwards to `http://localhost:11434/api/chat`

Default model: `llama3` (constant `DEFAULT_MODEL`).

**To change the default Ollama model:**
```python
DEFAULT_MODEL = "mistral"
```

**To use a different Ollama URL** (e.g. remote Ollama instance):
```python
OLLAMA_URL = "http://192.168.1.100:11434/api/chat"
```

---

## Shared Prompt Patterns — All Options

All three options share the same core system prompt philosophy:

**Idea generation:** `"Return ONLY a valid JSON array. No markdown, no preamble. Start with [ end with ]."` — enforces clean JSON parse.

**Stress test:** `"Return ONLY valid JSON. No markdown."` — same pattern.

**Failure handling:** Both callers strip ` ```json` and ` ``` ` fences before parsing, and slice from the first `[` or `{` to the last `]` or `}` to tolerate any prefix/suffix noise from the model.

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add a focus area | `FOCUS_OPTIONS` list in `cli.py`; corresponding select in `app.html` |
| Change the model (Option B) | `call_claude()` `model=` parameter |
| Change max tokens | `call_claude()` `max_tokens=` / `callModel()` in HTML |
| Change research prompt | `research_target()` content string |
| Disable web search | Replace `research_target()` with a plain `call_claude()` call |
| Add a field to each idea | Idea schema in `generate_ideas()` system prompt |
| Add a new verdict label | `stress_test()` system prompt schema + `display_results()` symbol map |
| Add an evaluation field | `stress_test()` system prompt schema + `save_output()` + `display_results()` |
| Change output directory | `save_output()` `filename` construction |
| Change allowed idea counts | `pick_idea_count()` validation |
| Change Ollama model | `DEFAULT_MODEL` in `option-c-ollama/proxy.py` |
| Change proxy port | `PORT` env var when running `proxy.py` |
| Add a new proxy route | Flask `@app.route()` decorator in `proxy.py` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
