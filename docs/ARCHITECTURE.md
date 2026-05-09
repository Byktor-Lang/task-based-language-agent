# ARCHITECTURE.md — Supervisor/Subagent Design

## Overview

This directory implements **Architecture A (Thin Supervisor + Three Specialists)** for the TBLT
lesson pipeline. The system replaces a single-context four-skill stack (the "conductor" skill
running in Claude Desktop/Cowork) with four Claude Code subagents that communicate through a
structured manifest contract.

---

## Why the Split Exists

The original Desktop/Cowork build runs all four skills in a single conversation context. This
works but has two structural limitations:

1. **Context bleed.** Each sub-skill's pedagogical logic runs in the same context as the
   orchestration logic. The conductor skill contained detailed prose instructions for suppressing
   Phase 0 in each sub-skill, injecting context headers, and managing what the teacher sees at
   each stage — because the model had to be told what *not* to do inside the same context where it
   was also being told what *to* do.

2. **No real gate boundaries.** The confirmation gates (Gate 1, Gate 2, Gate 3) are implemented as
   conversational instructions. A model following them in a single long context can lose track of
   gate state. There is no structural guarantee that Stage 2 cannot start before Stage 1 produces a
   manifest.

The supervisor/subagent architecture addresses both:

- **Context isolation:** each specialist runs in its own subagent call. It cannot see the
  orchestrator's elicitation logic, the other specialists' artifacts, or the conversation history
  from prior stages. Its Phase 0 has been removed — it has no elicitation logic to run.
- **Scoped tools:** the reflective specialist has no Bash tool. It cannot accidentally write to the
  session log. The orchestrator has no log-write logic; it delegates that to the specialists.
- **Real gate boundaries:** the Agent tool call for Stage 2 does not happen until after Gate 2 is
  confirmed. This is structural, not instructional.

---

## Agent Roles

### `tblt-orchestrator` (supervisor)

Owns the teacher conversation from start to finish. Responsibilities:

- **Phase -2:** Silent class-profile load; writes back `class-profile.md` if absent (Gate 1 only)
- **Phase 0 (0a–0h):** Sequential multi-turn input elicitation with 3-option recommendations
- **Phase 1:** PVS lock and Shared Context Block assembly; Gate 1
- **Stage delegation:** Invokes each specialist and quality gate via the Agent tool — execution order: activity-specialist (Stage 1) → tblt-inspector (post-Stage-1 quality gate) → [optional revision pass: activity-specialist] → pre-task specialist (Stage 2) → reflective specialist (Stage 3)
- **Manifest parsing:** Parses the YAML manifest returned by each specialist; runs per-stage and cross-stage integrity checks
- **Activity derivation record:** Extracts `actual_pvs_items_used` and `actual_grammar_deployed` from the Stage 1 (activity) manifest; holds and passes both to Stage 2 and Stage 3 specialists
- **Gate 2 / Gate 3:** Handoff records and teacher confirmation prompts between stages; Gate 2 displays the activity derivation summary to the teacher
- **Gate 3 complication resolution:** Extracts or generates complication/outcome; presents 3-option choices to teacher
- **Phase 7:** Pipeline Summary Card with integrated integrity verification

The orchestrator does NOT implement any exercise generation logic, Paso design logic, or register-shift
table logic. Those live exclusively in the specialists.

### `tblt-inspector` (post-Stage-1 quality gate)

Evaluates the HTML artifact and YAML manifest from Stage 1 against the Ludic-Language
framework and returns a prioritized revision list. Invoked by the orchestrator between
Stage 1 and Gate 2. Isolated subagent context (tools: Read, Write only). Responsibilities:

- Step 0: Constraint layer — linguistic boundaries check + pivot classification (Soft/Hard)
- Kill Criteria: five conditions requiring structural revision before any rubric feedback
- Layer A (Lee's Hardware): four pedagogical validity criteria, each with 0–3 anchors and an
  observable indicator (`information_gap`, `interaction_dependency`, `linguistic_targeting`,
  `linguistic_payoff_alignment`)
- Layer B (Schell's Software): four engagement quality criteria, each with 0–3 anchors and an
  observable indicator (`curiosity_surprise`, `flow_curve`, `meaningful_choice`, `ending_payoff`)
- Extended Discourse Check: three binary checks on Paso 3/5 (not scored; NO answers become
  optional improvement suggestions)
- Convergence determination: CONVERGED | FEEDBACK | ESCALATE (max 2 rounds; same criterion
  failing in round 1 = ESCALATE)
- Iteration history: writes one YAML entry per run to `inspector-exchange.md`; reads on round 1
  to detect repeated failures
- Report output: YAML header (machine-readable verdict + scores) followed by Markdown revision
  list (consumed by activity specialist on revision pass)

### `tblt-activity-specialist` (Stage 1 — execution order position; `stage: 2` in manifest)

Produces the main communicative-task worksheet. Runs first in the pipeline. Isolated subagent
context. Responsibilities:

- Phase -1: Session-log fetch, anti-repetition, and low-rating signal (reads `## TBLT Activity Log`)
- Phase 1–1.5: Topic breakdown and engagement design pass (5 lenses)
- Phase 2 Step A: Vocabulary distribution table (mandatory gate)
- Phase 2 Step B: Non-item Spanish audit (mandatory gate)
- Phase 2.5: Success Criteria Coverage Table (mandatory soft-flag gate)
- Phase 3: Activity construction (internal)
- Phase 4: A4 HTML artifact
- Phase 5a: Split feedback collection (quality + engagement + note)
- Phase 5b: Session-log append (`## TBLT Activity Log`)
- Manifest emission: YAML manifest including `complication_candidate`, `actual_pvs_items_used`,
  and `actual_grammar_deployed` fields

### `tblt-pretask-specialist` (Stage 2 — execution order position; `stage: 1` in manifest)

Produces the pre-task vocabulary worksheet. Runs second in the pipeline with both the Shared
Context Block and the Activity Derivation Block. Isolated subagent context. Responsibilities:

- Phase -1: Session-log fetch and anti-repetition rule (reads `## Pre-Task Vocab Log`)
- Phase 1–1b: Vocabulary classification and distribution pre-commitment (prioritizes PVS items
  from Activity Derivation Block when present)
- Phase 2: Vocabulary distribution table + vocabulary audit (mandatory gate)
- Phase 3: Content pre-generation plan (mandatory gate; uses activity grammar deployment for frames)
- Phase 4: A4 HTML artifact + answer key
- Phase 5a: Teacher feedback collection (rating 1–5 + note)
- Phase 5b: Session-log append (`## Pre-Task Vocab Log`)
- Manifest emission: YAML manifest as final output block

### `tblt-reflective-specialist` (Stage 3)

Produces the post-task written-expansion worksheet. Isolated subagent context with minimal
permissions (no Bash — no log I/O in this specialist). Responsibilities:

- Phase 0: Input validation (complication pre-confirmed; Phase 1b suppressed)
- Phase 1: Genre selection, scenario anchor, phrase-pair selection, level calibration
- Phase 2: Vocabulary audit (mandatory gate)
- Phase 2.5: Writing Prompt Coverage Check (mandatory soft-flag gate)
- Phase 3: Content pre-generation plan (mandatory gate)
- Phase 4: A4 HTML artifact
- Manifest emission: YAML manifest (no Phase 5a, no Phase 5b)

---

## The Manifest Contract

Every specialist returns a YAML manifest as the **last block** of its output. The manifest is the
only structured return value the orchestrator consumes; all other specialist output (artifacts,
feedback prompts, gate displays) flows through to the teacher transparently.

The manifest records:
- Which PVS items appeared in the artifact (`pvs_coverage`)
- Whether each grammar structure is present (`grammar_coverage`)
- Whether any non-PVS Spanish was detected (`non_pvs_spanish_detected`)
- Whether the transfer goal was echoed (`transfer_goal_echoed`)
- Which success criteria the artifact addresses (`success_criteria_addressed`)
- Feedback rating and note from Phase 5a
- Log-write outcome from Phase 5b
- Any integrity issues the specialist flagged itself (`flags`)
- For Stage 2: a complication candidate extracted from the Paso design

See `docs/MANIFEST_SCHEMA.md` for the full schema, per-stage variations, examples, and the
orchestrator's verification rules.

---

## Line of Responsibility

| Responsibility | Owner |
|---|---|
| Teacher conversation (all gate prompts, recommendations, summary card) | `tblt-orchestrator` |
| PVS construction and freezing | `tblt-orchestrator` (Phase 1a) |
| Cross-stage PVS consistency | `tblt-orchestrator` (manifest verification) |
| Cross-stage grammar thread | `tblt-orchestrator` (manifest verification) |
| Transfer goal echo verification | `tblt-orchestrator` (manifest verification) |
| Success criteria union coverage | `tblt-orchestrator` (manifest verification) |
| Complication/outcome resolution | `tblt-orchestrator` (Gate 3) |
| Activity derivation record (actual_pvs_items_used, actual_grammar_deployed) | `tblt-activity-specialist` (emit) + `tblt-orchestrator` (hold and route) |
| Class profile read/write | `tblt-orchestrator` (Phase -2 / Step 1d) |
| Exercise generation and pedagogical quality | Specialist for each stage |
| Per-stage vocabulary audit | Specialist for each stage |
| Per-stage coverage tables | Specialist for each stage |
| Session-log read (anti-repetition) | `tblt-pretask-specialist`, `tblt-activity-specialist` |
| Session-log write — Pre-Task Vocab Log | `tblt-pretask-specialist` |
| Session-log write — TBLT Activity Log | `tblt-orchestrator` (Phase 5b moved to orchestrator after Inspector implementation) |
| Activity quality evaluation (Lee + Schell rubric) | `tblt-inspector` |
| Inspector iteration history — write | `tblt-inspector` (one entry per run to `inspector-exchange.md`) |
| Inspector iteration history — read | `tblt-inspector` (reads round 0 entry on round 1 to detect repeated failures) |
| No log operations | `tblt-reflective-specialist` |

---

## Log Files

| File | Purpose | Owner | Location |
|---|---|---|---|
| `spanish-activity-log.md` | Pedagogical session record — topic, Paso structure, vocab count, grammar structures, teacher quality/engagement ratings, notes. Read by `tblt-activity-specialist` Phase -1 (anti-repetition) and `tblt-orchestrator` Phase -2 (class profile context). | `tblt-orchestrator` writes TBLT Activity Log section (Phase 5b, after Inspector implementation). `tblt-pretask-specialist` writes Pre-Task Vocab Log section. | `%USERPROFILE%\Documents\spanish-tblt\` (Windows) / `~/Documents/spanish-tblt/` (Mac/Linux) |
| `inspector-exchange.md` | Inspector iteration history — which criteria failed each round, what verdict was reached, per session_key. Written and read only by `tblt-inspector`. Never teacher-facing. Entries accumulate across sessions; Inspector reads only the entry matching the current session_key + round 0 on round 1 runs. | `tblt-inspector` (writes one YAML entry per run; reads on round 1). `tblt-orchestrator` resolves the file path and passes it to the Inspector. | Same folder as `spanish-activity-log.md` |
| `class-profile.md` | Class constants — ACTFL level, interest hooks, target register, scaffolding defaults, classroom constraints. Create-if-not-exists at Gate 1; never overwritten by orchestrator. | `tblt-orchestrator` (create-only at Step 1d) | Same folder as `spanish-activity-log.md` |
