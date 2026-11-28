---
name: impl-journal
description: >
  Keep one implementation journal per run — the durable record of the decisions the plan did not
  make, the time each execution took, plus what shipped. Use for "journal this run", "start the
  implementation journal", "record the decisions we made", "where is the decision log". Not for
  generic notes (note-write), not for searching notes (note-search), not for review or QA findings
  (defect-report), not for design docs (design-write).
---

An implementation journal is one file per run, written by one writer, holding the durable record of
decisions the plan did not make, of the time each execution took, and of what shipped. The plan and
the design say what is supposed to happen; the journal says what actually happened when execution
met something the plan had left open, in the order it happened and with the evidence that it did.

## When a run journals

- A `code` run journals when it has more than two tasks or more than one project.
- A `task-graph` apply journals on every apply, however small.

## Where it lives

| Run | Path | Lifecycle |
|---|---|---|
| `code` run | `.notes/code/{YYYY}-{MM}-{DD}[-{TICKET}]-{slug}/{stem}-journal.md` | Never committed. Archived by the one `mv` that moves the run directory, so the stem travels with it. |
| `task-graph` apply | `<change dir>/journal.md` | Committed with the change and travels through `openspec archive`. |

Both use the note frontmatter shape: `tags`, `projects`, `created`, `updated`, `related`,
`description`. `related:` entries are sibling-relative, so moving or archiving the directory keeps
them valid.

## One writer

The journal belongs to the run's orchestrator — the session that dispatches the work — and to
nobody else.

- A node or subagent reports a decision in its result and never edits the file; the orchestrator
  transcribes it.
- A read-only reviewer holds no `edit` permission, so its findings arrive in chat and the
  orchestrator transcribes them too.

One writer per file is what keeps the numbering and the state columns coherent while nodes run in
parallel.

## What goes in

An entry is a decision execution forced, one of:

- a library or interface choice the design left open;
- a deviation from the design;
- a discovery that changes how a node executes.

Nothing else: no narration, no diffs, no restatement of the plan. `design.md` (or the design note)
holds the decisions made before apply; this file holds the apply-time ones. A gate verdict is not
an entry — its durable record is the marker or summary on the task line — though a decision that
rests on a verdict states the finding itself, never a path to it.

## Execution log

A `task-graph` apply also journals time: every dispatch of a node gets one row in `## Execution
log`, holding the node id, its agent, the start and end timestamps, and the minutes spent.

- Timestamps are the literal output of `date '+%Y-%m-%d %H:%M:%S %Z (%z)'`, printed when the
  dispatch starts and again when its tasks are marked.
- Minutes are end minus start, in whole minutes, rounded to the nearest; anything under half a
  minute is `0`.
- A fix-round resume is a new dispatch: its own row, same node id. Parallel dispatches each keep
  their own row.
- The log is not a decision: it carries no `D` id and never replaces the marker on the task line.

A `code` run has no execution log — its unit is the change unit, recorded in `Applies`. The table
is append-only, like the rest of the file.

## Ids

- `D<n>` — a decision, in report order across the run.
- `R<round>.F<n>` — a review finding: `round` is the review round, `n` is in-round.
- `Q<n>` — a QA defect, sequential across every QA pass, in report order.

Numbering runs sequentially across the run and is never restarted per pass. The journal is
append-only: never renumber a row and never rewrite one. A decision that supersedes an earlier one
is a new `D` entry carrying `Supersedes D<n>`.

## Write boundaries

`write` the whole file — never patch single cells — at:

- plan set;
- each change unit closed;
- the change-set close, or a resume.

Read it at a resume and before the final report. Rows accrue between writes; the write replaces the
file with the accumulated state.

## Skeleton

```markdown
---
tags: [code]
projects: [<project>]
created: {YYYY-MM-DD}
updated: {YYYY-MM-DD}
related: [recon-<project>.md, evidence-<project>-<owner>.md]
description: <change> — implementation journal
---
# <change title>
<intent, two lines> · status: shipped | partial | blocked
## Status
| Task | Project | Owner | State | Evidence |
|---|---|---|---|---|
| T1 | <project> | <owner> | done | evidence-<project>-<owner>.md |
## Applies
| # | Change unit | Date | Source | Result | Commits |
|---|---|---|---|---|---|
| 1 | <unit> | <date> | `ses_…` / `node <id>` | shipped | `abc1234` |
## Execution log
| Node | Agent | Start | End | Minutes |
|---|---|---|---|---|
| 1 | golang-eng | 2026-11-21 09:12:04 WIB (+0700) | 2026-11-21 09:58:37 WIB (+0700) | 47 |
## Decisions
| # | Decision | Rejected | Why | Effect | Source |
|---|---|---|---|---|---|
| D1 | <choice> | <alternative> | <why it lost> | <path or task id> | `ses_…` / `node <id>` |
## Coordination log
| When | Source | Agent | Project | Task | Brief |
|---|---|---|---|---|---|
| plan | `ses_…` / `node <id>` | <owner> | <project> | T1 | <anchor> |
## Findings ledger
| Round | ID | Tier | Location | Owner | State |
|---|---|---|---|---|---|
| 1 | R1.F1 | 🔴 bug | <path>:<line> | <owner> | fixed |
## Outcome
<what shipped, what was deferred, user follow-ups>
```

In every `Source` cell a `code` run writes the producing `ses_…`; a `task-graph` apply writes
`node <id>`. Under a `task-graph` apply `related:` names `tasks.md` and `design.md` (as the example
journal does) and omits the evidence files. Omit a section the run has no content for — never leave
an empty table. Under a `task-graph` apply the Findings ledger stays empty or omitted: gate verdicts
live on the task line, not in a table.

## Rules

- One writer per file.
- `.notes/` never enters a commit and sits outside review. A change-dir journal is committed and
  sits outside `apply.tracks`.
- A PR or MR body composes from `Applies` and the findings ledger.
- The final `status:` is one of `shipped`, `partial` or `blocked`.
