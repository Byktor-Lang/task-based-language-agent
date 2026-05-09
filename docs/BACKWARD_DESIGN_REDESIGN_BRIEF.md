# BACKWARD DESIGN REDESIGN BRIEF
# For: Future Claude session implementing the new pipeline execution order
# Project root: C:\Users\azuaje.MSMC\Documents\spanish-tblt\tblt-v2\

---

## 1. WHAT THIS DOCUMENT IS

This brief captures a design decision made in a prior session and gives you everything
you need to implement it without re-reading that conversation. Read this entire document
before touching any file. The changes are bounded and surgical — do not refactor anything
beyond what is explicitly described here.

---

## 2. PROJECT CONTEXT (read this first)

### What exists

Four Claude Code subagents in `.claude/agents/`:

| File | Agent name | Role |
|---|---|---|
| `tblt-orchestrator.md` | `tblt-orchestrator` | Supervisor: collects inputs, runs gates, invokes specialists |
| `tblt-pretask-specialist.md` | `tblt-pretask-specialist` | Stage 1: pre-task vocabulary worksheet |
| `tblt-activity-specialist.md` | `tblt-activity-specialist` | Stage 2: main TBLT activity worksheet |
| `tblt-reflective-specialist.md` | `tblt-reflective-specialist` | Stage 3: post-task reflective loop worksheet |

Supporting docs:
- `docs/MANIFEST_SCHEMA.md` — YAML manifest contract between specialists and orchestrator
- `docs/ARCHITECTURE.md` — current design rationale
- `docs/MIGRATION_NOTES.md` — source-to-agent mapping and log schema
- `tests/smoke-test-prompts.md` — 5 end-to-end test prompts

### Current execution order (what you are CHANGING)

```
Gate 1 → pretask-specialist → Gate 2 → activity-specialist → Gate 3 → reflective-specialist
```

All three specialists receive the same Shared Context Block (general teacher inputs only).
The pre-task specialist has no knowledge of what the activity will actually contain.
The reflective specialist has no knowledge of what the activity actually contained.

### What MUST NOT change

- The source skills in `C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-task-based\`
  are read-only. Never touch them.
- Every specialist's internal phases (Phase -1 through Phase 5b, manifest emission) stay
  exactly as written. You are not rewriting specialist pedagogy — only changing what
  information they receive and in what order they are called.
- The PVS Integrity Rule: the numbered PVS (#01–#N) is still frozen at Phase 1 and never
  modified between stages.
- The confirmation gate structure: three gates still exist. Their content and position
  in the sequence change (see Section 4).

---

## 3. THE USER'S BACKWARD DESIGN MODEL (the requirement)

### Core principle

The main TBLT activity is the design anchor. Everything is derived from it:
- The pre-task prepares students for what the activity **actually contains**.
- The reflective loop consolidates what the activity **actually practiced**.

### Input layers

Two layers of information flow to both the pre-task specialist and the reflective specialist:

**Layer 1 — General inputs (from the teacher, assembled by orchestrator in Phase 0/1)**
- Transfer goal
- Success criteria
- Grammar structures (teacher-provided)
- Full PVS (teacher-provided, frozen)
- Topic, learner profile, ACTFL level, task type, target register, main task description

**Layer 2 — Activity derivation record (from the activity specialist's manifest)**
- `actual_pvs_items_used`: the specific PVS items that appear in the activity as built
  (not just a boolean coverage flag — a list of which items)
- `actual_grammar_deployed`: the grammar structures as they were actually deployed in
  the activity text (e.g., not just "present tense reflexive verbs" but how that structure
  appears in the specific Paso instructions and sentence frames)

Both layers are passed to BOTH the pre-task specialist AND the reflective specialist.
Neither specialist loses the general context. The activity derivation record is an
additional input, not a replacement.

### Why both layers matter

The general inputs ground the pedagogy and ensure coherence with the teacher's intent.
The activity derivation record grounds the content in the actual artifact students will use.
The pre-task specialist uses the derivation record to prioritize which vocabulary and grammar
constructions to introduce (the ones students are about to encounter). The reflective
specialist uses it to anchor the writing prompt and phrase pairs to what students specifically
practiced, not just what the topic generally involves.

### New execution order

```
Gate 1 → activity-specialist → Gate 2 → pretask-specialist → Gate 3 → reflective-specialist
```

The complication/outcome resolution (currently part of Gate 3) still happens just before
the reflective specialist is invoked — it just now sits at Gate 3, after the pre-task
specialist completes. The complication_candidate from the activity manifest is held by
the orchestrator from Gate 2 forward and used at Gate 3.

---

## 4. EXACT CHANGES REQUIRED

### 4.1 — tblt-activity-specialist.md

**Location:** `.claude/agents/tblt-activity-specialist.md`

**Change:** Extend the manifest to include two new fields in the `Manifest Emission` section.

Current manifest block (near bottom of file):

```yaml manifest
stage: 2
specialist: tblt-activity-specialist
...
complication_candidate: "..." | null
flags: []
```

Add after `complication_candidate`:

```yaml
actual_pvs_items_used: ["#01", "#03", "#05", ...]
  # List only the PVS items that ACTUALLY APPEAR in the Paso instructions, sentence frames,
  # model dialogue, or vocabulary tables of the generated HTML artifact.
  # This is a subset of pvs_coverage.items_used — items_used tracks full coverage for
  # integrity checks; actual_pvs_items_used tracks which items have a VISIBLE PRESENCE
  # in the activity text itself (i.e., a student doing the activity will encounter them).
  # If all items appear visibly, this list equals items_used.

