# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

GNAP (Git-Native Agent Protocol) — a specification for coordinating AI agent teams using git as the only transport/storage layer. No server required.

**Status:** Draft v4, March 2026. This is a protocol spec repo, not a traditional software project.

## Architecture

```
AgentHQ (Application Layer)  — budgets, dashboards, workflows (separate repo)
GNAP (Protocol Layer)        — 4 entities: Agent, Task, Run, Message
Git  (Transport)             — push/pull/commit = the message bus
```

**Protocol (GNAP core):** Defined in `README.md`. Four JSON entities live in `.gnap/` — `agents.json`, `tasks/*.json`, `runs/*.json`, `messages/*.json`. Protocol version in `.gnap/version`.

## Key Files

| File | Purpose |
|------|---------|
| `README.md` | GNAP protocol spec — the 4 entities, schemas, state machines, transport, onboarding |
| `ONBOARDING.md` | Step-by-step agent onboarding guide (includes workspace clone step) |
| `WORKFLOW.md` | Per-agent role workflows for the spec-extraction loop (Worker, Reviewer, Spec Writer) |
| `setup.md` | Spec-extraction loop design: goal, agents, task state flow, artifacts, bootstrap |
| `CONTRIBUTING.md` | Contribution guidelines |
| `examples/.gnap/` | Reference data: `agents.json`, `tasks/`, `runs/`, `messages/`, `version` |
| `docs/gnap-overview.svg` | Visual diagram embedded in README hero section |
| `workspace/` | Clone of `iobuhov/web-widgets` — the widget source repo being processed |
| `workspace/.gnap/` | Live GNAP state: 53 extract tasks (EX-001…EX-053), agent roster, runs, messages |

## Core Protocol Concepts

**Task states:** `backlog → ready → in_progress → review → done` (also `blocked`, `cancelled`). Reverse transitions: `review → in_progress` (reject), `blocked → ready` (unblocked).

**Commit convention:** `<agent-id>: <action> <entity> [details]` — git history IS the audit log.

**Heartbeat loop:** Each agent polls on `heartbeat_sec` interval (default 300s): pull → check status → check tasks → check messages → work → commit → push.

**Conflict resolution:** git merge/rebase. On conflict: pull + rebase + retry push (max 3).

## Spec-Extraction Loop

This repo runs a Ralph loop that extracts widget specifications from `workspace/` (iobuhov/web-widgets). Three agent roles coordinate via `workspace/.gnap/`:

- **Worker** — explores widget source files, fills `workspace/_workspace/{widget}/draft.md`
- **Reviewer** — reviews drafts, sends directives to Worker, creates synthesize tasks on consensus
- **Spec Writer** — reads approved draft, writes `workspace/spec-alpha/{widget}.md`

See `WORKFLOW.md` for per-agent workflows and `setup.md` for full design.

## Editing Guidelines

- This is a spec repo. Changes to `README.md` change the protocol definition.
- JSON schemas in the spec docs are normative — keep entity examples consistent across all docs.
- The four GNAP entities (Agent, Task, Run, Message) are intentionally minimal. Resist adding new protocol-level entities.
- `examples/` should always be valid instances of the schemas defined in the spec.
- Operational concerns (budget enforcement, stall detection, concurrency control) belong in AgentHQ, not the protocol.
- `workspace/` is a git submodule-style clone — commit changes to it separately and push to `iobuhov/web-widgets`.
