# meta-prompt-engine — Modification Guide
**File:** `meta-prompt-engine.html` (single-file browser application, ~38KB)  
**No build step. No server. No dependencies.**  
**External resources:** IBM Plex Mono and IBM Plex Sans from Google Fonts (graceful fallback if unavailable)

---

## Who This Document Is For

Engineers and developers who want to extend the prompt architecture, add model support, change the UI, or embed the engine's logic in another tool. This is not a usage guide — the README covers that. This document uses the actual function names, variable names, and data structures from the source.

---

## File Structure

Everything is in one `.html` file in this order:

```
<head>
  Google Fonts link
  <style> — all CSS
</head>
<body>
  UI markup
    #domain               select — 8 domain options
    #use-case             select — 5 use case options
    #model-target         select — 4 model options
    #key-row              API key input row (hidden for agnostic)
    #agnostic-note        shown when model-agnostic is selected
    #task-desc            textarea — task description
    #component-toggles    6 toggle buttons (role/context/task/format/chain/guard)
    #generate-btn         submit button
    #status               status text span
    #output-card          result card (hidden until first generation)
      #panel-anatomy      anatomy tab — #anatomy-blocks
      #panel-raw          raw tab — #raw-text, #copy-btn
      meta pills          #meta-model, #meta-domain, #meta-use, #meta-blocks
  <script>
    MODEL_CONFIG          model registry object
    TAG_CONFIG            block type registry object
    activeToggles         Set — currently enabled block keys
    rawPromptText         string — assembled raw prompt for copy
    updateKeyUI()         show/hide API key row based on selected model
    buildMetaSystemPrompt() system prompt sent to the model
    callClaude()          Anthropic SSE caller
    callOpenAI()          OpenAI SSE caller
    callGemini()          Google SSE caller
    buildAgnosticPrompt() local assembly — no API call
    renderAnatomy()       renders parsed blocks into #anatomy-blocks
    showSkeleton()        loading state placeholder
    assembleRaw()         builds the copyable raw prompt string
    updateMeta()          updates the four meta pills
    setStatus()           updates #status text and CSS class
    escHtml()             HTML escaper
    parseJSON()           JSON parser with markdown fence fallback
    generate-btn click    main event handler — orchestrates the full flow
</script>
</body>
```

---

## Data Structures

### `MODEL_CONFIG`

Object keyed by model value string. Drives all model-switching UI behavior.

```javascript
const MODEL_CONFIG = {
  claude: {
    label: 'Claude (Anthropic)',
    keyLabel: 'Anthropic API Key',
    keyPlaceholder: 'sk-ant-...',
    keyNote: '...HTML link to console.anthropic.com...',
    endpoint: 'https://api.anthropic.com/v1/messages',
    model: 'claude-sonnet-4-6',
  },
  gpt4: {
    label: 'GPT-4o (OpenAI)',
    keyLabel: 'OpenAI API Key',
    keyPlaceholder: 'sk-...',
    keyNote: '...HTML link to platform.openai.com...',
    endpoint: 'https://api.openai.com/v1/chat/completions',
    model: 'gpt-4o',
  },
  gemini: {
    label: 'Gemini 2.0 Flash (Google)',
    keyLabel: 'Google AI API Key',
    keyPlaceholder: 'AIza...',
    keyNote: '...HTML link to aistudio.google.com...',
    endpoint: null,   // URL built dynamically in callGemini()
    model: 'gemini-2.0-flash',
  },
  agnostic: {
    label: 'Model-Agnostic',
    keyLabel: null,   // no key field shown
    keyNote: null,
    endpoint: null,
    model: null,
  },
};
```

### `TAG_CONFIG`

Object keyed by block type string. Drives both the toggle buttons and the anatomy view rendering.

```javascript
const TAG_CONFIG = {
  role:    { label: 'ROLE',    cls: 'tag-role',    desc: 'Sets the model\'s identity and authority domain' },
  context: { label: 'CONTEXT', cls: 'tag-context', desc: 'Frames the situation, prior work, and known state' },
  task:    { label: 'TASK',    cls: 'tag-task',    desc: 'The primary instruction — what to produce' },
  format:  { label: 'FORMAT',  cls: 'tag-format',  desc: 'Output structure, length, and delivery constraints' },
  chain:   { label: 'CHAIN',   cls: 'tag-chain',   desc: 'Hook for feeding this output into the next prompt' },
  guard:   { label: 'GUARD',   cls: 'tag-guard',   desc: 'Constraints, exclusions, and failure modes to avoid' },
};
```

