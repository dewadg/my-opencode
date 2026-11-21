# task-graph schema

An OpenSpec workflow schema that replaces the flat task checklist with an explicit work graph.
`specs` is unchanged from `spec-driven`; `design` carries the project's design rules;
`proposal.md` is dropped; `tasks.md` is the graph, `graph.md` is its one-shot snapshot, and
`journal.md` is the apply-time decision log.

## Install

OpenSpec resolves schemas at `<project>/openspec/schemas/<name>/`. This directory is the staging
copy; a dot-directory is not discoverable by the CLI, so install with a plain recursive copy.

```bash
cp -R <this-directory> <project>/openspec/schemas/task-graph
```

Then either:

```bash
openspec new change <name> --schema task-graph
```

or set it as the project default in `openspec/config.yaml`:

```yaml
schema: task-graph
```

Either way, add both gate-evidence patterns to the project's `.gitignore` once, since a project may
root its change dir at either `openspec/` or `.openspec/`. Evidence is scratch output, so it must
never enter the repository:

```
openspec/changes/**/evidence/
.openspec/changes/**/evidence/
```

Artifacts: `specs` and `design` are roots; `tasks` requires both; `graph` requires `tasks`.
`apply.requires` is `[tasks]`, and `apply.tracks` stays `tasks.md`, so OpenSpec's checkbox progress
and archive gate are unchanged. The `graph` artifact is kept for the template, but `graph.md` is a
plan snapshot — it is written once, right after `tasks.md`, and nothing gates on it.

The `task-graph` skill draws the plan as the board below. It is the standard view; nothing else
reads `tasks.md`.

## Design artifact

`design.md` carries the project's design rules, not just the upstream headings. Required sections:

- Context, Goals / Non-Goals
- Flow — mermaid when a flow, schema or state changes; IDE-safe types only (flowchart,
  sequenceDiagram, stateDiagram-v2, erDiagram), quoted labels, no `;`/`#`, never `end` as an id
- Decisions
- Shared contracts — types, interfaces, migrations, API shapes, packages touched, and every new or
  renamed foreign key / id-holding column or payload field under the project's naming rule
- Test scenarios — `TC-…` table with scenario, given, when, then. Each row cites the requirement
  name it proves; the spec deltas cite requirement names, never `TC-…` ids, because `specs` and
  `design` are both roots and a delta's id reference could name a row that does not exist yet
- Risks / Trade-offs, Migration Plan
- Docs to update — every doc the change touches, at the project's paths; `none` when nothing changes
- Open Questions

Rules: code flow is explained with the `call-graph` skill; scenarios come from the `test-scenario`
skill; HTTP tests come from the `hurl-write` skill when the project uses hurl, with change-only
checks in the project's scratch location and lasting asserts in its regression file; tests run in
the local environment only, never staging or production.

## `tasks.md` grammar

First line, exact:

```
<!-- openspec-graph: 1 -->
```

A file without the marker is not a plan; nothing reads it, and legacy or archived changes are left
alone.

```
# Tasks

### <id>. <title> — <agent>
Depends on: <none | id, id, …>
Owns: <none | path, path, …>
Resource: <tag, tag, …>                    # optional
Gate: <unit | review | qa | human | none>  # optional, default unit
Evidence: evidence/<file>.md               # required for review, qa, human; scratch, never committed

- [ ] <id>.<n> <text> Check: <command> Covers: <TC-…, TC-… | all>
```

- Node ids are unique positive integers. Gaps are allowed: inserting a node never renumbers the
  rest, so existing ids stay stable for evidence and TC references.
- Task ids are prefixed by their node id. Node `2` owns `2.1`, `2.2`.
- `Depends on:` names node ids only. Waves, groups, ranges and prose are not edges.
- `Owns:` lists every file the node writes.
- `Resource:` names non-file conflicts (`db:test`, `port:8080`, `build`). Nodes sharing a tag never
  run together.
