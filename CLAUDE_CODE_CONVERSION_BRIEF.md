# Conversion Brief — TBLT Skills to Supervisor/Subagent Architecture

**Audience:** Claude Code, running locally on the user's Windows laptop.
**Goal:** Convert four existing TBLT lesson-design skills into a supervisor/subagent system, where the supervisor is named `tblt-orchestrator` and three specialist subagents handle the three lesson stages.
**Architecture target:** Architecture A from the prior analysis — Thin Supervisor + Three Specialists. (Architectures B and C are out of scope for this pass; they may be added later.)

---

## 1. Source material

The user has four existing skills that currently run together as a single-context skill stack in Claude Desktop and Cowork. The canonical location for these source skills on this laptop is:

```
C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-task-based\
```

This directory contains four subfolders, each holding one `SKILL.md`:

| Subfolder | Role today | Approx. lines |
|---|---|---|
| `conductor` | Pipeline orchestrator: input elicitation, PVS lock, gates, summary | ~1,184 |
| `spanish-pretask-vocab` | Stage 1 — pre-task vocabulary worksheet | ~702 |
| `tblt-spanish-activity` | Stage 2 — main communicative-task worksheet | ~831 |
| `reflective-loop` | Stage 3 — post-task written-expansion worksheet | ~652 |

These four skills will be **read but not modified.** The originals must remain intact and functional in their current location — they are the active build for Desktop/Cowork users.

The conversion produces a new, parallel set of agent files for Claude Code that share the same pedagogical logic but use real subagent isolation.

A second copy of these skills also exists at `C:\Users\azuaje.MSMC\Downloads\New_Skills\`. That copy is **not canonical** and should be ignored. The user has confirmed `Documents\spanish-tblt\spanish-task-based\` as the single source of truth going forward.

---

## 2. Workspace setup

The user is launching Claude Code with the source skills mounted read-only via `--add-dir`. Expect this layout when the session starts:

```
Project root (cwd):
  C:\Users\azuaje.MSMC\Documents\spanish-tblt\tblt-v2\          ← write all output here

Additional read-only directory (passed via --add-dir):
  C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-task-based\
    ├── conductor\
    │   └── SKILL.md
    ├── spanish-pretask-vocab\
    │   └── SKILL.md
    ├── tblt-spanish-activity\
    │   └── SKILL.md
    └── reflective-loop\
        └── SKILL.md
