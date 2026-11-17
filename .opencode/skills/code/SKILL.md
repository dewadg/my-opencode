---
name: code
description: >
  Coding and programming work, broken into tasks and delegated to specialized subagents chosen by
  each project's tech stack. Works on a single repo or across several projects in a workspace.
  Use when the user wants code written or changed — e.g. "implement TICKET-123", "add endpoint X",
  "fix the bug in Y", "refactor Z", "wire the frontend to the new API".
  Not for diagnosing an unexplained bug (investigate the cause first) or for scoping a feature
  before building it.
---
# Code

Break coding instruction into tasks, orchestrate specialized subagents. Output: working, tested, reviewed code.

**Input**: change description. May be ticket number, may name projects.

## Steps

1. **If no input, ask what to work on**

   Use the **`question` tool** with an open-ended prompt (the free-form answer is always available):
   > "What do you want to work on?"

   **IMPORTANT**: do NOT proceed without understanding what user wants built.

2. **List impacted projects**

   From description, or determine yourself. Goal: know each impacted project's tech stack (Go, React, JavaScript, infra) — that selects the subagent.

3. **Plan the work**

   Track the plan as a checklist in your reply (or in a note file) — single tracking system for this skill, no parallel todo list.

   Decide changes before writing code, record as tasks in dependency order (shared contracts before consumers; backend before frontend calling it). Every behavioral change gets a test task alongside when the project supports one.

4. **Delegate, then verify and review once for the whole change**

   OpenCode subagents cannot spawn other subagents (nesting depth is one), so the main thread
   launches every task itself. Loop implementation tasks in dependency order.

   a. **ALWAYS delegate project-level investigation, coding, testing, and browser verification to a
      subagent** with the `subagent` tool. Never edit, build, test, or drive a browser inline.
      Supply the agent ID, a short description, and a complete prompt. Keep the returned
      `sessionID` for each task.

   b. **Concurrency.** One subagent per project per task — two agents in the same project collide on
      the test database, dev-server port, build output, lockfile, or migration state. Parallel is
      for independent projects and for read-only work (any number of explorers). Serialize anything
      that mutates a shared resource inside one project: builds, full suites, dev servers,
      migrations, generated artifacts.

   c. **Pipeline order — implement, full suite, review, behavioral QA last.** Each expensive
      stage runs once, in this order. (1) The implementing subagent self-checks before reporting:
      narrow build, lint, and unit tests for the packages it touched. (2) After every implementing
      subagent in a project has finished, resume that project's last implementing subagent (its
      saved `sessionID`) to run the full project build and suite once. (3) The batch review and
      its bounded fix round (d–g below). (4) The QA agent runs behavioral verification once, on the
      final reviewed code — endpoints via hurl, UI via browser, mobile via device. QA runs last on
      purpose: review fixes cannot invalidate its evidence, and no fix round re-triggers a QA pass.
      Unit-test suites and build pipelines are always the implementing agents' job; the QA agent
      never runs them. Never two agents building or testing the same project; never behavioral
      verification alongside an implementing agent in that project.

   d. **No review until every implementing subagent finished.** Independent tasks run parallel; each
      subagent pauses when done. Review is batch-level, never per-subagent or per-project — separate
      reviews burn duplicate rounds and miss cross-boundary mismatches (frontend calling backend
      contract).

   e. **One review pass: single `code-reviewer` subagent, single invocation, full combined diff
      across every touched project.** Reviewer gets the whole working-tree diff of all touched repos
      in one briefing, not one diff per agent.

   f. Review returns findings: split by project, **resume the same implementing subagents** (pass the
      saved `sessionID` back to the `subagent` tool with the findings verbatim, each its slice only),
      not fresh spawns — resumed agents apply fixes from context already held. Fixes for different
      projects run parallel; same-project findings go to that project's agent in one message. Each
      fix self-checks narrow — no full-suite rerun for review fixes unless a fix crosses package
      boundaries, and then the project's last implementing agent reruns the full suite once, alone.

   g. All fixes in: re-review the full combined diff once, same way. Bound at two rounds total —
      findings survive the second pass, stop, bring to user.

   h. **QA pass, once, after the review loop closes.** One brief to one QA subagent covering every
      touched project: behavioral verification on the running app against the acceptance criteria.
      QA defects loop back exactly like review findings — resume the project's implementing agent
      with the defect verbatim, it fixes and narrow re-checks, then the QA agent re-verifies only
      the failed scenarios, never the whole pass. Defects surviving one fix round: stop, bring to
      user.

   i. **If something genuinely unclear**: use the **`question` tool** to clarify, then continue.

