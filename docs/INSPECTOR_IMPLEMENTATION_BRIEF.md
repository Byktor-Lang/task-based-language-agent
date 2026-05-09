# INSPECTOR IMPLEMENTATION BRIEF
# For: Future Claude session implementing the tblt-inspector agent
# Project root: C:\Users\azuaje.MSMC\Documents\spanish-tblt\tblt-v2\

---

## FIRST PROMPT TO SELF (copy this verbatim as your opening instruction)

```
Read this entire file before touching anything:
C:\Users\azuaje.MSMC\Documents\spanish-tblt\tblt-v2\docs\INSPECTOR_IMPLEMENTATION_BRIEF.md

Then read these four files in full:
1. .claude/agents/tblt-orchestrator.md
2. .claude/agents/tblt-activity-specialist.md
3. .claude/agents/tblt-pretask-specialist.md
4. .claude/agents/tblt-reflective-specialist.md

Then read:
5. docs/MANIFEST_SCHEMA.md
6. docs/ARCHITECTURE.md

Do not edit any file until you have read all six. The brief tells you
exactly what to create and what to modify, in what order, and what must
not change. Work through the implementation sequence in Section 10 exactly.
```

---

## 1. WHAT THIS DOCUMENT IS

This brief captures all design decisions made in the inspector design session and
gives a future implementation session everything it needs to build and integrate
the `tblt-inspector` agent without re-reading the design conversation.

The brief is self-contained. Do not infer anything not stated here. When in doubt,
implement the minimal change described — do not extend or refactor beyond scope.

---

## 2. PROJECT CONTEXT

### File locations

| File | Path |
|---|---|
| Orchestrator | `.claude/agents/tblt-orchestrator.md` |
| Activity specialist | `.claude/agents/tblt-activity-specialist.md` |
| Pre-task specialist | `.claude/agents/tblt-pretask-specialist.md` |
| Reflective specialist | `.claude/agents/tblt-reflective-specialist.md` |
| Manifest schema | `docs/MANIFEST_SCHEMA.md` |
| Architecture doc | `docs/ARCHITECTURE.md` |
| Activity log | `%USERPROFILE%\Documents\spanish-tblt\spanish-activity-log.md` |
| Inspector exchange log (NEW) | `%USERPROFILE%\Documents\spanish-tblt\inspector-exchange.md` |

### Current pipeline execution order (already implemented)

The backward design change from `docs/BACKWARD_DESIGN_REDESIGN_BRIEF.md` is
already in place. The execution order is:

```
Gate 1 → tblt-activity-specialist → Gate 2 → tblt-pretask-specialist
       → Gate 3 → tblt-reflective-specialist → Phase 7
```

The activity specialist runs first because it is the design anchor. Its manifest
produces an Activity Derivation Record (`actual_pvs_items_used`,
`actual_grammar_deployed`) that the pre-task and reflective specialists receive
as a second input block alongside the Shared Context Block.

### Design decisions confirmed for this implementation

1. **No Simulator (Option B).** No persona-based simulation agent exists anywhere
   in the pipeline — not as a blocking gate, not as a teacher-facing display, not
   as optional output.

2. **Lee + Schell for both agents.** The `tblt-activity-specialist` (creator) and
   the `tblt-inspector` (evaluator) both operate within the Ludic-Language framework:
   Lee's TBLT structural principles (Hardware) and Schell's engagement lenses
   (Software). The Inspector uses no external theoretical framework.

3. **Separate logs.** The class activity log (`spanish-activity-log.md`) is
   teacher-facing and written only by the activity specialist. The inter-agent
   exchange file (`inspector-exchange.md`) is written and read only by the
   Inspector and orchestrator. The two files never overlap.

4. **Backward design is out of scope for this implementation.** Three options for
   strengthening backward design verification were identified (pre-flight gate,
   Inspector layer, Phase 7 cross-stage audit) but are not implemented here.
   Backward design verification remains as the specialist's internal Phase 2.5
   gate and the orchestrator's existing cross-stage manifest checks.

---

## 3. WHAT THIS IMPLEMENTATION ADDS

Three changes, no more:

1. **NEW file:** `.claude/agents/tblt-inspector.md` — the Inspector agent
2. **MODIFY:** `.claude/agents/tblt-orchestrator.md` — insert Inspector invocation
   after Stage 1, suppress Phase 5a/5b on specialist invocation, move feedback
   collection to post-Inspector, update Gate 2 display
3. **MODIFY:** `.claude/agents/tblt-activity-specialist.md` — add Phase 4.5
   (revision intake for Inspector-driven revision passes)

