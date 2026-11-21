<!-- openspec-graph: 1 -->

# Tasks

### 1. <!-- Node title --> — <!-- agent -->
Depends on: none
Owns: <!-- path, path -->
<!-- Resource: db:test, build -->
Gate: unit

- [ ] 1.1 <!-- What this task does --> Check: `<!-- command -->` exits 0. Covers: TC-<!-- id -->

<!--
One node per subagent invocation. Node ids are unique positive integers; gaps are allowed,
so inserting a node never renumbers the rest. Metadata lines must match the grammar exactly:

  Depends on: none | 1, 2
  Owns: none | path, path
  Resource: db:test, port:8080           (optional; nodes sharing a tag never run together)
  Gate: unit | review | qa | human | none
  Evidence: evidence/<file>.md            (required for review, qa and human; scratch output,
                                          never committed, never archived)

Tasks: `- [ ] <id>.<n> <text> Check: <command> Covers: <TC-... | all>`
`Covers:` names ids from design.md's Test scenarios table; `all` is legal only on a review
or QA gate. `Check:` must state the outcome that proves the task.

No ## Wave headings. Layers and the ready set are computed from Depends on, Owns and
Resource; the `task-graph` skill draws the board. Put shared contracts first, review and
QA gates last. graph.md is written once, after this file exists.
-->
