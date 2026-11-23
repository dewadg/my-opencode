# task-graph schema

An OpenSpec workflow schema that replaces the flat task checklist with an explicit work graph.
`specs` is unchanged from `spec-driven`; `design` carries the project's design rules;
`proposal.md` is dropped; `tasks.md` is the graph and `graph.md` is its one-shot snapshot.

Execution runs in three stages. **Build** writes the code and its tests and only compiles the
touched packages, so builds stay parallel instead of serializing on a test database. **Verify**
runs each project's suite once, from the project's own build owner, and a QA node proves the
behavioral scenarios on the running app. **Review** is a static read, concurrent with QA. Gate
findings are collected per layer and dispatched to each project's builder once, for one fix round.

## Install

OpenSpec resolves schemas at `<project>/openspec/schemas/<name>/` first, then at
`${XDG_DATA_HOME}/openspec/schemas/<name>/` (`~/.local/share` on Unix). This directory is the
staging copy; a dot-directory is not discoverable by the CLI, so install with a plain recursive
copy — project-local by default, or user-level for every project:

```bash
cp -R <this-directory> <project>/openspec/schemas/task-graph
cp -R <this-directory> "${XDG_DATA_HOME:-$HOME/.local/share}/openspec/schemas/task-graph"
```

A user-level copy serves every project and is overridden by a project-local one. Confirm with
`openspec schemas`: it lists `task-graph`, annotated `(user override)` when the userspace copy is
the one resolved.

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
- Test scenarios — `TC-…` table with scenario, given, when, then, `Requirement` and `Proof`.
  Each row cites the requirement name it proves; the spec deltas cite requirement names, never
  `TC-…` ids, because `specs` and `design` are both roots and a delta's id reference could name a
  row that does not exist yet. `Proof` is `unit | integration | api | ui | device | manual`:
  `unit` and `integration` route to the verify node; `api`, `ui`, `device` and `manual` route to QA
  (`ui` is web, `device` is mobile), so the column decides which gate proves each scenario
- Verification — per project, the migrate, build, typecheck, lint and suite commands the verify
  stage runs, or `none`; and which behavior the suite proves versus which only QA proves
- Observability — log events, levels and fields, trace spans, metric names and labels, and what must
  never be logged. Review checks this against the code instead of taste
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
<!-- openspec-graph: 2 -->
```

A file without the marker is not a plan; nothing reads it, and legacy or archived changes are left
alone. Marker `1` is the previous grammar: gate semantics changed, so a v1 file is read-only and
never rewritten under these rules.

```
# Tasks

### <id>. <title> — <agent>
Depends on: <none | id, id, …>
Owns: <none | path, path, …>
Resource: <tag, tag, …>                                # optional
Gate: <build | verify | qa | review | human | none>    # optional, default build
Evidence: evidence/<file>.md                           # required for verify, qa, review, human; scratch

- [ ] <id>.<n> <text> Check: <command> Covers: <TC-…, TC-… | all>
```

- Node ids are unique positive integers. Gaps are allowed: inserting a node never renumbers the
  rest, so existing ids stay stable for evidence and TC references.
- Task ids are prefixed by their node id. Node `2` owns `2.1`, `2.2`.
- `Depends on:` names node ids only. Waves, groups, ranges and prose are not edges.
- `Owns:` lists every file the node writes.
- `Resource:` names non-file conflicts: `db:test`, `port:<n>`, `emulator`, `metro`, `build`
  (generated clients, lockfiles). Tag only what the node actually holds; nodes sharing a tag never
  run together.
- `Gate:` marks the node type and derives its stage. Absent, it is `build`. `unit` is v1-only and
  reads as `build`; new plans never write it.
- `Evidence:` points inside the change at `evidence/<file>.md`. It is required for `verify`, `qa`,
  `review` and `human`, names scratch output written while the gate verifies, and is never
  committed; see [Evidence](#evidence).
- `Covers:` names the TCs a node implements. A `build` node names its own; `all` is legal on
  `verify`, `qa` and `review`.
- Checkbox parsing follows OpenSpec: `[x]` is done; every other marker is unfinished.
- `Check:` is split from `Covers:` on the last ` Covers: `; the text before it contains exactly one
  ` Check: `.

`Owns:` paths are comma-separated. `Depends on:` and `Covers:` are comma-separated. Fenced code
blocks are ignored when the file is read.

### Gates and stages

| Gate | Stage | Owner | Does | Writes |
|---|---|---|---|---|
| `build` (default) | 1 build | the project's builder | writes the planned code and its tests; compiles, parses or runs the narrowest structural check of the touched files, test files included | yes |
| `none` | 1 build | any | no check; a spike or a manual step | yes |
| `verify` | 2 verify | the project's build owner, resumed | runs every command in design's Verification table; fixes mechanical failures | only the paths of the build nodes it depends on |
| `qa` | 2 verify | the behavioral QA agent | the behavioral scenarios on the running app; never fixes | no |
| `review` | 3 review | the review agent | static read of the diff and code; never runs a project command | no |
| `human` | 3 review | the user | decision | no |

A build node's `Check:` is the project's compile, parse or narrowest structural check of the touched
files — a validator or a `grep` for required content where there is no compiler — test files
included, and never a behavior test. A verify node reports each command's exit code; a qa node
reports scenario pass/fail; a review node reports findings by file and severity plus confirmation
that no project command ran.

Gate nodes carry `Owns: none`. A verify node writes only inside the paths of the build nodes it
depends on, and those are never ready at the same time, so the shared-file rule holds. `qa` and
`review` both depend only on `verify` and run concurrently.

Fast path: one project and at most three nodes may put `Gate: verify` on a builder node instead of
adding a verify node; review stays separate. A fused node keeps its own `Owns:` list and writes only
those files, so the `Owns: none` gate rule does not apply to it.

### Example

```markdown
<!-- openspec-graph: 2 -->

