# Spec-Extraction Ralph Loop

## Goal

Automatically extract technical and product specifications from Mendix pluggable widget source code and produce one clean markdown spec document per widget under `spec-alpha/`.

The AI infers hidden behavior and usage from source files. The process is source-driven: every source file must be documented before a spec can be written.

---

## Source Repository

**Upstream:** https://github.com/iobuhov/web-widgets
**Widget location:** `packages/pluggableWidgets/` (55+ widgets)

### Extraction Sources (per widget)

| Source | Role |
|--------|------|
| `src/{Widget}.xml` | Property definitions, captions, types, enumerations, defaults, groups |
| `typings/{Widget}Props.d.ts` | Type shapes, enums — auto-generated from XML |
| `src/{Widget}.editorConfig.ts` | Behavioral constraints — prop visibility rules |
| `src/{Widget}.tsx` | Main component logic |
| `src/components/` | Sub-component behavior |
| `CHANGELOG.md` | History — reveals implied requirements |
| `src/**/__tests__/*.spec.tsx` | Behavioral evidence — interactions, states |
| `e2e/*.spec.js` | End-to-end behavioral evidence |

---

## Artifacts

```
_tmp/{widget}/
  draft.md          ← research document (findings per source file)

spec-alpha/
  {widget-name}.md  ← final clean spec (produced from draft)
```

### Draft Format (`draft.md`)

One section per source file. Each section answers:

1. What is the purpose of this file?
2. What kind of logic is described in this file?
3. What part of behavior can be documented from this file?
4. Is it user-facing?
5. What new did you learn from this file?

**Done condition:** Every source file has a section with all 5 questions answered.

### Spec Format (`spec-alpha/{widget}.md`)

```markdown
# {WidgetName}

## Purpose
{One paragraph: what problem this widget solves and when to use it}

## User Scenarios

### [P1] {Scenario name}
**Given** ...
**When** ...
**Then** ...

#### Edge Cases
- ...

## Functional Requirements
- FR-001: System MUST ...
- FR-002: Users MUST be able to ...

## Props Reference

| Name | Type | Default | Caption | Description |
|------|------|---------|---------|-------------|

## Changelog
{Last 3 versions from CHANGELOG.md}

## Open Questions
> Things that could not be determined from code alone — requires human review
- [ ] ...
```

---

## Agents

### Worker
- Explores widget source files one by one
- Fills `_tmp/{widget}/draft.md` — one section per file, answering the 5 questions
- Receives `directive` messages from Reviewer, addresses gaps, updates draft
- Owns the `extract/{widget}` GNAP task

### Reviewer
- Triggered when `extract/{widget}` task enters `review` state
- Reads draft critically — checks that every source file is covered and all 5 questions have enough context to produce a spec
- If gaps found: sends `directive` message to Worker, task returns to `todo`
- If satisfied: creates `synthesize/{widget}` GNAP task and marks `extract/{widget}` as `done`
- Max 10 Worker-Reviewer cycles per widget; if consensus not reached → task → `blocked`, human agent flagged via GNAP message

### Spec Writer
- Picks up `synthesize/{widget}` GNAP task
- Reads `_tmp/{widget}/draft.md`
- Produces clean, formal `spec-alpha/{widget}.md`
- Specialization: formal language and spec format preservation
- Marks `synthesize/{widget}` task `done`

---

## GNAP Task State Flow

```
[extract/{widget}]
  todo → in_progress (Worker) → review (Reviewer)
               ↑________________________| (gaps found, max 10 cycles)
                                        → done (consensus reached)
                                        → blocked (10 cycles exceeded, human flagged)

[synthesize/{widget}]   (created by Reviewer on consensus)
  todo → in_progress (Spec Writer) → done
```

### Communication

| From | To | Mechanism | Purpose |
|------|----|-----------|---------|
| Reviewer | Worker | GNAP `directive` message | List files needing deeper analysis |
| Reviewer | Spec Writer | GNAP task (`synthesize/{widget}`) | Trigger synthesis |
| Reviewer | Human | GNAP `alert` message | Escalate blocked widgets |

---

## Bootstrap Script

A one-time script reads `packages/pluggableWidgets/` from the upstream repo and creates:
- One `.gnap/tasks/extract/{widget}.json` per widget with `state: todo`
- `.gnap/agents.json` with Worker, Reviewer, and Spec Writer agent entries

Re-running the script is idempotent — skips widgets that already have a task.

---

## Loop Mechanics (Ralph Loop)

Each iteration:
1. Agent pulls latest git state
2. Checks GNAP for tasks assigned to it in `state: todo`
3. Picks next task, sets `state: in_progress`
4. Does work, commits results
5. Updates task state, pushes
6. Loop continues until no `todo` tasks remain

**State lives in `.gnap/` and `_tmp/` — fresh agent context each iteration.**

---

## Open Decisions

- [ ] **Widget scope:** All 55+ widgets in first run, or a filtered subset to validate the loop?
- [ ] **Stop hook implementation:** Claude Code hook vs. external shell loop (`while :; do ...`)
- [ ] **Draft update strategy:** When Worker receives a Reviewer directive, does it append new findings or rewrite the relevant section?
