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
   digest file at `<session dir>/recon-{project}.md` containing: the entry points the change touches,
   the project's conventions, the narrow and full test/build commands, cross-project contract touch
   points, and a `file:line` index. Every later brief cites that file and does not restate its
   content. A single-project, small change skips recon.

3. **Plan the work**

   Track the plan as a checklist in your reply (or in a note file) — single tracking system for this skill, no parallel todo list.

   More than two tasks: journal the plan (see Session artifacts) and have each brief cite its section
   anchor; when journaling applies the journal is the plan — no separate plan file. Two tasks or
   fewer: an in-chat checklist is enough.

   Decide changes before writing code, record as tasks in dependency order (shared contracts before consumers; backend before frontend calling it). Every behavioral change gets a test task alongside when the project supports one.

4. **Delegate, then verify and review once for the whole change**

   OpenCode subagents cannot spawn other subagents (nesting depth is one), so the main thread
   launches every task itself. Loop implementation tasks in dependency order.

   a. **ALWAYS delegate project-level investigation, coding, testing, and browser verification to a
      subagent** with the `subagent` tool. Never edit, build, test, or drive a browser inline.
      Supply the agent ID, a short description, and a complete prompt. Keep the returned
      `sessionID` for each task. Implementing subagents write their full record to
      `evidence-{project}-{agent}.md`.

   b. **Concurrency.** One subagent per project per task — two agents in the same project collide on
      the test database, dev-server port, build output, lockfile, or migration state. Parallel is
      for independent projects and for read-only work (any number of explorers). Serialize anything
      that mutates a shared resource inside one project: builds, full suites, dev servers,
      migrations, generated artifacts.

   c. **Order the work, then close each unit.** Sequence tasks by dependency. What must pass before
      a unit is done is the project's own call — its `AGENTS.md`, CI config, `Makefile` or scripts,
      existing practice. Unless the project defines its own gates, the default is: the implementing
      subagent self-checks narrow (build, lint, unit tests for what it touched), then runs the
      project build and suite once; then one batch review with a fix round bounded at two passes;
      then behavioral verification last. Two agents never build or test the same project
      concurrently; the implementing subagent runs builds and suites, never the QA agent.

   d. **Batch every review.** Independent tasks run parallel; each subagent pauses when done. When
      the project defines a review convention, follow it; otherwise review is batch-level, never
      per-subagent or per-project — separate reviews burn duplicate rounds and miss cross-boundary
      mismatches (frontend calling backend contract).

   e. **One review pass: a single review-agent subagent, single invocation, scope not payload.**
      The brief carries each repo root, the output of `git diff --stat`, the acceptance criteria
      reference, an exclusion list, and an instruction to run `git diff` itself. NEVER paste diff
      text into a brief. Exclusion list, unless the excluded file is itself the change's source of
      truth: lockfiles, generated code (API clients, schema types, mocks, API docs), vendored
      directories, minified bundles, snapshots, `.notes/`. If the combined diff exceeds roughly
      1500 changed lines, review it in directory slices sequentially inside that one reviewer
      session, reporting once at the end.

   f. Review returns findings: split by project, **resume the same implementing subagents** (pass the
      saved `sessionID` back to the `subagent` tool with the findings verbatim, each its slice only),
      not fresh spawns — resumed agents apply fixes from context already held. Fixes for different
      projects run parallel; same-project findings go to that project's agent in one message. Each
      fix self-checks narrow — by default no full-suite rerun for review fixes unless a fix crosses
      package boundaries, and then the project's last implementing agent reruns the full suite once.

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

One subagent per project per task — two agents editing the same project concurrently conflict.
Independent projects run parallel.

## Briefing subagents

Give each subagent: goal, files or package to work in, the search scope to confine itself to (the
files, packages, or directories it should grep rather than read wide), applicable project
conventions, what "done" means (the narrow build, lint, and unit commands for its packages pass, and
the behavior under test is covered). Point at source files, acceptance criteria, and the recon digest
— not restated spec. Scope to what asked — no opportunistic refactors.

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

- **Tests**: testing skill exists for language (example: Go unit-test skill): use it. Otherwise match existing test style in that project.
- **Comments**: comments, test names, story descriptions explain code behavior and intent. No review tags, ticket numbers, task ids.
- **Generated code**: never hand-edit generated files (API clients, schema types, mocks, API docs). Change source of truth, re-run generator.

## Session artifacts

More than two tasks, or more than one project: journal the run. Two tasks or fewer in one project:
skip — the chat report is enough. When it applies, the journal REPLACES a separate plan file; still
one tracking system.

