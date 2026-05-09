# Task-Based Language Agent

A multi-agent Claude Code pipeline that produces complete TBLT lesson packages for 9th-grade Spanish. Given a topic and class context, it generates three coordinated A4 worksheets (a main communicative task, a pre-task vocabulary exercise, and a post-task written reflection), evaluated against Lee's TBLT principles and Schell's engagement framework before delivery.

---

## What It Does

A teacher supplies a topic and class details. The pipeline freezes a **Permitted Vocabulary Set (PVS)** —the exact words students may use— before any worksheet is generated. Three specialist agents run in sequence, each producing one worksheet. A quality gate scores the main activity for pedagogical validity and engagement; it triggers a revision pass if the activity falls short. Every worksheet is verified to draw from the same vocabulary set, grammar structures, and transfer goal before the pipeline closes.

---

## Agents

| Agent | Role |
|---|---|
| `tblt-orchestrator` | Collects inputs, freezes the PVS, manages teacher confirmation gates, runs cross-stage integrity checks, produces the final summary card. |
| `tblt-activity-specialist` | Generates the main communicative-task worksheet. Runs first; its vocabulary and grammar deployment anchor the two stages that follow. |
| `tblt-inspector` | Scores the Stage 1 artifact against Lee's TBLT criteria and Schell's engagement lenses. Blocks Stage 2 until the activity meets threshold; triggers a revision pass if it does not. |
| `tblt-pretask-specialist` | Generates the pre-task vocabulary worksheet using the actual items and grammar from Stage 1. |
| `tblt-reflective-specialist` | Generates the post-task written-expansion worksheet using the confirmed complication and outcome from Stage 1 as the writing anchor. |

---

## Pedagogical Framework

The pipeline is built on two interlocking frameworks. Lee's TBLT hardware (information gap, interaction dependency, linguistic targeting, linguistic payoff alignment) defines whether an activity qualifies as a task. Schell's engagement lenses (curiosity and surprise, flow curve, meaningful choice, ending payoff) determine whether students will invest in it. The inspector scores both; low scores on either produce revision requirements, not suggestions.

The main activity runs before the pre-task because it is the design anchor. Vocabulary, grammar, and scenario used in Stage 1 propagate forward; the pre-task and reflective worksheets prepare and extend what students will actually encounter, not the topic in the abstract.

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI installed and authenticated
- An Anthropic account with access to Claude Sonnet or above

---

## Installation

### Project scope

The agents are installed in `.claude/agents/`. Running Claude Code from this directory activates them.

### User-wide

**macOS / Linux**
```bash
cp .claude/agents/*.md ~/.claude/agents/
```

**Windows**
```powershell
Copy-Item ".\.claude\agents\*.md" "$env:USERPROFILE\.claude\agents\"
```

---

## Usage

### Full three-stage pipeline

```
Build me a full TBLT lesson on Las tareas del hogar for my 9th-grade class.
```

```
Run the complete pipeline for La rutina diaria — Spanish 2, ACTFL Intermediate Low.
```

The orchestrator elicits inputs one step at a time. Supplying all inputs in one message skips the elicitation.

### Single-stage requests

Individual specialists activate without the orchestrator:

```
Build me a pre-task vocabulary worksheet for La comida.
I need a main communicative activity for hotel booking.
Create a reflective writing worksheet for the travel task we did.
```

---

## Output

Worksheets are saved automatically as dated HTML files:

```
[YYYY-MM-DD]_[topic-slug]_activity.html
[YYYY-MM-DD]_[topic-slug]_pretask.html
[YYYY-MM-DD]_[topic-slug]_reflective.html
```

Default location: `~/Documents/spanish-tblt/lessons/`

A failed save does not halt the pipeline; the HTML remains in the chat.

---

## Session Log

Each session appends a structured record to `spanish-activity-log.md`: topic, vocabulary count, grammar structures, Paso design, teacher ratings. Specialists read this log before generating to avoid repeating recent topics or exercise formats.

---

## Documentation

| File | Contents |
|---|---|
| `docs/ARCHITECTURE.md` | Agent responsibilities, design rationale, manifest contract |
| `docs/MANIFEST_SCHEMA.md` | Full YAML manifest schema with per-stage variations and examples |
| `docs/MIGRATION_NOTES.md` | What changed from the earlier single-context build |
| `tests/smoke-test-prompts.md` | Test prompts for verifying pipeline behavior |

# task-based-language-agent
A Claude Code multi-agent pipeline that generates complete TBLT lesson packages for 9th-grade Spanish. Three specialist agents produce coordinated worksheets (communicative task, vocabulary preparation, and written reflection), scored against Lee's TBLT criteria and Schell's engagement lenses before delivery.
