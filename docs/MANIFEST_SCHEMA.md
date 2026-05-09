# MANIFEST_SCHEMA.md — Specialist Manifest Contract

Every specialist returns a YAML manifest as the **last block** of its output.
The orchestrator (`tblt-orchestrator`) parses this manifest to run integrity checks
and populate the Phase 7 Pipeline Summary Card. No pedagogical data ever travels
back to the orchestrator as prose — the manifest is the only structured return value.

---

## Canonical Schema

```yaml
stage: 1                              # 1, 2, or 3
specialist: tblt-pretask-specialist   # tblt-pretask-specialist | tblt-activity-specialist
                                      #   | tblt-reflective-specialist
artifact_format: html                 # always 'html' for all three specialists
artifact_location: inline             # 'inline' if Phase 4b save failed; absolute file path
                                      # (e.g. C:\...\lessons\2026-05-03_las-tareas_activity.html)
                                      # if Phase 4b succeeded

pvs_coverage:
  total_items: 18                     # N from the Shared Context Block PVS
  items_used: ["#01", "#02", ...]     # every PVS item that appears in at least one exercise
  items_unused: []                    # items NOT appearing in any exercise
                                      # MUST be empty for stages 1 and 2
                                      # MAY be non-empty for stage 3 if the register-shift
                                      # naturally selects a subset — orchestrator decides
                                      # whether to surface this to the teacher

grammar_coverage:
  structure_1: present                # present | absent — for grammar[0]
  structure_2: present                # present | absent — for grammar[1]

non_pvs_spanish_detected: false       # true if the specialist detected any Spanish word
                                      # in the artifact that is not in the PVS or the
                                      # grammar function-word allowlist; triggers an
                                      # integrity flag in the orchestrator

transfer_goal_echoed: true            # true | false — whether {transfer_goal} is explicitly
                                      # referenced in the artifact
                                      # Required for Stage 3 (checklist Position 1)
                                      # Stage 1 and Stage 2: include if a criterion or frame
                                      # directly references it; otherwise false is acceptable

success_criteria_addressed:           # list of criterion identifiers (short labels or
  - "criterion_1_id"                  # positional IDs like "SC1", "SC3") for criteria that
  - "criterion_3_id"                  # are meaningfully primed or assessed in this stage.
                                      # The orchestrator takes the UNION across all three
                                      # stages to verify full coverage.

phase_5a_rating: 4                    # integer 1–5, or null if no feedback was collected
                                      # Stage 3 has no Phase 5a — always null
phase_5a_note: "..."                  # free text, or null if no note / no feedback
                                      # Stage 3 has no Phase 5a — always null

log_write: ok                         # ok | manual_fallback | not_applicable
                                      # ok: Phase 5b completed successfully (✓ Session logged.)
                                      # manual_fallback: Phase 5b failed (⚠ Auto-log failed)
                                      # not_applicable: this specialist has no Phase 5b
                                      # (always 'not_applicable' for tblt-reflective-specialist)

flags: []                             # list of integrity issues the specialist detected
                                      # during its own run (string messages, or empty list)
                                      # Example: ["pvs_item_#07_absent_from_exercises",
                                      #           "grammar_structure_2_absent"]
```

---

## Stage-Specific Extensions

### Stage 2 Only — Complication Candidate

`tblt-activity-specialist` adds one optional field used by the orchestrator for
Gate 3 complication resolution:

```yaml
complication_candidate: "..."    # The specific problem extracted from the Paso design
                                 # (a scenario card, Student B role conflict, or explicit
                                 # instruction framing a problem). Null if no extractable
                                 # complication was found in the Paso design.
```

This field is populated by the specialist after generating the HTML artifact. The
orchestrator uses it in Branch B of the Gate 3 complication resolution logic.

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

---

## Per-Stage Summary

| Field | Stage 1 | Stage 2 | Stage 3 |
|---|---|---|---|
| `stage` | `1` | `2` | `3` |
| `specialist` | `tblt-pretask-specialist` | `tblt-activity-specialist` | `tblt-reflective-specialist` |
| `pvs_coverage.items_unused` | Must be `[]` | Must be `[]` | May be non-empty |
| `transfer_goal_echoed` | Include if applicable | Include if applicable | **Required** — must be `true` |
| `phase_5a_rating` | Integer 1–5 or null | Integer 1–5 or null | **Always null** |
| `phase_5a_note` | Text or null | Text or null | **Always null** |
| `log_write` | `ok` or `manual_fallback` | `ok` or `manual_fallback` | **Always** `not_applicable` |
| `complication_candidate` | Not present | Optional string or null | Not present |
| `actual_pvs_items_used` | Not present | List of PVS item numbers (subset of `items_used`) | Not present |
| `actual_grammar_deployed` | Not present | One string per grammar structure describing deployment | Not present |

---

## Manifest Emission Instructions (for specialists)

The manifest is emitted as the **final output block** after all other content.
Format it as a fenced YAML code block with the label `manifest`:

