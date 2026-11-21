---
name: task-graph
description: >
  Draw an OpenSpec `task-graph` plan as the one standard ASCII layer board: layers, node state,
  owner, task progress, dependencies, gates, the next batch and the critical path. Use when the
  user says "show the task graph", "plan board", "where is this change in its plan", "what can run
  next", or "visualise the plan". Reads `tasks.md` and nothing else. Not for code call paths (use
  `call-graph`), not for searching notes (use `note-search`), and not for writing or editing the
  plan.
---
# Task graph

One canonical picture of a change's plan, so every session and every change is drawn the same way.
Plain text: it has to read on an SSH session, in a diff, and pasted into a ticket.

The plan is `tasks.md`, written to the `task-graph` OpenSpec schema's grammar. That file is the only
input. Nothing here validates the plan — it renders what the file says.

## When invoked

Resolve the change:

- A change was named: use it.
- No name: find `openspec/changes/*/tasks.md` (or `.openspec/changes/*/`) carrying the
  `<!-- openspec-graph: 1 -->` marker. If several have unfinished tasks, or none does, ask once
  which change the user means.

Then read `tasks.md` in full. Never infer a node, an edge, an owner or a TC that the file does not
contain.

## Derive each node

Fields come straight from the file:

| Field | Source |
|---|---|
| layer | `1 + max(layer of each `Depends on:` id)`; roots are layer 1 |
| state | see the table below |
| owner | the agent after the em dash in the `###` heading |
| progress | `[x]` tasks / total tasks under that node |
| deps | its `Depends on:` ids, `None` for roots |
| gate | `Gate:` when it is `review`, `qa` or `human` |

| State | Condition |
|---|---|
| `done` | every task under the node is `[x]` |
| `running` | some `[x]` and some not |
| `failed` | a task carries `(failed: …)` |
| `skipped` | the heading carries `(skipped: …)` |
| `blocked` | a `Depends on:` node is not `done` (a `skipped` dependency counts as satisfied) |
| `ready` | no `[x]`, every dependency `done`, no `(failed: …)` |

Glyphs: `▪` ready, `▸` running, `✓` done, `⊘` blocked, `✕` failed, `—` skipped.

## Emit the board

One fenced block, no prose around it:

```text
<change> · <N> nodes · <L> layers · <R> ready · <U> running · <B> blocked
L1  ▪ 1  <title>                          <owner>  <done>/<total>  ←None
L2  ⊘ 2  <title>                          <owner>  0/1  ←1   {review}
    ▪ 3  <title>                          <owner>  0/2  ←1,2
next: <ids or none> · deferred: <ids or none> · blocked: <ids or none> · critical path 1→2→3
```

- Header: omit state counts that are zero; always print nodes and layers.
- One line per node, sorted by `(layer, id)`. `L<n>` in the 4-column left field, printed only on the
  first row of its layer and blank on continuation rows.
- Columns: state glyph, id, title, owner, tasks done/total, `←` dependencies, `{gate}` only for
  `review`, `qa` and `human`.
- Titles truncate to 36 characters with `…`; pad the owner column to the longest owner; keep every
  row within 80 columns, dropping the gate tag before the dependencies.
- Footer: `next` is the ready set in `(layer, id)` order, capped at 4 and skipping a node that shares
  an `Owns:` path or a `Resource:` tag with one already listed — say which nodes the cap or a clash
  deferred. `blocked` lists nodes whose dependencies are unfinished. `critical path` is the longest
  dependency chain, drawn as `1→2→3→4`.
- No colour, no box drawing, no Mermaid. The board has to survive `cat`, a narrow terminal and a
  copy-paste into a ticket.

## Expand a node

When the user asks about one node, print its heading, owner, evidence path and gate first, then the
tasks one per line:

```text
<glyph> 2.1 <task text> · Covers: TC-…, TC-…
```

Evidence is scratch, so its file may already be gone after archive or cleanup; print the path as
written anyway.

## Rules

- The file is the truth. If the marker is missing or `tasks.md` does not exist, say so and stop —
  never substitute a guessed plan.
- No arithmetic beyond what the tables above state. Do not reorder the plan, invent a batch policy,
  or hide nodes behind a filter the user did not ask for.
- Read-only: never edit `tasks.md` or any other file from this skill.