The pre-task specialist, reflective specialist, manifest schema, and all other
files are untouched.

---

## 4. THE INSPECTOR'S FUNCTION

The Inspector reviews the HTML artifact and YAML manifest produced by the
`tblt-activity-specialist` at the decision point before Gate 2. Its sole job is
to evaluate the output against the Ludic-Language framework (Lee's TBLT
structural principles + Schell's engagement lenses) and return specific,
actionable feedback on how the output can be improved. It does not approve or
reject — it provides a prioritized revision list. The specialist acts on that
feedback and re-emits. This loop runs until convergence criteria are met or
the maximum round count is reached.

The Inspector does not check backward design. It does not simulate students.
It does not collect teacher feedback. It does not write to the class activity log.

---

## 5. INSPECTOR POSITION IN THE PIPELINE

```
Stage 1: tblt-activity-specialist
  Phases -1 through 4 only (Phase 5a and 5b suppressed on orchestrator invocation)
  → emits HTML artifact + YAML manifest
        ↓
tblt-inspector [round 0]
  → reads HTML artifact + YAML manifest + Shared Context Block
  → writes round 0 entry to inspector-exchange.md
  → returns YAML report header + Markdown revision list
        ↓
  IF converged → orchestrator collects Phase 5a feedback from teacher
                 orchestrator writes Phase 5b log entry
                 → Gate 2

  IF feedback → tblt-activity-specialist [revision pass]
                  Phase 4.5 + Phase 3 reconstruction + Phase 4 HTML only
                  (all other phases suppressed)
                  → re-emits HTML artifact + YAML manifest
                        ↓
                tblt-inspector [round 1]
                  → reads round 0 entry from inspector-exchange.md
                  → checks for repeated failures
                  → writes round 1 entry to inspector-exchange.md
                  → returns report
                        ↓
                  IF converged → Phase 5a, Phase 5b, Gate 2
                  IF same criterion fails again → escalate to teacher at Gate 2
```

---

## 6. INSPECTOR AGENT SPECIFICATION

### Agent file header

```
---
name: tblt-inspector
description: >
  Post-generation quality gate for tblt-activity-specialist. Evaluates the
  HTML artifact and YAML manifest against the Ludic-Language framework
  (Lee's TBLT principles + Schell's engagement lenses) and returns a
  prioritized revision list. Invoked by tblt-orchestrator after Stage 1,
  before Gate 2. Do not invoke directly.
tools: Read, Write
model: inherit
---
```

### Inputs (passed in invocation prompt by orchestrator)

| Input | What it contains |
|---|---|
| HTML artifact | Full content of the activity HTML file from Stage 1 |
| YAML manifest | Full manifest emitted by tblt-activity-specialist |
| Shared Context Block | Full block from orchestrator — provides `{level}`, `{learner_profile}`, `{task_type}`, `{user_grammar}` |
| Iteration count | Integer: 0 (first run) or 1 (second run) |
| Session key | Date + topic slug, e.g. `2026-05-05_la-rutina-diaria` — used to scope iteration history |
| Exchange file path | Resolved absolute path to `inspector-exchange.md` |

### Step 0 — Constraint Layer (runs before rubric scoring)

Two checks. Failures are included in the feedback report before Layer A.
The specialist must resolve constraint violations before other revisions.

**Linguistic boundaries check.** Verify that:
- Target grammar structures are deployed at complexity appropriate for `{level}`.
  A Novice class should not be required to produce complex embedded clauses;
  an Intermediate class should not be limited to one-word responses.
- The Paso structure matches the declared `{task_type}`. A declared partner
  interview that produces an individual checklist violates this constraint.

If either check fails, state specifically what the mismatch is and what
change would resolve it.

**Pivot classification.** If a surprise element (Paso pivot) is present,
classify it:
- *Soft Pivot* — adds complexity within the existing task frame. Safe at
  any proficiency level.
- *Hard Pivot* — reframes the task entirely mid-activity. Appropriate at
  Intermediate and above only.

If the activity contains a Hard Pivot AND `{level}` is Novice or Novice-Mid,
flag: state that a Hard Pivot at Novice level risks L1 reversion and disrupts
linguistic scaffolding, and recommend converting it to a Soft Pivot with a
specific instruction on how.

### Kill Criteria

Five conditions that generate the highest-priority revision instructions,
regardless of rubric scores. Kill criteria are addressed before any Layer A
or Layer B feedback.

| Criterion | How to check it |
|---|---|
| Task is individually solvable | Could one student complete the full worksheet without a partner? |
| Only yes/no answers required | Does task resolution require no elaboration beyond binary responses? |
| No negotiation required | Does any Paso contain a decision point where partners must align on an answer? |
| Exploit: task completable without target language | Could game mechanics (tallying, checking boxes, pointing) replace Spanish production? |
| Speaking avoidable | Are all Pasos write-only with no moment requiring spoken Spanish? |

Kill criteria do not halt the feedback loop. They produce specific structural
revision instructions. Example format: "Kill criterion — [name]. [Specific
instruction naming the Paso and what to change]."

