# task-graph schema

An OpenSpec workflow schema that replaces the flat task checklist with an explicit work graph.
`specs` and `design` are unchanged from `spec-driven`; `proposal.md` is dropped; `tasks.md` is the
graph and `graph.md` is its generated view.

## Install

OpenSpec resolves schemas at `<project>/openspec/schemas/<name>/`. This directory is the staging
copy; a dot-directory is not discoverable by the CLI.

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

Artifacts: `specs` and `design` are roots; `tasks` requires both; `graph` requires `tasks`.
`apply.requires` is `[tasks, graph]`, and `apply.tracks` stays `tasks.md`, so OpenSpec's checkbox
progress and archive gate are unchanged.

## `tasks.md` grammar

First line, exact:

```
<!-- openspec-graph: 1 -->
```

A file without the marker is ignored by the checker; legacy and archived changes are never parsed.

```
# Tasks

### <id>. <title> — <agent>
Depends on: <none | id, id, …>
Owns: <none | path, path, …>
Resource: <tag, tag, …>                    # optional
Gate: <unit | review | qa | human | none>  # optional, default unit
Evidence: <repo-relative path>             # required for review, qa, human

- [ ] <id>.<n> <text> Check: <command> Covers: <TC-…, TC-… | all>
```

- Node ids are continuous from 1; task ids are prefixed by their node id. Node `2` owns `2.1`, `2.2`.
- `Depends on:` names node ids only. Waves, groups, ranges and prose are errors.
- `Owns:` lists every file the node writes.
- `Resource:` names non-file conflicts (`db:test`, `port:8080`, `build`). Nodes sharing a tag never
  run together.
- `Gate:` marks the node type. `review`, `qa` and `human` require `Evidence:`.
- `Covers: all` is legal only on `review` and `qa` gates.
- Checkbox parsing follows OpenSpec: `[x]` is done; every other marker is unfinished.
- `Check:` is split from `Covers:` on the last ` Covers: `; the text before it contains exactly one
  ` Check: `.

`Owns:` paths are comma-separated. `Depends on:` and `Covers:` are comma-separated. Fenced code
blocks are ignored by the parser.

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
Evidence: .notes/add-order-export/review-round-1.md

- [ ] 3.1 `make test` and `make lint` exit 0; findings by file and severity. Covers: all
```

## Semantics

**Edges.** `Depends on:` is the only edge source. A node may start when every dependency is `done`.
Transitive dependencies are implicit.

**Layers.** `layer(n) = 1 + max(layer(dep))`; roots are layer 1. Computed, never written.

**States.**

| State | Condition |
|---|---|
| `blocked` | some dependency is not `done` |
| `ready` | dependencies `done`, no task touched |
| `running` | some task `[x]`, or a task carries `(blocked: …)` |
| `done` | every `<id>.<n>` task `[x]` and the gate is satisfied |
| `failed` | a task carries `(failed: …)`, or gate evidence records a failure |
| `skipped` | heading carries `(skipped: <reason>)`; boxes tick with the same reason |

**Batches.** The ready set is packed deterministically:

```
ready = nodes with state == ready
batch = []
for node in ready sorted by (layer, id):
    if len(batch) == max:               defer(node, "max")
    elif owns_overlap(node, batch):     defer(node, "owns")
    elif resource_overlap(node, batch): defer(node, "resource")
    else: batch.append(node)