~~~
```yaml manifest
stage: 1
specialist: tblt-pretask-specialist
artifact_format: html
artifact_location: inline
pvs_coverage:
  total_items: 18
  items_used: ["#01", "#02", "#03", "#04", "#05", "#06", "#07", "#08",
               "#09", "#10", "#11", "#12", "#13", "#14", "#15", "#16",
               "#17", "#18"]
  items_unused: []
grammar_coverage:
  structure_1: present
  structure_2: present
non_pvs_spanish_detected: false
transfer_goal_echoed: false
success_criteria_addressed: ["SC1", "SC3", "SC4"]
phase_5a_rating: 4
phase_5a_note: "students needed extra time on the matching exercise"
log_write: ok
flags: []
```
~~~

Emit the manifest only after Phase 5b (or Phase 4 for Stage 3, which has no feedback or log phase).
Do not emit a partial manifest mid-run. If Phase 5b fails (manual fallback), still emit the manifest
with `log_write: manual_fallback`.

---

## Orchestrator Verification Rules

After **each stage**, the orchestrator runs per-stage checks on the returned manifest.
After **Stage 3**, the orchestrator runs cross-stage checks across all three manifests.

### Per-Stage Checks

| Check | Condition | Action |
|---|---|---|
| PVS fence | `non_pvs_spanish_detected: true` | Integrity flag — surface to teacher |
| PVS coverage (stages 1–2) | `pvs_coverage.items_unused` is not empty | Integrity flag — list missing items |
| Grammar coverage | Either `structure_1` or `structure_2` is `absent` | Integrity flag |
| Specialist flags | `flags` list is non-empty | Surface each flag to teacher |

### Cross-Stage Checks (after Stage 3)

| Check | Rule | Failure action |
|---|---|---|
| **Full PVS coverage** | Every PVS item appears in `items_used` for Stage 1 **or** Stage 2 (or both) | Note in Phase 7 summary |
| **Grammar thread** | Both grammar structures are `present` in at least 2 of the 3 stage manifests | Note in Phase 7 summary |
| **PVS fence** | `non_pvs_spanish_detected: false` for all three stages | Note in Phase 7 summary with stage numbers |
| **Transfer goal** | `transfer_goal_echoed: true` for Stage 3 | Note in Phase 7 summary |
| **Criteria coverage** | Union of `success_criteria_addressed` across all three stages covers all {M} teacher-selected criteria | Note uncovered criteria in Phase 7 summary |

Any failure produces an integrity note in the Phase 7 Pipeline Summary Card under the
`INTEGRITY VERIFICATION` section. The orchestrator surfaces the note to the teacher but
does not block pipeline completion — the teacher decides whether to revise.

---

## Examples

### Minimal passing manifest (Stage 1, 10-item PVS)

```yaml manifest
stage: 1
specialist: tblt-pretask-specialist
artifact_format: html
artifact_location: inline
pvs_coverage:
  total_items: 10
  items_used: ["#01", "#02", "#03", "#04", "#05", "#06", "#07", "#08", "#09", "#10"]
  items_unused: []
grammar_coverage:
  structure_1: present
  structure_2: present
non_pvs_spanish_detected: false
transfer_goal_echoed: false
success_criteria_addressed: ["SC2", "SC4"]
phase_5a_rating: 5
phase_5a_note: null
log_write: ok
flags: []
```

### Stage 2 manifest with complication candidate and a flag

```yaml manifest
stage: 2
specialist: tblt-activity-specialist
artifact_format: html
artifact_location: inline
pvs_coverage:
  total_items: 15
  items_used: ["#01", "#02", "#03", "#04", "#05", "#06", "#07", "#08",
               "#09", "#10", "#11", "#12", "#13", "#14", "#15"]
  items_unused: []
grammar_coverage:
  structure_1: present
  structure_2: absent
non_pvs_spanish_detected: false
transfer_goal_echoed: false
success_criteria_addressed: ["SC1", "SC2", "SC3", "SC5"]
phase_5a_rating: 3
phase_5a_note: "Paso 3 ran long"
log_write: ok
complication_candidate: "la reservación del restaurante está doble reservada"
flags: ["grammar_structure_2_absent_from_pasos"]
```

### Stage 3 manifest (no feedback, no log)

```yaml manifest
stage: 3
specialist: tblt-reflective-specialist
artifact_format: html
artifact_location: inline
pvs_coverage:
  total_items: 15
  items_used: ["#01", "#03", "#05", "#07", "#09", "#11", "#13", "#15"]
  items_unused: ["#02", "#04", "#06", "#08", "#10", "#12", "#14"]
grammar_coverage:
  structure_1: present
  structure_2: present
non_pvs_spanish_detected: false
transfer_goal_echoed: true
success_criteria_addressed: ["SC1", "SC2", "SC4", "SC5"]
phase_5a_rating: null
phase_5a_note: null
log_write: not_applicable
flags: []
```