```

At session start, **verify the path** by running `dir` (or `ls`, either works in Claude Code) on the additional directory and confirm the four expected subdirectories are present. If they are not — for example, if the four folders are nested one level deeper inside a wrapper folder like `New_Skills` — stop and ask the user to confirm the source path before reading anything.

Both forward and backslash path styles are acceptable when you reference paths in your output. Be consistent within a single file.

**Hard rule:** never write to the source directory at `Documents\spanish-tblt\spanish-task-based\`. All output goes into the project root at `Documents\spanish-tblt\tblt-v2\`. All reads from the source directory are read-only.

---

## 3. Output structure

Produce the following directory tree in the project root (`C:\Users\azuaje.MSMC\Documents\spanish-tblt\tblt-v2\`):

```
.
├── .claude\
│   └── agents\
│       ├── tblt-orchestrator.md
│       ├── tblt-pretask-specialist.md
│       ├── tblt-activity-specialist.md
│       └── tblt-reflective-specialist.md
├── docs\
│   ├── ARCHITECTURE.md
│   ├── MANIFEST_SCHEMA.md
│   └── MIGRATION_NOTES.md
├── tests\
│   └── smoke-test-prompts.md
└── README.md
```

Naming conventions:

- **Supervisor** is named `tblt-orchestrator` (per the user's decision). Its frontmatter `name:` field, file name, and all cross-references must use exactly this string.
- **Specialists** are named `tblt-pretask-specialist`, `tblt-activity-specialist`, `tblt-reflective-specialist`. The `tblt-` prefix scopes the agents to this domain so they don't collide with other agents the user may have installed.
- The three specialists derive their content from the three sub-skills (`spanish-pretask-vocab`, `tblt-spanish-activity`, `reflective-loop`) but are renamed to reflect their role inside the new architecture rather than their original standalone identity.

---

## 4. What each output file must contain

### 4.1 `.claude\agents\tblt-orchestrator.md`

This is the supervisor. Its system prompt must:

1. **Inherit the conductor's responsibilities** for: class-profile load (Phase -2), sequential input elicitation (Phase 0a–0h), PVS lock and Shared Context Block assembly (Phase 1), the three teacher confirmation gates, integrity verification *across manifests* (not across artifacts), and the final summary card (Phase 7). The Teacher Input Recommendations Rule and the Disambiguation Rule from the conductor are also preserved.

2. **Drop everything that the new architecture makes obsolete.** Specifically: remove the conductor's "Phase 0 Suppression Rule" prose (suppression is now structural — the specialists' system prompts simply don't have a Phase 0), remove the Context Recovery Rule (subagents can't lose context the same way a single-context skill can), and trim the parts of the conductor that re-stated sub-skill mechanics for orientation.

3. **Use the Agent tool to delegate** to the three specialists. Each delegation passes the Shared Context Block (or the relevant subset) as input. The orchestrator never re-implements specialist logic.

4. **Verify integrity over manifests.** After each specialist returns, the orchestrator parses the manifest (schema in §4.5 and `docs\MANIFEST_SCHEMA.md`) and runs the integrity checks. Cross-stage checks (e.g., PVS consistency across all three stages) run after Stage 3.

5. **Own the teacher conversation.** All teacher-facing text — gate prompts, recommendation triplets, summary card — is emitted by the orchestrator. Specialists do not converse with the teacher.

The orchestrator's frontmatter:

```yaml
---
name: tblt-orchestrator
description: >
  TBLT lesson pipeline supervisor. Use when a teacher requests a complete
  three-part TBLT lesson (pre-task vocab + main task + reflective loop) for
  9th-grade Spanish. Collects shared inputs once, freezes the Permitted
  Vocabulary Set, and delegates to three specialist subagents in order with
  teacher confirmation gates between stages.
tools: Read, Write, Edit, Agent, Bash
model: inherit
---
```

The `Agent` tool is required so the orchestrator can invoke subagents. `Bash` is needed only for the class-profile write-back at Step 1d (file creation if the profile does not exist) and the optional log-file paths used by the specialists; if you can constrain it further with hooks or wrappers, prefer that.

The orchestrator's system prompt should be **substantially shorter than the original 1,184-line conductor** — target 300–450 lines. The reduction comes from removing the prose that exists today only to mediate the inside of each stage.

### 4.2 `.claude\agents\tblt-pretask-specialist.md`

Source: `spanish-pretask-vocab\SKILL.md`.

Construct the system prompt by:

1. Copying the source SKILL.md verbatim as a starting point.
2. **Removing Phase 0** (input collection). The orchestrator now provides all inputs via the invocation prompt. Replace the Phase 0 section with a short "Inputs (provided by orchestrator)" block that lists the variables the specialist receives: `{topic}`, `{vocabulary}` (already numbered as PVS), `{grammar}`, `{level}`, `{main_task}`, `{transfer_goal}`, `{success_criteria}`, `{target_register}`.
3. **Keeping Phase −1** (session-log fetch and anti-repetition). This logic is local to the specialist and must continue to run. On Windows, the source SKILL.md already resolves the log file at `%USERPROFILE%\Documents\spanish-tblt\spanish-activity-log.md` by default (or at `$SPANISH_TBLT_LOG_DIR\spanish-activity-log.md` if that environment variable is set). On this laptop, `%USERPROFILE%` resolves to `C:\Users\azuaje.MSMC\`, so the default path lands at `C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-activity-log.md` — exactly the parent directory of the source skills. No environment-variable override is needed. Preserve this resolution logic verbatim.
4. **Keeping all mandatory gates intact**: Phase 2 (vocabulary distribution table + audit), Phase 3 (content plan), Phase 4 (HTML artifact), Phase 5a (teacher feedback), Phase 5b (log append). The specialist still emits these to its own output stream; the orchestrator forwards the artifact and feedback prompt to the teacher and captures the rating.
5. **Adding a final manifest emission step** (new): after Phase 5b, the specialist returns a structured manifest per §4.5. This manifest is what the orchestrator actually consumes; the artifact and feedback flow through the orchestrator transparently.

Frontmatter:

```yaml
---
name: tblt-pretask-specialist
description: >
  Stage 1 specialist: produces the pre-task vocabulary worksheet for a
  9th-grade Spanish TBLT lesson. Invoked by tblt-orchestrator with all
  inputs pre-supplied. Do not invoke directly for full lesson packages.
