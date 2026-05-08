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

State lives in `workspace/.gnap/` and `workspace/_tmp/`. Each agent runs on a `/loop` heartbeat — fresh context per iteration, no cross-widget memory.

---

## Worker

**Heartbeat:** 300s  
**Writes:** `workspace/_tmp/{widget}/draft.md`

### [First run] — no directive in messages

```
1. git pull --rebase  (in workspace/)
2. Read workspace/.gnap/agents.json  → confirm status: active
3. Read workspace/.gnap/tasks/       → find todo task assigned to "worker"
4. Set task state → "in_progress", commit + push
5. Read workspace/.gnap/messages/    → no directives for "worker"
6. List all source files under workspace/packages/pluggableWidgets/{widget}/
7. For each source file, check imports — follow local workspace packages (present in workspace/packages/); skip external npm packages
8. For each source file, create a section in workspace/_tmp/{widget}/draft.md answering:
   1. What is the purpose of this file?
   2. What kind of logic is described in this file?
   3. What part of behavior can be documented from this file?
   4. Is it user-facing?
   5. What new did you learn from this file?
   Each section: max 2 paragraphs, 256 words total.
9. Set task state → "review", commit + push
10. Stop — wait for next heartbeat
```

### [Follow-up run] — directive found in messages

```
1. git pull --rebase  (in workspace/)
2. Read workspace/.gnap/agents.json  → confirm status: active
3. Read workspace/.gnap/tasks/       → find todo task assigned to "worker"
4. Set task state → "in_progress", commit + push
5. Read workspace/.gnap/messages/    → directive found for "worker"
6. Read the directive — it lists specific files or sections needing deeper analysis
7. Re-read the named files, update the relevant sections in workspace/_tmp/{widget}/draft.md
8. Mark the directive message as read_by: ["worker"], commit
9. Post a status message confirming gaps were addressed ({N} = max existing message id + 1):
```

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

```
10. Set task state → "review", commit + push
11. Stop — wait for next heartbeat
```

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

### Per-iteration workflow

```
1. git pull --rebase  (in workspace/)
2. Read workspace/.gnap/agents.json  → confirm status: active
3. Read workspace/.gnap/tasks/       → find tasks in "review" state
4. Read workspace/_tmp/{widget}/draft.md
5. Evaluate draft (see criteria below)
6. If gaps found → send directive to Worker, return task to "todo"
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

Create `workspace/.gnap/messages/{N}.json` (`{N}` = max existing message id + 1):

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

Set task `state` back to `todo` and increment `review_cycles` in the task file.

### If satisfied (consensus reached)

1. Create `workspace/.gnap/tasks/SY-{N}.json` (synthesize task, `{N}` = max existing SY task id + 1):

```json
{
  "id": "SY-{N}",
  "title": "Write spec: {widget}",
  "assigned_to": ["spec-writer"],
  "state": "todo",
  "priority": 1,
  "created_by": "reviewer",
  "created_at": "{ISO-timestamp}",
  "updated_at": "{ISO-timestamp}",
  "desc": "Read workspace/_tmp/{widget}/draft.md and produce a clean spec at workspace/spec-alpha/{widget}.md.",
  "parent": "EX-{N}",
  "tags": ["synthesize", "{widget}"]
}
```

2. Set extract task `state` → `done`
3. Post a status message notifying the Spec Writer (`{N}` = max existing message id + 1):

```json
{
  "id": "{N}",
  "from": "reviewer",
  "to": ["spec-writer"],
  "at": "{ISO-timestamp}",
  "type": "info",
  "text": "EX-001: draft approved for accordion-web. Synthesize task SY-001 is todo."
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

### Per-iteration workflow

```
1. git pull --rebase  (in workspace/)
2. Read workspace/.gnap/agents.json  → confirm status: active
3. Read workspace/.gnap/tasks/       → find in_progress synthesize task (priority) or todo synthesize task
4. Set task state → "in_progress", commit + push
5. Execute (see below)
6. Set task state → "done", commit + push
```

### Execution

1. Read `workspace/_tmp/{widget}/draft.md` in full
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
backlog → todo → in_progress → review → done
            ↑                   │
            └───────────────────┘  (reviewer rejects)
            │
         blocked → todo              (unblocked)
            ↓
         cancelled
```

| State | Meaning |
|-------|---------|
| `backlog` | Not yet prioritized |
| `todo` | Prioritized, waiting for agent to pick up |
| `in_progress` | Agent is working on it |
| `review` | Work done, waiting for review |
| `done` | Completed (terminal) |
| `blocked` | Cannot proceed (see `blocked_reason`) |
| `cancelled` | Will not be done (terminal) |

Reverse transitions:
- `review → todo` — reviewer rejects, agent reworks
- `blocked → todo` — unblocked, agent picks up again

### This Loop

```
[extract/EX-NNN]
  todo → in_progress (Worker)
        → review (Worker, draft complete)
        → todo (Reviewer returns for gaps)            ← up to 10 cycles
        → done (Reviewer satisfied)
        → blocked (10 cycles exceeded)

[synthesize/SY-NNN]   created by Reviewer on consensus
  todo → in_progress (Spec Writer)
        → done
```

## Communication Reference

| From | To | File | Type | Purpose |
|------|----|------|------|---------|
| Reviewer | Worker | `workspace/.gnap/messages/{N}.json` | `directive` | List gaps to address |
| Reviewer | Spec Writer | `workspace/.gnap/tasks/SY-{N}.json` | task | Trigger synthesis |
| Reviewer | Human | `workspace/.gnap/messages/{N}.json` | `alert` | Escalate blocked widget |
