---
name: security-audit
description: >
  Security guidance and source-only vulnerability review for codebases, APIs, services, CLI
  tools, libraries, and daemons: threat-model a surface, trace a trust boundary to its sink,
  judge whether a candidate is a real vulnerability, or run a full six-phase audit with a
  coverage ledger, independent verification, and a report note. Use for security questions,
  focused reviews, triage, and when the user says "security audit this codebase", "find
  security vulnerabilities", "pen-test the repo", "threat model X", or "review this service for
  authz bugs". Run the complete workflow only when the user explicitly asks for an audit,
  pen-test, full/comprehensive/end-to-end review, or report artifacts; otherwise stay in
  guidance mode and touch only the part that answers the question. Not for adding CI scans (use
  gitlab-security-scan), not for reviewing a diff (use code-reviewer), not for a compliance or
  controls assessment (use compliance-auditor), not for writing the fix (use code), not for
  formatting findings as a table (use defect-report).
---

# Security Audit

Find vulnerabilities that violate a real trust boundary, then give owners the source evidence, the
reproduction to run themselves, the priority, and the smallest effective fix. The audit is defensive
and **source-only**: it reads the target and never builds, runs, tests, fuzzes, or probes it. A
candidate without a concrete affected principal, resource, or security outcome is not a confirmed
finding.

