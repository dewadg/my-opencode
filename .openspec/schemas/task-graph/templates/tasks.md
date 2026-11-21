<!-- openspec-graph: 1 -->

# Tasks

### 1. <!-- Node title --> — <!-- agent -->
Depends on: none
Owns: <!-- path, path -->
<!-- Resource: db:test, build -->
Gate: unit

- [ ] 1.1 <!-- What this task does --> Check: `<!-- command -->` exits 0. Covers: TC-<!-- id -->

<!--
One node per subagent invocation. Metadata lines must match the grammar exactly:

  Depends on: none | 1, 2
  Owns: none | path, path
  Resource: db:test, port:8080          (optional; nodes sharing a tag never run together)
  Gate: unit | review | qa | human | none
  Evidence: .notes/<change>/<file>.md   (required for review, qa and human)

Tasks: `- [ ] <id>.<n> <text> Check: <command> Covers: <TC-... | all>`
`Covers: all` is legal only on a review or QA gate.

No ## Wave headings. Layers and run batches are computed from Depends on, Owns and
Resource. Put shared contracts first, review and QA gates last.
-->
