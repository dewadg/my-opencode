<!-- openspec-graph: 2 -->

# Tasks

### 1. <!-- Node title --> — <!-- agent -->
Depends on: none
Owns: <!-- path, path -->
<!-- Resource: build -->
Gate: build

- [ ] 1.1 <!-- What this task does --> Check: `<!-- compile/parse command -->` exits 0. Covers: TC-<!-- id -->

<!--
One node per subagent invocation. Node ids are unique positive integers; gaps are allowed,
so inserting a node never renumbers the rest. Metadata lines must match the grammar exactly:

  Depends on: none | 1, 2
  Owns: none | path, path
  Resource: db:test, port:8080, emulator, metro, build  (optional; tag only what the node holds)
  Gate: build | verify | qa | review | human | none  (default build; `unit` is v1-only)
  Evidence: evidence/<file>.md                       (required for verify, qa, review and human;
                                                      scratch output, never committed, never archived)

Gates and stages: build/none stage 1, verify/qa stage 2, review/human stage 3. qa and review
both depend only on verify and run concurrently.

Check: per gate, name the outcome that proves the task:
  build   the project's compile, parse or narrowest structural check (validator, required-content
          grep) on the touched files, test files included, exits 0; never a behavior test
  verify  every command in design's Verification table, each exit code reported
  qa      the scenario pass/fail shape
  review  findings by file and severity, and no project command run

Tasks: `- [ ] <id>.<n> <text> Check: <command> Covers: <TC-... | all>`
`Covers:` names ids from design.md's Test scenarios table; a build node names its own, and
`all` is legal on verify, qa and review.

No ## Wave headings. Layers and the ready set are computed from Depends on, Owns and
Resource; the `task-graph` skill draws the board. Put shared contracts first, then one verify
node per project, then review and QA. graph.md is written once, after this file exists.
-->
