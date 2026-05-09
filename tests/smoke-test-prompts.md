# Smoke Test Prompts

Paste any of these into Claude Code to verify the system works end-to-end.
Each prompt is followed by its expected behavior so you can spot regressions.

---

## Test 1 — Full Pipeline, All Inputs in One Message (Fast-Track)

**Prompt:**

```
Build me a complete three-part TBLT lesson on Las tareas del hogar.

Topic: Las tareas del hogar
Vocabulary: barrer, pasar la aspiradora, lavar los platos, limpiar el baño,
sacudir los muebles, quitar el polvo, sacar la basura, fregar el suelo,
ordenar los cuartos, hacer la cama
Grammar: present tense reflexive verbs; frequency expressions (nunca, a veces,
siempre, todos los días)
ACTFL level: Novice
Learner profile: 9th grade; interest hooks: TikTok trends, school sports, group chats
Task type: Partner interview — each student asks the other how often they do household chores
Complication: one partner has to do all chores because their roommate is away
Target register: Informal text reply to a friend (audience: a peer)
Transfer goal: Students will be able to discuss household responsibilities with a partner in order to negotiate fair division of chores in a real shared living situation.
Success criteria:
  1. Sustains the interview in Spanish for the full task without reverting to English
  2. Uses frequency expressions to make specific claims (not just "a veces")
  3. Asks at least one clarifying question when a partner's response is unclear
  4. Negotiates a fair chore division given the complication
```