```

`owns_overlap` compares resolved paths: equal, directory prefix, or the same Go package directory.
Default `max` is 4. A deferred node is eligible, not blocked.

**Correlation.** The task id prefix is the join key between a node and its tasks. OpenSpec flattens
checkboxes to sequential ids; the checker correlates by source `line`, never by that id.

## Planning rules

- **Parallelism.** Add an edge only when the dependent cannot pass a `Check:` without the
  dependency's output, and name that output — a path, an interface, or a `TC-…`. If it cannot be
  named, drop the edge. Independent roots are the default. A change with four or more nodes should
  have at least one layer with two nodes.
- **Self-containment.** A node owns every file it writes plus its tests and fixtures. It must pass
  its checks with only its ancestors' work in the tree. Tests and docs for a node's behavior land in
  that node, never in a final catch-all.
- **Shared files.** If two nodes must write the same file, merge them or extract the file into an
  earlier contract node.
- **Gates.** Review and QA nodes verify only; they do not fix. Findings reopen the owning node.

## Checker contract

`bin/openspec-graph` — Node, no runtime dependencies. Planned; not implemented.

| Command | Output | Exit |
|---|---|---|
| `check <change> [--changed] [--diff [--base <ref>]] [--json]` | `{errors, warnings}`; `--diff` adds `G018` | 1 on errors |
| `next <change> [--max N] [--explain] [--json]` | `{batch, deferred, blocked, ready}` | 0, or 1 without a marker |
| `render <change> [--check]` | writes `graph.md`, or diffs | 1 on drift or parse error |
| `baseline [--archived] [--json]` | counts per code over unmarked files | 0 |

### Errors

| Code | Condition |
|---|---|
| `G001` | unknown marker version |
| `G002` | malformed node heading |
| `G003` | duplicate or non-continuous node id |
| `G004` | missing `Depends on:` or `Owns:` |
| `G005` | dependency names an unknown node |
| `G006` | cycle |
| `G007` | malformed `Depends on:` (wave, group, range, prose) |
| `G008` | duplicate task id inside a node |
| `G009` | task without `Check:` or without `Covers:` |
| `G010` | `Covers:` names a TC absent from `design.md` and the main specs (`openspec/specs/**/spec.md`) |
| `G011` | a TC in `design.md` is covered by no node |
| `G012` | `Covers: all` on a non-gate node |
| `G013` | `review`/`qa`/`human` gate without `Evidence:` |
| `G014` | `review`/`qa` node does not depend on every implementation node it inspects |
| `G015` | doc obligation missing: migration → `docs/database.md`; sync jobs → `docs/background-jobs.md`; new domain → `docs/architecture.md` + `doc.go` |
| `G016` | `graph.md` source hash does not match `tasks.md` |
| `G017` | task id prefix does not match the enclosing node id |
| `G018` | `check --diff` finds a changed file outside the union of the change's `Owns:` paths |

### Warnings

| Code | Condition |
|---|---|
| `W001` | redundant transitive dependency |
| `W002` | node always deferred by an `Owns:`/`Resource:` clash |
| `W003` | node depends on a higher id (readability only) |
| `W004` | fully serial graph: four or more nodes and every layer has width 1 |
| `W005` | edge without a visible contract |
| `W006` | node has no behavioral check and `Gate` is not `none` |
| `W007` | `Gate: unit` node covers no TC |
| `W008` | `Owns:` path shared by two nodes where neither is an ancestor of the other |
| `W009` | a task `Check:` names a path owned by a node that is not an ancestor |

## `graph.md`

`openspec/changes/<change>/graph.md`, generated by `render`, committed, never edited. It is a view,
not a source of truth.

```
<!-- generated by openspec-graph render; do not edit -->
# Graph — <change>
Source: tasks.md@<sha256 first 12 hex> · Nodes: N · Edges: E · Layers: L · Width: W · Ratio: R · Ready: N
```

Required sections, in order:

- `## Graph` — Mermaid `flowchart LR`; rectangle = node, hexagon = gate; green done, blue ready, grey
  dashed blocked.
- `## Layers` — table: layer, node, task range, owner, gate, depends on, state.
- `## Next batch` — table: node, owner, why, feeds; plus deferred and blocked lines.
- `## Conflicts` — pairs: owns, resource, max.
- `## Coverage` — TC → nodes.
- `## Critical path`.
- `## Warnings` — optional; code, node, detail.

`Width` is the largest layer; `Ratio` is `nodes / critical-path length`. Determinism: nodes by id,
edges by `(from, to)`, tables by id, so `render --check` produces an empty diff when nothing changed.
Mermaid labels are quoted, `;` and `#` stripped, `end` never used as an id.

## Parsed model

```json
{
  "schema": 1,
  "change": "add-order-export",
  "sourceHash": "7f3a1c9d2b04",
  "nodes": [{
    "id": 1,
    "title": "Migration and entity",
    "owner": "golang-eng",
    "dependsOn": [],
    "owns": ["internal/pkg/entity/export.go"],
    "resources": ["db:test"],
    "gate": "unit",
    "evidence": null,
    "layer": 1,
    "state": "ready",
    "tasks": [{"id": "1.1", "text": "…", "check": "…", "covers": ["TC-101"], "done": false}]
  }],
  "edges": [{"from": 1, "to": 2}]
}
```

## Integration

- `tasks.md` stays at `openspec/changes/<change>/tasks.md`; `apply.tracks` and checkbox progress are
  unchanged.
- `openspec validate --archived` remains the completion gate.
- `openspec/config.yaml` `rules.tasks` describes the grammar; apply-time doctrine goes in
  `operations.apply.guidance`, because `rules` is injected only into `instructions <artifact>`.
- `graph.md` is committed and travels with the change when archived.
- Only the loader keys OpenSpec 1.16 accepts are used in `schema.yaml`: `name`, `version`,
  `description`, `artifacts[]` (`id`, `generates`, `description`, `template`, `instruction`,
  `requires`) and `apply` (`requires`, `tracks`, `instruction`). Unknown keys are silently stripped,
  so no gate or validator fields may be added there.
