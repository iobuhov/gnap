# Spec-Extraction Workflow

This document describes the workflow for each agent role in the spec-extraction Ralph loop. All agents operate on the `workspace/` repository and coordinate through `workspace/.gnap/`.

---

## Overview

```
[Worker] ──────────────────────────────────────────────┐
   │  explores source files, fills draft.md             │
   │  ← receives directives from Reviewer               │
   ▼                                                     │
[Reviewer] ─────────────────────────────────────────────┤
   │  reviews draft, sends gaps back to Worker           │
   │  creates synthesize task when satisfied             │  max 10 cycles
   ▼                                                     │
[Spec Writer] ──────────────────────────────────────────┘
      reads draft, writes clean spec-alpha/{widget}.md
```

State lives in `workspace/.gnap/` and `workspace/_workspace/`. Each agent runs on a `/loop` heartbeat — fresh context per iteration, no cross-widget memory.

---

## Worker

**Heartbeat:** 300s  
**Writes:** `workspace/_workspace/{widget}/draft.md`

### Trigger

Pick up any task where:
- `assigned_to` includes `worker`
- `state` is `ready`

Take the lowest `priority` number (highest priority).

### Per-iteration workflow

```
1. git pull --rebase  (in workspace/)
2. Read workspace/.gnap/agents.json  → confirm status: active
3. Read workspace/.gnap/messages/    → any directives for "worker"?
4. Read workspace/.gnap/tasks/       → find ready task
5. Set task state → "in_progress", commit + push
6. Execute (see below)
7. Set task state → "review", commit + push
```

### Execution

**First run (no draft exists):**

1. List all source files under `workspace/packages/pluggableWidgets/{widget}/`
2. For each source file, check its imports. If a import points to a local workspace package — a package present in `workspace/packages/` — follow it and include that package's source files in the exploration as well. External npm packages (not found in `workspace/packages/`) are out of scope.
3. For each source file (widget and local dependencies), read it and create a section in `_workspace/{widget}/draft.md` answering:
   1. What is the purpose of this file?
   2. What kind of logic is described in this file?
   3. What part of behavior can be documented from this file?
   4. Is it user-facing?
   5. What new did you learn from this file?

   Each answer must be concise — no more than two paragraphs and 256 words total per file section.
4. When all source files have sections → set task state to `review`

**Follow-up run (Reviewer directive received):**

1. Read the directive message — it lists specific files or sections needing deeper analysis
2. Re-read the named files, update the relevant sections in `draft.md`
3. Mark the directive message as `read_by: ["worker"]`, commit
4. Post a status message confirming the gaps were addressed:

```json
{
  "id": "{N}",
  "from": "worker",
  "to": ["reviewer"],
  "at": "{ISO-timestamp}",
  "type": "status",
  "text": "EX-001: gaps addressed. Updated editorConfig.ts section with behavioral constraints. CHANGELOG v2.3.0 findings added."
}
```

5. Set task state back to `review`

### Commit convention

```
worker: in_progress EX-001
worker: review EX-001 — draft complete, 8 source files covered
worker: update EX-001 — addressed reviewer gaps in editorConfig section
```

---

## Reviewer

**Heartbeat:** 300s  
**Writes:** directives to Worker, creates `synthesize/{widget}` tasks

### Trigger

Find any task where:
- `state` is `review`
- `tags` includes `extract`

### Per-iteration workflow

```
1. git pull --rebase  (in workspace/)
2. Read workspace/.gnap/agents.json  → confirm status: active
3. Read workspace/.gnap/tasks/       → find tasks in "review" state
4. Read _workspace/{widget}/draft.md
5. Evaluate draft (see criteria below)
6. If gaps found → send directive to Worker, return task to "in_progress"
7. If satisfied  → create synthesize task, mark extract task "done"
8. If cycle limit reached → mark task "blocked", alert human
9. commit + push
```

### Review criteria

For each source file section in `draft.md`, verify:

- [ ] All 5 questions are answered (not left blank or marked "unknown")
- [ ] Answers have enough context to write a spec — not just "this file defines props"
- [ ] Behavioral constraints are captured (what props affect other props)
- [ ] User-facing vs. internal distinction is clear
- [ ] Changelog entries are reflected in the findings

### If gaps found

Create `workspace/.gnap/messages/{N}.json`:

```json
{
  "id": "{N}",
  "from": "reviewer",
  "to": ["worker"],
  "at": "{ISO-timestamp}",
  "type": "directive",
  "text": "EX-001: gaps found. Please revisit: (1) editorConfig.ts — behavioral constraints not captured; (2) CHANGELOG.md — v2.3.0 changes not reflected in findings."
}
```