### Layer A — Lee's Hardware (Pedagogical Validity)

All four criteria must score ≥ 2. Any criterion below 2 generates a required
revision instruction.

**Criterion 1 — Information Gap (0–3)**

| Score | Anchor |
|---|---|
| 0 | No gap. One student could answer all questions alone. |
| 1 | Gap is optional. Students could skip the exchange. |
| 2 | Gap is structurally needed for at least one Paso. |
| 3 | Gap is essential across all major Pasos. No Paso is completable without partner input. |

Observable indicator: Remove one student from the pair mentally. Can the
remaining student complete more than 50% of the activity? If yes → score ≤ 1.

**Criterion 2 — Interaction Dependency (0–3)**

| Score | Anchor |
|---|---|
| 0 | Task individually completable. |
| 1 | Partner adds convenience but is not required. |
| 2 | At least 2 Pasos require partner input to proceed. |
| 3 | Every Paso builds on the partner's previous response. Dependency is cumulative. |

Observable indicator: Identify which Pasos stall if the partner gives no response.
Count them. Score 2 requires at least 2 stalling Pasos; score 3 requires all Pasos.

**Criterion 3 — Linguistic Targeting: Target Structure Deployment (0–3)**

| Score | Anchor |
|---|---|
| 0 | Both target grammar structures absent from student production slots. |
| 1 | Structures appear in instructions or chrome only — not in student output requirements. |
| 2 | At least one structure is required in student production (sentence frames or dialogue). |
| 3 | Both structures required in student production AND appear in Paso 3 live exchange slots. |

Observable indicator: Read every sentence frame and model dialogue exchange in the
HTML. Are both `{user_grammar}` structures present in slots students must fill or
speak? If neither appears in Paso 3 specifically → cannot score 3.

**Criterion 4 — Linguistic Payoff Alignment (0–3)**

| Score | Anchor |
|---|---|
| 0 | Task resolvable without any Spanish. Gestures or pointing suffice. |
| 1 | Target language incidental. Game mechanics could replace Spanish production. |
| 2 | Target language required to exchange the key information that closes the gap. |
| 3 | Without target forms (vocabulary + grammar), the central problem stated in Phase 1.5.1 cannot be resolved. |

Observable indicator: Identify the Paso 5 resolution condition — the moment where
the activity's driving problem is answered. Does it require Spanish production using
target vocabulary and at least one target grammar structure? If the resolution could
happen through tallying numbers or pointing at a board without Spanish → score ≤ 1.

### Layer B — Schell's Software (Engagement Quality)

All four criteria must score ≥ 1. At least one must score 3. Criteria scoring 0
generate required revision instructions. Criteria scoring 1 or 2 generate optional
improvement suggestions placed after required revisions in the report.

**Criterion 5 — Curiosity and Surprise Capacity (0–3)**

| Score | Anchor |
|---|---|
| 0 | All partner responses predictable. Only 1–2 answers exist. |
| 1 | Some unpredictability, but limited to surface variation. |
| 2 | At least one Paso produces genuinely open responses (≥ 4 plausible distinct answers). |
| 3 | Paso 3 contains a prompt where ≥ 5 distinct answers are plausible and a student could genuinely be surprised. |

Observable indicator: List the 3 most likely partner answers to the Paso 3 central
interview question. If those 3 answers cover more than 80% of plausible responses,
the surprise capacity is low → score ≤ 1.

**Criterion 6 — Flow Curve Integrity (0–3)**

| Score | Anchor |
|---|---|
| 0 | Difficulty flat or declining across Pasos. |
| 1 | Difficulty climbs but has at least one jump of more than 2 points between adjacent Pasos. |
| 2 | Difficulty climbs monotonically with at most one small dip allowed. |
| 3 | Smooth climb. Paso 5 is at least 2 difficulty points harder than Paso 1. No cliffs. |

Observable indicator: Map each Paso to a 1–5 cognitive demand scale calibrated to
9th grade. Reference scale: categorization table = 1, list generation = 2, partner
interview = 3, evaluation/rating = 4, class synthesis/argumentation = 5. Check the
sequence for cliffs (jump > 2) or flat lines.

