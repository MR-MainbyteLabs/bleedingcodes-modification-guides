# meta-prompt-engine — Modification Guide
**File:** `meta-prompt-engine.html` (single-file browser application)  
**No build step. No server. No dependencies.**  
**External resources:** IBM Plex Mono and IBM Plex Sans from Google Fonts (graceful fallback if unavailable)

---

## Who This Document Is For

Engineers and developers who want to extend the prompt architecture, add model support, change the UI, or embed the engine's logic in another tool. This is not a usage guide — the README covers that. This document tells you where the code lives inside the HTML file and exactly where to make each class of change.

---

## File Structure

Everything lives in one `.html` file in this order:

```
<head>
  Google Fonts link
  <style> — all CSS
</head>
<body>
  UI markup — controls and output panels
  <script>
    Configuration constants
    Meta-prompt system prompt builders (one per model)
    Model-Agnostic assembler
    API caller functions (one per provider)
    SSE streaming parsers (one per provider)
    UI render functions (anatomy view, raw view)
    Event handlers (form submit, copy, tab switch)
  </script>
</body>
```

All modification targets are inside the `<script>` block. The CSS and HTML markup are straightforward — search by element ID or class when adjusting the UI.

---

## Prompt Architecture — The Six Block Types

Every generated prompt is composed of up to six typed blocks. These types are defined by the meta-prompt system prompt sent to the model, and by the JSON schema the model is instructed to return.

| Block | Purpose |
|---|---|
| `ROLE` | Model identity, expertise, and authority domain |
| `CONTEXT` | Situation framing, prior state, what was already done |
| `TASK` | Primary instruction — single deliverable, scoped and unambiguous |
| `FORMAT` | Output structure, length, markup requirements |
| `CHAIN` | Hook making this output directly usable as the next prompt's input |
| `GUARD` | Explicit exclusions, failure modes to avoid, quality bars |

`GUARD` is off by default and must be enabled per generation via the Components checkboxes. All others are enabled by default.

### JSON Schema Returned by the Model

The system prompt instructs every model to return only valid JSON matching this schema:

```json
{
  "title": "string",
  "purpose": "string",
  "blocks": [
    {
      "type": "ROLE | CONTEXT | TASK | FORMAT | CHAIN | GUARD",
      "content": "string",
      "note": "string — one sentence explaining the engineering decision"
    }
  ]
}
```

The `note` field is what populates the anatomy view — the per-block engineering rationale, not just what the block says.

### Adding a New Block Type

1. Add the new type string to the block type enum in the system prompt builders (search for `ROLE | CONTEXT | TASK | FORMAT | CHAIN | GUARD`).
2. Add a checkbox for it in the Components section of the HTML markup.
3. Add a case for it in the anatomy view render function that processes the `blocks` array.
4. Update the system prompt text to explain to the model what this block type means and when to use it.

---

## Model Support

### Supported Models

| Label | Model string | Provider | API endpoint |
|---|---|---|---|
| Claude | `claude-sonnet-4-6` | Anthropic | `https://api.anthropic.com/v1/messages` |
| GPT-4o | `gpt-4o` | OpenAI | `https://api.openai.com/v1/chat/completions` |
| Gemini 2.0 Flash | `gemini-2.0-flash` | Google | `https://generativelanguage.googleapis.com/v1beta/models/...` |
| Model-Agnostic | — | None (local assembly) | No network request |

### Model-Specific System Prompt Conventions

The system prompt adapts per model before every request. The conventions instructed:

- **Claude** — Use XML tags (`<role>`, `<task>`, etc.) for block delimiters
- **GPT-4o** — Use Markdown headings (`## ROLE`, `## TASK`, etc.)
- **Gemini** — Use concise structure, minimize nesting
- **Model-Agnostic** — Assembled locally from the meta-prompt architecture, no model call

Changing the target model changes the structural conventions in the output, not just a label.

### Adding a New Model

1. Add a new `<option>` in the Target Model `<select>` element in the HTML markup.

2. Write a system prompt builder function for the new model:
```javascript
function buildSystemPromptMyModel(domain, promptUse, components) {
    // Same structure as existing builders
    // Specify syntax conventions for the new model
    return `You are a prompt architect...`;
}
```