# Tasks

### 1. Migration and entity — golang-eng
Depends on: none
Owns: internal/db/migrations/00033_export.sql, internal/pkg/entity/export.go, internal/pkg/entity/export_test.go
Gate: build

- [ ] 1.1 Add the migration and entity, with tests for the row mapping; Check: `go build ./internal/pkg/entity/` and `go test -run '^$' ./internal/pkg/entity/...` exit 0. Covers: TC-101

### 2. Export endpoint — golang-eng
Depends on: 1
Owns: internal/exports/service.go, internal/exports/service_test.go
Gate: build

- [ ] 2.1 Stream a CSV for a date range, with table tests; Check: `go build ./internal/exports/` and `go test -run '^$' ./internal/exports/...` exit 0. Covers: TC-102

### 3. Verify — golang-eng
Depends on: 1, 2
Owns: none
Resource: db:test
Gate: verify
Evidence: evidence/verify-go.md

- [ ] 3.1 Run `make test`, `make vet` and `make lint`. Check: every exit code reported; mechanical failures fixed. Covers: all

### 4. Review — code-reviewer
Depends on: 3
Owns: none
Gate: review
Evidence: evidence/review-round-1.md

- [ ] 4.1 Static read: `git diff --stat`, then `git diff`. Check: findings by file and severity; no project command run. Covers: all

### 5. Behavioral check — qa-eng
Depends on: 3
Owns: none
Gate: qa
Evidence: evidence/qa-report.md

- [ ] 5.1 Export a CSV from the running app, then from the mobile flow. Check: scenario pass/fail. Covers: TC-103
```

## Semantics

**Edges.** `Depends on:` is the only edge source. A node may start when every dependency is `done`
or `skipped`. Transitive dependencies are implicit.

**Layers.** `layer(n) = 1 + max(layer(dep))`; roots are layer 1. Computed, never written.

**Stages.** A node's stage derives from its `Gate:`: `build`, `none` and a v1 `unit` are stage 1;
`verify` and `qa` are stage 2; `review` and `human` are stage 3. Order is build → verify → review.
`qa` and `review` both depend only on `verify` and run concurrently, so a layer may hold one of
each. Verify fixes only mechanical failures inline; review and QA never fix — their findings go back
to the build node that owns the code.

**Invalidation.** Any changed file reopens every gate node downstream of it. A review fix reopens
the verify node for the touched packages, then review scoped to the fix diff; QA re-runs only its
failed scenario ids. One fix round per build node: findings surviving it go to the user.

**States.**

| State | Condition |
|---|---|
| `blocked` | some dependency is not `done` or `skipped` |
| `ready` | dependencies `done` or `skipped`, no task ticked |
| `running` | some tasks ticked and some not |
| `done` | every `<id>.<n>` task `[x]`; vacuously true for a node with no tasks |
| `failed` | a task, or the node's gate evidence, carries `(failed: …)` |
| `skipped` | heading carries `(skipped: <reason>)`; boxes tick with the same reason |

**Ready set.** The batch is every ready node, in `(layer, id)` order, minus a node that shares an
`Owns:` path or a `Resource:` tag with one already selected. There is no size cap:

```
for node in ready sorted by (layer, id):
    if owns_overlap(node, batch):       defer(node, "owns")
    elif resource_overlap(node, batch): defer(node, "resource")
    else: batch.append(node)