**Criterion 7 — Meaningful Choice Carry-Forward (0–3)**

| Score | Anchor |
|---|---|
| 0 | No consequential student decision exists in any Paso. |
| 1 | A choice exists but does not affect any later Paso. |
| 2 | A choice point exists and a later Paso references it. |
| 3 | The carry-forward is explicit in the instruction text. Students are told "You will use this in Paso N." |

Observable indicator: Find the meaningful-choice Paso (Phase 1.5.3 anchor). Does
the text of the referenced later Paso explicitly name the earlier decision? If the
connection exists only implicitly → score 2 at most.

**Criterion 8 — Ending and Payoff (0–3)**

| Score | Anchor |
|---|---|
| 0 | Activity ends with open sharing or time's up. No designed conclusion. |
| 1 | Paso 5 has a synthesis activity but no reference to earlier Paso outputs. |
| 2 | Paso 5 references at least one earlier Paso output by name. |
| 3 | Paso 5 references at least 2 earlier Paso outputs by name AND the ending type is reveal, verdict, tally, or public commitment. |

Observable indicator: Read Paso 5 instructions only. Count how many earlier Pasos
are named by number or by their output. Identify the ending type from these four:
reveal (pairs share most-surprising finding), verdict (class votes), tally/profile
(shared visual artifact builds during Paso 5), public commitment (each pair states
one action). If Paso 5 describes none of these → score ≤ 1.

### Extended Discourse Check

Three binary checks on Paso 3 and Paso 5. Not scored. Output is YES or NO for each.
NO answers become optional improvement suggestions in the revision list.

| Check | Paso | What to look for in instruction text |
|---|---|---|
| Justification required | Paso 3 or 5 | Does an instruction say "explain why" or "give a reason for your answer"? |
| Disagreement structurally possible | Paso 4 or 5 | Does any instruction say "if your partner disagrees" or create a decision point where partners can hold different positions? |
| Multi-turn response required | Paso 3 | Does the model dialogue show at least 2 exchanges — not one question and one answer? |

### Convergence Rule

The Inspector's verdict is CONVERGED when all of the following are true:
- All Kill Criteria are resolved (all NO)
- All Layer A scores ≥ 2
- All Layer B scores ≥ 1
- At least one Layer B score = 3

Maximum 2 rounds. If iteration count = 1 and the same Layer A criterion that
failed in round 0 is still failing, do not produce a revision list. Instead,
set verdict to ESCALATE and describe the specific failure. The orchestrator
will surface this to the teacher at Gate 2 with three options (revise manually,
proceed and accept the gap, adjust a Shared Context Block input).

### Output Format

The Inspector emits a report in two parts:

**Part 1 — YAML header** (machine-readable; orchestrator extracts verdict from here)

```yaml
inspector_report:
  session_key: "[date]-[topic-slug]"
  round: 0
  constraint_layer:
    linguistic_boundaries: pass | flag
    linguistic_boundaries_detail: "..." | null
    pivot_type: soft | hard | none
    pivot_flag: "..." | null
  kill_criteria:
    individually_solvable: true | false
    yes_no_only: true | false
    no_negotiation: true | false
    exploit_detected: true | false
    speaking_avoidable: true | false
    any_triggered: true | false
  layer_a:
    information_gap: 0-3
    interaction_dependency: 0-3
    linguistic_targeting: 0-3
    linguistic_payoff_alignment: 0-3
    all_pass: true | false
  layer_b:
    curiosity_surprise: 0-3
    flow_curve: 0-3
    meaningful_choice: 0-3
    ending_payoff: 0-3
    all_pass: true | false
  extended_discourse:
    justification_required: true | false
    disagreement_possible: true | false
    multi_turn_paso3: true | false
  convergence:
    verdict: converged | feedback | escalate
    revisions_required: 0-5
    improvements_optional: 0-3
```

**Part 2 — Markdown revision list** (human-readable; specialist consumes this)

```
## INSPECTOR REVISION LIST — Round [N]

### Required Revisions
R1. **[Paso N] — [Criterion name]**
    [Specific actionable instruction. Names the exact element to change
    and what to change it to. One to three sentences.]

R2. **[Paso N] — [Criterion name]**
    [...]

### Optional Improvements
I1. **[Paso N] — [Check name]**
    [Suggestion for improving an element that passed minimum threshold
    but could be stronger.]
```

The YAML header and Markdown revision list are emitted as a single response,
YAML block first, then the Markdown section. The orchestrator reads the YAML
for the verdict field. The specialist reads the Markdown for revision instructions.