tools: Read, Write, Edit, Bash
model: inherit
---
```

`Bash` is needed for session-log read (Phase −1) and append (Phase 5b). No `Agent` tool — specialists do not invoke other agents.

### 4.3 `.claude\agents\tblt-activity-specialist.md`

Source: `tblt-spanish-activity\SKILL.md`. Apply the same transformation as §4.2:

- Remove Phase 0; replace with an "Inputs (provided by orchestrator)" block.
- Keep Phase −1 anti-repetition logic. The same Windows log-path resolution applies (see §4.2 step 3).
- Keep all mandatory gates (vocabulary distribution, content plan, Pasos generation, coverage table, complication anchor, feedback).
- Add manifest emission as the final step.

Frontmatter mirrors §4.2 with `name: tblt-activity-specialist` and an updated description.

### 4.4 `.claude\agents\tblt-reflective-specialist.md`

Source: `reflective-loop\SKILL.md`. Apply the same transformation:

- Remove Phase 0; replace with an "Inputs (provided by orchestrator)" block. **Important:** the orchestrator passes a confirmed `{complication_final}` and `{outcome}` from its Phase 5a/5b, so the reflective-loop's Phase 1b (complication generation) is also dropped — the value is pre-supplied. This matches the existing "Phase 0 Suppression Rule" behavior in the conductor.
- This specialist has no Phase −1 / Phase 5b log operations in the original skill. Confirm by reading the source. If absent, do not add them.
- Add manifest emission as the final step.

Frontmatter mirrors §4.2 with `name: tblt-reflective-specialist`. Tools: `Read, Write, Edit` — no `Bash` needed if there is no log I/O, which keeps this specialist's permissions minimal.

### 4.5 Manifest schema

Every specialist returns a YAML manifest as the last block of its output. The orchestrator parses this and runs cross-stage verification against it.

Document the schema in `docs\MANIFEST_SCHEMA.md` and reference it from each specialist's system prompt. Canonical shape:

```yaml
stage: 1                              # or 2, or 3
specialist: tblt-pretask-specialist   # or activity / reflective
artifact_format: html                  # always 'html' for these three
artifact_location: inline              # or file path if written to disk
pvs_coverage:
  total_items: 18                      # N from Shared Context Block
  items_used: ["#01", "#02", ...]
  items_unused: []                     # MUST be empty for stages 1 and 2;
                                       # may be non-empty for stage 3 if
                                       # register-shift naturally selects
                                       # a subset (orchestrator decides)
grammar_coverage:
  structure_1: present | absent
  structure_2: present | absent
non_pvs_spanish_detected: false        # true triggers an integrity flag
transfer_goal_echoed: true | false     # only required for stage 3 position 1
success_criteria_addressed: ["criterion_1_id", "criterion_3_id", ...]
phase_5a_rating: 4                     # null for stage 3 (no rating prompt)
phase_5a_note: "..."                   # null if no note
log_write: ok | manual_fallback | not_applicable
flags: []                              # any integrity issues the specialist
                                       # detected during its own run