```

`owns_overlap` compares resolved paths: equal, directory prefix, or the same Go package directory.
A deferred node is eligible, not blocked. The board's `next:` footer prints the first 4 ids and
appends `+<n>` for the rest; `deferred:` names conflict-deferred nodes only.

**Width.** Two `build` nodes in the same project may run together when their `Owns:` paths are
disjoint and they share no `Resource:` tag: build's only command is a compile, so no database, port
or suite is shared. From verify onward, one node per project at a time — two `verify` or `qa` nodes
never overlap in one project. QA nodes are per project, so they run concurrently across projects; a
flow that spans projects gets its own node depending on both projects' verify nodes, carrying the
`Resource:` tags of every client it drives so conflict-only selection serializes it behind their QA
nodes.

**Correlation.** The task id prefix is the join key between a node and its tasks. OpenSpec flattens
checkboxes to sequential ids, so correlate by line, never by that id.

## Evidence

Gate evidence is scratch. The gate's agent writes it at
`openspec/changes/<change>/evidence/<file>.md`, relative to the change directory, the planner reads
it during apply, and it exists nowhere else: never committed, never archived. The path must be
exactly `evidence/<name>.md`: no `..`, no leading `/`, no other directory.

The name is the agent's to choose, and the usual shape follows the stage: a verify node reports
status and each command's exit code (`verify-go.md`), a QA node reports scenario pass/fail
(`qa-report.md`), a review node reports findings by file and severity (`review-round-1.md`).

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
  pass/fail, counts, exit codes — and the record written into `tasks.md` states that substance.
  Naming `evidence/...` is backing for it, never the sole record of the verdict.
- After archive, `Evidence:` lines are historical pointers whose targets are gone by design. Do not
  rewrite, remove or renumber them.

## Planning rules

- **Width.** Every node that can start without another's output shares that layer. Extract a shared
  contract into an early node instead of serializing its consumers. Add a `Depends on:` edge only
  when the dependent needs a named output — a path, an interface, or a `TC-…` — and name it in the
  task text. A serial edge must name its output; a fully serial plan needs a reason per edge.
- **Resources.** Tag `Resource:` only for a genuinely shared resource — `db:test`, `port:<n>`,
  `emulator`, `metro`, `build` (generated clients, lockfiles) — and only what the node actually
  holds. Over-tagging serializes the plan.
- **Project width.** Two `build` nodes in one project may run together when their `Owns:` paths are
  disjoint and they share no `Resource:` tag. From verify onward, one node per project at a time.
  QA is per project, so it runs concurrently across projects; a flow spanning projects gets its own
  node depending on both projects' verify nodes.
- **Self-containment.** A node owns every file it writes plus its tests and fixtures. It must pass
  its checks with only its ancestors' work in the tree. Tests and docs for a node's behavior land in
  that node, never in a final catch-all.
- **Shared files.** If two nodes must write the same file, merge them or extract the file into an
  earlier contract node. Two ready nodes sharing an `Owns:` path is a planning defect, not a runtime
  condition.
- **Gate ownership.** A verify node belongs to the project's last implementing build owner, resumed;
  it runs the project's suite once, after that project's builds, and fixes only mechanical failures.
  `qa` belongs to the behavioral QA agent, driving the `api | ui | device | manual` rows; `review`
  belongs to the static review agent, which touches no project command.
- **Fast path.** One project and at most three nodes may put `Gate: verify` on a builder node instead
  of adding a verify node; review stays separate. A fused node keeps its own `Owns:` list and writes
  only those files, so the `Owns: none` gate rule does not apply to it.
- **Fix rounds.** Review and QA report only; they do not fix. Their findings are batched per project
  and dispatched once, so each build node gets one fix round; survivors go to the user.

### Fullstack shape

A fullstack app, web or mobile, follows one shape:

- Node 1, or the first layer, is the API/schema contract: OpenAPI, shared types, migration and
  payload shapes. Backend, web and mobile each depend on that contract, never on the backend
  implementation, so all three build in parallel.
- Then one build node per stack — API, web, mobile — as separate projects and agents, running
  together.
- Then one `verify` node per stack for its suite, lint and typecheck. Then one `qa` node per changed
  client — API via hurl (`Resource: db:test, port:8080`), web via browser (`browser:web`), mobile
  via device (`device:mobile`) — plus one cross-stack end-to-end node depending on every verify node
  it crosses, with `Resource: db:test, browser:web, device:mobile`.
- A flow-spanning QA node carries the `Resource:` tags of every client it drives, so conflict-only
  selection runs it after their QA nodes; it still depends only on `verify`.
- Review is one node over the combined diff, and it reads the contract seam first.

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

- Header: `<change> · <N> nodes · <L> layers · <b> build · <v> verify · <r> review`. Zero stage
  counts are omitted; nodes and layers always print. `qa` rolls into verify and `human` into review.
- One line per node, sorted by `(layer, id)`. `L<n>` sits in a 4-column left field, printed only on
  the first row of its layer and blank on continuation rows.
- Columns: state glyph, id, title, owner, tasks done/total, `←` dependencies (`←None` for roots),
  and a `{gate}` tag for `verify`, `qa`, `review` and `human` nodes; `build` carries none.
- Glyphs: `▪` ready, `▸` running, `✓` done, `⊘` blocked, `✕` failed, `—` skipped.
- Titles truncate to 36 characters with `…`; the owner column pads to the longest owner; a row stays
  within 80 columns, dropping the gate tag first and the dependencies only if it still overflows.
- Footer: `next: … · deferred: … · blocked: … · critical path 1→2→3`, `none` where empty.

```text
template-resources · 6 nodes · 5 layers · 3 build · 2 verify · 1 review
L1  ✓ 1  Template content                js-eng          6/6  ←None
L2  ✓ 2  Registry, resources and tests   golang-eng      4/4  ←1
L3  ✓ 3  Guide list and docs             api-documenter  2/2  ←2
L4  ✓ 4  Verify                          golang-eng      1/1  ←1,2,3   {verify}
L5  ✓ 5  Review                          code-reviewer   1/1  ←4       {review}
    ✓ 6  Render and publish              qa-eng          3/3  ←4       {qa}