### Iteration History — inspector-exchange.md

**Location:** Same folder as the activity log —
`%USERPROFILE%\Documents\spanish-tblt\` (Windows) or
`~/Documents/spanish-tblt/` (Mac/Linux).

**Format:** YAML entries, one per Inspector run, appended chronologically.

**What the Inspector writes after each run:**

```yaml
- session_key: "2026-05-05_la-rutina-diaria"
  round: 0
  failed_layer_a: ["information_gap", "linguistic_targeting"]
  failed_layer_b: []
  kill_criteria_triggered: []
  verdict: feedback
  timestamp: "2026-05-05T14:32:00"
```

**Round 1 read rule:** When `iteration_count = 1`, the Inspector reads
`inspector-exchange.md`, finds the entry matching the current `session_key`
with `round: 0`, and extracts `failed_layer_a`, `failed_layer_b`, and
`kill_criteria_triggered`. If the same criterion appears in the round 1
scoring AND was in the round 0 failed list, the verdict becomes ESCALATE
for that criterion. The Inspector still evaluates all other criteria normally
and provides feedback for those.

**If the file does not exist:** create it on first write. If it cannot be
read on round 1, proceed with round 1 evaluation without the comparison
check and note in the report that iteration history was unavailable.

---

## 7. ORCHESTRATOR CHANGES

### 7.1 Stage 1 invocation modification

The orchestrator currently invokes `tblt-activity-specialist` and instructs it
to return the HTML artifact and run Phase 5a (teacher feedback). Change the
instruction appended to the invocation prompt to suppress Phase 5a and 5b:

Add this instruction to the end of the Stage 1 invocation prompt:

```
ORCHESTRATOR INSTRUCTION — Phase 5a and Phase 5b suppressed on this invocation.
Do not run Phase 5a (do not ask the teacher for feedback ratings).
Do not run Phase 5b (do not write to the session log).
Deliver the HTML artifact and emit the YAML manifest as your final output.
The orchestrator will handle feedback collection and logging after Inspector
evaluation completes.
```

No other change to the Stage 1 invocation prompt.

### 7.2 New Inspector invocation block

After Step 3 in the current Stage 1 section (parse manifest, extract
`{activity_pvs_used}`, `{activity_grammar_deployed}`, `{complication_candidate}`),
and after Step 4 (per-stage integrity checks), insert the following new block
before Gate 2.

Label it: **Inspector Evaluation Loop**

```
Inspector Evaluation Loop:

1. Resolve the exchange file path:
   - If SPANISH_TBLT_LOG_DIR is set, use that directory.
   - Otherwise use %USERPROFILE%\Documents\spanish-tblt\ (Windows) or
     ~/Documents/spanish-tblt/ (Mac/Linux).
   - File: inspector-exchange.md in that folder.

2. Invoke tblt-inspector via the Agent tool. Include in the invocation prompt:
   - The full HTML artifact content from Stage 1
   - The full YAML manifest from Stage 1
   - The full Shared Context Block
   - iteration_count: 0
   - session_key: [YYYY-MM-DD]-[topic-slug]
     where topic-slug = {topic} lowercased with spaces replaced by hyphens
   - exchange_file_path: [resolved absolute path]

3. Parse the returned YAML header. Extract:
   - {inspector_verdict} = inspector_report.convergence.verdict
   - {inspector_scores_a} = layer_a scores (all four criteria)
   - {inspector_scores_b} = layer_b scores (all four criteria)
   - {inspector_kill_status} = kill_criteria.any_triggered
   - {inspector_escalations} = list of criteria where verdict = escalate (if any)

4. Branch on {inspector_verdict}:

   CONVERGED:
   → Proceed to Phase 5a collection (Section 7.3 below).

   FEEDBACK:
   → Invoke tblt-activity-specialist via the Agent tool for a revision pass.
     Include in the invocation prompt:
       - The full Shared Context Block
       - The Inspector's Markdown revision list (Part 2 of the report)
       - This instruction: "This is a revision pass. Proceed to Phase 4.5
         only. Apply all Required Revisions in the revision list. Then
         re-run Phase 3 (internal) and Phase 4 (HTML output). Emit the
         revised YAML manifest. Do not run any other phase."
   → After the specialist returns the revised artifact and manifest:
       - Re-parse the manifest. Update {manifest_stage1} with the revised
         manifest. Re-extract {activity_pvs_used}, {activity_grammar_deployed},
         {complication_candidate} from the revised manifest.
       - Invoke tblt-inspector again with iteration_count: 1 and the same
         session_key. Pass the revised HTML artifact and revised manifest.
   → Parse the returned YAML header again. Extract {inspector_verdict}.
   → Branch:
       CONVERGED: → Proceed to Phase 5a collection (Section 7.3).
       ESCALATE: → Proceed to Phase 5a collection, but also carry
                    {inspector_escalations} to Gate 2 display (Section 7.4).
       FEEDBACK (same criterion failing again): → treat as ESCALATE.

   ESCALATE (from round 0 directly, rare):
   → Proceed to Phase 5a collection. Carry {inspector_escalations} to Gate 2.
