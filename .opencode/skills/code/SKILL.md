---
name: code
description: >
  Write or change code in one repo or several: break the work into tasks, delegate to the
  project's own stack agents, then verify against its gates. Use for "implement TICKET-123", "add
  endpoint X", "fix the bug in Y", "wire the frontend to the new API". Not for diagnosing an
  unexplained bug (investigate its cause first) or scoping a feature before building it.

---
# Code

Break coding instruction into tasks, orchestrate specialized subagents. Output: working code, verified against the project's own gates.

**Input**: change description. May be ticket number, may name projects.

## Steps

1. **If no input, ask what to work on**

   Use the **`question` tool** with an open-ended prompt (the free-form answer is always available):
   > "What do you want to work on?"

   **IMPORTANT**: do NOT proceed without understanding what user wants built.

2. **List impacted projects**

   From description, or determine yourself. Goal: know each impacted project's tech stack (Go, React, JavaScript, infra) — that selects the subagent.

   **Pre-flight recon.** When the change spans more than one project, or has more than two tasks,
   launch one read-only explorer subagent per project, in parallel. Its single deliverable is a recon
   digest file at `<session dir>/recon-{project}.md` (see **Session directory** below) containing: the entry points the change touches,
   the project's conventions, the narrow and full test/build commands, cross-project contract touch
   points, and a `file:line` index. Every later brief cites that file and does not restate its
   content. A single-project, small change skips recon.

3. **Plan the work**

   Track the plan as a checklist in your reply (or in a note file) — single tracking system for this skill, no parallel todo list.

   More than two tasks: the primary records the plan and each brief cites its section anchor; that
   record is the plan — no separate plan file. Two tasks or fewer: an in-chat checklist is enough.

   Decide changes before writing code, record as tasks in dependency order (shared contracts before consumers; backend before frontend calling it). Every behavioral change gets a test task alongside when the project supports one.

4. **Delegate, then verify and review once for the whole change**

   OpenCode subagents cannot spawn other subagents (nesting depth is one), so the main thread
   launches every task itself. Loop implementation tasks in dependency order.

   a. **ALWAYS delegate project-level investigation, coding, testing, and browser verification to a
      subagent** with the `subagent` tool. Never edit, build, test, or drive a browser inline.
      Supply the agent ID, a short description, and a complete prompt. Keep the returned
      `sessionID` for each task. Implementing subagents write their full record to
      `evidence-{project}-{agent}.md`. Every brief states that file changes go through the `edit`
      tool — `sed`/`awk` only for a uniform multi-file rewrite, never a Python script or `python3`
      heredoc patching files.

   b. **Concurrency.** One subagent per project per task — two agents in the same project collide on
      the test database, dev-server port, build output, lockfile, or migration state. Parallel is
      for independent projects and for read-only work (any number of explorers). Serialize anything
      that mutates a shared resource inside one project: builds, full suites, dev servers,
      migrations, generated artifacts.

   c. **Order the work, then close each unit.** Sequence tasks by dependency. What must pass before
      a unit is done is the project's own call — its `AGENTS.md`, CI config, `Makefile` or scripts,
      existing practice. The implementing subagent discovers the checks the project already defines
      and self-checks narrow with only those — format, lint, type-check, build, unit tests for the
      packages it touched — reporting each with an exit code or as absent; a project with none is "no
      project checks defined", not an invitation to add tooling. Unit tests are the per-task gate;
      isolate the unit scope where the runner allows it, else defer the suite. The project's full
      build and suite run once at close-out after that project's implementation tasks; the
      integration/E2E tier only where the project enforces it — never per task, never on a review fix.
      Then one batch review with a fix round bounded at two passes; then behavioral verification
      last. Builds and suites belong to the implementing subagent, never the QA agent.

   d. **Batch every review.** Independent tasks run parallel; each subagent pauses when done. When
      the project defines a review convention, follow it; otherwise review is batch-level, never
      per-subagent or per-project — separate reviews burn duplicate rounds and miss cross-boundary
      mismatches (frontend calling backend contract).

   e. **One review pass: a single review-agent subagent, single invocation, scope not payload.**
      The brief carries each repo root, the output of `git diff --stat`, the acceptance criteria
      reference, an exclusion list, the `evidence-{project}-{agent}.md` paths for the projects
      under review, and an instruction to run `git diff` itself and cross-check the code against
      that evidence. NEVER paste diff text into a brief. The brief forbids project commands — no
      build, lint, format, type-check, test, suite, dev server, container. Exclusion list, unless the
      excluded file is itself the change's source of truth: lockfiles, generated code (API clients,
      schema types, mocks, API docs), vendored directories, minified bundles, snapshots, `.notes/`.
      If the combined diff exceeds roughly 1500 changed lines, review it in directory slices
      sequentially inside that one reviewer session, reporting once at the end.

   f. Review returns findings: split by project, **resume the same implementing subagents** (pass the
      saved `sessionID` back to the `subagent` tool with the findings verbatim, each its slice only),
      not fresh spawns — resumed agents apply fixes from context already held. Fixes for different
      projects run parallel; same-project findings go to that project's agent in one message. Each
      fix self-checks narrow — format, lint, type-check, touched-package build, unit tests — no suite
      run on a review fix; a cross-package fix is re-verified in a fresh close-out run after the fix
      round.

   g. All fixes in: re-review the full combined diff once, same way; when the project defines no
      review convention, bound it at two rounds — findings surviving the second pass stop and come to
      the user.

   h. **Behavioral verification, where the project calls for it, once, last.** One brief to one QA
      subagent covering every touched project: verification on the running app against the acceptance
      criteria. QA defects loop back exactly like review findings — resume the project's implementing
      agent with the defect verbatim, it fixes and narrow re-checks, then the QA agent re-verifies
      only the failed scenarios, never the whole pass. Reuse the same QA session for the
      re-verification round: brief it with the failed scenario IDs only, not the whole pass and not a
      fresh full brief. Defects surviving one fix round: stop, bring to user.

   i. **If something genuinely unclear**: use the **`question` tool** to clarify, then continue.

