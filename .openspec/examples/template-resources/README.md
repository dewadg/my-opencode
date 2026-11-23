# Example — template-resources

A rewrite of the archived change at
`artifact/openspec/changes/archive/2026-11-21-glab50-template-resources/tasks.md` in the
`task-graph` format.

## What changed

- `## Wave 1` / `## Wave 2` removed. The two waves became edges: node 2 depends on node 1,
  node 3 depends on node 2, and the gates depend on the builds they prove.
- The plan is six nodes in three stages: builds 1–3, verify 4, then review 5 and QA 6, both
  depending only on 4 and running concurrently. The marker is `<!-- openspec-graph: 2 -->`.
- Builders run only the structural check — compile, parse, or a content `grep` where there is no
  compiler — never a behavior test. Node 1's `grep`/`du` checks are that check for static assets;
  node 2 writes the tests and compiles them with `go test -run '^$'`. The
  suite runs once, in node 4, which is `Gate: verify` and depends on 1, 2 and 3. It runs `make test`,
  `make vet`, `make lint` and `make web-build`, reports each exit code, and fixes mechanical
  failures in the same session.
- Review (node 5) is a static read of the diff: no `make`, `go test` or lint command. QA (node 6)
  drives the behavioral scenarios on the running app: boards at their sizes, the four `?state=`
  screens, and every resource through `resources/list` and `resources/read`. The suite line that
  used to sit in the QA report moved to `evidence/verify-go.md`.
- `Resource: db:test` sits on node 4, the verify node that runs `make test`, not on the build nodes:
  a build compiles and never opens the test database.
- Node 2's original `Depends on: nothing` was not accurate for completion: its tests read node 1's
  files. The edge makes that explicit.
- Node 3 depends on 2 only. Listing 1 as well would be a transitive edge, because 2 already
  depends on 1.
- The original `TC-830` reference is kept. It is not in this change's `design.md`; it resolves from
  `openspec/specs/authoring-guide/spec.md`, so a plan may cite a TC that lives in a main spec rather
  than in the change design.
- Every task is `[x]`, matching the archived state, so the board shows every node done and no
  runnable batch.

The three `evidence/*.md` files are kept only to show each stage's report shape; in a real change
that directory is gitignored and deleted before archive.

`graph.md` is the plan snapshot: its `Source:` hash records which `tasks.md` produced it, and it
shows the layers, stage counts, coverage and critical path, not run state.