actual_grammar_deployed: ["...", "..."]
  # For each grammar structure in {user_grammar}, describe specifically how it appears
  # in the activity text. Use concrete language:
  #   - NOT: "present tense reflexive verbs"
  #   - YES: "present tense reflexive verbs in first-person sentence frames (e.g., 'Yo me ducho...')"
  #   - NOT: "frequency expressions"
  #   - YES: "frequency expressions (nunca, a veces, siempre) as adverbs in Paso 3 interview prompts"
  # If a grammar structure is absent from the activity (grammar_coverage: absent), describe
  # the absence: "not deployed — absent from all Pasos".
```

Also update the prose description in the `Manifest Emission` section to tell the specialist
to populate these fields based on its Phase 4 HTML output.

**What does NOT change in this file:** Everything else. All phases, all internal reasoning,
all halt conditions, the HTML rules, Phase 5a, Phase 5b — untouched.

---

### 4.2 — tblt-orchestrator.md

**Location:** `.claude/agents/tblt-orchestrator.md`

This is the largest change. Work through it section by section.

#### 4.2.1 — Update the Pipeline Architecture Map

Replace the current ASCII map:

```
STAGE 1  — Pre-Task Vocabulary Introduction    [tblt-pretask-specialist]
GATE 2   — Handoff: Stage 1 → Stage 2          [ORCHESTRATOR — GATE 2]
STAGE 2  — Main TBLT Activity                  [tblt-activity-specialist]
GATE 3   — Complication Resolution + Handoff   [ORCHESTRATOR — GATE 3]
STAGE 3  — Reflective Loop                     [tblt-reflective-specialist]
```

With:

```
STAGE 1  — Main TBLT Activity                  [tblt-activity-specialist]
GATE 2   — Handoff: Stage 1 → Stage 2          [ORCHESTRATOR — GATE 2]
STAGE 2  — Pre-Task Vocabulary Introduction    [tblt-pretask-specialist]
GATE 3   — Complication Resolution + Handoff   [ORCHESTRATOR — GATE 3]
STAGE 3  — Reflective Loop                     [tblt-reflective-specialist]
```

Update the role/responsibility table at the top (Stage 1 / Stage 2 labels) to match.

#### 4.2.2 — Rename Stage sections

The current file has sections: `## Stage 1`, `## Gate 2`, `## Stage 2`, `## Gate 3`,
`## Stage 3`. Rename them to reflect the new order:

- Old `## Stage 1 — Pre-Task Vocabulary Introduction` → `## Stage 1 — Main TBLT Activity`
- Old `## Stage 2 — Main TBLT Activity` → `## Stage 2 — Pre-Task Vocabulary Introduction`
- Gate 2 and Gate 3 headings stay the same; their content changes (see below).

#### 4.2.3 — Stage 1 invocation (was Stage 2 / activity-specialist)

Move the activity-specialist invocation to Stage 1. The invocation prompt it receives
is the same as the current Stage 2 invocation — just the Shared Context Block with
activity-specialist variable names. No Activity Derivation Block at this point (it hasn't
been built yet).

After the activity-specialist returns:
1. Parse the manifest. Store as `{manifest_stage1}` (was `{manifest_stage2}`).
2. Extract and store the two new fields:
   - `{activity_pvs_used}` = `manifest_stage1.actual_pvs_items_used`
   - `{activity_grammar_deployed}` = `manifest_stage1.actual_grammar_deployed`
3. Also extract `{complication_candidate}` from the manifest (same as before — hold it
   for Gate 3).
4. Run per-stage integrity checks (same checks, same flag format as before).
5. Collect teacher feedback (quality + engagement + note). Store as `{stage1_quality}`,
   `{stage1_engagement}`, `{stage1_note}`.
6. Capture log-write outcome as `{stage1_log}`.

#### 4.2.4 — Gate 2 content (after activity-specialist, before pre-task-specialist)

Gate 2 handoff record shows:
- PVS carried forward (all N items unchanged)
- Grammar carried forward
- Transfer goal carried
- Success criteria carried
- **NEW:** Activity derivation summary — show the teacher which PVS items appeared in
  the activity and how each grammar structure was deployed. Use this format:

```
Activity derivation (passed to pre-task specialist):
  PVS items present in activity: [list from {activity_pvs_used}]
  Grammar deployed:
    {grammar[0]}: [{activity_grammar_deployed[0]}]
    {grammar[1]}: [{activity_grammar_deployed[1]}]
```

- Stage 1 quality/engagement rating

Gate 2 confirmation line: "Confirm to begin Stage 2 (Pre-Task Vocabulary Introduction)."

#### 4.2.5 — Stage 2 invocation (was Stage 1 / pretask-specialist)

Move the pretask-specialist invocation to Stage 2. The invocation prompt now contains
TWO blocks:

**Block 1 — Shared Context Block (unchanged from current design):**
Same as current Stage 1 invocation: full Shared Context Block with pretask-specialist
variable names (`{topic}`, `{vocabulary}` = full PVS, `{grammar}`, `{level}`, etc.).

**Block 2 — Activity Derivation Block (NEW):**

```
ACTIVITY DERIVATION BLOCK — from Stage 1 (tblt-activity-specialist)
════════════════════════════════════════════════════════════════════
Purpose: These are the vocabulary items and grammar structures that
ACTUALLY APPEAR in the main TBLT activity your students will do after
this pre-task. Design your exercises to introduce and prime these
specifically — they are a subset of the full PVS above.

PVS items that appear in the activity:
  {activity_pvs_used}  ← list of PVS item numbers, e.g. ["#01","#03","#07"]

How each grammar structure is deployed in the activity:
  {grammar[0]}: {activity_grammar_deployed[0]}
  {grammar[1]}: {activity_grammar_deployed[1]}

Instruction to specialist:
- Treat the items in this block as your PRIMARY vocabulary design targets.
  Prioritize them in exercise selection and item placement.
- PVS items NOT in this list should still appear in at least one exercise
  (full PVS coverage is required) but may be treated as secondary.
- Use the grammar deployment descriptions above to select exercise types
  and sentence frames that prime the specific constructions students will
  encounter, not just the abstract grammar label.
════════════════════════════════════════════════════════════════════
```