Set task `state` back to `in_progress` and increment a `review_cycles` counter in the task file.

### If satisfied (consensus reached)

1. Create `workspace/.gnap/tasks/SY-{N}.json` (synthesize task):

```json
{
  "id": "SY-{N}",
  "title": "Write spec: {widget}",
  "assigned_to": ["spec-writer"],
  "state": "ready",
  "priority": 1,
  "created_by": "reviewer",
  "created_at": "{ISO-timestamp}",
  "updated_at": "{ISO-timestamp}",
  "desc": "Read _workspace/{widget}/draft.md and produce a clean spec at spec-alpha/{widget}.md.",
  "parent": "EX-{N}",
  "tags": ["synthesize", "{widget}"]
}
```

2. Set extract task `state` → `done`
3. Post a status message notifying the Spec Writer:

```json
{
  "id": "{N}",
  "from": "reviewer",
  "to": ["spec-writer"],
  "at": "{ISO-timestamp}",
  "type": "info",
  "text": "EX-001: draft approved for accordion-web. Synthesize task SY-001 is ready for pickup."
}
```

### If cycle limit reached (10 cycles)

Set task `state` → `blocked`, add `blocked_reason`, send an alert:

```json
{
  "from": "reviewer",
  "to": ["human"],
  "type": "alert",
  "text": "EX-001 blocked after 10 review cycles. Manual review required. Widget: accordion-web."
}
```

### Commit convention

```
reviewer: review EX-001 — gaps found, returning to worker
reviewer: done EX-001 — draft approved, created SY-001
reviewer: block EX-001 — 10 cycles exceeded, human review required
```

---

## Spec Writer

**Heartbeat:** 300s  
**Writes:** `workspace/spec-alpha/{widget}.md`

### Trigger

Pick up any task where:
- `assigned_to` includes `spec-writer`
- `state` is `ready`
- `tags` includes `synthesize`

### Per-iteration workflow

```
1. git pull --rebase  (in workspace/)
2. Read workspace/.gnap/agents.json  → confirm status: active
3. Read workspace/.gnap/tasks/       → find ready synthesize task
4. Set task state → "in_progress", commit + push
5. Execute (see below)
6. Set task state → "done", commit + push
```

### Execution

1. Read `_workspace/{widget}/draft.md` in full
2. Synthesize findings into `spec-alpha/{widget}.md` using the template below
3. Use formal, precise language — this is a specification, not a summary
4. Every claim must be traceable to a finding in the draft
5. If a section cannot be filled from the draft, add it to `## Open Questions`

### Spec template

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
> Could not be determined from source code alone — requires human review
- [ ] ...
```

### Commit convention

```
spec-writer: in_progress SY-001
spec-writer: done SY-001 — spec written for accordion-web
```

---

## Task State Reference

### GNAP States

```
backlog → ready → in_progress → review → done
            ↑          ↑           │
            │          └───────────┘  (reviewer rejects)
            │
         blocked → ready              (unblocked)
            ↓
         cancelled
```

| State | Meaning |
|-------|---------|
| `backlog` | Not yet prioritized |
| `ready` | Prioritized, waiting for agent to pick up |
| `in_progress` | Agent is working on it |
| `review` | Work done, waiting for review |
| `done` | Completed (terminal) |
| `blocked` | Cannot proceed (see `blocked_reason`) |
| `cancelled` | Will not be done (terminal) |

Reverse transitions:
- `review → in_progress` — reviewer rejects, agent reworks
- `blocked → ready` — unblocked, agent picks up again

### This Loop

```
[extract/EX-NNN]
  ready → in_progress (Worker)
        → review (Worker, draft complete)
        → in_progress (Reviewer returns for gaps)     ← up to 10 cycles
        → done (Reviewer satisfied)
        → blocked (10 cycles exceeded)

[synthesize/SY-NNN]   created by Reviewer on consensus
  ready → in_progress (Spec Writer)
        → done
```

## Communication Reference

| From | To | File | Type | Purpose |
|------|----|------|------|---------|
| Reviewer | Worker | `workspace/.gnap/messages/{N}.json` | `directive` | List gaps to address |
| Reviewer | Spec Writer | `workspace/.gnap/tasks/SY-{N}.json` | task | Trigger synthesis |
| Reviewer | Human | `workspace/.gnap/messages/{N}.json` | `alert` | Escalate blocked widget |