next: none · deferred: none · blocked: none · critical path 1→2→3→4→5
```

A fullstack app, where L1 is the contract and the three stacks build against it:

```text
shop-app · 12 nodes · 4 layers · 4 build · 7 verify · 1 review
L1  ✓ 1  API and schema contract         api-documenter  2/2  ←None
L2  ✓ 2  API service and tests           golang-eng      4/4  ←1
    ✓ 3  Web client                      js-eng          5/5  ←1
    ✓ 4  Mobile client                   js-eng          5/5  ←1
L3  ✓ 5  Verify API                      golang-eng      1/1  ←2       {verify}
    ✓ 6  Verify web                      js-eng          1/1  ←3       {verify}
    ✓ 7  Verify mobile                   js-eng          1/1  ←4       {verify}
L4  ✓ 8  QA API                          qa-eng          2/2  ←5       {qa}
    ✓ 9  QA web                          qa-eng          2/2  ←6       {qa}
    ✓ 10 QA mobile                       qa-eng          2/2  ←7       {qa}
    ✓ 11 Review                          code-reviewer   1/1  ←5,6,7   {review}
    ✓ 12 End-to-end purchase flow        qa-eng          2/2  ←5,6,7   {qa}
next: none · deferred: none · blocked: none · critical path 1→2→5→8
```

No ANSI colour, and the same `tasks.md` draws the same board. `## Coverage` maps each `TC-…` to the
nodes that name it under `Covers:`; a node with `Covers: all` is not listed there. Table cells keep
`\|` escaping, while the board only collapses line breaks in a title.

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
- The schema is `version: 2` and plans carry `<!-- openspec-graph: 2 -->`. The `task-graph` skill
  reads marker `1` or `2`; v1 is read-only, `unit` renders as build, and an archived plan is never
  rewritten under the new gate rules.
- `openspec validate --archived` remains the completion gate.
- `openspec/config.yaml` `rules.tasks` describes the grammar; apply-time doctrine goes in
  `operations.apply.guidance`, because `rules` is injected only into `instructions <artifact>`. The
  schema's own `apply.instruction` carries the execution loop inline; project-specific doctrine
  still belongs in `operations.apply.guidance`.
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
