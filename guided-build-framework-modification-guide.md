# guided-build-framework — Modification Guide
**Type:** Prompt framework (Markdown files, no code)  
**Components:** `framework/Idea_Generator.md`, `framework/idea_evaluation_system.md`  
**Examples:** `examples/`

---

## Who This Document Is For

Anyone who wants to adapt, extend, or re-target the framework for a different domain, evaluation criteria, or build style. This is not a usage guide — the README covers that. This document tells you what each file controls, what each section does, and exactly where to make each class of change.

---

## What the Framework Actually Is

Two prompt files that work together in sequence:

```
Idea_Generator.md         → paste as USER MESSAGE → get idea list
idea_evaluation_system.md → paste as SYSTEM PROMPT → evaluate + wrap ideas
```

Neither file contains code. Both are prompt engineering artifacts — structured instructions that shape AI behavior across a session. The modification guide treats them as configurable systems with distinct parameters.

---

## File 1 — `framework/Idea_Generator.md`

**Role:** User message that produces a structured list of project ideas.  
**Paste as:** User message in a new chat session.

### Section Map

**Domain list (top of file — the primary customization point):**
```
system monitoring
automation engines
Linux infrastructure tooling
serial/UART communication
log analysis
CLI wrappers
```
This is the only section explicitly marked for user editing. Replace with your own domain interests. The framework works for any technical domain.

**Anti-patterns (DO NOT section):**
Tells the model what to avoid generating — generic beginner projects, fake enterprise architectures, unnecessary frameworks, ideas requiring huge ecosystems before being useful.

**Target directions:**
The expansion list the model should push toward — data transformation, visualization, simulations, procedural generation, interactive terminal experiences, etc. These are the creative counterweight to the domain list.

**Core goal:**
Defines what makes a good idea in this framework: worth finishing, incrementally growable, teaches reusable skills, produces visible progress quickly, avoids excessive setup.

**Output strategy:**
Enforces phased output — the model must stop after each phase and wait. Prevents overwhelming single-response dumps.

**Phases 1–4:**
- Phase 1: Generate 12–15 ideas in structured format
- Phase 2: Connection and roadmap design (2–3 roadmaps)
- Phase 3: Master project system (combined tree structure)
- Phase 4: Execution layer (status, risk, failure points per idea)

**Per-idea format (Phase 1):**
```
[IDEA X] Title
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

### Changing the Domain List

This is the standard customization. Replace the six domain lines at the top with your own areas. Examples:

```
# Game development focus:
2D game mechanics
physics simulation
procedural level generation
sprite tooling
save system design

# Data science focus:
data visualization
ETL pipeline tooling
statistical modeling
CSV/JSON transformation
exploratory analysis tools
```

### Changing Idea Count

Phase 1 defaults to 12–15 ideas. To change:
```
# Current:
Generate 12–15 highly distinct ideas.

# Change to:
Generate 8–10 highly distinct ideas.
```

### Removing a Phase

If you only want idea generation without roadmap design or the execution layer, delete Phase 2, 3, or 4 sections entirely. The STOP instruction at the end of Phase 1 will still hold.

### Changing the Per-Idea Format

Add or remove fields from the Phase 1 format block. Any field you add here will be populated by the model for every idea. Removing fields reduces response length.

**Example — add a "Stack" field:**
```
[IDEA X] Title
Domain:
Description:
Stack:               ← new
Difficulty (1–5):
...
```

### Tightening the Execution Layer (Phase 4)

Phase 4 currently asks for status tracking, failure points, scope creep warnings, and best-project recommendations. If Phase 4 is too verbose, reduce it to just the fields you use:

```
PHASE 4 — EXECUTION LAYER