After the pretask-specialist returns:
1. Parse the manifest. Store as `{manifest_stage2}`.
2. Run per-stage integrity checks (same as before).
3. Collect teacher feedback (rating 1–5 + note). Store as `{stage2_rating}`,
   `{stage2_note}`.
4. Capture log-write outcome as `{stage2_log}`.

#### 4.2.6 — Gate 3 content (after pre-task-specialist, before reflective-specialist)

Gate 3 still handles complication/outcome resolution. The `{complication_candidate}` was
extracted from the Stage 1 (activity) manifest and held since Gate 2. Use it here exactly
as the current Gate 3 Branch B logic works — nothing changes about the complication
resolution logic itself.

Gate 3 handoff record shows:
- PVS carried forward
- Grammar carried forward
- Complication (final)
- Outcome (final)
- Stage 2 (pre-task) rating

Gate 3 confirmation line: "Confirm to begin Stage 3 (Reflective Loop)."

#### 4.2.7 — Stage 3 invocation (reflective-specialist — unchanged role)

The reflective-specialist invocation now also passes the Activity Derivation Block.
Use the same block format as defined in 4.2.5, with this instruction variant:

```
ACTIVITY DERIVATION BLOCK — from Stage 1 (tblt-activity-specialist)
════════════════════════════════════════════════════════════════════
Purpose: These are the vocabulary items and grammar structures that
ACTUALLY APPEARED in the main TBLT activity students completed.
Your consolidation activity should verify and extend the specific
skills students practiced in that activity.

PVS items that appeared in the activity:
  {activity_pvs_used}

How each grammar structure was deployed in the activity:
  {grammar[0]}: {activity_grammar_deployed[0]}
  {grammar[1]}: {activity_grammar_deployed[1]}

Instruction to specialist:
- Anchor the writing prompt, scenario, and phrase pairs to these
  specific vocabulary items and grammar constructions.
- The general PVS and grammar structures (in the Shared Context Block
  above) remain present as the broader instructional frame.
- The activity derivation record tells you what students ACTUALLY DID —
  use it to make the reflective loop feel continuous with the activity,
  not like a separate generic writing task.
════════════════════════════════════════════════════════════════════
```

The rest of the Stage 3 invocation prompt is identical to the current version (complication,
outcome, all variable names for reflective-specialist).

#### 4.2.8 — Phase 7 Pipeline Summary Card

Update the card to reflect the new stage order:

- Document 1 is now the Main TBLT Activity (was Document 2)
- Document 2 is now the Pre-Task Vocabulary Introduction (was Document 1)
- Document 3 remains the Reflective Loop

Update label text accordingly. The data fields pulled from manifests follow the
renaming: `{manifest_stage1}` = activity manifest, `{manifest_stage2}` = pretask manifest.

Update log write line: `{stage1_log}` = activity log · `{stage2_log}` = pre-task log ·
Stage 3 (no log).

#### 4.2.9 — Cross-stage integrity check update

The cross-stage PVS coverage check currently reads:
"Every PVS item appears in items_used for Stage 1 OR Stage 2 (or both)."

After the rename, Stage 1 = activity-specialist and Stage 2 = pretask-specialist.
The check logic is identical — just confirm the variable names in the implementation
point to the right manifests.

#### 4.2.10 — Confirmation Gate Rule (Critical Rule #3)

Update the rule text to reflect the new gate assignments:
1. After the Shared Context Block (Step 1d) — before activity-specialist runs
2. After Stage 1 (activity) completes (Gate 2) — before invoking pre-task specialist
3. After complication and outcome are confirmed (Gate 3) — before invoking reflective
   specialist

---

### 4.3 — tblt-pretask-specialist.md

**Location:** `.claude/agents/tblt-pretask-specialist.md`

**Change:** Add the Activity Derivation Block to the Inputs section and update Phase 1
to use it.

In the `## Inputs` section, add after the existing inputs table:

```
### Activity Derivation Block (when invoked via orchestrator)

When invoked by `tblt-orchestrator`, the invocation prompt also contains an
Activity Derivation Block with two fields:

| Variable | Description |
|---|---|
| `{activity_pvs_items_used}` | List of PVS item numbers (#NN) that appear in the main TBLT activity as built |
| `{activity_grammar_deployed}` | How each grammar structure is specifically deployed in the activity text |

**Design priority rule:** Treat the items in `{activity_pvs_items_used}` as your
PRIMARY vocabulary targets. Select exercise types and item placements that introduce
these items first. All other PVS items (those not in the derivation list) still require
full coverage in at least one exercise, but treat them as secondary in sequencing decisions.

Use `{activity_grammar_deployed}` in Phase 3 exercise planning: select sentence frames
and model sentences that prime the specific constructions students will encounter, not
just the abstract grammar label.

**When invoked standalone** (without orchestrator): this block is absent. Apply standard
Phase 1 classification and exercise selection using only the PVS and grammar structures
provided.
```

In the `## Phase 1` section, add to Step 1c (exercise type selection) a prioritization note:

```
If the Activity Derivation Block is present, cross-reference the {activity_pvs_items_used}
list when committing to the Phase 1b distribution. Items in the derivation list should
appear in Paso 1 unless the cognitive load ceiling (10 items) requires deferring some.
Items NOT in the derivation list are still required to appear in at least one exercise
but may be distributed to Paso 2 or later.
```

**What does NOT change in this file:** All phases, all internal reasoning, all exercise
types, all HTML rules, all output delivery sequence. The Activity Derivation Block is
additional context, not a structural change to the workflow.

---

### 4.4 — tblt-reflective-specialist.md

**Location:** `.claude/agents/tblt-reflective-specialist.md`

**Change:** Add the Activity Derivation Block to the Inputs section and reference it
in Phase 1 (scenario anchor and phrase-pair selection).

In the `## Inputs` section, add after the existing inputs table:

```
### Activity Derivation Block (when invoked via orchestrator)

When invoked by `tblt-orchestrator`, the invocation prompt also contains an
Activity Derivation Block with two fields:

| Variable | Description |
|---|---|
| `{activity_pvs_items_used}` | List of PVS item numbers (#NN) that appeared in the main TBLT activity |
| `{activity_grammar_deployed}` | How each grammar structure was specifically deployed in the activity text |

**Design priority rule:** Anchor the writing prompt scenario, the phrase-pair selection
(Phase 1d), and the self-correction checklist to the vocabulary and grammar constructions
in this block. The consolidation activity should feel continuous with the main TBLT
activity — students should recognize the vocabulary and structures, not encounter them
for the first time.

The general PVS and grammar structures (in the Shared Context Block) remain the broader
instructional frame. The Activity Derivation Block tells you what students ACTUALLY DID.

**When invoked standalone** (without orchestrator): this block is absent. Apply standard
Phase 1 design using only the PVS and grammar structures provided.
```

In Phase 1c (scenario anchor) and Phase 1d (phrase-pair selection), add a note:

```
If the Activity Derivation Block is present, prefer scenario elements and phrase pairs
that draw from {activity_pvs_items_used} and mirror the constructions described in
{activity_grammar_deployed}. This makes the scenario feel like a direct continuation
of the activity students just completed.
```

**What does NOT change in this file:** All phases, all HTML rules, all constraints
(C01–C07), all checklist positions, Phase 0 validation, the register-shift table logic.

---

### 4.5 — docs/MANIFEST_SCHEMA.md

**Location:** `docs/MANIFEST_SCHEMA.md`

**Change:** Add the two new fields to the Stage 2 section (which is now the activity
specialist, still labeled Stage 2 in manifest terms — the stage number in the manifest
corresponds to the specialist, not the execution order position).

Wait — important clarification: the manifest `stage:` field is **specialist-specific**,
not execution-order-specific. The activity-specialist always emits `stage: 2` regardless
of whether it runs first or second in the pipeline. Do NOT renumber the stage fields.

