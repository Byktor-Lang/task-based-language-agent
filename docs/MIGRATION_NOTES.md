# MIGRATION_NOTES.md — Source-to-Agent Mapping

## Skill-to-Agent Name Mapping

| Source skill (Desktop/Cowork) | New agent (Claude Code) | Source path |
|---|---|---|
| `conductor` | `tblt-orchestrator` | `spanish-task-based/conductor/SKILL.md` |
| `spanish-pretask-vocab` | `tblt-pretask-specialist` | `spanish-task-based/spanish-pretask-vocab/SKILL.md` |
| `tblt-spanish-activity` | `tblt-activity-specialist` | `spanish-task-based/tblt-spanish-activity/SKILL.md` |
| `reflective-loop` | `tblt-reflective-specialist` | `spanish-task-based/reflective-loop/SKILL.md` |

The `tblt-` prefix scopes all agents to this domain so they don't collide with other agents
the user may have installed.

---

## Sections Dropped and Why

### From `conductor` → `tblt-orchestrator`

| Section dropped | Why |
|---|---|
| **Phase 0 Suppression Rule** (Rule #2 in Critical Conductor Rules) | Suppression is now structural. The specialists' system prompts simply don't have a multi-turn Phase 0 elicitation sequence. The rule existed to instruct a single-context model to skip a section of its own loaded SKILL.md content. |
| **Context Recovery Rule** (Rule #9 in Critical Conductor Rules) | Subagents can't lose context the same way a single-context skill can. Each specialist invocation starts fresh with the full Shared Context Block supplied in the invocation prompt. There is no mid-pipeline context degradation scenario for a subagent. |
| **Sub-skill loading steps** (Phases 2a, 4a, 6a — `view` tool calls) | The `view`/`Read` tool calls that loaded each SKILL.md file in the single-context build are replaced by Agent tool invocations. The specialist's logic is in its own system prompt. |
| **Injection headers** (Phases 2b, 4b, 6b — `⚠ PHASE 0 IS SUPPRESSED` blocks) | These were prompts sent to the model within the same context to override Phase 0. In the new architecture, Phase 0 has been removed from the specialists entirely. The injection content is now the Agent tool invocation prompt, which simply lists the pre-supplied inputs. |
| **Sub-skill execution prose** (Phases 2c, 4c, 6c — "Execute the sub-skill…" instructions) | These were step-by-step instructions for running the sub-skill within the same context ("Show the teacher every mandatory gate…"). In the new architecture, the specialist runs its own phases; the orchestrator just invokes it and parses the manifest. |
| **Reference Skills table** (bottom of conductor) | The file-path table pointing to `/mnt/skills/user/…` paths is no longer needed. Agent invocation uses agent names, not file paths. |

**Nothing pedagogically substantive was removed from the orchestrator.** All input-elicitation
phases (0a–0h), gate logic (Gates 1–3), complication extraction, Phase 7 summary card, the
Teacher Input Recommendations Rule, and the PVS Integrity Rule are fully preserved.

### From `spanish-pretask-vocab` → `tblt-pretask-specialist`

| Section dropped | Why |
|---|---|
| **Phase 0 — Collect Inputs** (multi-field elicitation) | The orchestrator collects all inputs in Phase 0 and passes them via the Shared Context Block. The specialist receives all inputs pre-supplied. For standalone invocations, the specialist now asks for all inputs in one shot (not multi-turn) if anything is missing — this is the same "ask all at once" behavior that the original Phase 0 already specified for standalone use. |

All other content — Phase -1 (anti-repetition), Phase 1 (vocabulary classification), Phase 1b
(distribution pre-commitment), Phase 2 (distribution gate), Phase 3 (content plan gate),
Phase 4 (HTML), Phase 5a (feedback), Phase 5b (log append), proficiency calibration tables,
exercise type catalog, bridge logic, language rules, all confirmation checklists — is preserved
verbatim.

### From `tblt-spanish-activity` → `tblt-activity-specialist`

| Section dropped | Why |
|---|---|
| **Phase 0 — Input Collection** | Same reason as above. Orchestrator pre-supplies all inputs. |

All other content — Phase -1 (anti-repetition + low-rating signal), Phase 1 (topic breakdown),
Phase 1.5 (engagement design pass, all 5 lenses), Phase 2 Step A (vocabulary distribution),
Phase 2 Step B (non-item Spanish audit), Phase 2.5 (coverage table), Phase 3 (activity
construction), Phase 4 (HTML), Phase 5a (split feedback), Phase 5b (log append), all design
rules, language rules, and checklists — is preserved verbatim.

### From `reflective-loop` → `tblt-reflective-specialist`

| Section dropped | Why |
|---|---|
| **Phase 0 — Collect / Validate Inputs** (standalone elicitation block) | Orchestrator pre-supplies all inputs. For orchestrator-mode, Phase 0 becomes a validation-only step. Standalone behavior (ask all at once if missing) is preserved. |
| **Phase 1b — Complication Generation** | `{complication}` is always pre-confirmed by the orchestrator at Gate 3 before this specialist is invoked. The source skill already specified Phase 1b runs "ONLY if `{complication}` was not supplied." Since the orchestrator always supplies it, Phase 1b would never run in pipeline mode. The section is removed to make this structural rather than conditional. For standalone invocations without a complication, Phase 1b still runs — the "when invoked standalone" note in the specialist preserves this. |

All other content — Phase 1a (genre selection + `{target_register}` override), Phase 1c (scenario
anchor), Phase 1d (phrase-pair selection with quality requirements), Phase 1e (level calibration),
Phase 2 (vocabulary audit), Phase 2.5 (writing prompt coverage), Phase 3 (content plan including
all five checklist positions and closing recognition line), Phase 4 (HTML rules, component mapping,
27-item checklist), and all constraints (C01–C07) — is preserved verbatim.

---

## Session Log Schema

The `spanish-activity-log.md` file is shared between the Desktop/Cowork build and the Claude Code
build. Both builds write the same format; no schema migration is needed.

### `## Pre-Task Vocab Log` table

Written by `spanish-pretask-vocab` (Desktop/Cowork) and `tblt-pretask-specialist` (Claude Code).
Read by both for Phase -1 anti-repetition.

**Table header:**

```markdown
## Pre-Task Vocab Log

| Date | Skill | Exercise Sequence | Topic | Level | Rating (1–5) | Teacher Notes |
|---|---|---|---|---|---|---|
```

**Row format:**

```
| [YYYY-MM-DD] | spanish-pretask-vocab | [exercise sequence, e.g. TF → ODD → FREQ] | [topic] | [level] | [rating or blank] | [note or blank] |
```

Source: `spanish-pretask-vocab/SKILL.md` Phase 5b.

Note: The `Skill` column always writes `spanish-pretask-vocab` in both the old and new builds.
This preserves cross-build continuity in the anti-repetition window.

### `## TBLT Activity Log` table

Written by `tblt-spanish-activity` (Desktop/Cowork) and `tblt-activity-specialist` (Claude Code).
Read by both for Phase -1 anti-repetition and low-rating signal.

**Table header:**

```markdown
## TBLT Activity Log

| Date | Skill | Topic | Learner Profile | Paso Structure | Vocab Count | Grammar Structures | Quality (1–5) | Engagement (1–5) | Teacher Notes |
|---|---|---|---|---|---|---|---|---|---|
```

**Row format:**

```
| [YYYY-MM-DD] | tblt-spanish-activity | [topic] | [learner profile] | [Paso structure] | [vocab count] | [grammar structures] | [quality 1–5 or blank] | [engagement 1–5 or blank] | [note or blank] |
```

Source: `tblt-spanish-activity/SKILL.md` Phase 5b. Note in source: "If the table exists but doesn't
yet have the Learner Profile and Engagement columns, append them on first write; preserve old rows
by leaving those cells blank."

### Log file path resolution (Windows — this laptop)

Both skills resolve the log file as follows:
1. If environment variable `SPANISH_TBLT_LOG_DIR` is set → use `$SPANISH_TBLT_LOG_DIR\spanish-activity-log.md`
2. Otherwise → use `%USERPROFILE%\Documents\spanish-tblt\spanish-activity-log.md`

On this laptop, `%USERPROFILE%` resolves to `C:\Users\azuaje.MSMC\`, so the default log path is:

```
C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-activity-log.md
```

No environment-variable override is needed. This is the same directory that contains the source
skills (`spanish-task-based\`), so both builds read from and write to the same log file.

### `reflective-loop` / `tblt-reflective-specialist` — no log operations

Confirmed by reading both the source `reflective-loop/SKILL.md` and the conductor's `SKILL.md`:

From conductor (Phase 6c): "Phase −1 — N/A: reflective-loop does not write to the session log and
has no fetch step" and "Phase 5 — N/A: reflective-loop does not collect feedback or append a log row."

This specialist has no Phase -1, no Phase 5a, and no Phase 5b. Its manifest always emits
`log_write: not_applicable`.

---

## Windows Log Path for This Laptop

```
C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-activity-log.md
```

Achieved via the standard `%USERPROFILE%\Documents\spanish-tblt\` resolution with no
environment variable override. Both the Desktop/Cowork build and the Claude Code build write
to this same path.

---

## Non-Canonical Duplicate

A second copy of the source skills exists at `C:\Users\azuaje.MSMC\Downloads\New_Skills\`.
This is **not canonical** — the single source of truth is `Documents\spanish-tblt\spanish-task-based\`.
The user may delete or archive the `Downloads\New_Skills\` copy at any time.

---

## Known Divergences Between Old and New Behavior

### 1 — Manifest emission (new behavior, no Desktop/Cowork analog)

The specialists in the Claude Code build emit a YAML manifest as their final output block.
This does not happen in the Desktop/Cowork build. The manifest is consumed only by the
orchestrator and is invisible to the teacher. No behavior change is visible in the teacher-facing
output.

### 2 — Complication candidate via manifest (Stage 2)

In the Desktop/Cowork build, the conductor performs complication extraction (Step 4d) by
examining the inline HTML artifact from Stage 2 within the same context. In the Claude Code
build, the specialist includes a `complication_candidate` field in its manifest, and the
orchestrator uses that field for Branch B of the Gate 3 complication resolution. The logical
behavior is identical; the mechanism is different.

### 3 — Phase 5a feedback flow through orchestrator

In the Desktop/Cowork build, the sub-skill emits the Phase 5a feedback prompt directly to the
teacher in the same conversation. In the Claude Code build, the specialist emits the feedback
prompt to the orchestrator's Agent tool result, and the orchestrator forwards it to the teacher.
The teacher-facing experience is the same; the routing is different.

### 4 — Session log `Skill` column (pretask)

Both builds write `spanish-pretask-vocab` in the `Skill` column of the Pre-Task Vocab Log. This
is intentional — it preserves cross-build continuity so the Phase -1 anti-repetition window reads
correctly regardless of which build generated a prior log entry.

---

## Future Work

The following items were identified during conversion but are out of scope for this pass
(per Section 10 of the conversion brief):

- **Architecture B** — adding an integrity-reviewer subagent that runs cross-stage checks in
  parallel rather than serially after Stage 3.
- **Architecture C** — Context Steward + Pedagogical Coordinator split.
- **Register-aware pre-task sequencing** — the `{target_register}` field is already passed to
  `tblt-pretask-specialist` (as informational only, per the source skill's own forward-compatibility
  note). A future pass could use it to select exercise types that prime the post-task register.
- **Hosted log backend** — currently both builds use a local Markdown file. A shared database or
  cloud-synced file would allow multi-device continuity.