```

### 7.3 Phase 5a collection (moved — now runs after Inspector convergence)

After the Inspector loop produces CONVERGED or ESCALATE, the orchestrator
collects teacher feedback using the same prompt format the specialist uses
in its own Phase 5a:

```
How does this activity look?
  Overall quality 1–5 (1 = needs major revision · 5 = ready to print): ___
  Engagement guess 1–5 (1 = students will tune out · 5 = students will lean in): ___
  Short note (optional): ___

Type as: "4 4 good variety but Paso 3 too long"
Type "skip" to log without feedback.
```

Collect and store: `{stage1_quality}`, `{stage1_engagement}`, `{stage1_note}`.
Apply the same input validation rules as the specialist's Phase 5a:
- Accept two numbers 1–5 optionally followed by a note, or "skip"
- On invalid input, re-prompt once
- If second response also invalid, treat as skip

### 7.4 Phase 5b logging (orchestrator handles — specialist no longer writes this)

After collecting Phase 5a feedback, the orchestrator writes the session log
entry directly. Format the row exactly as the specialist's Phase 5b specifies:

```
| [YYYY-MM-DD] | tblt-spanish-activity | [topic] | [learner profile] | [Paso structure] | [vocab count] | [grammar structures] | [quality or blank] | [engagement or blank] | [note or blank] |
```

Use the manifest data for Paso structure, vocab count, and grammar structures.
Use `{stage1_quality}`, `{stage1_engagement}`, `{stage1_note}` for the last
three columns.

Resolve the log file path the same way the specialist does (SPANISH_TBLT_LOG_DIR
or standard folder). Create the file and table header if absent (same schema as
the specialist's Phase 5b). Append the row. Confirm: `✓ Session logged.`

If write fails, surface the manual fallback line as the specialist does.

### 7.5 Gate 2 display changes

The Gate 2 handoff record currently shows:
- PVS carried forward
- Grammar carried forward
- Transfer goal carried
- Activity derivation summary
- Stage 1 quality/engagement rating

Add one new section after the activity derivation summary and before the
quality/engagement rating:

```
Inspector evaluation:
  Layer A (Lee): Gap [score] · Dependency [score] · Targeting [score] · Payoff [score]
  Layer B (Schell): Curiosity [score] · Flow [score] · Choice [score] · Ending [score]
  Rounds: [1 or 2]
  [If ESCALATE: ⚠ Escalated criterion: [name] — [one-line description of failure].
   Options at Gate 2: (A) revise manually before proceeding, (B) proceed and
   accept the known gap, (C) adjust [specific Shared Context Block field].]
```

If no escalation: show scores only, no warning.
If escalation: show scores plus the three-option prompt. Wait for teacher
response before displaying the Gate 2 confirmation line. Store teacher's
escalation decision for the pipeline summary card (Phase 7).

---

## 8. ACTIVITY SPECIALIST CHANGES

### Phase 4.5 — Revision Intake

Add a new phase to `tblt-activity-specialist.md` between Phase 4 (HTML Output)
and Phase 5a (Collect Feedback).

**Phase 4.5 runs only when the specialist is invoked by the orchestrator with
a revision prompt containing an Inspector revision list.** On a normal (first)
invocation, Phase 4.5 is skipped entirely.

Phase 4.5 instruction text (add to the specialist's workflow section):

```
## Phase 4.5 — Inspector Revision Intake (runs only on revision pass)

This phase activates when the orchestrator invokes the specialist with a
revision prompt that contains an Inspector revision list. It does not run
on normal (first) invocations.

When Phase 4.5 is active:

1. Read the Required Revisions list from the Inspector's Markdown revision
   list. Each revision entry specifies a Paso number, a criterion name, and
   a specific instruction.