`cls` is the CSS class applied to the block type badge in the anatomy view. Each has a corresponding color defined in `<style>`.

### Global State

```javascript
let activeToggles = new Set(['role','context','task','format','chain']);  // guard off by default
let rawPromptText = '';    // updated after each generation; used by copy button
let keyVisible = false;    // tracks show/hide state of API key input
```

---

## System Prompt — `buildMetaSystemPrompt(domain, useCase, model, blocks)`

This is the only prompt sent to the model. It is rebuilt on every generation from the current UI selections.

**Inputs:**
- `domain` — value from `#domain` select (e.g. `"code"`, `"system"`)
- `useCase` — value from `#use-case` select (e.g. `"followup"`, `"chain"`)
- `model` — value from `#model-target` select (e.g. `"claude"`, `"gpt4"`)
- `blocks` — array of active block key strings (e.g. `["role","context","task","format"]`)

**Use case → description mapping (inside the function):**
```javascript
const useCaseMap = {
  followup:  'a follow-up prompt intended to run after the described task is complete...',
  parallel:  'a parallel prompt that runs alongside the described task...',
  decompose: 'a decomposition prompt that breaks the described task into...',
  review:    'a review prompt designed to critique, audit, or QA the output...',
  chain:     'a chaining prompt that takes the output of the described task...',
};
```

**Model → syntax convention mapping:**
```javascript
const modelMap = {
  claude:   'Claude (Anthropic). Use XML-style tags for structural blocks where helpful...',
  gpt4:     'GPT-4o or o-series (OpenAI). Use markdown headings for structure. Avoid XML tags...',
  gemini:   'Gemini 2.0 Flash (Google). Keep instructions concise and well-structured...',
  agnostic: 'any capable LLM. Write in model-agnostic plain language with no model-specific syntax or tags.',
};
```

**JSON schema instructed in the system prompt:**
```json
{
  "title": "short descriptive title (5-8 words)",
  "purpose": "one sentence: what this prompt achieves and when to use it",
  "blocks": [
    {
      "type": "role|context|task|format|chain|guard",
      "content": "the actual prompt text for this block",
      "note": "one sentence explaining the engineering decision behind this block"
    }
  ]
}
```

### Changing the Model String

To point Claude at a different model, change it in `MODEL_CONFIG.claude.model` and in `callClaude()`:

```javascript
// MODEL_CONFIG:
model: 'claude-opus-5',

// callClaude() body:
model: 'claude-opus-5',
```

### Changing max_tokens

Each caller has its own `max_tokens`. All three currently use `2000`:

```javascript
// callClaude:
max_tokens: 2000,

// callOpenAI:
max_tokens: 2000,

// callGemini:
generationConfig: { maxOutputTokens: 2000 },
```

### Adding a New Use Case

1. Add an `<option>` in the `#use-case` select:
```html
<option value="audit">Audit (security/compliance review)</option>
```

2. Add to `useCaseMap` in `buildMetaSystemPrompt()`:
```javascript
audit: 'a security and compliance audit prompt that reviews the described task for risks, vulnerabilities, and policy violations',
```

3. Add to `uLabels` in `updateMeta()`:
```javascript
const uLabels = { ..., audit: 'Audit' };
```

### Adding a New Domain

1. Add an `<option>` in the `#domain` select:
```html
<option value="electronics">Electronics & Hardware</option>
```

2. Add to `dLabels` in `updateMeta()`:
```javascript
const dLabels = { ..., electronics: 'Electronics & Hardware' };
```

3. Optionally add a domain-specific note in `buildMetaSystemPrompt()` if the domain needs special instruction framing. Currently the domain is inserted as plain text — no per-domain branching exists in the function.

---

## API Callers

### `callClaude(apiKey, systemPrompt, userMessage, onChunk)`

```javascript
headers: {
  'x-api-key': apiKey,
  'anthropic-version': '2023-06-01',
  'anthropic-dangerous-direct-browser-access': 'true',  // required for browser CORS
}
body: {
  model: 'claude-sonnet-4-6',
  max_tokens: 2000,
  stream: true,
  system: systemPrompt,
  messages: [{ role: 'user', content: userMessage }],
}
```

SSE field extracted: `evt.delta.text` when `evt.type === 'content_block_delta'` and `evt.delta.type === 'text_delta'`.

The `anthropic-dangerous-direct-browser-access` header is required for direct browser-to-API calls (bypasses the normal CORS block). Do not remove it.

### `callOpenAI(apiKey, systemPrompt, userMessage, onChunk)`