```

The orchestrator's integrity check after Stage 3 uses these manifests to verify:

- Every PVS item appears in `items_used` for at least one of stages 1 and 2.
- Both grammar structures are `present` in at least two of the three stages.
- `non_pvs_spanish_detected` is `false` for all three stages.
- `transfer_goal_echoed` is `true` for stage 3.
- The union of `success_criteria_addressed` covers all criteria the teacher selected.

Any failure produces an integrity flag the orchestrator surfaces to the teacher in the summary card, exactly as the conductor does today.

---

## 5. Documentation files

### 5.1 `docs\ARCHITECTURE.md`

A concise (300–500 word) document explaining:

- The supervisor/subagent split and why it exists (context isolation, scoped tools, real gate boundaries).
- The role of each agent.
- How the manifest contract works.
- Where the line is between orchestrator responsibility and specialist responsibility (the orchestrator owns the teacher conversation, the PVS, and cross-stage integrity; specialists own pedagogical generation and per-stage gates).

### 5.2 `docs\MANIFEST_SCHEMA.md`

The full manifest schema from §4.5 plus examples and the orchestrator's verification rules.

### 5.3 `docs\MIGRATION_NOTES.md`

A short document covering:

- The mapping from old skill names to new agent names.
- Which sections of each source skill were dropped (Phase 0 in all three specialists, Phase 1b in `reflective-loop`, the Context Recovery and Phase 0 Suppression rules in the conductor) and why.
- The shared `spanish-activity-log.md` schema, confirming both the old Desktop/Cowork build and the new Claude Code build write the same format. **Do not invent or modify the schema** — read the existing log-handling sections of the source skills (`spanish-pretask-vocab` Phase −1 and Phase 5b, and the corresponding sections in `tblt-spanish-activity`) and document the format that's already in use.
- The Windows-specific log-resolution path for this laptop: `C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-activity-log.md`, achievable via the source skills' default `%USERPROFILE%\Documents\spanish-tblt\` resolution with no environment-variable override.
- A note that `Downloads\New_Skills` is a non-canonical duplicate the user may delete or archive.
- Known divergences between old and new behavior, if any. Flag rather than silently change.

---

## 6. Tests

### 6.1 `tests\smoke-test-prompts.md`

A handful of natural-language prompts the user can paste into Claude Code to verify the system works end-to-end. Suggested coverage:

- A full pipeline run with all inputs provided in one message (fast-track path).
- A multi-turn run where the orchestrator walks through Phase 0a–0h.
- A two-stage request (e.g., "skip the reflective loop") to confirm the disambiguation rule.
- A single-stage request (e.g., "just the pre-task") to confirm the orchestrator does NOT engage and the specialist runs standalone.
- A run with intentionally form-focused success criteria to confirm the orchestrator's restate-and-confirm behavior survived the migration.

For each prompt, note the expected behavior so the user can spot regressions.

---

## 7. README

A top-level `README.md` covering: what this directory is, how to install the agents (copy `.claude\agents\*.md` to `%USERPROFILE%\.claude\agents\` for user-wide deployment, or keep at project scope for testing), how to invoke the orchestrator, and a pointer to `docs\ARCHITECTURE.md` for design rationale. Mention that on this laptop the user-wide path resolves to `C:\Users\azuaje.MSMC\.claude\agents\`.

---

## 8. Execution sequence for Claude Code

Work through these steps in order. Do not skip ahead.

1. **Verify the workspace.** Run `dir` (or `ls`) on the additional directory passed via `--add-dir` — expected to be `C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-task-based\`. Confirm the four expected subdirectories: `conductor`, `spanish-pretask-vocab`, `tblt-spanish-activity`, `reflective-loop`. If any are missing or nested one level deeper, stop and ask.

2. **Read the four source skills in full.** Use the Read tool on each `SKILL.md`. Do not summarize from memory — every transformation in this brief depends on faithful reading of the source.

3. **Confirm the log-file schema** by reading the Phase −1 and Phase 5b sections of `spanish-pretask-vocab\SKILL.md` and `tblt-spanish-activity\SKILL.md`. Capture the exact section names, table column headers, and field names so `MIGRATION_NOTES.md` documents them correctly. Confirm the Windows resolution path noted in §4.2 step 3.

4. **Create the directory structure** in §3 inside the project root (`C:\Users\azuaje.MSMC\Documents\spanish-tblt\tblt-v2\`). Create empty placeholder files first so the structure is visible.

5. **Write `.claude\agents\tblt-orchestrator.md`.** Build it from the conductor's system prompt minus the obsolete sections (per §4.1). Verify by re-reading the source conductor that you haven't dropped anything pedagogically substantive.

6. **Write the three specialists** in order: `tblt-pretask-specialist.md`, `tblt-activity-specialist.md`, `tblt-reflective-specialist.md`. For each one, apply the transformation in §4.2–4.4. Add the manifest emission step at the end.

7. **Write `docs\MANIFEST_SCHEMA.md`** before finalizing the agents — the agents reference it.

8. **Write `docs\ARCHITECTURE.md` and `docs\MIGRATION_NOTES.md`.**

9. **Write `tests\smoke-test-prompts.md` and `README.md`.**

10. **Self-check** by re-reading each output file with fresh eyes:
    - Does each specialist's frontmatter `tools:` list contain exactly what it needs and nothing more?
    - Does the orchestrator's frontmatter include the `Agent` tool?
    - Are all four `name:` fields exactly the strings specified in §3?
    - Does `MIGRATION_NOTES.md` document the log schema by reference to the actual source, not by invention?
    - Does `MIGRATION_NOTES.md` correctly note the Windows log-resolution path for this laptop?
    - Is the manifest schema referenced consistently from all three specialists?

11. **Report back to the user** with: a tree view of what was produced, the line count of `tblt-orchestrator.md` versus the original conductor (target: 300–450 lines), and any integrity concerns surfaced during the conversion (e.g., source-skill sections that were ambiguous and required a judgment call).

---

## 9. Rules that must not be violated

These are absolute. If any of them comes into tension with another instruction in this brief, stop and ask the user.

- **Never modify the source skills** at `C:\Users\azuaje.MSMC\Documents\spanish-tblt\spanish-task-based\`. All output goes into the project root at `C:\Users\azuaje.MSMC\Documents\spanish-tblt\tblt-v2\`; the additional directory is read-only.
- **Never touch `Downloads\New_Skills`.** It is a non-canonical duplicate. Do not read from it, write to it, or reference it in any output file except in `MIGRATION_NOTES.md` where it is mentioned as a duplicate the user may discard.
- **Never invent the log schema.** Read it from the source.
- **Never invent pedagogical logic.** All gates, distribution rules, vocabulary fences, and proficiency calibrations come from the source skills verbatim. The transformation removes Phase 0 and adds manifest emission; nothing else changes.
- **Never reduce the PVS integrity guarantee.** The conductor's PVS Integrity Rule (Rule 1) — that the numbered PVS may never be modified, expanded, or reduced between stages — must survive the migration unchanged. The orchestrator enforces it by passing the same PVS to every specialist and verifying via manifests that no specialist added or removed items.
- **Never co-mingle teacher conversation with specialist invocation.** The orchestrator talks to the teacher; specialists generate artifacts and emit manifests. If the source conductor mixes these (e.g., a teacher gate that lives inside what should now be a specialist), surface it and ask the user how to handle it.
- **Preserve the three confirmation gates.** Gate 1 (after Shared Context Block), Gate 2 (after Stage 1), Gate 3 (after Stage 2 / before Stage 3). These are the teacher's control points and must remain teacher-facing.

---

## 10. Out of scope for this pass

Do not attempt any of the following in this conversion:

- Architecture B (integrity-reviewer subagent) or Architecture C (Context Steward + Pedagogical Coordinator). These are deliberate future steps.
- Changing pedagogical content (exercise types, Paso structures, rubric formats).
- Porting to a hosted database or non-filesystem log backend.
- Translating any user-facing strings out of their existing language (specialist artifacts stay Spanish; orchestrator-level text stays English; per the source skills' language rules).
- Removing the Desktop/Cowork build. The user explicitly wants a parallel deployment, not a replacement.
- Acting on the `Downloads\New_Skills` duplicate. Cleanup of that folder is the user's decision and happens outside this conversion.

If any of these come up during conversion as obvious wins, note them in `MIGRATION_NOTES.md` under a "Future Work" section but do not implement them.

---

End of brief.
