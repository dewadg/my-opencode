# Example — glab50-template-resources

A rewrite of the archived change at
`artifact/openspec/changes/archive/2026-11-21-glab50-template-resources/tasks.md` in the
`task-graph` format.

## What changed

- `## Wave 1` / `## Wave 2` removed. The two waves became edges: node 2 depends on node 1,
  node 3 depends on node 2, and gates 4 and 5 depend on 1, 2 and 3.
- Node 2's original `Depends on: nothing` was not accurate for completion: its tests read node 1's
  files. The edge makes that explicit.
- Node 3 depends on 2 only. Listing 1 as well would be a transitive edge, because 2 already
  depends on 1.
- Review and QA are `Gate` nodes with committed `Evidence:` files under `evidence/`, not prose-only
  last-wave groups and not `.notes/`.
- `journal.md` is the apply-time decision log: entries are numbered `D1`, `D2`, ... and each names
  the node that made the decision.
- The original `TC-830` reference is kept. It is not in this change's `design.md`; it resolves from
  `openspec/specs/authoring-guide/spec.md`, so a plan may cite a TC that lives in a main spec rather
  than in the change design.
- Every task is `[x]`, matching the archived state, so the board shows every node done and no
  runnable batch.

`graph.md` is the plan snapshot: its `Source:` hash records which `tasks.md` produced it, and it
shows the layers, coverage and critical path, not run state.