3. Add a new API caller function:
```javascript
async function callMyModelAPI(apiKey, systemPrompt, userMessage, onChunk) {
    const response = await fetch("https://api.mymodel.com/v1/completions", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            "Authorization": `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
            model: "my-model-name",
            messages: [{ role: "user", content: userMessage }],
            system: systemPrompt,
            stream: true,
        }),
    });
    // SSE parsing for this provider's format
}
```

4. Add a new SSE parser for this provider's streaming format (see SSE Streaming section below).

5. Add the dispatch case in the main form submit handler that routes to your new caller based on the selected model value.

6. Add an API key input field in the UI for the new provider.

---

## SSE Streaming

All three real API callers implement SSE streaming via `ReadableStream`. Each provider uses a different event format — the parsers are separate functions.

### Provider SSE Formats

**Anthropic:**
```javascript
// Event type: content_block_delta
// Target field: event.delta.text (when event.type === "text_delta")
const data = JSON.parse(line.replace("data: ", ""));
if (data.type === "content_block_delta" && data.delta?.type === "text_delta") {
    onChunk(data.delta.text);
}
```

**OpenAI:**
```javascript
// Standard chat completion chunks
// Target field: choices[0].delta.content
const data = JSON.parse(line.replace("data: ", ""));
const text = data.choices?.[0]?.delta?.content;
if (text) onChunk(text);
```

**Google:**
```javascript
// Gemini SSE
// Target field: candidates[0].content.parts[0].text
const data = JSON.parse(line.replace("data: ", ""));
const text = data.candidates?.[0]?.content?.parts?.[0]?.text;
if (text) onChunk(text);
```

### Live Parse Strategy

As chunks arrive, the engine attempts a live JSON parse on the accumulated text on each chunk. If `JSON.parse()` throws (incomplete JSON mid-stream), it falls back to displaying raw accumulated text. Once a full valid JSON is received, the anatomy view renders from the parsed structure.

To change fallback display behavior, find the `try { JSON.parse(...) } catch` block in the streaming handler and edit the `catch` branch.

---

## Model-Agnostic Mode

Model-Agnostic mode assembles a structured prompt locally without any network request or API key. It uses the same block type architecture and reads the same domain/promptUse/components configuration as the live model callers, but builds the content deterministically from the meta-prompt template rather than asking a model to generate it.

This mode is useful for: testing the tool without API keys, building prompts offline, or generating consistent baseline prompts for comparison.

To change what the Model-Agnostic assembler produces, find the `assembleModelAgnostic()` function and edit the per-block content templates directly.

---

## Domains

The Domain selector controls how the system prompt frames the task. Current options:

`General`, `Code`, `Writing/Docs`, `Data`, `Research`, `System Design`, `Creative`, `QA`

Each domain value is passed to the system prompt builder and used to add domain-specific framing to the meta-prompt instructions.

### Adding a New Domain

1. Add a new `<option>` in the Domain `<select>` element.
2. In each system prompt builder function, find the domain switch/conditional and add a case for the new domain value with appropriate framing instructions.

---

## Prompt Use Cases

The Prompt Use selector determines how the generated prompt is framed structurally. Current options:

| Value | What it means |
|---|---|
| `follow-up` | Continues from a previous session — CONTEXT block is emphasized |
| `parallel` | One of several independent prompts for the same task |
| `decompose` | Breaks a large task into sub-tasks |
| `review` | Critiques or evaluates output from a previous prompt |
| `chain` | Output feeds directly into the next prompt — CHAIN block is required |

The prompt use value is passed into the system prompt builder and changes the structural emphasis of the generated output.

### Adding a New Prompt Use

1. Add a `<option>` in the Prompt Use `<select>`.
2. In each system prompt builder, add the new case and describe to the model how to frame the output for this use.

---

## Output Views

The result is displayed in two tabs:

**Anatomy view** — Renders each block as a labeled card. Each card shows the block type, the block content, and the `note` field (the engineering rationale). Block type labels are styled with distinct colors.

**Raw view** — Shows the full assembled prompt text, ready to copy. This is what you paste into a new session.

### Changing Block Label Colors

Find the CSS block (or inline style) that maps block type strings to colors. The anatomy view applies a class or style per block type — search for `ROLE`, `CONTEXT`, etc. in the `<style>` section.

### Adding a Third View

Add a new tab button in the tab strip markup and a corresponding panel `<div>`. In the render function that fires after a complete JSON is received, add logic to populate the new panel.

---

## API Keys

Keys are collected via `<input type="password">` fields in the UI. They are held in JavaScript memory for the session duration only. They are sent exclusively to the respective provider's API endpoint via `fetch()`. They are never written to localStorage, sessionStorage, cookies, or any other persistence mechanism.

To verify: search the `<script>` block for any `localStorage` or `sessionStorage` calls — there are none.

---

## Hosting

The file is fully self-contained. To host:

- **GitHub Pages** — push to a repo with Pages enabled; the `.html` file serves directly.
- **Netlify Drop** — drag the file to [netlify.com/drop](https://netlify.com/drop).
- **Any static host** — no server-side processing required.

The only outbound connections from a loaded page are:
1. Google Fonts (optional, graceful fallback)
2. API calls to whichever provider the user selects, using the key they entered

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add a new block type | System prompt builders + Components checkbox + anatomy view renderer |
| Add a new model provider | New system prompt builder + new API caller + new SSE parser + new model `<option>` + API key input |
| Change model-specific syntax conventions | System prompt builder for that model (XML vs Markdown vs concise) |
| Change what Model-Agnostic mode generates | `assembleModelAgnostic()` function |
| Add a new domain | Domain `<select>` option + domain case in each system prompt builder |
| Add a new prompt use case | Prompt Use `<select>` option + case in each system prompt builder |
| Change block label colors in anatomy view | CSS block — find color map by block type string |
| Change JSON schema returned by model | System prompt schema definition + anatomy view renderer |
| Change SSE chunk handling / fallback display | Streaming handler `try/catch` block |
| Change context window / max tokens | API caller `body` payload — `max_tokens` field |
| Add a third output view tab | Tab markup + panel `<div>` + render logic in the completion handler |
| Change default Components selection | Initial checked state of Components checkboxes in HTML |
| Remove Google Fonts dependency entirely | Delete the `<link>` to fonts.googleapis.com; update font-family stack in CSS |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
