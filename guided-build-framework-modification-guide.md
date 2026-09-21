# Guided Build Framework — Modification Guide

**Repo:** [BleedingCodes/guided-build-framework](https://github.com/BleedingCodes/guided-build-framework)
**Maintained by:** MainbyteLabs

---

## What This Guide Covers

This guide covers all four editions in the guided-build-framework repo:

| Edition | File | Type |
|---|---|---|
| Core | `Idea_Generator.md` + `idea_evaluation_system.md` | Two-file, system prompt + user message |
| 3D Python Games | `Python3d-games-idea-gener-n-tester.md` | Single self-contained file |
| Public API + Python | `Free-Public-API-n-Python-idea-gener-n-tester.md` | Single self-contained file |
| Novelty-Weighted | `novelty-weighted-idea-gener-n-tester.md` | Single self-contained file — customize domain list before use |

It covers: what is shared across all editions, what is domain-specific to each, how the Novelty-Weighted edition differs structurally from the others, how to safely modify any edition, and how to build a new edition from scratch.

---

## Architecture Overview

### Core Edition (Two Files)

The Core edition separates the two jobs into two files:

- `Idea_Generator.md` — pasted as a **user message**. Generates ideas. The domain list at the top is the only thing you configure.
- `idea_evaluation_system.md` — pasted as a **system prompt**. Evaluates and stress-tests the ideas from the generator.

These two files work together but are not aware of each other at the prompt level — the user bridges them by copying ideas from Phase 1 output into the system prompt session.

### Domain Editions (Single File)

The 3D Games, Public API, and Novelty-Weighted editions merge both jobs into one file pasted as a **user message**. The AI session starts, generates ideas (Phase 1), hard-stops, then waits for stress-test selection (Phase 2), hard-stops again, then generates a wrapped build prompt on request (Phase 3).

The single-file design means:
- no system prompt needed
- all three phases are embedded and gated by hard stops
- the AI enforces its own role through the embedded system role block at the top

### Novelty-Weighted Edition — Structural Differences

The Novelty-Weighted edition is not a domain lock like 3D Games or Public API. It changes the evaluation axis. Three things differ structurally from every other edition:

**1. Phase 2 runs 16 evaluations, not 15.**
Evaluation 1 is a dedicated Novelty Audit that runs before the Reality Check. It classifies each idea as GENUINE NOVELTY, INCREMENTAL VARIATION, or COSMETIC RESKIN — with no hedging allowed. A COSMETIC RESKIN with no clear fix path is DROP regardless of how easy the project would be to finish.

**2. The scoring formula has 7 positive scores, not 6.**
Novelty is added as a seventh positive score (10 = Genuine Novelty, 5–7 = Incremental Variation, 1–3 = Cosmetic Reskin). The formula becomes:
```
FINAL SCORE = (sum of 7 positive scores) − (sum of 3 friction scores)
```
This is a deliberate change from the base formula. Do not apply the base 6-score formula to this edition.

**3. The domain list is a user-supplied blank, not a fixed seed list.**
The Phase 1 domain list contains a placeholder (`[YOUR DOMAIN LIST HERE]`) with example seeds below it. Users must replace the placeholder with their own interests before pasting the file. The README calls this out explicitly. Do not remove the placeholder or replace it with a fixed domain — the whole point of this edition is that it works for any domain.

These three differences are load-bearing. Everything else in the Novelty-Weighted edition follows the same structure as the other single-file editions.

---

## Shared Structure — All Editions

These elements are identical or near-identical across all three editions. Do not change them without understanding the downstream effects.

### The Three-Phase Gate Structure

```
Phase 1 — Idea Generation     → HARD STOP → awaits: "stress test ideas [numbers]"
Phase 2 — Stress Test         → HARD STOP → awaits: "wrap idea [number]"
Phase 3 — Wrapped Prompt      → ends
```

The hard stops are what prevent the AI from running ahead. They are literal output blocks the AI is instructed to print exactly, followed by an instruction to stop completely and add no commentary. Do not soften this language — "stop completely" and "DO NOT begin Phase 2" are load-bearing.

### The Stress Test Evaluations (Phase 2)

The Core, 3D Games, and Public API editions run 15 evaluations. The Novelty-Weighted edition runs 16 — the same 15 plus a Novelty Audit prepended as evaluation 1. The category names and evaluation order are otherwise fixed across all editions. Only the domain-specific language inside each evaluation changes.

| # | Evaluation |
|---|---|
| 1 | Reality Check |
| 2 | Motivation Collapse Analysis |
| 3 | Hidden Complexity Analysis |
| 4 | Value Analysis |
| 5 | Momentum & Feedback Loop Analysis |
| 6 | Scope Classification (TOO BIG / TOO SMALL / MISCOPED / VALID) |
| 7 | Force Simplification |
| 8 | Execution Risk Analysis |
| 9 | Infrastructure Complexity Rule |
| 10 | Ambition Balancing Rule |
| 11 | Scoring |
| 12 | Ranking |
| 13 | Final Decision (BUILD NOW / DELAY / DROP) |
| 14 | Execution Plans (BUILD NOW ideas only) |
| 15 | Quick Strike Summary |

### Scoring Formula

**Core, 3D Games, and Public API editions:**
```
Positive scores (1–10 each):
    Execution Likelihood
    Learning Value
    Reusability
    Clarity
    Visible Progress
    Scope Control

Friction scores (1–10 each, higher = worse):
    Setup Friction
    Debugging Friction
    Maintenance Friction

FINAL SCORE = (sum of 6 positive scores) − (sum of 3 friction scores)
```

**Novelty-Weighted edition:**
```
Positive scores (1–10 each):
    Execution Likelihood
    Novelty  ← added (10 = Genuine Novelty, 5–7 = Incremental Variation, 1–3 = Cosmetic Reskin)
    Learning Value
    Reusability
    Clarity
    Visible Progress
    Scope Control

Friction scores (1–10 each, higher = worse):
    Setup Friction
    Debugging Friction
    Maintenance Friction

FINAL SCORE = (sum of 7 positive scores) − (sum of 3 friction scores)
```

Do not change either formula. Do not apply the 6-score formula to the Novelty-Weighted edition — the Novelty score is structural, not optional.

### The Wrapped Prompt Rules (Phase 3)

All editions enforce:
- **Guide-first coding rule** — the AI must not write project code until the user attempts it, asks for it, or is blocked
- **Live build guidance rules** — small steps, fast test cycles, stable working states, preserve working code
- **Incremental expansion rules** — one feature at a time, complexity warnings before any major addition

These are what make a wrapped prompt session behave differently from a plain "build this for me" session. Do not remove or weaken them.

### The System Role Block

All domain editions open with a "SYSTEM ROLE (ENFORCED FOR THIS ENTIRE SESSION)" block. This shapes the AI's behavior for the entire session before any phase runs. It defines what the AI prioritizes, what it does not prioritize, and what it assumes about the user. This block must stay at the top of any single-file edition.

---

## What Is Domain-Specific

These elements differ between editions and are where most customization happens.

### 1. The Absolute Constraints Block

Each edition has a non-negotiable constraints block that defines what every generated idea must satisfy. This is the most important domain-specific block — it is what prevents the AI from generating ideas that cannot survive the domain's real execution conditions.

| Edition | Core constraint |
|---|---|
| Core | None — domain is user-defined |
| 3D Games | Every idea must open a real rendered window and use real 3D geometry. Terminal/ASCII output is banned outright. |
| Public API | Every idea must make a real HTTP request to a real live API and use the real response. Mocked or hardcoded data is banned outright. |
| Novelty-Weighted | Every idea must have a genuine novelty angle — stated explicitly, not implied. Ideas that cannot clear this bar are labeled SAFE BASELINE, never smuggled in as novel. |

When building a new edition, write this block first. It is the single clearest statement of what the domain is and what it refuses to tolerate.

### 2. The Idea Format Fields

Each edition's idea format includes the shared base fields plus domain-specific additions.

**Shared base fields (all editions):**
```
Domain:
Description:
Difficulty (1–5):
Dependencies:
Estimated Time:
Visible Result:
Smallest Working Version:
Why it's interesting / what it teaches:
Why someone would continue using it:
Scope Risk:
Connects To:
```

**3D Games edition adds:**
```
Library/Engine (Ursina / Panda3D / PyOpenGL+moderngl / Pyglet / raylib-py / other):
Install/Platform Risk (Linux-specific):
```

**Public API edition adds:**
```
API(s) Used:
Auth Type (none / free API key / free-tier signup / OAuth):
Auth/Rate-Limit Risk:
```

**Novelty-Weighted edition adds:**
```
Novelty Lens Used:           [Cross-domain transplant / Constraint removal / Inversion / Underserved niche / Itch-scratching / SAFE BASELINE]
Nearest Existing Alternative:
What's Actually New:
Groundbreaking Potential (1–5):
Smallest Working Version (must retain the novel part):   ← replaces the standard Smallest Working Version field
```

Note: the Novelty-Weighted edition replaces the standard `Smallest Working Version` field with a version that includes an explicit constraint — the smallest version must still contain the novel part. This is intentional: if simplification strips the novelty out, the idea's "new part" is actually the hard part and must be front-loaded, not deferred.

When building a new edition, identify the 1–3 fields that expose the domain's specific failure modes. Those are the fields to add.

### 3. The Idea Generation Domain List

Each edition seeds Phase 1 with a list of domain-specific categories to push variety. These are the starting points the AI uses to avoid generating 12 variations of the same idea.

**3D Games domain categories (examples):**
- procedural terrain / voxel worlds
- physics-driven mini-games
- camera and control experimentation
- particle systems and generative-art 3D toys

**Public API domain categories (examples):**
- alert and watcher bots (price watchers, RSS/news watchers)
- API mashups (combine two or more free APIs)
- data logging / tracking tools (poll on a schedule, build local dataset)
- chat/notification bot integrations (Discord/Slack webhooks)

For a new edition, write 10–12 category seeds. Make them specific enough that two different categories produce structurally different engineering problems — not just different topics.

### 4. Quantity and Variety Requirements

All editions target 12–15 ideas and require the same four buckets: quick wins, deep systems, creative/generative projects, and practically useful projects. The domain editions add one domain-specific variety constraint:

- 3D Games: include at least one PyOpenGL/moderngl idea (not all Ursina)
- Public API: include at least a couple of no-auth ideas (not all key-required)

The Novelty-Weighted edition replaces one of the four standard buckets and adds two hard limits:
- Replaces "practically useful projects" bucket with "at least 5 ideas rated Groundbreaking Potential 4–5"
- Adds: at most 3 SAFE BASELINE ideas — clearly labeled, included only for contrast, not padded to inflate the list
- Adds: label which novelty lens produced each idea; if no lens fits, it's probably not novel

For a new edition, identify the equivalent constraint — the one that prevents the AI from defaulting to the easiest approach for every idea.

### 5. Domain-Specific Risk Categories in Phase 2

The stress test evaluations use domain-specific language for the most important risk types. The evaluation structure is fixed; the risk vocabulary inside each evaluation is not.

**3D Games — domain-specific risks called out in Phase 2:**
- GPU/driver issues on Linux (OpenGL version, Wayland vs X11)
- asset sourcing (free low-poly models, textures, licensing)
- camera/controls complexity (coordinate space, rotation math)
- packaging a 3D Python game for distribution
- "invisible progress syndrome" — blank/black screen debugging

**Public API — domain-specific risks called out in Phase 2:**
- API key signup delays, email verification, dashboard registration
- secret management (env vars, `.env` file, not committing keys to git)
- rate limits and throttling (free-tier daily caps)
- pagination and inconsistent JSON schemas
- APIs that quietly change or shut down

**Novelty-Weighted — domain-specific risks called out in Phase 2:**
- Novelty erosion — how simplification could quietly turn the project into a clone of its nearest existing alternative
- Cosmetic novelty — an idea that sounds original but differs from an existing thing only in surface details (language choice, UI skin, naming)
- Novel-but-pointless — genuinely uncommon but with no practical value once built
- Front-loading failure — deferring the novel part to "later" and building only the generic scaffolding first, then abandoning before the interesting part ships

These risks are surfaced explicitly in evaluation 1 (Novelty Audit), evaluation 8 (Execution Risk Analysis adds `Highest novelty-erosion risk:`), evaluation 15 (execution plan adds `What the novel part looks like at hour 1:`), and Phase 3 (the wrapped prompt includes novelty-protection language).

For a new edition, identify the 4–6 failure modes that are specific to your domain and invisible to someone who hasn't hit them before. Those go into evaluations 2 (Motivation Collapse), 3 (Hidden Complexity), and 8 (Execution Risk Analysis).

### 6. The Execution Plan Format (Phase 2, Evaluation 14)

Each edition's BUILD NOW execution plan includes a domain-specific setup step.

**3D Games:**
```
Hour 1 task:
First file to create:
Library setup command(s):
Smallest working version:
What "done" means:
Maximum allowed build time:
What NOT to add:
Biggest stall risk:
```

**Public API:**
```
Hour 1 task:
First file to create:
API setup steps (key signup / no-auth verification):
Smallest working version:
What "done" means:
Maximum allowed build time:
What NOT to add:
Biggest stall risk:
```

**Novelty-Weighted:**
```
Hour 1 task:
First file to create:
Smallest working version:
What the novel part looks like at hour 1 (build this before the plumbing, not after):
What "done" means:
Maximum allowed build time:
What NOT to add:
Biggest stall risk:
```

The key addition is `What the novel part looks like at hour 1:` — this forces the novel angle to be built first, not deferred until generic scaffolding is complete. This is the single most important field in the novelty edition's execution plan.

For a new edition, identify the single most important domain-specific first action and encode it as a field here.

### 7. The Wrapped Prompt's First Steps (Phase 3)

Each edition's guide-first coding rule specifies what the AI must verify before any code is written. This is domain-specific.

**3D Games — verify first:**
> Confirm the library is installed and a blank window/scene renders successfully

**Public API — verify first:**
> Confirm the API key is obtained (or no-auth access is verified) and a single test request succeeds — even via `curl` or the browser, before any Python is written

**Novelty-Weighted — verify first:**
> Confirm that the novel part of the idea is the first thing being built, not deferred. If the first working version only contains generic scaffolding, stop and restructure so the novel angle ships before anything else.

For a new edition, the first verification step must be the one thing that, if skipped, causes the most common first-session failure.

---

## How to Modify an Existing Edition Safely

### Changing the domain category list (Phase 1)

Safe to change. Add, remove, or rewrite categories freely. The only rule: each category should produce a structurally different engineering problem, not just a different theme.

### Adding a new idea format field

Safe to add. Add it to the idea format block in Phase 1 and add corresponding language to the stress test evaluations where it's relevant (usually evaluations 2, 3, and 8). Keep it to 1–2 new fields — more than that makes the format harder to scan.

### Changing the quantity requirements (12–15 ideas, four buckets)

Change with care. The four-bucket requirement (quick wins, deep systems, creative, practical) is what forces idea variety. Removing it tends to produce lists heavy on one type. The 12–15 count is the minimum needed for meaningful selection — going below 10 weakens the Phase 1 output.

### Changing the stress test evaluations

Do not remove evaluations. The base 15 evaluations are a unit — removing any one creates blind spots that show up as failed builds later. You can add domain-specific language inside an evaluation, or add an extra evaluation for a domain-specific concern, but the base 15 must remain. The Novelty-Weighted edition already uses 16 — do not remove the Novelty Audit (evaluation 1) from that edition under any circumstances; it is what makes the edition structurally different from the others.

### Changing the scoring formula

Do not change the formula. It is referenced by name in the output and recognized by repeat users. If you want to weight certain scores differently, add a note below the formula explaining the weighting — don't change the formula itself.

### Changing the hard stop blocks

Do not soften the hard stop blocks. "Stop completely" and "DO NOT begin Phase 2" are the literal instructions the AI follows. Softening them ("pause here" or "wait for instructions") produces inconsistent behavior where the AI sometimes runs ahead into the next phase.

### Changing the Phase 3 wrapped prompt rules

Safe to adjust the live build guidance and incremental expansion rules for domain-specific behavior. The guide-first coding rule (no code before the user attempts it) must stay — it is the core behavior that makes a wrapped session different from a plain build session.

---

## How to Build a New Edition from Scratch

Use this checklist. Build in this order — the constraints block must exist before you write the idea categories, or the categories will not be correctly constrained.

**Step 1 — Define the domain in one sentence.**
What is the technology or domain? What is the non-negotiable output constraint? (Example: "Python tools that talk to real live APIs — no mocked data." "Real rendered 3D in Python — no terminal output.")

**Step 2 — Write the Absolute Constraints block.**
List every constraint that every generated idea must satisfy. Be specific about what is banned outright. This is the most important block in the file.

**Step 3 — Write the Library/Technology Reality block.**
List the specific tools, libraries, APIs, or platforms the user can realistically use. This prevents ideas that require unavailable or pay-gated resources.

**Step 4 — Write 10–12 domain category seeds.**
Each category should produce a structurally different engineering problem. Vary the interaction model, the output type, and the engineering challenge.

**Step 5 — Add domain-specific idea format fields.**
Identify 1–3 fields that expose your domain's specific failure modes. Add them to the base field list.

**Step 6 — Write the domain-specific variety requirement.**
What is the one constraint that prevents the AI from defaulting to the easiest approach for every idea? Add it to the "Quantity and Variety" section.

**Step 7 — Write the domain-specific risk vocabulary for Phase 2.**
List 4–6 failure modes specific to your domain. Weave them into evaluations 2 (Motivation Collapse), 3 (Hidden Complexity), and 8 (Execution Risk Analysis).

**Step 8 — Write the domain-specific execution plan field.**
Replace `Library setup command(s):` or `API setup steps:` with the equivalent for your domain.

**Step 9 — Write the Phase 3 first-verification step.**
What is the one thing that must be confirmed working before any code is written?

**Step 10 — Assemble the full file.**
Structure: Usage comment block → System Role → Absolute Constraints → Library/Technology Reality → Phase 1 → Hard Stop → Phase 2 (all 15 evaluations) → Hard Stop → Phase 3 → Final Rules.

**Step 11 — Test it.**
Paste the file as a user message in a fresh AI chat session. Verify: Phase 1 stops correctly, the idea format includes your domain fields, Phase 2 runs all 15 evaluations with domain-appropriate language, Phase 2 stops correctly, Phase 3 generates a wrapped prompt that enforces guide-first coding.

---

## File Naming Convention

Follow the existing pattern for new editions:

```
[Domain-keyword]-idea-gener-n-tester.md
```

Examples:
```
Python3d-games-idea-gener-n-tester.md
Free-Public-API-n-Python-idea-gener-n-tester.md
novelty-weighted-idea-gener-n-tester.md
```

Place new edition files in `/framework/` and update the repo README to add a row to the Framework Editions table.

---

## Checklist After Any Change

- [ ] Phase 1 hard stop block is intact and outputs exactly the specified text
- [ ] Phase 2 hard stop block is intact and outputs exactly the specified text
- [ ] All 15 stress test evaluations are present (16 for the Novelty-Weighted edition — Novelty Audit must be evaluation 1)
- [ ] Scoring formula is unchanged (6-score formula for Core/3D/API editions; 7-score formula for Novelty-Weighted)
- [ ] Guide-first coding rule is present in Phase 3
- [ ] Novelty-Weighted edition: domain list placeholder is intact — not replaced with a fixed list
- [ ] New edition file added to `/framework/`
- [ ] README Framework Editions table updated with new file link
- [ ] README Repository Structure section updated

---

*Maintained by MainbyteLabs | github.com/MR-MainbyteLabs*