2. Apply all Required Revisions during Phase 3 internal reconstruction.
   Do not re-run Phase -1 (log fetch), Phase 1 (topic breakdown),
   Phase 1.5 (engagement design pass), Phase 2 (vocabulary distribution),
   or Phase 2.5 (success criteria coverage). These phases ran on the
   initial invocation and their results stand. The vocabulary distribution
   and Spanish audit from Phase 2 are frozen — do not alter item-to-Paso
   assignments unless the revision explicitly requires a vocabulary
   redistribution to fix a kill criterion.

3. Re-run Phase 4 (HTML output) incorporating the Phase 3 revisions.
   Apply the Pre-Output Checklist as normal.

4. Emit the revised YAML manifest as the final output.

5. Do not run Phase 5a or Phase 5b on a revision pass. These are handled
   by the orchestrator after Inspector evaluation.

If a Required Revision conflicts with a Phase 2 vocabulary distribution
decision (e.g., the revision requires adding an item to Paso 1 that Phase 2
assigned to Paso 3), prefer the revision and note the conflict in the manifest
flags field.
```

### Workflow overview update

In the specialist's workflow overview table, add Phase 4.5 as a conditional row:

```
Phase 4.5 → Inspector revision intake    (conditional — only on revision pass)
```

Place it between Phase 4 and Phase 5a in the workflow table.

---

## 9. LOG ARCHITECTURE

Two files. No overlap.

### spanish-activity-log.md (unchanged)

- **Owner:** Activity specialist (writes), orchestrator (writes after this
  implementation, since Phase 5b moves to orchestrator)
- **Purpose:** Pedagogical record of all sessions — topic, Paso structure,
  vocabulary count, grammar structures, teacher quality rating, teacher
  engagement rating, teacher note
- **Read by:** Activity specialist Phase -1 (anti-repetition and low-rating
  signals), orchestrator Phase -2 (class profile)
- **Location:** `%USERPROFILE%\Documents\spanish-tblt\spanish-activity-log.md`
- **Schema:** Unchanged from current implementation
- **The Inspector never reads or writes this file**

### inspector-exchange.md (new)

- **Owner:** Inspector (writes), Inspector (reads on round 1)
- **Purpose:** Iteration history for the current and past Inspector runs —
  which criteria failed each round, what verdict was reached
- **Read by:** Inspector on round 1 to check for repeated failures
- **Location:** Same folder as the activity log —
  `%USERPROFILE%\Documents\spanish-tblt\inspector-exchange.md`
- **Format:** YAML list, entries appended chronologically, scoped by session_key
- **The teacher never sees this file**
- **Not deleted between sessions** — entries accumulate. The Inspector
  reads only the entry matching the current session_key + round 0.

---

## 10. WHAT DOES NOT CHANGE

These files and behaviors are untouched by this implementation:

1. `tblt-pretask-specialist.md` — no changes
2. `tblt-reflective-specialist.md` — no changes
3. `docs/MANIFEST_SCHEMA.md` — no new fields (the Inspector reads existing
   manifest fields; it does not add new ones)
4. `docs/ARCHITECTURE.md` — update the pipeline diagram and add the Inspector
   to the agent table, but do not change any architectural principle
5. All specialist internal phases (-1 through 5b) except:
   - Activity specialist Phase 5a/5b (now suppressed on orchestrator invocation;
     orchestrator handles both)
   - Activity specialist (new Phase 4.5 added — conditional only)
6. The PVS Integrity Rule — the Inspector never modifies vocabulary
7. The three-gate confirmation structure — gates 1, 2, and 3 still exist;
   Gate 2 content is extended (Inspector scores + optional escalation) but
   its confirmation requirement is unchanged
8. The class profile read/write logic (Phase -2 and Step 1d)
9. Standalone invocation behavior of any specialist — when invoked without
   the orchestrator, Phase 4.5 does not activate and Phase 5a/5b run normally
10. The Activity Derivation Block mechanism — `actual_pvs_items_used` and
    `actual_grammar_deployed` continue to flow to the pre-task and reflective
    specialists unchanged. If the Inspector triggers a revision pass, the
    orchestrator re-extracts these fields from the revised manifest before
    Gate 2.

---

## 11. OUT OF SCOPE (future work)

Three backward design strengthening options were identified in the design session
but are not implemented here:

- **Option 1 — Pre-flight gate:** Orchestrator checks transfer goal and success
  criteria coherence before Gate 1 (before any specialist runs)
- **Option 2 — Inspector BD layer:** Inspector checks success criteria coverage,
  transfer goal echo, and derivation record richness from the manifest — EXCLUDED
  because the Inspector's function is Lee + Schell evaluation only
- **Option 3 — Phase 7 cross-stage audit:** Orchestrator verifies backward design
  chain across all three manifests at Pipeline Summary Card time

These are independent additions that do not conflict with the Inspector. They
can be implemented in a separate session using the three options described in
the design conversation.

---

## 12. IMPLEMENTATION SEQUENCE

Work in this exact order:

1. Read all six files listed in the First Prompt (Section 0) before writing
   anything.

2. Create `.claude/agents/tblt-inspector.md` — new file, full agent specification
   from Section 6 of this brief. This file has no dependencies on the other edits.

3. Edit `.claude/agents/tblt-activity-specialist.md`:
   - Add Phase 4.5 to the workflow overview table (one row, conditional)
   - Add the Phase 4.5 instruction block between Phase 4 and Phase 5a
   - Add an orchestrator-mode note to Phase 5a: "When invoked via orchestrator,
     Phase 5a is suppressed. The orchestrator collects feedback after Inspector
     evaluation."
   - Add an orchestrator-mode note to Phase 5b: "When invoked via orchestrator,
     Phase 5b is suppressed. The orchestrator writes the log entry directly."

4. Edit `.claude/agents/tblt-orchestrator.md`:
   - In Stage 1 invocation prompt, add the Phase 5a/5b suppression instruction
     (Section 7.1)
   - After Step 4 of Stage 1 (per-stage integrity checks), insert the Inspector
     Evaluation Loop block (Section 7.2)
   - After the Inspector loop, insert the Phase 5a collection block (Section 7.3)
   - After Phase 5a collection, insert the Phase 5b logging block (Section 7.4)
   - In Gate 2 handoff record, add the Inspector evaluation section (Section 7.5)

5. Edit `docs/ARCHITECTURE.md`:
   - Add `tblt-inspector` to the agent table with its role
   - Update the pipeline diagram to show the Inspector between Stage 1 and Gate 2
   - Add `inspector-exchange.md` to the file table with its purpose and owner

6. Verify consistency (do not skip):
   - The orchestrator's Stage 1 section invokes the activity specialist with the
     Phase 5a/5b suppression instruction
   - The orchestrator's Inspector loop invokes `tblt-inspector` via the Agent tool
   - Phase 4.5 in the specialist is marked conditional (revision pass only)
   - Phase 5a and 5b in the specialist have orchestrator-mode suppression notes
   - The Gate 2 handoff record includes the Inspector evaluation section
   - The Inspector agent file includes all eight rubric criteria with 0–3 anchors
     and observable indicators
   - The Inspector's output format specifies YAML header first, then Markdown
     revision list
   - The convergence rule (max 2 rounds, same criterion = escalate) is in the
     Inspector's system prompt
   - The iteration history write/read protocol references `inspector-exchange.md`
     by the resolved path passed from the orchestrator

---

## 13. EDGE CASES AND DECISIONS ALREADY MADE

**Q: What if the Inspector cannot read inspector-exchange.md on round 1?**
A: Proceed with round 1 evaluation without the comparison check. Note in the
report that iteration history was unavailable. Do not fail or halt.

**Q: What if the revised manifest has different actual_pvs_items_used than the
original?**
A: Re-extract and update {activity_pvs_used} and {activity_grammar_deployed}
from the revised manifest. These updated values flow to Gate 2 and to the
pre-task and reflective specialists. The Activity Derivation Block that
downstream specialists receive reflects the approved revised activity.

**Q: Does the Inspector run when the activity specialist is invoked standalone
(without the orchestrator)?**
A: No. The Inspector is invoked only by the orchestrator. Standalone specialist
invocations run Phase 5a and 5b normally and do not interact with the Inspector.

**Q: Can the Inspector's revision instructions ask the specialist to change
vocabulary distribution?**
A: Only if a Kill Criterion requires it. Normal rubric feedback should work
within the existing Phase 2 vocabulary distribution. The PVS Integrity Rule
is not suspended for Inspector revisions.

**Q: What does the manifest_stage1 variable contain after a revision pass?**
A: The revised manifest — the one from the second specialist invocation, not
the first. The orchestrator updates {manifest_stage1} with the revised manifest
and uses it for all downstream operations including the Activity Derivation Block
passed to Stage 2 and Stage 3.

**Q: What happens to the teacher feedback rating if the activity was revised?**
A: The teacher rates the final approved artifact (the one that passed the Inspector).
Phase 5a runs after Inspector convergence, so the teacher never rates a draft
that was subsequently revised. This is the correct behavior.

**Q: Does the manifest stage: number change?**
A: No. The activity specialist always emits stage: 2. This is a specialist
identifier, not an execution-order position. The backward design brief already
established this convention.