In the `## Stage-Specific Extensions` section under `### Stage 2 Only — Complication Candidate`,
add the two new fields:

```yaml
actual_pvs_items_used: ["#01", "#03", "#05", ...]
  # PVS items with a visible presence in the activity HTML (instructions, frames,
  # dialogue, tables). Subset of or equal to pvs_coverage.items_used.
  # Passed by orchestrator to pre-task and reflective specialists as derivation input.

actual_grammar_deployed: ["...", "..."]
  # One string per grammar structure in {user_grammar}, describing the specific
  # constructions as they appear in the activity text. Used by pre-task specialist
  # for exercise frame design and by reflective specialist for phrase-pair selection.
  # Format: "[grammar label] as [specific deployment description]"
  # Example: "present tense reflexive verbs in first-person sentence frames (Yo me ducho...)"
```

Update the Per-Stage Summary table to show these fields for Stage 2.

---

### 4.6 — docs/ARCHITECTURE.md

**Location:** `docs/ARCHITECTURE.md`

**Change:** Update the execution order section and the Line of Responsibility table.

In the execution order: note that activity-specialist runs first (Stage 1 position in
execution order, Stage 2 specialist identity), pre-task specialist runs second, reflective
runs third.

In the agent roles section for `tblt-orchestrator`, update Stage delegation to describe
the new order.

Add to the Line of Responsibility table:

| Activity derivation record (actual_pvs_items_used, actual_grammar_deployed) | `tblt-activity-specialist` (emit) + `tblt-orchestrator` (hold and route) |

---

## 5. WHAT IS NOT CHANGING — DO NOT TOUCH

These items must remain exactly as they are:

1. **Phase 0 of the orchestrator** — all of 0a through 0h. Input elicitation is unchanged.
   The teacher still provides vocabulary, grammar, goals, and all other inputs at the start.

2. **Phase 1 of the orchestrator** — PVS lock, Shared Context Block assembly, Gate 1.
   The PVS is still frozen before any specialist runs.

3. **All specialist internal phases** — every phase from Phase -1 through manifest emission
   in all three specialists. The only additions are: new fields in the activity-specialist
   manifest, and new input blocks in the pretask and reflective specialists' Inputs sections.

4. **The Vocabulary Fence rule** — still enforced in all specialists.

5. **The gate structure** — three gates still exist. Their content changes slightly
   (Gate 2 now shows derivation summary; Gate 3 still handles complication resolution);
   the count and confirmation requirement do not change.

6. **The manifest stage: numbers** — activity-specialist always emits `stage: 2`,
   pretask-specialist always emits `stage: 1`, reflective-specialist always emits `stage: 3`.
   These are specialist identifiers, not execution-order positions.

7. **The session log schema** — both log tables and their write rules are unchanged.

8. **The class-profile.md read/write logic** — Phase -2 and Step 1d write-back are unchanged.

9. **Standalone invocation behavior** — when any specialist is invoked directly without
   the orchestrator, it behaves exactly as before. The Activity Derivation Block is simply
   absent and the specialist falls back to its standard exercise selection logic.

---

## 6. IMPLEMENTATION SEQUENCE

Work in this order to avoid writing something that depends on content you haven't read yet:

1. Read all four agent files and both MANIFEST_SCHEMA.md and ARCHITECTURE.md in full.
2. Edit `tblt-activity-specialist.md` — add the two new manifest fields and their
   population instructions.
3. Edit `docs/MANIFEST_SCHEMA.md` — document the two new fields in the Stage 2 section.
4. Edit `tblt-orchestrator.md` — this is the largest change; work through sections
   4.2.1 through 4.2.10 in order.
5. Edit `tblt-pretask-specialist.md` — add Activity Derivation Block to Inputs and
   Phase 1 prioritization note.
6. Edit `tblt-reflective-specialist.md` — add Activity Derivation Block to Inputs and
   Phase 1 anchoring note.