> Source: adapted from cloudflare/security-audit-skill
> (https://github.com/cloudflare/security-audit-skill), MIT — see [LICENSE](LICENSE).
>
> **What changed for OpenCode.** Upstream assumes Claude-Code-style delegation and an OS-enforced
> sandbox for running target code. This copy drops all target-code execution: the sandbox, the
> per-agent write roots with their promotion procedure and ID rules, and write isolation are gone, and
> anything settleable only by running code, a build, a test, a browser, an emulator, a fuzzer, or a
> live/shared/lab environment becomes a `needs_validation` record. Delegation is the `subagent` tool
> from the main thread; independence comes from fresh sessions, not filesystem isolation; artifacts
> land in `.notes/` in this repo's note format. `report-schema.json` and both validators are
> upstream's, unchanged.

## Files — read when you reach the phase

| File | Read at |
|---|---|
| `RECONNAISSANCE.md` | Phase 1 |
| `HUNTING.md` | Phase 2, and before every coverage-critic wave |
| `ATTACK-CLASSES.md` | Phase 1 companion selection, and Phase 2 hunter prompts |
| `VALIDATION-AND-REPORTING.md` | Phases 3–6 |
| `report-schema.json` | Phases 2–6, immediately before writing any record |
| `MEMORY-SAFETY-AND-BINARY.md` | only when recon selects a native/binary boundary |
| `AI-AND-LLM.md` | only when a model participates in a trust decision |
| `WEB-PROTOCOL-AND-AUTH.md` | only when HTTP framing, cache, or identity is the boundary |
| `CLIENT-SIDE.md` | only when browser sources, sinks, or storage are in scope |
| `SUPPLY-CHAIN-AND-RELEASE.md` | only when dependencies, CI, release, or updates are in scope |
| `CLOUD-AND-DEPLOYMENT.md` | only when cloud identity, IaC, containers, or runtime config are in scope |
| `PROTOCOLS-RPC-AND-MESSAGING.md` | only when RPC, queues, brokers, or streaming are in scope |
| `RESOURCE-EXHAUSTION-AND-AVAILABILITY.md` | only when untrusted work consumes shared resources |
| `DATA-ISOLATION-AND-LIFECYCLE.md` | only when tenant scope, copies, export, backup, or deletion are in scope |
| `DESKTOP-MOBILE-AND-LOCAL-IPC.md` | only when a native app, deep link, or local IPC channel is in scope |

Companions are opt-in. Selecting one is a reconnaissance decision backed by a source-visible
boundary, never by a language or dependency name.

## Operating modes

This skill is guidance by default. Loading it does not authorize the complete audit workflow or file
creation.

- **Guidance mode** — security questions, focused reviews, methodology, triage, or investigation of
  a specific finding: use only the relevant parts of this skill. Do not run all six phases, create a
  note, or write run artifacts. You may delegate focused reads with the `subagent` tool; they return
  results to the current task.
- **Full audit mode** — the user explicitly asks to audit or pen-test a codebase, asks for a full,
  comprehensive, or end-to-end security review, or requests report artifacts: run all six phases and
  write the artifacts defined below.

If the request could mean either mode, ask one focused question with the `question` tool before
creating files or starting the complete workflow.

## Orchestration

The workflow runs in the **main thread**, under whichever primary is active, which plans, briefs,
and verifies returned reports. It has to: OpenCode nesting depth is one, so a subagent cannot spawn a subagent.
Every hunter, coverage critic, and verifier is exactly one `subagent` call from the main thread and
gets a fresh session.

**Pre-run guard.** If the invoking agent has no `subagent` capability, stop here — before setup,
before `run-metadata.json`, before the run directory, and before the note — and hand back to the main
thread (or ask the user to run the audit there). That hand-back writes nothing at all: it is not a run
outcome. The one permitted pre-Phase-1 write is the budget-gate exit's `run-metadata.json`, which
records terminal state (b) under Cost budget. Guidance-mode reads need no such guarantee.

- **Route by capability, not by name.** A read-only tracing agent for narrow source traces; a
  general investigator for broad multi-file sweeps; a security-review agent for boundary judgement
  and independent verification; the language engineer for stack semantics the trace depends on.
  (The roster is whatever agents OpenCode loads; pick by the capability each brief needs.)
- **Independence is session freshness.** The session that verifies a finding is never the session
  that found it. Use a distinct `subagent` call and never reuse a hunter's session for verification.
- **The parent owns the shared files.** Only the main-thread agent writes the report note,
  `run-metadata.json`, `architecture.md`, `coverage-ledger.json`, `findings.json`,
  `FINDINGS-DETAIL.md`, and `NEEDS-VALIDATION.md`. Hunters, critics, and verifiers return structured
  records as text and write nothing. This is a role rule, prompt-enforced, not an OS guarantee.
- **Budget counts `subagent` invocations.** One invocation is one agent call, whatever it reads.

## Evidence rule — source only

Read-only source inspection is the only activity permitted against the target, in both modes.

- **Allowed:** the `read`, `grep`, and `glob` tools over source, tests, build definitions, manifests,
  infrastructure code, lockfiles, migrations, and local documentation. Quote the line.
- **Not allowed:** running the target's build, tests, scripts, CLIs, servers, browsers, emulators,
  fuzzers, or fixture processing; installing or fetching dependencies; contacting endpoints,
  identity providers, registries, brokers, cloud APIs, or any shared, live, or lab environment. Do
  not probe production identities, other users' data, or control planes.
- **`confirmed` rests on two things:** a complete repository-relative source trace from a real
  lower-trust entrypoint to the affected sink or boundary decision, plus a bounded, source-derivable
  observed result — what the code deterministically does with that input, expressed as an outcome,
  not as a guess about a live system. No runtime output is required or claimed.
- **Anything that can only be settled by execution becomes `needs_validation`**, carrying the exact
  unresolved fact and a safe, owner-run validation plan: the smallest command, test, or observation
  the owner can run at their discretion. The audit never runs it.

A candidate without source that reaches a real boundary is not a finding. A source-level hardening
gap with no affected principal or resource is a hardening note, not a finding.

## Full audit setup

Resolve these before reconnaissance:

- **Skill directory** — the directory containing this `SKILL.md`; it holds the schema and validators.
  Never copy them into the output.
- **Target** — the absolute repository root under review.
- **Repo name** — a stable identifier from the directory or local Git remote.
- **Notes root** — `.notes/` at the audited repo root, or wherever that project's `AGENTS.md`
  overrides it. Confirm it is ignored by version control before writing; if it is not, stop and ask
  the user, because the audit dirties the worktree whose dirty state it records in `source_ref`.
- **Slug** — a short dash-slug of target repo plus scope, e.g. `payments-api-auth`. One slug per run.
- **Run directory** — `<notes-root>/security-audit/{YYYY}-{MM}-{DD}-{slug}/`, or
  `<notes-root>/security-audit/{YYYY}-{MM}-{DD}-{TICKET}-{slug}/` when a ticket is known: ticket
  segment first, then slug; the date prefix is always there, so runs sort chronologically.
- **Report note** — `<run-dir>/{same stem}-report.md`, e.g.
  `.notes/security-audit/2026-11-17-PAY-431-payments-api-auth/2026-11-17-PAY-431-payments-api-auth-report.md`,
  written with the `note-write` skill's file mechanics and frontmatter schema. The leaf name carries
  no `security-audit` — the namespaced parent directory supplies it — so the note's frontmatter
  `tags` MUST include `security-audit`, keeping it findable by `note-search`. This note is the only
  report artifact; do not also emit a `REPORT.md`.
- **Source ref** — the reviewed commit and whether the worktree is dirty. Unreviewed generated or
  modified files are not another revision.

`{YYYY}`, `{MM}`, and `{DD}` are zero-padded. When a same-day prior run with the same slug exists,
append `-run-2`, then `-run-3`, to the **directory** name:
`.notes/security-audit/2026-11-17-payments-api-auth-run-2/`. The note stem mirrors the directory, so
it inherits the suffix.

Before delegation, the parent writes `run-metadata.json` with at least `run_id`, `repo`, `target`,
`source_ref`, `profile`, `scope_paths`, `budget` (null if unset), `execution_policy: "source-only"`,
`slug`, `note_path`, `run_dir`, selected companion files, prior-run paths, and
`run_status: "in_progress"`. Update metadata only when those facts change; candidate state belongs in
the coverage ledger and `findings.json`.

The run directory holds `run-metadata.json`, `architecture.md`, `coverage-ledger.json`,
`findings.json`, `FINDINGS-DETAIL.md`, and `NEEDS-VALIDATION.md`. The note carries the report body;
its `related` entries are sibling-relative — `run-metadata.json`, `architecture.md`,
`coverage-ledger.json`, `findings.json`, `FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md` — so moving the
directory never breaks them. A finished run moves as one `mv` to
`<notes-root>/security-audit/archive/{YYYY}-{MM}-{DD}[-{TICKET}]-{slug}/`: no renames, links intact.

## Full audit planning

Coverage, prior-run, profile, and budget rules apply only in full audit mode.

### Coverage and prior runs

No one pass is complete. Build a deterministic coverage plan before hunting and update it after every
subagent result. `RECONNAISSANCE.md` defines the stable coverage units; `HUNTING.md` defines
coverage-critic waves. The parent alone updates the ledger.

Find prior runs by globbing `<notes-root>/security-audit/*/coverage-ledger.json` and
`<notes-root>/security-audit/archive/*/coverage-ledger.json` for ledgers, and
`<notes-root>/security-audit/*/*-report.md` plus the same under `archive/` for notes. Read every
compatible prior `coverage-ledger.json` and `findings.json` before planning:

1. Compare the relevant current source with each prior record and unit. A prior source ref alone is
   not evidence that a path is unchanged.
2. Carry a prior `confirmed` record into the current candidate set only when its relevant source and
   conditions are unchanged and its evidence still meets the current contract. Link it to a current
   ledger unit seeded `planned`, preserve its fingerprint, exclude only that carried root cause from
   hunters, and send the carried record through the current final verification path; the verifier
   that re-checks it becomes that unit's assignment owner and moves it to `candidate`.
3. When relevant source for a prior `confirmed` record changed, create a current planned revalidation
   unit. Do not put that record on the hunter exclusion list. It remains confirmed only if current
   independent verification establishes the current path and result.
4. Make prior `needs_validation`, `deferred`, `blocked`, `out_of_scope`, and changed-source units
   current work. A still-external `needs_validation` record may be carried only after the current
   source trace is checked and linked by fingerprint to a current `planned` unit whose verifier
   re-check supplies its owner and evidence; the record keeps the unresolved blocker. These prior
   states never suppress a current unit.
5. A prior same-source covered unit may inform priority, but it stays visible in the current ledger.
   A prior `rejected` record suppresses only the unchanged failed claim, not coverage of its unit;
   changed evidence creates current work.
6. Read the prior profile and scope. A prior `quick` or scoped ledger contributes only its recorded
   evidence and gaps, never an implied "rest is fine."

If no prior ledger exists, say so in the final coverage statement. Never imply that one run exhausts
the target.

### Run profiles and scope

Pick a profile from the user's request or propose one from the target's size and stakes. Record it in
`run-metadata.json` (`profile`, `scope_paths`) and state it in the report. Default is `standard`.

- **`quick`** — a bounded pass for small targets, re-runs, or a fast first look. Coarsen ledger units
  to surface × boundary × attack class (subsystem uses the fixed canonical
  `profile/quick/all-in-scope-subsystems` identifier), run exactly one hunter wave followed by
  exactly one final coverage-critic pass, and give each candidate one fresh verifier for both
  candidate validation and final record verification. Do not launch a follow-up hunter wave: record
  the critic's accepted discoveries and reassignments as `deferred`.
- **`standard`** — the workflow as written.
- **`deep`** — for high-stakes or large targets. Split ledger units per subsystem and lifecycle mode,
  run critic waves to a clean pass, keep candidate validation and final record verification as
  separate fresh subagents, and give `prior_covered_same_source` units an independent second pass.

A **scoped run** audits a subset: named paths, one subsystem, one companion domain, or the diff
between two source refs. Seed ledger units only for in-scope surfaces and record everything else as
`out_of_scope` — never as `covered`. A scoped or `quick` run presents itself as partial coverage.

Profiles change breadth and redundancy, never the evidence bar. Do not scale away the candidate gate,
the source-only boundary, `needs_validation` discipline, schema validation, or independent
verification of `confirmed` records.

#### Cost budget

The ledger makes spend countable: one unit is roughly one hunter assignment, and one surviving
candidate is one or two verifier assignments depending on profile. When the user sets a budget — or
the parent proposes one for a large target — record `budget` in `run-metadata.json` as a maximum
number of `subagent` invocations across all phases.

Apply the strict budget gate before launching any reconnaissance subagent. Reserve the four baseline
reconnaissance calls, one final post-wave critic for `quick` or one post-wave plus one distinct
final-clean critic for `standard`/`deep`, and at least one verifier call. Add focused reconnaissance
only after repeating this gate for each extra call. If the requested budget cannot fund that minimum,
launch no subagent: ask for a larger budget, narrower scope, or different profile. If the request
stays unchanged, the run ends before Phase 1 — a pre-run exit, not a mid-phase stop: create
`<run-dir>/` holding only `run-metadata.json` with `run_status: "incomplete"` and
`incomplete_reason: "budget_cannot_fund_reconnaissance_and_reserves"`, create no report note and no
other run artifact, and state the gap and the minimum the gate needs in the reply to the user.

Spend it in this order:

1. Count reconnaissance, every post-wave critic, and the separate final-clean critic as `subagent`
   invocations.
2. **Reserve critics and validation before hunting.** For `quick`, reserve its one post-wave final
   critic. Before every `standard` or `deep` hunter wave, reserve one immediate post-wave critic plus
   one distinct final-clean critic. Also reserve verifier cost from the profile (about 1 or 2
   subagents per expected candidate; when in doubt reserve 30% of the balance after critic
   reservation). Never assign hunters into either reserve.
3. Assign hunters to units in priority order until the hunting allowance is spent. Spend the
   reserved post-wave critic immediately after that wave; keep the final-clean and validation
   reserves intact.
4. Before a later wave, reserve its new post-wave critic again. If the remaining budget cannot cover
   the required critic calls and validation reserve, launch no hunters from that wave, mark its
   planned units `deferred` with reason `budget_cannot_reserve_critics_and_validation`, and use the
   retained final-clean critic to record the resulting gap.

Before wave 1, update the pre-recon estimate with seeded units, implied hunter count, mandatory critic
calls, validation reserve, and whether the remaining budget covers the plan. If it clearly cannot,
say so and propose a tighter scope or a coarser profile instead of silently thinning evidence. If
later facts consume the required final-critic reserve, launch no hunters, mark all planned work
deferred, set the run incomplete with reason `critic_budget_exhausted`, and make no complete-coverage
claim.

A strict total-subagent budget can still be exceeded by an unexpectedly large candidate set or by a
material Phase 5 replacement that needs another independent verifier. If the remaining budget cannot
validate every candidate, stop hunting, validate candidates in fingerprint order while the budget
permits, and set `run_status: "incomplete"` plus
`incomplete_reason: "validation_budget_exhausted"`. Keep each unvalidated fingerprint linked to a
`candidate` ledger unit with that unresolved reason. Do not put an unvalidated candidate in
`findings.json`, relabel it `needs_validation`, or report the run as complete. Phase 6 may produce a
partial report only if its first section states that candidate validation is incomplete and lists the
affected fingerprints and units. Never exceed a user-set strict budget silently.

## Core principles

### Require a boundary and result

For every candidate, name the lower-trust principal, accepted input or action, intended control,
crossed boundary, affected principal or resource, and concrete source-derived or owner-observable
result. Do not elevate a missing best practice, guessed deployment behavior, generic parser crash, or
self-impact into a security finding.

### Use source-derived evidence

The source trace establishes both the path and the result. Follow the accepted input through parsing,
identity, authorization, normalization, state, derived copies, and the final sink, and state what the
code deterministically does at each step. Stop at the smallest source-visible outcome that settles the
invariant: a missing check on the path, an unbound selector, a control that a sibling path omits, a
value that reaches a sink unencoded. Do not extend reasoning past the boundary result, and never
describe a runtime effect the source does not establish.

### Respect source visibility

Deployment controls, proxy behavior, provider settings, browser headers, identity policy, broker
ACLs, packaging, and topology are real controls. If they are required and absent from the repository,
do not assume either presence or absence. Use `needs_validation` with the exact missing fact and a
safe owner-run plan.

### Separate priority from certainty

Only `confirmed` records receive severity. Likelihood and impact must reflect the demonstrated
conditions and result; overall severity cannot exceed demonstrated impact. `needs_validation` means a
specific source-grounded boundary hypothesis is blocked, not a low-confidence confirmed vulnerability,
and it has no severity.

Calibrate overall severity with these anchors:

- **critical** — an unauthenticated actor gains code execution, full data-store access, or takeover of
  arbitrary accounts.
- **high** — an actor fully defeats an explicit security control with real consequences:
  authentication bypass, cross-tenant read or write, stored script execution affecting other users,
  authenticated code execution, or an unauthenticated remote stop of a shared service.
- **medium** — a real boundary violation with limited blast radius, uncommon preconditions, or
  consequences confined to a narrow resource set.
- **low** — disclosure of non-secret internals, or an effect requiring sustained effort for minimal
  gain.
- **informational** — a confirmed but minimal-impact observation, useful mainly as a prerequisite
  inside a larger finding.

The high/medium discriminator: does the source demonstrably defeat an explicit control for an action
with real consequences, or only weaken it? If you cannot state the concrete damage the code permits,
the severity is lower than it feels.

### Recommend the smallest effective source fix

For each confirmed finding, identify the invariant the code must enforce and the narrowest source
change that enforces it at the last trusted decision point. Prefer specific repository-relative
changes and regression tests over generic hardening advice. The audit describes fixes; it does not
modify target source.

## Full audit workflow

Follow all six phases in order:

1. **Reconnaissance** — map the source, trust boundaries, companion selections, prior evidence, and
   initial deterministic coverage ledger with [RECONNAISSANCE.md](RECONNAISSANCE.md).
2. **Coverage-led hunting waves** — assign fresh hunters from the ledger and collect structured
   candidate results with [HUNTING.md](HUNTING.md), [ATTACK-CLASSES.md](ATTACK-CLASSES.md), and the
   selected domain companions.
3. **Candidate validation** — consolidate fingerprints and give every candidate to a fresh source
   verifier as defined in [VALIDATION-AND-REPORTING.md](VALIDATION-AND-REPORTING.md).
4. **Structured output** — write all final `confirmed`, `needs_validation`, and `rejected` records to
   `findings.json`; validate it with `report-schema.json` and `validate-findings.cjs`, and validate
   the coverage claim with `validate-coverage-ledger.cjs`.
5. **Independent record verification** — use fresh subagents to verify final source claims and
   reconcile corrections or state changes.
6. **Target-neutral report** — derive the report note, `FINDINGS-DETAIL.md`, and
   `NEEDS-VALIDATION.md` from the final records, with no live-probe instructions.

Do not end the run before one of exactly two terminal states: (a) all Phase 6 artifacts are written
and both validators pass, or (b) `run_status: "incomplete"` is recorded with its exact reason and the
gap is disclosed in the report. Never stop mid-phase.

## No penalty for a clean audit

A clean run may have zero confirmed records. State that result, the coverage limits, and the
unvalidated leads without inventing LOW findings to look thorough. Reporting an honest clean pass
with named gaps is a better outcome than padding the report.

## Anti-patterns

1. Checklist deviations presented as vulnerabilities.
2. Defense-in-depth advice with no reachable boundary violation.
3. Reaching for execution — a build, a test, a browser, an emulator, a fuzzer, a live or shared
   environment — instead of recording `needs_validation` with the exact unresolved fact.
4. Guessing provider, proxy, browser, identity, or deployment behavior not present in source.
5. Treating intended same-principal authority or self-impact as a cross-boundary result.
6. Reporting a parser or runtime effect stronger than the source establishes.
7. Emitting prose-only hunter results that cannot be deduplicated or verified.
8. Re-reporting carried same-source prior confirmed records or using them as exemplars that anchor the
   hunt.
9. Assigning severity to `needs_validation` records.
10. Writing the report before independent verification or letting prose and JSON disagree.
11. Claiming runtime confirmation the audit never obtained.