- `Gate:` marks the node type. `review`, `qa` and `human` require `Evidence:`.
- `Evidence:` points inside the change at `evidence/<file>.md`. It names scratch output, written
  while the gate verifies and never committed; see [Evidence](#evidence).
- `Covers: all` is legal only on `review` and `qa` gates.
- Checkbox parsing follows OpenSpec: `[x]` is done; every other marker is unfinished.
- `Check:` is split from `Covers:` on the last ` Covers: `; the text before it contains exactly one
  ` Check: `.

`Owns:` paths are comma-separated. `Depends on:` and `Covers:` are comma-separated. Fenced code
blocks are ignored when the file is read.

### Example

```markdown
<!-- openspec-graph: 1 -->

# Tasks

### 1. Migration and entity — golang-eng
Depends on: none
Owns: internal/db/migrations/00033_export.sql, internal/pkg/entity/export.go
Resource: db:test
Gate: unit

- [ ] 1.1 Add the migration and entity; Check: `make migrate` then `go test -race ./internal/pkg/entity/...` exits 0. Covers: TC-101

### 2. Export endpoint — golang-eng
Depends on: 1
Owns: internal/exports/service.go, internal/exports/service_test.go
Gate: unit

- [ ] 2.1 Stream a CSV for a date range; Check: `go test -race ./internal/exports/...` exits 0. Covers: TC-102

### 3. Review — code-reviewer
Depends on: 1, 2
Owns: none
Gate: review
Evidence: evidence/review-round-1.md

- [ ] 3.1 `make test` and `make lint` exit 0; findings by file and severity. Covers: all
```

## Semantics

**Edges.** `Depends on:` is the only edge source. A node may start when every dependency is `done`
or `skipped`. Transitive dependencies are implicit.

**Layers.** `layer(n) = 1 + max(layer(dep))`; roots are layer 1. Computed, never written.

**States.**

| State | Condition |
|---|---|
| `blocked` | some dependency is not `done` or `skipped` |
| `ready` | dependencies `done`, no task ticked |
| `running` | some tasks ticked and some not |
| `done` | every `<id>.<n>` task `[x]`; vacuously true for a node with no tasks |
| `failed` | a task, or the node's gate evidence, carries `(failed: …)` |
| `skipped` | heading carries `(skipped: <reason>)`; boxes tick with the same reason |

**Ready set.** The board's `next:` footer lists the ready nodes in `(layer, id)` order, capped at 4
and skipping a node that shares an `Owns:` path or a `Resource:` tag with one already listed:

```
for node in ready sorted by (layer, id):
    if len(batch) == 4:                 defer(node, "max")
    elif owns_overlap(node, batch):     defer(node, "owns")
    elif resource_overlap(node, batch): defer(node, "resource")
    else: batch.append(node)
```

`owns_overlap` compares resolved paths: equal, directory prefix, or the same Go package directory.
A deferred node is eligible, not blocked.

**Correlation.** The task id prefix is the join key between a node and its tasks. OpenSpec flattens
checkboxes to sequential ids, so correlate by line, never by that id.

## Evidence

Gate evidence is scratch. The gate's agent writes it at
`openspec/changes/<change>/evidence/<file>.md`, relative to the change directory, the planner reads
it during apply, and it exists nowhere else: never committed, never archived. The path must be
exactly `evidence/<name>.md`: no `..`, no leading `/`, no other directory.

- The project's `.gitignore` carries `openspec/changes/**/evidence/`, and the archive step empties
  it before the change is archived ([Archive](#archive)), so the repository never carries it.
- A report is prose the planner reads, not a machine artifact. The `evidence/` directory may hold
  screenshots, HTML dumps and other binaries, which is exactly why it is not persisted; the named
  `evidence/<file>.md` is the prose index the planner reads.
- The durable record of a gate is `tasks.md`, not the report: the `[x]` tick, and `(failed: reason)`
  on the task line for a failure. A failing gate must write that marker: a report alone is deleted,
  so a verdict that exists only in `evidence/` does not survive.
- A gate's verdict is summarized from its report, never recorded as a bare pointer to it. The gate
  agent reads its report and returns the substance — findings by file and severity, scenario
  pass/fail, counts, exit codes — and the record written into `tasks.md` or the journal states that
  substance. Naming `evidence/...` is backing for it, never the sole record of the verdict.
- After archive, `Evidence:` lines are historical pointers whose targets are gone by design. Do not
  rewrite, remove or renumber them.

## Journal

Apply-time decisions live at `openspec/changes/<change>/journal.md`, relative to the change,
committed and archived with it, unlike `evidence/`, which is scratch and never committed. It is
not an artifact: OpenSpec does not generate it, list it or gate on it, and `apply.tracks` stays
`tasks.md`.

`design.md` holds the decisions made before apply. The journal holds the ones apply forces: a
library or interface choice the design left open, a deviation from `design.md`, a discovery that
changes how a node executes. A gate's verdict is written into `evidence/` while the gate runs; the
record that survives is the `tasks.md` marker. A `D` entry that rests on a gate verdict states the
finding itself, not a path to it.

```markdown
<!-- openspec-journal: 1 -->
# Journal — <change>

## D1 · node 2 · <decision title>
Date: 2026-11-21 · Owner: <agent>
Decided: <what was chosen and why>.
Rejected: <alternative> — <why it lost>.
Effect: <none | spec/<path>, design.md, <contract path>, task <id>.<n>>.
```

- One writer: the applying orchestrator appends. Nodes may run in parallel, so a node's agent
  reports the decision in its result and never edits the file itself.
- Entries are `D1`, `D2`, ... in order. Never reuse a number and never edit a past entry:
  supersede it with a new one carrying `Supersedes: D<n>`.
- One entry per decision, no narration, no diffs. A decision that changes a spec or design file
  updates that file in the same node; the entry names the path.
- A node that made no decision adds no entry.

## Planning rules

- **Parallelism.** Add an edge only when the dependent cannot pass a `Check:` without the
  dependency's output, and name that output — a path, an interface, or a `TC-…`. If it cannot be
  named, drop the edge. Independent roots are the default. A change with four or more nodes should
  have at least one layer with two nodes.
- **Self-containment.** A node owns every file it writes plus its tests and fixtures. It must pass
  its checks with only its ancestors' work in the tree. Tests and docs for a node's behavior land in
  that node, never in a final catch-all.
- **Shared files.** If two nodes must write the same file, merge them or extract the file into an
  earlier contract node. Two ready nodes sharing an `Owns:` path is a planning defect, not a runtime
  condition.
- **Gates.** Review and QA nodes verify only; they do not fix. Findings reopen the owning node.

## `graph.md`

`openspec/changes/<change>/graph.md`, written once after `tasks.md`, committed, never edited and
never regenerated. It is a plan snapshot, not a live view: `tasks.md` changes during apply, and a
re-render would pin the snapshot to the hottest file in the change.

```
<!-- plan snapshot; do not edit -->
# Graph — <change>

Source: tasks.md@<sha256 first 12 hex> · <N> nodes · <L> layers · <E> edges
```

then the board as one fenced block, then `## Coverage`. The `Source:` hash is provenance only — it
records which `tasks.md` the snapshot came from.

Board grammar:

- Header: `<change> · <N> nodes · <L> layers · <R> ready · <U> running · <B> blocked`. Zero state
  counts are omitted; nodes and layers always print.
- One line per node, sorted by `(layer, id)`. `L<n>` sits in a 4-column left field, printed only on
  the first row of its layer and blank on continuation rows.
- Columns: state glyph, id, title, owner, tasks done/total, `←` dependencies (`←None` for roots),
  and a `{gate}` tag for `review`, `qa` and `human` nodes only.
- Glyphs: `▪` ready, `▸` running, `✓` done, `⊘` blocked, `✕` failed, `—` skipped.
- Titles truncate to 36 characters with `…`; the owner column pads to the longest owner; a row stays
  within 80 columns, dropping the gate tag first and the dependencies only if it still overflows.
- Footer: `next: … · deferred: … · blocked: … · critical path 1→2→3`, `none` where empty.

```text
template-resources · 5 nodes · 4 layers
L1  ✓ 1  Template content                js-eng          6/6  ←None
L2  ✓ 2  Registry, resources and tests   golang-eng      4/4  ←1
L3  ✓ 3  Guide list and docs             api-documenter  2/2  ←2
L4  ✓ 4  Review                          code-reviewer   1/1  ←1,2,3   {review}
    ✓ 5  Render and publish              qa-eng          3/3  ←1,2,3   {qa}
next: none · deferred: none · blocked: none · critical path 1→2→3→4
```

No ANSI colour, and the same `tasks.md` draws the same board. `## Coverage` maps each `TC-…` to the
nodes that name it under `Covers:`; table cells keep `\|` escaping, while the board only collapses
line breaks in a title.

## Archive

Evidence does not outlive the change, so emptying it is an archive step, not a cleanup left to
habit.

- Before `openspec archive`, empty the change's evidence directory: `rm -rf <change-dir>/evidence`.
  Delete rather than move — nothing reads it afterwards, and the durable record is `tasks.md`.
- The rule ships as project `operations.archive.guidance`, because `schema.yaml` has no archive
  hook: its loader accepts `name`, `version`, `description`, `artifacts[]` and `apply` only. The
  `openspec instructions archive` workflow injects that guidance.

```yaml
operations:
  archive:
    guidance:
      - Empty the change's evidence directory before archiving: rm -rf <change-dir>/evidence.
      - Delete evidence rather than moving it; nothing reads it after archive.
```

- A change archived with its evidence left behind is un-purged, not corrupt: the directory is
  gitignored, so it is untracked junk on disk. Delete it then; the `Evidence:` lines stay as
  historical pointers ([Evidence](#evidence)).

## Integration

- `tasks.md` stays at `openspec/changes/<change>/tasks.md`; `apply.tracks` and checkbox progress are
  unchanged.
- `openspec validate --archived` remains the completion gate.
- `journal.md` is apply-time output, not an artifact and not a tracked task file: `openspec status`,
  `validate` and archive's task gate ignore it, and it travels with the change when archived. The
  rule lives in `apply.instruction` because it is workflow grammar, not project doctrine;
  `operations.apply.guidance` still extends it per project.
- `openspec/config.yaml` `rules.tasks` describes the grammar; apply-time doctrine goes in
  `operations.apply.guidance`, because `rules` is injected only into `instructions <artifact>`. The
  schema's own `apply.instruction` stays short and points there.
- Archive doctrine goes in `operations.archive.guidance`, the sibling of
  `operations.apply.guidance`: `openspec instructions archive` injects it, and neither lives in
  `schema.yaml`.
- `graph.md` is committed and travels with the change when archived; it is never regenerated.
- `evidence/` never travels with the change: it is gitignored and emptied by the archive step
  ([Archive](#archive)), so `Evidence:` lines in an archived change point at files that are gone.
- Only the loader keys OpenSpec 1.16 accepts are used in `schema.yaml`: `name`, `version`,
  `description`, `artifacts[]` (`id`, `generates`, `description`, `template`, `instruction`,
  `requires`) and `apply` (`requires`, `tracks`, `instruction`). Unknown keys are silently stripped,
  so no gate or validator fields may be added there.
