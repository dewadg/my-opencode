---
status: draft
created: 2026-11-21
updated: 2026-11-21
tags: [openspec, graph, tasks, schema, workflow, validator, spec-driven]
projects: [my-opencode, stocko, artifact]
related:
  - ../../karambyte/stocko/.notes/2026-11-21-graph-engineering-openspec.md
description: Task-graph schema for OpenSpec changes — group nodes, Depends on edges, Owns/Resource conflict sets, gates, computed layers and run batches, generated graph.md; no waves, no schema fork.
---

# OpenSpec task graph — tasks.md schema v1

> The canonical definition of the `tasks` artifact for graph-driven OpenSpec changes. Proposal,
> design and spec artifacts are unchanged.

## Context

OpenSpec's `tasks` artifact is a flat checklist: `apply.tracks: tasks.md` and `openspec instructions
apply --json` see individual checkboxes with no group, dependency or owner information. The work
already has a graph shape — groups of work, dependencies, file ownership, verification gates — but it
lives in prose that no tool can validate. This spec defines a strict, Markdown-native grammar for that
graph so it stays the single source of truth OpenSpec already tracks, while a small CLI can check it,
compute layers, pack run batches and render a view.

## Goals

- A `tasks.md` that is both human-reviewable and machine-parseable, with no second file to drift.
- Nodes, edges, write sets, resource conflicts and gates as explicit fields.
- Layers and parallel batches computed per run; nothing scheduled by hand.
- A validator with stable error codes, a `next` command that returns the runnable batch, and a
  generated visual.
- Forward-only adoption: marked files are enforced; legacy and archived files are untouched.
- No schema fork, no new artifact types; `proposal.md`, `design.md` and `specs/**` stay as they are.

## Non-goals

- Conditional edges, loops, retries or a durable run store. v1 is an acyclic graph per change.
- Cross-change or portfolio edges (capability → change → ticket).
- A GUI, daemon or database; commands are read-only except `render`.
- Rewriting archived changes.
- Replacing OpenSpec's checkbox progress or archive gate.

## Flow

```mermaid
flowchart LR
  T["tasks.md"] --> P["parse"]
  P --> C{"check"}
  C -->|errors| F["exit 1"]
  C -->|ok| L["layers"]
  L --> N["next: pack ready set"]
  N --> A["apply: one agent per node"]
  A --> G{"Check + gate"}
  G -->|pass| D["tick, node done"]
  G -->|fail| R["(failed: reason)"]
  D --> N
  D --> X["archive"]
  P --> V["render graph.md"]
```

## Decisions

### D1: A node is a group of work
- **Choice**: one `### N. <title> — <agent>` group is one graph node; `N.x` checkboxes are its
  internal sequence.
- **Why**: the group is the unit a subagent takes, the unit that declares `Owns:`, and the unit that
  runs in parallel. Task-level nodes would mirror the 32 booleans OpenSpec already stores without
  adding a scheduling decision.
- **Alternatives**: task-per-node (noisy, duplicates progress); hybrid with cross-group task edges
  (rejected — split the group instead).

### D2: Waves are deleted
- **Choice**: no `## Wave` headings. Layers are computed from `Depends on:`; each run's batch is
  computed from conflicts plus a concurrency cap.
- **Why**: a wave duplicates the edge information and can disagree with it. Parallelism is a run-time
  condition (DB, ports, build cache, agent capacity), not a property of the plan.
- **Alternatives**: authored batches with a `wave >= layer` invariant (rejected — second ordering
  source); pure topological rendering with no cap (rejected — unbounded fan-out).

### D3: The graph is derived from Markdown
- **Choice**: parse nodes from headings and metadata lines in `tasks.md`; no sidecar file, no fenced
  YAML block.
- **Why**: `apply.tracks` needs the checkboxes, and a duplicate representation drifts. The file is
  already the progress source; make it the structure source too.
- **Alternatives**: `graph.yaml` sidecar; YAML block inside the file; YAML-only tasks file (breaks
  `apply.tracks` and archive validation).

### D4: `graph.md` is a generated view, not a schema artifact
- **Choice**: `openspec/changes/<change>/graph.md`, written by `render`, committed, stamped with the
  source hash.
- **Why**: registering it as an artifact would let a stale render read as complete, and apply never
  revisits artifact creation.
- **Alternatives**: schema artifact with `requires: [tasks]`; CI-only comment (not reviewable).

### D5: Opt-in marker
- **Choice**: the first line `<!-- openspec-graph: 1 -->` opts a file into the grammar. Files without
  it are ignored by `check`; archives are never rewritten.
- **Why**: 148 archived changes exist; a migration is not the point. Forward-only adoption.
- **Alternatives**: enforce everywhere (baseline noise, rewrites); a config flag (per-repo, not
  per-change).

### D6: No schema fork
- **Choice**: keep `schema: spec-driven`; `tasks.md` keeps its path, `apply.tracks` and checkboxes.
- **Why**: the artifact DAG is fine for documents; only the content of the tasks artifact changes.
- **Alternatives**: a `graph-driven` schema with `contracts.md`/`test-plan.md` (rejected — more
  artifacts, no execution semantics gained).

### D7: Review and QA are explicit nodes
- **Choice**: `Gate: review` and `Gate: qa` nodes live in `tasks.md` with an `Evidence:` path.
- **Why**: a gate that exists only in prose cannot be checked; `glab53`-style manual gates work but
  are invisible to tooling.
- **Alternatives**: implicit apply steps (invisible, drift); schema artifacts (file existence is not
  a verdict).

### D8: `Covers: all` is reserved for gates
- **Choice**: only `Gate: review` and `Gate: qa` nodes may write `Covers: all`.
- **Why**: those nodes inspect the whole diff; an implementation node claiming `all` hides what it
  actually covers.
- **Alternatives**: forbid `all` (forces enumerating every TC on a review node, which adds no
  information); allow anywhere (coverage becomes meaningless).

## Configurations

### 1. Opt-in marker

First line of the file, exact:

```
<!-- openspec-graph: 1 -->
```

A different version number is error `G001`. A file without the marker is not parsed.

### 2. File shape

```
<!-- openspec-graph: 1 -->

# Tasks

### <id>. <title> — <agent>
Depends on: <none | id, id, …>
Owns: <none | path, path, …>
Resource: <tag, tag, …>          # optional
Gate: <unit | review | qa | human | none>   # optional, default unit
Evidence: <repo-relative path>   # required when Gate is review, qa or human

- [ ] <id>.<n> <text> Check: <command> Covers: <TC-…, TC-… | all>
```

Nodes may appear in any order; ids must be continuous from 1. `# Tasks` is the only required heading;
`##` headings are ignored. Fenced code blocks are ignored while parsing.

### 3. Node fields

| Field | Required | Grammar | Default | Error if |
|---|---|---|---|---|
| heading | yes | `### <id>. <title> — <owner>`; owner is kebab-case | — | missing `—`, non-numeric id, duplicate id |
| `Depends on:` | yes | `none` or `id` (`, ` `id`)* | — | unknown id, prose, self-edge, cycle |
| `Owns:` | yes | `none` or path (`, ` path)* | — | empty |
| `Resource:` | no | tag (`, ` tag)*; tag `[a-z][a-z0-9:-]*` | none | malformed tag |
| `Gate:` | no | `unit` \| `review` \| `qa` \| `human` \| `none` | `unit` | unknown value |
| `Evidence:` | gate ≠ unit | repo-relative path | — | missing path for review/qa/human |
| task | ≥1 per node | `- [ ] <id>.<n> <text> Check: <cmd> Covers: <ids>` | — | missing `Check:`, missing `Covers:`, duplicate task id |

Task line parsing: split on the **last** occurrence of ` Covers: `; the text before it must contain
exactly one ` Check: `, which splits the description from the command. Backticks around a command are
stripped but not required. `[x]` means done; every other marker is unfinished (OpenSpec rule).

### 4. Semantics

**Edges.** `Depends on:` is the only edge source. A node may start when every dependency is `done`.
Transitive dependencies are implicit; listing a transitive dependency is warning `W001`.

**Layers.** `layer(n) = 1 + max(layer(dep))`; roots are layer 1. Computed, never written.

**States.**

| State | Condition |
|---|---|
| `blocked` | some dependency is not `done` |
| `ready` | dependencies `done`, no task touched |
| `running` | some task `[x]`, or a task carries `(blocked: …)` |
| `done` | every task `[x]` and the gate is satisfied |
| `failed` | a task carries `(failed: …)`, or the gate evidence records a failure |
| `skipped` | heading carries `(skipped: <reason>)`; its boxes tick with the same reason |

**Batches.** Deterministic packing:

```
ready  = nodes with state == ready
batch  = []
for node in ready sorted by (layer, id):
    if len(batch) == max:            defer(node, "max")
    elif owns_overlap(node, batch):  defer(node, "owns")
    elif resource_overlap(node, batch): defer(node, "resource")
    else: batch.append(node)
```

`owns_overlap` compares resolved paths: equal, one a directory prefix of the other, or both in the
same Go package directory. `resource_overlap` compares tag sets. Default `max` is 4; `--max 1`
serializes. A deferred node is eligible, not blocked; `next --explain` names the reason.

### 5. Invariants

Errors (exit 1):

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
| `G010` | `Covers:` names a TC absent from `design.md` |
| `G011` | a TC in `design.md` is covered by no node |
| `G012` | `Covers: all` on a non-gate node |
| `G013` | `review`/`qa`/`human` gate without `Evidence:` |
| `G014` | `review`/`qa` node does not depend on every implementation node it inspects |
| `G015` | doc obligation missing: migration → `docs/database.md`; sync jobs → `docs/background-jobs.md`; new domain → `docs/architecture.md` + `doc.go` |
| `G016` | `graph.md` source hash does not match `tasks.md` |

Warnings (exit 0): `W001` redundant transitive dependency; `W002` node always deferred by an
`Owns:`/`Resource:` clash; `W003` node depends on a higher id (readability only).

`check --changed` scopes every rule to files in the diff that carry the marker.

### 6. graph.md

Path `openspec/changes/<change>/graph.md`. First lines:

```
<!-- generated by openspec-graph render; do not edit -->
# Graph — <change>
Source: tasks.md@<sha256-of-tasks.md, first 12 hex> · Nodes: N · Edges: E · Layers: L · Ready: R
```

Required sections, in order: `## Graph` (Mermaid flowchart), `## Layers` (table: layer, node, owner,
gate, depends on, state), `## Next batch` (table: node, owner, why, feeds; plus deferred and blocked
lines), `## Conflicts` (pairs: owns, resource, max), `## Coverage` (TC → nodes), `## Critical path`.

Determinism: nodes sorted by id, edges by `(from, to)`, tables by id, so `render --check` produces an
empty diff when nothing changed. Mermaid labels are quoted, `;` and `#` stripped, `end` never used as
an id; run `mmdc` when installed.

### 7. CLI contract

`bin/openspec-graph` — Node, no runtime dependencies.

| Command | Output | Exit |
|---|---|---|
| `check <change> [--changed] [--json]` | `{errors:[{code,node,line,message}], warnings:[…]}` | 1 on errors |
| `next <change> [--max N] [--explain] [--json]` | `{batch:[id], deferred:[{id,reason}], blocked:[{id,waitingOn}], ready:[id]}` | 0, or 1 if no marker |
| `render <change> [--check]` | writes `graph.md`, or diffs with `--check` | 1 on drift or parse error |
| `baseline [--archived] [--json]` | counts per code over unmarked files | 0 always |

Canonical parsed model (shared by all commands):

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

### 8. OpenSpec integration

- `tasks.md` stays at `openspec/changes/<change>/tasks.md`; `apply.tracks` and the checkbox format are
  unchanged.
- `openspec validate --archived` remains the completion gate; `openspec status` remains the artifact
  gate.
- `openspec/config.yaml` `rules.tasks` describes the grammar; apply-time doctrine goes in
  `operations.apply.guidance`, since `rules` is injected only into `instructions <artifact>`.
- `graph.md` is committed and travels with the change when archived.
- No change to `proposal.md`, `design.md` or `specs/**`.

## Testing

### Pre-test requirements

- Fixture directory `test/fixtures/graph/`, no database, no network.
- One valid 5-node fixture and one fixture per error code.

### Scenario

| Given | When | Then |
|---|---|---|
| valid 5-node fixture | `check` | exit 0, no errors |
| node with `Depends on: Wave 1` | `check` | `G007`, line reported |
| dependency on a missing id | `check` | `G005` |
| 1 → 2 → 1 | `check` | `G006` |
| two ready nodes with overlapping `Owns:` | `next` | second is `deferred: owns` |
| two ready nodes share `Resource: db:test` | `next` | second is `deferred: resource` |
| 6 ready nodes, `--max 2` | `next` | 2 in batch, 4 `deferred: max` |
| `Covers: all` on `Gate: unit` | `check` | `G012` |
| review node without `Evidence:` | `check` | `G013` |
| `graph.md` older than `tasks.md` | `render --check` | exit 1, hash mismatch |
| file without the marker | `check` | skipped, not an error |
| unmarked legacy `tasks.md` | `baseline` | counted, never fails |

## Open Questions

- [ ] Where does the validator live when implemented — `bin/openspec-graph.mjs` per repo, or one
  shared package? Blocks P0 only.
- [ ] Do `Resource:` tags stay free-form, or become a fixed vocabulary (`db`, `port`, `build`,
  `browser`)? Free-form is the default; an observed clash can promote a tag.
- [ ] Default `--max`: 4, or derived from the number of available implementation agents?
- [ ] Should `graph.md` be committed for every marked change, or only for cross-domain changes?
  Current default: every marked change, so `render --check` has something to compare.

## Todo

Not started — this spec documents the format first. Wave headings below are this spec's delivery
layers, not the change-task format described above.

### Wave 1 — Parser and checker
- [ ] [js-eng] Implement `bin/openspec-graph.mjs` parser + `check` with codes `G001–G015`, `verify` against fixtures — `test/fixtures/graph/`
- [ ] [js-eng] Unit tests for every code and for the valid fixture — same directory

### Wave 2 — Compute and render
- [ ] [js-eng] Add `next` (layers, states, deterministic packing, `--max`, `--explain`) and `render`/`--check` with `graph.md` — `bin/openspec-graph.mjs`
- [ ] [js-eng] Tests for packing order, deferral reasons, determinism and hash staleness — `test/fixtures/graph/`

### Wave 3 — Adoption
- [ ] [general] Wire `rules.tasks`, `operations.apply.guidance`, `make graph-check` and CI `check --changed` + `render --check` in the adopting repo
- [ ] [code-reviewer] Review the CLI and the first marked `tasks.md` against this spec
