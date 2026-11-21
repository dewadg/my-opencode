# Example — glab50-template-resources

A rewrite of the archived change at
`artifact/openspec/changes/archive/2026-11-21-glab50-template-resources/tasks.md` in the
`task-graph` format.

## What changed

- `## Wave 1` / `## Wave 2` removed. The two waves became edges: node 2 now depends on node 1, and
  nodes 4 and 5 depend on 1, 2 and 3.
- Node 2's original `Depends on: nothing` was not accurate for completion: its tests read node 1's
  files. The edge makes that explicit.
- Review and QA are `Gate` nodes with `Evidence:` paths instead of prose-only last-wave groups.
- The original `TC-830` reference is kept. It is not in this change's `design.md`; it resolves from
  `openspec/specs/authoring-guide/spec.md`, so the checker must read the main specs as well as the
  change design.
- Every task is `[x]`, matching the archived state. The graph therefore shows all nodes done and an
  empty next batch.

`graph.md` is hand-rendered from `tasks.md` because `openspec-graph render` is not implemented yet.
Its `Source:` hash matches the file above.