**Expected behavior:**
- `tblt-orchestrator` recognizes all 8 inputs are provided verbatim.
- Skips Phase 0 elicitation; jumps to Phase 0h (Review Collected Inputs).
- Displays the inputs summary and waits for YES.
- After YES, builds PVS (#01–#10), assembles Shared Context Block, displays Gate 1.
- After Gate 1 YES, invokes `tblt-activity-specialist` via Agent tool (Stage 1 — runs FIRST).
- Activity specialist runs Phase -1 (silent log fetch), shows Phase 2 Step A (vocabulary distribution), Phase 2 Step B (non-item Spanish audit), Phase 2.5 (success criteria coverage), produces HTML artifact, Phase 5a feedback prompt (split: quality + engagement).
- Orchestrator forwards artifact and feedback prompt to teacher; collects split rating (quality + engagement + note). Stores as `{stage1_quality}`, `{stage1_engagement}`, `{stage1_note}`.
- Orchestrator parses manifest; extracts `{activity_pvs_used}` and `{activity_grammar_deployed}` from `actual_pvs_items_used` and `actual_grammar_deployed` fields.
- Orchestrator displays Gate 2 handoff record — includes Activity Derivation summary showing which PVS items appear in the activity and how each grammar structure was deployed. Waits for YES.
- After Gate 2 YES, invokes `tblt-pretask-specialist` via Agent tool (Stage 2 — runs SECOND), passing both the Shared Context Block AND the Activity Derivation Block.
- Pre-task specialist runs Phase -1, shows Phase 2 distribution table (PVS items from derivation block prioritized in Paso 1), Phase 3 content plan (frames designed to prime specific constructions from activity), produces HTML artifact + answer key, Phase 5a feedback prompt.
- Orchestrator forwards artifact and feedback prompt to teacher; collects rating + note. Stores as `{stage2_rating}`, `{stage2_note}`.
- After rating, orchestrator displays Gate 3 complication/outcome prompt with 3 options (complication_candidate extracted from Stage 1 activity manifest).
- After teacher chooses complication + outcome, orchestrator displays Gate 3 handoff record (shows Stage 2 rating) and waits for YES.
- After Gate 3 YES, invokes `tblt-reflective-specialist` with Shared Context Block AND Activity Derivation Block.
- Reflective specialist shows Phase 2 audit, Phase 2.5, Phase 3 plan (scenario and phrase pairs anchored to activity derivation), produces HTML.
- Orchestrator displays Phase 7 Pipeline Summary Card: Document 1 = Main TBLT Activity, Document 2 = Pre-Task Vocabulary Introduction, Document 3 = Reflective Loop. Integrity verification section shows log writes as `{stage1_log} (activity) · {stage2_log} (pre-task) · — Stage 3 (no log)`.

---

## Test 2 — Full Pipeline, Multi-Turn (Orchestrator Walks Through Phase 0)

**Prompt:**

```
I want a complete TBLT lesson package — pre-task, main activity, and reflective loop — for my 9th-grade Spanish class.
```

**Expected behavior:**
- `tblt-orchestrator` recognizes this as a full three-stage request.
- Runs Phase -2 silently (attempts to read class-profile.md; proceeds silently if absent).
- Displays Phase 0a preview message listing the 8 input fields with a "ready?" prompt.
- Walks through 0b (topic), 0c (learner profile + ACTFL level), 0d.1 (vocabulary — no recommendations), 0d.2 (grammar — no recommendations), 0d.3 (task type — 3 options), 0d.4 (complication — optional), 0d.5 (target register — 3 options), 0e (transfer goal — 3 options), 0f (success criteria pool of 9 candidates), 0g (main task description — 3 phrasings), 0h (review all inputs).
- For every field except vocabulary and grammar, presents exactly 3 options grounded in prior inputs.
- After Phase 0h confirmation, continues with PVS lock, Gate 1, and the rest of the pipeline as in Test 1.

**Regression check:** confirm that vocabulary and grammar elicitation steps show NO recommendations (just the question + the ⚠ warning).

---

## Test 3 — Two-Stage Request (Disambiguation Rule)

**Prompt:**

```
Can you build me a pre-task activity and a main TBLT activity for La rutina diaria? I don't need the reflection piece this time.
```

**Expected behavior:**
- `tblt-orchestrator` recognizes this as a two-stage request (not all three).
- Applies the Disambiguation Rule: asks "Should I skip the Reflective Loop entirely, or run the full pipeline?"
- If the teacher confirms "skip the reflective loop": orchestrator runs Phase 0, Phase 1, Stage 1, Gate 2, Stage 2 only. Does NOT invoke `tblt-reflective-specialist`. Does NOT display Gate 3.
- Phase 7 summary card shows only Stages 1 and 2, noting "Stage 3 skipped at teacher's request."

**Regression check:** orchestrator does NOT silently skip Stage 3 without confirming with the teacher. It ASKS first.

---

## Test 4 — Single-Stage Request (Specialist Runs Standalone, Orchestrator Does NOT Engage)

**Prompt:**

```
Just build me a pre-task vocabulary worksheet for La comida. Here's my vocabulary list:
la manzana, el pan, la leche, el queso, el pollo, el arroz, las verduras, el jugo, el agua, la carne
Grammar: present tense of -ar verbs; indefinite articles (un, una)
Level: Novice
Main task: Students will interview a partner about their favorite foods and build a class profile.
Transfer goal: Students will be able to describe food preferences to a partner in order to find common ground when planning a meal together.
Success criteria:
  1. Names at least 3 foods they prefer and explains why
  2. Asks a partner what they like and records the answer
  3. Sustains the conversation in Spanish without English
```

**Expected behavior:**
- `tblt-orchestrator` is NOT invoked. The Disambiguation Rule says: single-activity request → activate the matching individual specialist directly.
- `tblt-pretask-specialist` runs standalone, beginning at Phase -1.
- Specialist runs through its full phase sequence: Phase -1 (silent log fetch), Phase 1–1b (vocabulary classification + distribution pre-commitment), Phase 2 (distribution table + vocabulary audit), Phase 3 (content plan), Phase 4 (HTML artifact + answer key), Phase 5a (feedback prompt), Phase 5b (log append), manifest emission.
- No Gate prompts, no handoff records, no Pipeline Summary Card.

**Regression check:** orchestrator must NOT interject and try to elicit additional inputs or run Gates. The specialist handles everything on its own.

---

## Test 5 — Form-Focused Success Criteria (Restate-and-Confirm Behavior)

**Prompt:**

```
Build a complete three-part TBLT lesson on Los viajes for my 9th-grade class.
Topic: Los viajes
Vocabulary: el avión, el hotel, el pasaporte, la maleta, el vuelo, la reservación,
el equipaje, facturar, aterrizar, despegar
Grammar: future tense; ir + a + infinitive
Level: Intermediate
Learner profile: 9th grade; interest hooks: travel, social media, music
Task type: Booking simulation — partners take turns as traveler and hotel receptionist
Target register: Formal complaint letter to the hotel manager (audience: hotel manager)
Transfer goal: Students will be able to handle a travel booking complication in Spanish in order to resolve problems at a hotel or airport in a real trip.
Success criteria:
  1. Uses future tense correctly in at least 80% of obligatory contexts
  2. Produces grammatically correct sentences throughout the interaction
  3. Negotiates a resolution to the booking problem with the receptionist
  4. Sustains the interaction in Spanish for the full task
```

**Expected behavior:**
- `tblt-orchestrator` reaches Phase 0f and detects that criteria 1 and 2 are form-focused
  (accuracy percentages, not communicative performances).
- Applies the restate-and-confirm behavior: proposes communicative restatements of criteria 1 and 2 with three alternative wordings each, asks teacher to confirm before advancing.
- Does NOT silently rewrite the criteria.
- After confirmation, stores the revised criteria and continues with Phase 0g–0h.
- When criteria are later passed to the specialists, confirms the specialists' Phase 2.5 coverage tables run against the revised (communicative) criteria, not the original form-focused ones.

**Regression check:** orchestrator presents 3 options for each form-focused criterion it flags. It does NOT just present one restatement. It waits for teacher confirmation before advancing to Phase 0g.

---

## Notes for Test Running

- Tests 1–3 and Test 5 require the orchestrator to be active (full pipeline or two-stage). Invoke via a message that matches the Trigger Examples in `tblt-orchestrator.md`.
- Test 4 requires invoking `tblt-pretask-specialist` directly (single-stage request).
- Each test's "Regression check" item calls out the most likely failure mode for that test. Check these explicitly.
- The session log at `C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-activity-log.md` will accumulate rows after each full test run. Inspect it to verify Phase 5b log appends are working correctly.