For each idea include:
Status: [ ] Not Started / [~] In Progress / [x] Completed
Most Likely Failure Point:
Minimum Finishable Version:
```

---

## File 2 — `framework/idea_evaluation_system.md`

**Role:** System prompt that turns the AI into a ruthless execution-focused evaluator and guided build mentor.  
**Paste as:** System prompt (Custom Instructions / System field).

### Section Map

**SYSTEM ROLE:** Defines the evaluator persona — execution-focused, completion-maximizing, scope-collapsing. States explicit priorities and anti-priorities.

**CORE EXECUTION PRINCIPLES (10 rules):** The foundational philosophy. Rule 8 is the key anti-pattern list: what the AI must actively prevent (scope creep, premature abstraction, architecture-first thinking, etc.).

**IDEA EVALUATION ENGINE (15 criteria):**

| Criterion | What it tests |
|---|---|
| 1. REALITY CHECK | Can one person build this? Is there a visible done state? |
| 2. MOTIVATION COLLAPSE ANALYSIS | Exact stage where motivation fails |
| 3. HIDDEN COMPLEXITY ANALYSIS | Underestimated setup, edge cases, state management |
| 4. VALUE ANALYSIS | Reusable skills, portfolio value, real vs busy work |
| 5. MOMENTUM & FEEDBACK LOOP | How quickly visible progress appears |
| 6. SCOPE CLASSIFICATION | Forced: TOO BIG / TOO SMALL / MISCOPED / VALID |
| 7. FORCE SIMPLIFICATION | 2–5 hour / 1-day / MVP / absolute done versions |
| 8. EXECUTION RISK ANALYSIS | Highest risks per category + how to reduce each |
| 9. INFRASTRUCTURE COMPLEXITY RULE | Penalizes distributed/heavy backend before useful output |
| 10. AMBITION BALANCING RULE | Ambitious projects stay viable if scope is constrained |
| 11. SCORING SYSTEM | 6 positive scores - 3 friction scores = final score |
| 12. RANKING | Forced rank, no ties |
| 13. FINAL DECISION | BUILD NOW / DELAY / DROP — no soft language |
| 14. TOP EXECUTION PLANS | Hour 1 task, first file, smallest working version, done definition |
| 15. QUICK STRIKE MODE | Compressed fast-use version of the full evaluation |

**PROMPT WRAPPING ENGINE:** Controls when and how surviving ideas are converted into guided build sessions. Key rules:
- Only BUILD NOW and DELAY ideas are eligible for wrapping
- The AI must present surviving ideas and ask which to wrap — it does not auto-wrap all of them
- Exception: if the user pastes a single idea immediately after evaluation, that is treated as implicit wrap intent

**WRAPPED PROMPT PURPOSE + EXECUTION STYLE:** Defines what a wrapped prompt must and must not do. The core constraint: the wrapped AI must not begin by writing code. It must guide the user through naming the first file, defining the first behavior, and asking the user to attempt the first lines themselves.

**GUIDE-FIRST CODING RULE:** Explicit sequencing — the AI may only provide code after: the user asks for it, makes an attempt and needs correction, is blocked, or a tiny syntax example is necessary.

**LIVE BUILD GUIDANCE RULES:** Governs the AI's behavior during active implementation — small steps, frequent testing, minimal cognitive overload, avoiding rewrites.

**INCREMENTAL DEVELOPMENT RULES:** One feature at a time, clear completion boundaries, explain complexity cost before adding anything.

**MECHANICS & TOOLING CLARITY RULES:** Explanation style — concise, implementation-focused, no academic theory, no more context than the current step needs.

**FAILURE PREVENTION RULES:** Active anti-patterns to suppress throughout the session.

### Changing the Scoring System

Criterion 11 defines the scoring formula:

```
FINAL SCORE =
(Execution + Learning + Reusability + Clarity + Visible Progress + Scope Control)
-
(Setup Friction + Debugging Friction + Maintenance Friction)
```

**To add a new positive score dimension** (e.g. "Portfolio Value"):

In criterion 11, add:
```
- Portfolio Value   1–10
```

And update the formula:
```
FINAL SCORE =
(Execution + Learning + Reusability + Clarity + Visible Progress + Scope Control + Portfolio Value)
-
(Friction Total)
```

**To change friction weighting** (e.g. double-weight Setup Friction):

```
FINAL SCORE =
(positive scores)
-
(Setup Friction × 2 + Debugging Friction + Maintenance Friction)
```

### Changing the Scope Classification Values

Criterion 6 forces each idea into one of four categories. To add a new classification:

```
Force EVERY idea into ONE category:

- TOO BIG
- TOO SMALL
- MISCOPED
- VALID
- VALID BUT SOLO-RISKY    ← new: valid scope but high solo-execution risk
```

Add the new value to criterion 13 (FINAL DECISION) as an eligible BUILD NOW sub-type or its own decision class.

### Changing the Final Decision Labels

Criterion 13 currently uses: `BUILD NOW`, `DELAY`, `DROP`.

To add a fourth classification:

```
13. FINAL DECISION

Classify each idea:

- BUILD NOW
- DELAY
- DROP
- ARCHIVE    ← new: interesting but wrong time, save for later
```

Update the PROMPT WRAPPING ENGINE to also allow ARCHIVE ideas to be wrapped on explicit user request.

### Loosening the Guide-First Coding Rule

The current rule requires the AI to ask the user to attempt the first lines before providing any code. To make it less strict:

Find the GUIDE-FIRST CODING RULE section and change:

```
# Current — strict:
The next AI may provide code only after one of these happens:
- the user asks for code
- the user makes an attempt and needs correction
- the user is blocked

# Looser version:
The next AI should prefer guided discovery but may provide
short code snippets (under 10 lines) to illustrate the next step.
Full implementations should still wait for user request or blockage.
```

### Adapting for a Non-Technical Domain

The framework is domain-agnostic. To use it for, say, writing or design projects:

In SYSTEM ROLE, replace references to "engineering" with the relevant domain:

```
# Current:
practical engineering skill development

# Writing adaptation:
practical craft development — structure, voice, editing discipline
```

In CORE EXECUTION PRINCIPLES, replace code-specific anti-patterns:

```
# Current:
speculative extensibility, architecture-first development

# Writing adaptation:
endless outlining before drafting, premature structural revision,
research rabbit holes before a working draft exists
```

In HIDDEN COMPLEXITY ANALYSIS (criterion 3), replace technical edge cases:

```
# Current:
setup overhead, environment problems, state management

# Writing adaptation:
scope creep into adjacent topics, research burden underestimation,
revision fatigue from unclear target audience
```

---

## Examples Directory

| File | What it shows |
|---|---|
| `example_input_ideas.md` | Five raw idea titles — the format to paste after evaluation system is loaded |
| `example_output_evaluation.md` | What the evaluation output looks like for those five ideas |
| `generated_wrapped_prompt_example.md` | A complete wrapped prompt for "Terminal Biome Generator" — ready to paste as a user message to start a guided build session |

The wrapped prompt example is the most useful reference for understanding what the output looks like and how the guide-first coding rule manifests in practice.

---

## Workflow Summary

```
1. Edit domain list in Idea_Generator.md
2. Paste Idea_Generator.md as user message → get idea list
3. Paste idea_evaluation_system.md as system prompt in new session
4. Send idea list as first user message
5. Review evaluation → decide which ideas to wrap
6. Receive wrapped prompt → paste as user message in new session → build
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Change the domain focus | Domain list at the top of `Idea_Generator.md` |
| Change idea count per generation | Phase 1 count in `Idea_Generator.md` |
| Add a field to the per-idea format | Phase 1 format block in `Idea_Generator.md` |
| Remove phases you don't use | Delete Phase 2/3/4 sections from `Idea_Generator.md` |
| Add a new scoring dimension | Criterion 11 formula in `idea_evaluation_system.md` |
| Change friction weighting | Criterion 11 formula — multiply friction terms |
| Add a new scope classification | Criterion 6 list + criterion 13 decision labels |
| Add a fourth decision label (e.g. ARCHIVE) | Criterion 13 + PROMPT WRAPPING ENGINE eligibility rule |
| Loosen guide-first coding rule | GUIDE-FIRST CODING RULE section |
| Adapt for non-technical domain | SYSTEM ROLE + principle language + criterion 3 complexity list |
| Change wrapped prompt default structure | WRAPPED PROMPT EXECUTION STYLE section |
| Change what "done" means | Criterion 7 simplification tiers + criterion 14 execution plan |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