```javascript
headers: { 'Authorization': `Bearer ${apiKey}` }
body: {
  model: 'gpt-4o',
  max_tokens: 2000,
  stream: true,
  messages: [
    { role: 'system', content: systemPrompt },
    { role: 'user',   content: userMessage },
  ],
}
```

SSE field extracted: `evt.choices?.[0]?.delta?.content`

### `callGemini(apiKey, systemPrompt, userMessage, onChunk)`

URL is built dynamically (key in query string):
```javascript
const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:streamGenerateContent?key=${apiKey}&alt=sse`;
```

Body uses Gemini's distinct schema:
```javascript
body: {
  system_instruction: { parts: [{ text: systemPrompt }] },
  contents: [{ role: 'user', parts: [{ text: userMessage }] }],
  generationConfig: { maxOutputTokens: 2000 },
}
```

SSE field extracted: `evt.candidates?.[0]?.content?.parts?.[0]?.text`

### Adding a New Model Provider

1. Add an entry to `MODEL_CONFIG`:
```javascript
mistral: {
  label: 'Mistral Large',
  keyLabel: 'Mistral API Key',
  keyPlaceholder: '...',
  keyNote: 'Get your key at <a href="https://console.mistral.ai" target="_blank">console.mistral.ai</a>.',
  endpoint: 'https://api.mistral.ai/v1/chat/completions',
  model: 'mistral-large-latest',
},
```

2. Add an `<option>` in `#model-target`:
```html
<option value="mistral">Mistral Large</option>
```

3. Write a new caller following the same signature:
```javascript
async function callMistral(apiKey, systemPrompt, userMessage, onChunk) {
  const res = await fetch('https://api.mistral.ai/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiKey}`,
    },
    body: JSON.stringify({
      model: 'mistral-large-latest',
      max_tokens: 2000,
      stream: true,
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user',   content: userMessage },
      ],
    }),
  });
  // SSE parsing — Mistral uses OpenAI-compatible format:
  // evt.choices?.[0]?.delta?.content
  // ... same reader loop as callOpenAI
}
```

4. Add the dispatch branch in the generate button handler:
```javascript
} else if (model === 'mistral') {
  fullText = await callMistral(apiKey, systemPrompt, userMessage, onChunk);
}
```

5. Add to `mLabels` in `updateMeta()`:
```javascript
const mLabels = { ..., mistral: 'Mistral Large' };
```

6. Add to `modelMap` in `buildMetaSystemPrompt()`:
```javascript
mistral: 'Mistral Large. Use markdown headings. Be explicit about output format.',
```

---

## Model-Agnostic Mode — `buildAgnosticPrompt(domain, useCase, blocks, task)`

No API call. Builds and returns a parsed result object directly, bypassing all streaming logic.

```javascript
// Returns:
{
  title: `${useCaseLabel[useCase]} prompt — ${domain}`,
  purpose: '...',
  blocks: blocks.map(key => ({
    type: key,
    content: blockDefs[key],   // hardcoded template per block type
    note: TAG_CONFIG[key]?.desc,
  })),
}
```

`blockDefs` contains one hardcoded template string per block type (`role`, `context`, `task`, `format`, `chain`, `guard`). The `task` template varies by `useCase` via a ternary chain. The `role` template varies by `domain`.

**To change what agnostic mode produces for a block type**, edit the corresponding string in `blockDefs` inside `buildAgnosticPrompt()`.

---

## Rendering

### `renderAnatomy(blocks, parsed, isStreaming)`

Clears `#anatomy-blocks` and rebuilds it. If `parsed.purpose` exists, prepends it as a muted text paragraph. For each block:

- Looks up `TAG_CONFIG[block.type]` for label, CSS class, and fallback desc
- Creates a `.block` div with a `.block-header` (tag badge + note text) and `.block-body` (content)
- If `isStreaming` is true and this is the last block, appends a `<span class="cursor">` blinking cursor

The streaming `onChunk` callback calls `renderAnatomy(partial.blocks, partial, true)` on each chunk if `parseJSON()` succeeds, otherwise shows raw accumulated text with a cursor.

### `assembleRaw(blocks)`

Builds the copyable text shown in the Raw tab:

```javascript
function assembleRaw(blocks) {
  return blocks.map(b => {
    const cfg = TAG_CONFIG[b.type] || { label: b.type.toUpperCase() };
    return `## ${cfg.label}\n${b.content}`;
  }).join('\n\n');
}
```

Output format: `## ROLE\n[content]\n\n## CONTEXT\n[content]\n\n...`