Session dir: `.notes/code/{YYYY}-{MM}-{DD}-{slug}/`, or
`.notes/code/{YYYY}-{MM}-{DD}-{TICKET}-{slug}/` when a ticket is known; the date prefix keeps `ls`
chronological. The journal note lives inside it, same stem plus `-journal.md` —
`.notes/code/{YYYY}-{MM}-{DD}-{TICKET}-{slug}/{YYYY}-{MM}-{DD}-{TICKET}-{slug}-journal.md` — which
satisfies `note-write`'s filename rule (`{YYYY}-{MM}-{DD}[-{TICKET}]-{slug}.md`); the repeated stem is
deliberate, and keeping the note inside makes the session one object: archiving is one `mv` of the
directory, no renames and no broken links. When the change ships or is abandoned and the journal is
final, move the whole directory to `.notes/code/archive/{YYYY}-{MM}-{DD}[-{TICKET}]-{slug}/`. Reuse the
existing directory when resuming or continuing the same change; never create a second one for it.
Suffix `-2` only when an unrelated change already took the name. Artifact names inside are fixed, and
`related:` entries are sibling-relative — never workspace-relative.

- `<session dir>/<stem>-journal.md` — the note, `note-write` frontmatter; the orchestrator only.
- `<session dir>/recon-{project}.md` — the read-only explorer, one per project.
- `<session dir>/evidence-{project}-{agent}.md` — that subagent, its own path.
- `<session dir>/review-round-{n}.md` — orchestrator only, when a round yields more than ~40 findings
  (the `defect-report` size rule).
- `<session dir>/assets/` — screenshots, hurl files, raw logs.

One writer per file; parallel implementers write separate evidence files. The ~15-line cap is on the
message back to the orchestrator, never on the record — commands, exit codes, failure log, and files
touched go to the subagent's evidence file. The journal is append-only; only `State` cells and the
`updated:` date change in place. IDs: `D<n>` decisions, `R<round>.F<n>` review findings, `Q<n>` QA
defects. Write at natural boundaries — when the plan is set, when a unit closes, when the change set
closes — and read at resume boundaries (a context break, a new session) and before the final report,
not every round. `.notes/` is outside review scope; it goes in the reviewer exclusion list.

```markdown
---
tags: [code]
projects: [<project>]
created: {YYYY-MM-DD}
updated: {YYYY-MM-DD}
related: [recon-<project>.md, evidence-<project>-<agent>.md]
description: <change> — session journal
---
# <change title>
<intent, two lines>
status: shipped | partial | blocked

## Status
`Status` is per change unit — append a row per unit, never overwrite.
| Task | Project | Owner | State | Evidence |
|---|---|---|---|---|
| T1 | <project> | <agent> | done | evidence-<project>-<agent>.md |
## Applies
| # | Change unit | Date | ses_ | Result | Commits |
|---|---|---|---|---|---|
## Decisions
| # | Decision | Rejected | Why | `ses_` |
|---|---|---|---|---|
| D1 | <choice> | <alternative> | <why> | `ses_…` |
## Coordination log
| When | `ses_` | Agent | Project | Task | Brief |
|---|---|---|---|---|---|
| plan | `ses_…` | <agent> | <project> | T1 | <anchor> |
## Findings ledger
| Round | ID | Tier | Location | Owner | State |
|---|---|---|---|---|---|
| 1 | R1.F1 | 🔴 bug | <path>:<line> | <agent> | fixed |
## Outcome
<what shipped, what was deferred, user follow-ups>
```

## Token discipline

Four rules govern every round:

- Cap what comes back — bound every report contract at ~15 lines.
- Cite files instead of restating them — paths, `file:line`, section anchors.
- Hand back paths instead of images and logs — screenshots, HTML, and full logs stay on disk.
- Resume sessions instead of respawning them — a saved `sessionID` carries its own context.
- A stage whose context re-read per turn exceeds roughly 100k is reading too wide — narrow that
  stage's brief before adding another round.

## Tool-output discipline

Each tool call is a turn, and every turn re-reads the agent's whole context — so tool-output size and
turn count, not report length, dominate the bill.

- Search before reading: `grep` or `glob` to locate, then `read` with `offset`/`limit` on the region
  that matters. Never read a whole file over ~500 lines; use `grep -n` with a pattern to scan instead.
- Batch: one `grep` over the tree beats five sequential greps; one `read` with a wider window beats
  three opened in a row. Every extra call is another full-context turn.
- Escalate cheaply: `git diff --stat` before `git diff`, `--name-only` before per-file diffs,
  `head`/`tail` on logs, `--filter`/`-q` on noisy commands. Narrow the command, not the answer.
- Never echo tool output into the reply or re-read a file you just wrote. Cite `path:line`; the
  orchestrator reads what it needs.
- Reports and briefs stay as specified — the message cap is not an excuse to move cost into calls.

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