## Choosing subagents

No fixed roster — pick from the agent definitions available in `.opencode/agents/` and
`~/.config/opencode/agents/`, so newly added agents work without a skill change. Enumerate with:

```bash
ls .opencode/agents ~/.config/opencode/agents 2>/dev/null
```

Route by capability, not name:

- Service or backend: engineering agent for that language
- Web frontend: engineering agent for that framework
- Test strategy, E2E, browser verification: QA / UI testing agent
- Review of any produced diff: code review agent
- Containers, compose, images: container agent
- Schema, migrations, query performance: database agent
- Cross-cutting design concerns: architecture review agent

Several fit: pick most specific. None fits: the built-in `general` subagent.

One subagent per project per task — two agents editing same project concurrently conflict. Independent projects run parallel.

## Briefing subagents

Give each subagent: goal, files or package to work in, applicable project conventions, what "done"
means (the narrow build, lint, and unit commands for its packages pass, and the behavior under test
is covered). Point at source files, not restated spec. Scope to what asked — no opportunistic
refactors.

Every brief asks for an evidence block in the report:

- Commands run, with exit codes.
- Failure output quoted verbatim, not paraphrased.
- Files touched.
- Anything not verified, and why.

Standing conventions every subagent follows:

- **Tests**: testing skill exists for language (example: Go unit-test skill): use it. Otherwise match existing test style in that project.
- **Comments**: comments, test names, story descriptions explain code behavior and intent. No review tags, ticket numbers, task ids.
- **Generated code**: never hand-edit generated files (API clients, schema types, mocks, API docs). Change source of truth, re-run generator.

## Verifying

You verify the returned work; you do not reproduce it. The implementing subagent runs the narrow
build, lint, and unit tests for what it touched before reporting. After all implementation in a
project is in, that project's last implementing subagent (resumed by `sessionID`) runs the full
project build and suite once. The QA agent then verifies behavior on the running app — endpoints,
browser flows, mobile — once for the whole change, after the review loop closes, so review fixes
never re-trigger it; it never runs unit-test suites.

Check the reports:

- Every claim cites the `sessionID` of the subagent that produced it. A claim you cannot attribute
  did not happen.
- Claimed files exist — `read` or `glob` them.
- Every criterion in the brief maps to a reported result.
- Commands carry exit codes and quoted failure output. "Should pass" is not a result.
- Each project's full-suite run is attributed to the implementing subagent that ran it, and its
  failures were addressed before behavioral verification started.
- The diff matches the planned scope — no opportunistic files.
- The project-verification report covers the acceptance criteria, with screenshots plus console and
  network findings for UI work.

Missing or contradicted evidence: report the gap, or resume that subagent with it. Do not run the
build, the tests, or the browser yourself — that is how a project ends up with two concurrent builds
and no attributable evidence.

No commit, push, tag, or PR unless user asked.

## Output

All tasks done, report back in chat:
- Changes grouped by project, files touched, with the `sessionID` behind each
- Verification status per task: evidence present, missing, or contradicted
- Test and build results as reported by the subagent that ran them — quote failures, exit codes
- Anything undone, deferred, knowingly out of scope
- Follow-ups user must action (migration to run, config or env var to set, deploy order across repos)

## Guardrails

- Context critically unclear: ask user — but prefer reasonable decisions, keep momentum.
- Verify each artifact file exists after writing, before next.
- Report outcomes faithfully. Tests fail: say so with output. Never claim done on unverified work.