**To change the raw format** (e.g. to XML tags for Claude outputs):
```javascript
function assembleRaw(blocks) {
  return blocks.map(b => {
    const tag = b.type.toLowerCase();
    return `<${tag}>\n${b.content}\n</${tag}>`;
  }).join('\n\n');
}
```

### `parseJSON(text)`

```javascript
function parseJSON(text) {
  try { return JSON.parse(text); } catch {}
  try { return JSON.parse(text.replace(/```json|```/g, '').trim()); } catch {}
  return null;
}
```

Two attempts: raw parse first, then strip markdown fences and retry. Returns `null` on both failures — the caller checks for `null` and handles it (either continues streaming or throws an error).

---

## Block Types — Adding a New One

1. Add to `TAG_CONFIG`:
```javascript
const TAG_CONFIG = {
  ...existing,
  persona: { label: 'PERSONA', cls: 'tag-persona', desc: 'Audience and voice constraints for the output' },
};
```

2. Add the CSS class in `<style>`:
```css
.tag-persona { background: #7c3aed; color: #fff; }
```

3. Add a toggle button in `#component-toggles`:
```html
<div class="toggle" data-key="persona">
  <div class="toggle-check"><svg ...checkmark svg...</svg></div>
  <span>Persona / audience</span>
</div>
```

4. Add to `blockDefs` in `buildAgnosticPrompt()`:
```javascript
persona: `Write for a [target audience] with [expertise level] background. Adjust vocabulary, assumed knowledge, and examples accordingly.`,
```

5. Add the type string to the block definitions in `buildMetaSystemPrompt()`:
```javascript
// In the block definitions section of the system prompt string:
- persona: defines the intended audience, their background, and the voice and vocabulary to use
```

6. Add the type to the schema enum comment in the system prompt:
```javascript
"type": "role|context|task|format|chain|guard|persona",
```

---

## Default Component State

`guard` is the only toggle that starts inactive. This is set two ways:

**In HTML** — the `guard` toggle div lacks the `active` class:
```html
<div class="toggle" data-key="guard">...</div>           <!-- no 'active' class -->
<div class="toggle active" data-key="role">...</div>     <!-- 'active' class present -->
```

**In JS** — `activeToggles` is initialized without `guard`:
```javascript
let activeToggles = new Set(['role','context','task','format','chain']);
```

To make a block default-off, remove it from both places. To make `guard` default-on, add `active` to its HTML div and add `'guard'` to the `Set`.

---

## API Key Security

Keys are held in the `value` of `#api-key` (`<input type="password">`). They are sent only inside the `fetch()` call for the selected provider. There are no `localStorage`, `sessionStorage`, `cookie`, or `IndexedDB` writes anywhere in the file. Verified: searching the source for any of these returns zero results.

The `autocomplete="off"` and `spellcheck="false"` attributes on `#api-key` prevent the browser from logging or suggesting the key.

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Change the Claude model string | `MODEL_CONFIG.claude.model` + `callClaude()` body |
| Change max tokens | `max_tokens` / `maxOutputTokens` in each caller |
| Add a new model provider | `MODEL_CONFIG` + `<option>` + new `call*()` function + dispatch branch + `mLabels` + `modelMap` |
| Add a new use case | `<option>` in `#use-case` + `useCaseMap` in `buildMetaSystemPrompt()` + `uLabels` in `updateMeta()` |
| Add a new domain | `<option>` in `#domain` + `dLabels` in `updateMeta()` |
| Add a new block type | `TAG_CONFIG` + CSS class + toggle HTML + `blockDefs` in agnostic + schema enum in system prompt |
| Change block badge color | `.tag-*` CSS class in `<style>` |
| Change raw prompt output format | `assembleRaw()` — currently `## LABEL\ncontent` |
| Change agnostic output for a block | `blockDefs[key]` string in `buildAgnosticPrompt()` |
| Change live parse fallback display | `onChunk` callback in the generate handler — the `else` branch |
| Make guard default-on | Add `active` class to guard toggle HTML + add `'guard'` to `activeToggles` Set |
| Change model syntax conventions | `modelMap` object in `buildMetaSystemPrompt()` |
| Change the model's JSON schema | Schema block in `buildMetaSystemPrompt()` return string + `renderAnatomy()` field access |
| Remove Google Fonts | Delete `<link>` tags in `<head>`; update font-family in CSS |
| Change streaming chunk poll behavior | SSE `for (const line of chunk.split('\n'))` loop in each caller |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