## Choosing subagents

No fixed roster — pick from the agent definitions OpenCode loads from the project and global config,
so a new agent works without a skill change. Enumerate once per session with `glob` over those config
directories' agent definitions; keep the result in the plan, re-list only if the config changes.

Route by capability, not name: backend → that language's engineering agent; web frontend → that
framework's engineering agent; test strategy, E2E, browser verification → QA / UI testing agent;
review of a produced diff → code review agent; containers, compose, images → container agent; schema,
migrations, query performance → database agent; cross-cutting design → architecture review agent.
Most specific fit wins; none fits → a general-purpose subagent.

## Briefing subagents

Give each subagent: goal, files or package to work in, the search scope to confine itself to (the
files, packages, or directories it should grep rather than read wide), applicable project
conventions, what "done" means (the project's existing narrow format, lint, type-check, build, and
unit checks for its packages pass, absent ones named, and the behavior under test is covered). Point
at source files, acceptance criteria, and the recon digest — not restated spec. Scope to what
asked — no opportunistic refactors.

Every brief carries this hard report contract, and the subagent follows it:

- **Files touched**: paths only.
- **Commands**: `cmd` → exit code, one line each.
- **Failures**: failing test/step IDs plus the assertion line, at most ~10 lines; the full log goes
  to a file path, never into the report.
- **Not verified**: one line each, with the reason.
- At most ~15 lines total.
- The full record goes to the evidence file named in the brief; the ~15 lines are its summary.
- No pasted diffs, no file bodies, no restated brief or plan.

Over roughly 200 lines of output means the brief was too broad — narrow the next brief rather than
accepting the dump.

Standing conventions every subagent follows:

- **Tests**: a testing skill exists for the language — `golang-unit-test` for Go — so use it. Otherwise match existing test style in that project.
- **Comments**: comments, test names, story descriptions explain code behavior and intent. No review tags, ticket numbers, task ids.
- **Generated code**: never hand-edit generated files (API clients, schema types, mocks, API docs). Change source of truth, re-run generator.

## Session directory

Everything this run produces goes into one session directory:

`.notes/code/{YYYY}-{MM}-{DD}-{slug}/`, or `{YYYY}-{MM}-{DD}-{TICKET}-{slug}/` when a ticket is
known — date prefix always, ticket segment before the slug. That `<session dir>` is what briefs and
evidence paths point at; the caller may name a different root per the project's own `AGENTS.md`, and
the layout below still applies.

- `{stem}-journal.md` — the run's journal, sharing the directory's stem.
- `recon-{project}.md` — one per project, from pre-flight recon.
- `evidence-{project}-{agent}.md` — one per subagent, one writer per file.
- `review-round-{n}.md` — the findings of review round `n`.
- `assets/` — screenshots and other non-markdown evidence.

`related:` entries are sibling-relative. Ids are numbered within the session:

| Prefix | Means | Numbering |
|---|---|---|
| `D<n>` | a decision taken for the user | sequential |
| `R<round>.F<n>` | a review finding | round segment = review round, `n` in-round |
| `Q<n>` | a QA defect | sequential across every QA pass, in report order — never restarted per pass |

Archive a finished session by one `mv` to `.notes/code/archive/{YYYY}-{MM}-{DD}[-{TICKET}]-{slug}/`:
the stem moves with it, so sibling-relative links stay valid. `.notes/` never enters a commit and is
outside review scope.

## Evidence files

One `evidence-{project}-{agent}.md` per subagent in the session directory, never shared between
parallel implementers. The ~15-line cap binds the message back, not the record: commands, exit
codes, failure logs and files touched all belong in the file. `.notes/` stays out of the review diff.

## Context discipline

Each tool call is a turn and every turn re-reads the whole context, so output size and turn count,
not report length, dominate cost.

- Cap what comes back: reports ~15 lines (see **Briefing subagents**), the full record to the
  evidence file. Cite `path:line` and section anchors instead of restating.
- Hand back paths, not payloads — screenshots, HTML and full logs stay on disk.
- Resume a saved `sessionID` instead of respawning: that agent already holds its context.
- Search before reading: `grep`/`glob` to locate, then `read` with `offset`/`limit`; never read a
  whole file over ~500 lines.
- Escalate cheaply: `git diff --stat` before `git diff`, `--name-only` before per-file diffs,
  `head`/`tail` on logs, `--filter`/`-q` on noisy commands.
- Batch — one `grep` over the tree beats five sequential greps, one wider `read` beats three.
- Never echo tool output into the reply or re-read a file you just wrote.
- A stage re-reading over ~100k per turn is reading too wide: narrow that brief, don't add rounds.

## Verifying

Verify the returned work by reading; never reproduce it.

- Every claim cites the `sessionID` of the subagent that produced it; an unattributable claim did
  not happen.
- Check claims cheaply: `grep`/`glob` to locate, `read` with `offset`/`limit`, `git diff --stat`
  before `git diff`.
- Every criterion in the brief maps to a reported result, and commands carry exit codes.
- A missing or contradicted item is a gap: report it, or request it from that same subagent in ~5
  lines — never close it by running the build, tests, or browser.
- A project's suite run, where it has one, is attributed to the implementing subagent that ran it.
- Review rounds are static: the reviewer reads the diff and the evidence, never re-runs a gate. A
  missing gate is a gap to resume the implementing agent for — not for the reviewer or for you.
- A review reply that is not `defect-report` output or the single line `No issues.` is a gap: resume
  that reviewer for the findings, never accept prose.
- Publication stops at the rung the user named for the run; unstated, commit on the branch. Pushing,
  PRs, merges and shared-environment changes need the user's naming or a recorded project convention.

## Output

All tasks done, report back in chat:
- Changes grouped by project, files touched, with the `sessionID` behind each
- Verification status per task: evidence present, missing, or contradicted
- Test and build results as reported by the subagent that ran them — quote failures, exit codes
- Anything undone, deferred, knowingly out of scope
- Follow-ups user must action (migration to run, config or env var to set, rollout order across repos)

## Guardrails

- Context critically unclear: ask user — but prefer reasonable decisions, keep momentum.
- Verify each artifact file exists after writing, before next.
- Report outcomes faithfully. Tests fail: say so with output. Never claim done on unverified work.