7. Edit `docs/ARCHITECTURE.md` — update execution order and Line of Responsibility.
8. Update `tests/smoke-test-prompts.md` — revise Test 1's expected behavior to show
   the new execution order (activity first, then pre-task, then reflective) and the
   Gate 2 derivation summary display.

After editing, verify these things are consistent across all files:
- The orchestrator's Stage 1 section invokes activity-specialist.
- The orchestrator's Stage 2 section invokes pretask-specialist with both blocks.
- The orchestrator's Stage 3 section invokes reflective-specialist with both blocks.
- The activity-specialist manifest template includes `actual_pvs_items_used` and
  `actual_grammar_deployed`.
- MANIFEST_SCHEMA.md documents both new fields.
- The pretask-specialist Inputs section describes the Activity Derivation Block.
- The reflective-specialist Inputs section describes the Activity Derivation Block.

---

## 7. EDGE CASES AND DECISIONS ALREADY MADE

**Q: Does the stage: number in the manifest change since execution order changed?**
A: No. stage: 1 = pretask-specialist, stage: 2 = activity-specialist, stage: 3 =
reflective-specialist. These are fixed identifiers, not execution-order positions.

**Q: Does the cross-stage PVS coverage check change?**
A: The logic is identical. The check is still "every PVS item appears in items_used for
the pretask manifest OR the activity manifest (or both)." Only the variable names in the
orchestrator's implementation need to point to the correctly renamed manifests.

**Q: What if the activity-specialist marks a PVS item as covered (items_used) but it
doesn't appear in actual_pvs_items_used?**
A: This is valid. items_used = appeared anywhere in the artifact (including answer key,
teacher notes section, etc.). actual_pvs_items_used = visible to a student doing the
activity. The fields serve different purposes. No integrity conflict.

**Q: What if actual_pvs_items_used is empty or null (specialist omits it)?**
A: Treat it as: all items in pvs_coverage.items_used are also in actual_pvs_items_used.
The orchestrator should handle a missing field gracefully — pass the full items_used
list to downstream specialists and note the field was not populated.

**Q: Does standalone pre-task invocation change?**
A: No. When invoked without the orchestrator, the pre-task specialist has no Activity
Derivation Block. It uses standard Phase 1 logic. The new Inputs section language
makes this conditional: "When invoked standalone (without orchestrator): this block
is absent."

**Q: Does the complication/outcome resolution move?**
A: It stays at Gate 3 (before the reflective specialist). The complication_candidate
is extracted from the activity manifest at Stage 1 and held by the orchestrator through
Gate 2 and Stage 2. At Gate 3, the orchestrator uses it exactly as the current Gate 3
Branch B logic works.

**Q: Can the pretask-specialist request a PVS expansion if it detects a gap in the
derivation block?**
A: No. The PVS is frozen at Phase 1. The Orchestrator-mode note in Phase 2 Step B of
the activity-specialist already handles this: APPROVE is rerouted as a Shared-Context-Block
PVS expansion request to the orchestrator. Nothing new needed here.

---

## 8. SUMMARY OF THE DESIGN IN ONE PARAGRAPH

The teacher provides goals, grammar, and vocabulary. The orchestrator collects these,
freezes the PVS, assembles the Shared Context Block, and at Gate 1 invokes the
activity-specialist FIRST. The activity-specialist builds the main communicative task
and returns its manifest with two new fields: the PVS items that visibly appear in the
activity and the specific grammar constructions as deployed. The orchestrator holds this
derivation record. At Gate 2, it invokes the pre-task specialist with BOTH the original
Shared Context Block AND the Activity Derivation Block, so the pre-task exercises
introduce exactly the vocabulary and grammar students are about to encounter. At Gate 3,
after complication/outcome are confirmed, the orchestrator invokes the reflective
specialist with BOTH the Shared Context Block AND the Activity Derivation Block, so
the consolidation activity is anchored to what students actually practiced, not just
the general topic. The reflective specialist runs last and closes the instructional loop.
