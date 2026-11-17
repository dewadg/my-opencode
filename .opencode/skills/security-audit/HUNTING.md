# Vulnerability Hunting

### Phase 2: Run coverage-led hunting waves

The parent assigns `planned` ledger units to fresh subagents: a general investigator for a broad
multi-file sweep, a security-review agent when the unit turns on boundary judgement, the language
engineer when the trace depends on stack semantics. Use enough focused hunters to cover the units
without combining unrelated boundaries. One hunter may own closely related units in one subsystem; no
unit may be silently unassigned because of a subagent-count limit — a unit the budget cannot reach is
explicitly `deferred` with reason `budget_cannot_reserve_critics_and_validation`.

When a budget or profile caps hunter count, assign units in priority order and record the ordering
rationale in the ledger. Rank by: (1) unauthenticated or lowest-trust entry surfaces before
authenticated ones; (2) boundaries protecting the most valuable resources (credentials, cross-tenant
data, code execution, release authority); (3) prior-run gaps, revalidation targets, and changed
source before same-source re-passes; (4) units whose class historically yields confirmed findings for
this target type over speculative ones. Ties break lexicographically by `coverage_id` so runs stay
deterministic.

Before launch, the parent changes assigned units to `in_progress` and sets a canonical lowercase
`agent_id` handle. Hunters read source and parent-provided context only, and return one structured
result through the `subagent` tool. They never write files, edit target source, or touch
`architecture.md`, `coverage-ledger.json`, `findings.json`, or any run artifact.

## Required hunter prompt

Every hunter prompt contains these parts in this order:

1. A two-sentence role preamble: the hunter's goal is to find source-grounded security invariant
   failures in its assigned units, and it must return exactly one JSON object matching the
   structured-result contract at the end of this prompt.
2. `architecture.md` verbatim.
3. Assigned coverage IDs, subsystem, boundary, repository-relative starting paths, and each unit's
   assignment block map from `coverage-ledger.json`.
4. The exact selected blocks, copied verbatim: each selected ordinary attack-class block from
   `ATTACK-CLASSES.md`, and from each selected companion its `Core discipline`, each chosen
   attack-class subsection, `Universal moves`, and `Validation rules`. Ordinary blocks are
   self-contained and carry no companion-style `Core discipline`, `Universal moves`, or
   `Validation rules` sections. Do not send block or companion names alone.
5. Explicit excluded ordinary and companion blocks with a reason for each exclusion.
6. The core hunting method below.
7. The core validation rules below.
8. Carried same-source prior confirmed exclusions, each limited to fingerprint, title, and root cause,
   plus peer-owned current coverage IDs that this hunter must not duplicate.
9. The structured-result contract, including the Structured hunter result block below and the
   `confirmed` and `needs_validation` branches of `report-schema.json` copied verbatim.

A prompt may select several companion blocks when the same path crosses several domains. Keep their
constraints together. Scope is the hunter's coverage obligation, not permission to duplicate excluded
work. If an unexpected different boundary appears, return it under `uncovered` so the parent creates
a stable ledger unit and assigns it in the next wave.

#### Core hunting method — include in every hunter prompt

```text
## Defensive vulnerability-finding method

Your goal is to find source-grounded security invariant failures and the smallest fix, not to expand
harm beyond the boundary result. Read code and reason about it. Do not build, run, test, fuzz, or
probe the target, and do not contact deployed endpoints, provider APIs, registries, identity systems,
message brokers, shared services, or other users. Anything that would only be settled by execution
belongs in a needs_validation record.

READ THE CODE AT DEPTH. Follow each assigned input through parsing, identity, authorization,
normalization, state, derived copies, and the final sink. Read sibling, legacy, batch, retry,
cancellation, migration, and error paths that produce the same effect. Compare sibling controls for
equivalence, not only presence, and compare what one component guarantees with what the next
component assumes.

WORK FROM A CONCRETE INVARIANT:
1. Name the lower-trust principal and starting capability.
2. Name the accepted value, action, state transition, or resource selector.
3. Locate the control that should reject, bind, isolate, limit, or revoke it.
4. Trace the exact source path after that decision.
5. Stop at the smallest source-visible outcome that settles the invariant: the missing or mismatched
   check, the unbound selector, the value that reaches the sink unchanged, the control a parallel
   path omits.
6. State a source-level change and regression case that enforce the invariant.

DEPTH BOUND: trace only paths that can reach your assigned boundary or whose guarantees that boundary
relies on. Stop a line of investigation as soon as the invariant is settled either way, and record
the result in your structured output — a covered, candidate, or blocked disposition, or an
`uncovered` entry — instead of continuing to search.

TEST SAD PATHS AND DISAGREEMENTS IN SOURCE. Check absent, empty, zero, negative, maximum, over-limit,
duplicate, mixed encoding, stale, revoked, reordered, concurrent, partially migrated, failed
dependency, and rollback handling only where the interface accepts them. Compare canonicalization and
units at every parser or policy handoff. For multi-step issues, treat each output as a prerequisite
and do not assume a later boundary. If any prerequisite is not established, record a blocker.

When a proposed high or critical candidate reveals a reusable root cause, search paths owned by the
assigned coverage IDs for lexical, structural, and logical variants. Consolidate the same root cause,
but establish each variant's conditions and impact independently. Do not investigate peer-owned
units. Return a variant with no current coverage unit as `uncovered`.

STATE THE RESULT THE SOURCE DETERMINES. A confirmed candidate needs a complete trace that ends in a
concrete source-visible outcome — a record that is returned, a branch that is taken without the
check, a selector that is trusted, a credential that is disclosed in output. Do not describe a
runtime effect the code does not establish. Do not invent output, status codes, stack traces, timing,
or sanitizer results.

IF IT NEEDS EXECUTION, IT IS needs_validation. A build, test, harness, fixture run, browser,
emulator, fuzzer, live endpoint, or shared environment could answer a question the source cannot.
Record that exact question and the smallest safe owner-run check — command, input, or observation —
that would settle it. Never run that check yourself.

A deployment, browser, provider, broker, OS, proxy, package, secret, or identity fact outside source
is not proof either way. If one such fact is decisive, return a needs_validation record with the
exact missing observation and safe owner-observed check.
```

#### Core validation rules — include in every hunter prompt

```text
## Candidate gate

1. A candidate needs a complete repository-relative source trace and evidence for the claimed root
   cause, including the strongest source-visible control.
2. A proposed confirmed record needs the source trace to reach a concrete, source-derivable boundary
   result: what the code deterministically does with the accepted input, and no visible preventing
   layer on that path.
3. Do not strengthen a missing check into code execution, ordinary work into shared availability, or
   a same-principal action into privilege gain.
4. If a required fact is outside source, use needs_validation. Name exact blockers; do not give it
   severity or speculative completion.
5. A missing best practice with no affected principal/resource is excluded or hardening, not a
   finding. A candidate disproved by source is not needs_validation.
6. Use the same source-derived fingerprint for the same root cause in every state. It must match
   `^[A-Za-z0-9][A-Za-z0-9._:/@+-]*$` and must not include a line, wave, agent, severity, or verdict.
7. Return an empty candidate array when nothing survives these gates.
```

## Source-only evidence boundary

The hunter reads; it does not run:

- **Allowed:** `read`, `grep`, and `glob` over source, tests, build definitions, manifests,
  infrastructure code, lockfiles, migrations, and local documentation. Reading a test or script to
  learn the intended behavior is allowed; executing it is not.
- **Disallowed:** target builds, test runs, scripts, CLIs, servers, browsers, emulators, fuzzers,
  fixture processing, dependency installation, live or deployed traffic, real accounts or
  credentials, shared queues, cloud resources, runners, registries, signing or release services,
  publishing, stress, saturation, or cost generation.

A missing execution capability does not erase a source-grounded candidate; represent the exact
execution-only question as a `needs_validation` blocker with an owner-run plan.

## Structured hunter result

Return exactly one JSON object, with no surrounding prose:

```json
{
  "units": [
    {
      "coverage_id": "one assigned ID",
      "disposition": "covered|candidate|blocked",
      "reviewed_paths": ["repo/relative/path"],
      "checks": [
        {
          "agent_id": "canonical owner of this check",
          "reviewed_paths": ["repo/relative/path owned by this check"],
          "invariant": "specific control checked for this unit",
          "method": "source",
          "result": "what the source establishes",
          "artifact": null
        }
      ],
      "candidate_fingerprints": [],
      "unresolved": []
    }
  ],
  "candidates": [],
  "hardening": ["concrete non-finding note"],
  "uncovered": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "reason": "why this needs its own deterministic coverage unit"
    }
  ]
}
```

Each `candidates` entry is schema-shaped except that it uses `proposed_verdict` in place of `verdict`:

- `proposed_verdict: "confirmed"`: include every field required by the `confirmed` branch of
  `report-schema.json` other than `verdict`: `fingerprint`, title, description, `root_cause`,
  `intended_behavior`, ordered `trace`, `evidence`, `conditions`, target-neutral `execution`,
  `remediation`, `severity`, and `confidence`. `execution` describes the reproduction an owner could
  run and the source-derivable result: `payloads` holds the minimal trigger input as the source
  defines it, `instructions` states the reproduction steps derived from the trace, and
  `observed_result` records what the source deterministically establishes — never invented runtime
  output. Overall severity must not exceed the demonstrated source-level impact.
- `proposed_verdict: "needs_validation"`: include every field required by that schema branch other
  than `verdict`: `fingerprint`, title, description, `claimed_root_cause`, ordered `trace`,
  `evidence`, nonempty `blockers`, and `validation_plan` with at least one applicable `local` or
  `deployment` step. Do not invent an inapplicable context. Do not include severity, execution,
  remediation, reason, or a confirmed `root_cause`. `local` is an owner-run check on a bounded
  fixture; `deployment` is an owner-observed configuration fact, not a request to probe a live
  target.

Every assigned coverage ID appears exactly once in `units`. A `covered` unit needs an owner, nonempty
`reviewed_paths` and `checks`, no unresolved fact, and no candidate. A `candidate` unit has the same
owned evidence and is the only state that carries linked fingerprints. A `blocked` unit is an owned
partial review with nonempty paths, checks, and unresolved facts but no fingerprint. All source paths
are repository-relative, never absolute or traversal paths. A trace with several entries begins at
`entrypoint`, ends at `sink`, and labels intermediate steps `propagation`. Every check has its own
agent handle and nonempty `reviewed_paths`; the unit-level list is exactly the union of those owned
paths. Every check uses `method: "source"` and `artifact: null`. Never return a local check or an
artifact path.

## Parent consolidation and ledger update

The parent validates each unit result, maps it to exactly one assigned `coverage_id`, and updates only
that ledger unit. Reject duplicate or absent IDs, unsafe unit or check agent handles, and any
`method: "local"` check or non-null `artifact`. Copy the unit's `reviewed_paths`, its `checks` into
the unit's `local_checks`, candidate fingerprints, and unresolved facts into the ledger. Retain each
hunter's `hardening` list in a parent bookkeeping field on the relevant units (outside the semantic
fields) so Phase 6 can report it. A failed, malformed, or unsupported conclusion leaves that unit
`planned` for reassignment. Untouched budget/profile units become unassigned `deferred` units with
empty evidence and a reason; do not hide partial evidence in `deferred`. Run
`validate-coverage-ledger.cjs` after the update; an invalid ledger cannot drive another assignment.
This per-unit contract allows one hunter to close one unit while returning a candidate or blocker for
another.

Consolidate candidate entries by fingerprint and then by root cause. One root cause that exposes
several entry paths or effects is one candidate with the strongest complete trace. Related but
independent missing controls use separate fingerprints. Record duplicate fingerprints in the relevant
ledger unit and do not send duplicate candidates to validation.

## Coverage-critic waves

Immediately after each hunter wave, spend the reserved invocation on one fresh coverage critic: a
security-review agent that reads source, writes nothing, and runs nothing. It receives
`architecture.md`, the full coverage ledger including each assignment block map, current candidate
fingerprints and states, and the prior-ledger gap summary. Require exactly this JSON:

```json
{
  "missing_units": [
    {
      "surface": "...",
      "boundary": "...",
      "subsystem": "...",
      "attack_class": "...",
      "starting_paths": ["repo/relative/path"],
      "selected_companion_blocks": ["FILE.md#section"],
      "excluded_blocks": [{"block": "FILE.md#section", "reason": "..."}],
      "reason": "source-backed coverage gap"
    }
  ],
  "reassign_ids": ["existing-id-that-did-not-close"],
  "resolved_prior_leads": ["fingerprint"],
  "stop": false
}
```

The critic checks for unmapped entry points, unchecked parallel paths, missing lifecycle modes,
selected companion classes without a unit, unjustified exclusions, units closed without paths/checks,
and prior `needs_validation` or changed-source gaps that no unit addresses. It proposes coverage, not
findings. `stop` is the critic's own assessment: `true` only when it accepts no `missing_units` and no
`reassign_ids`; the parent's loop condition below, not `stop` alone, decides whether another wave
runs. For each fingerprint in `resolved_prior_leads`, the parent marks the linked unit or prior-lead
entry resolved and records the critic's source-backed reason.

The parent rejects proposed units outside the review scope or source-only boundary, derives canonical
IDs for accepted units, and deduplicates them against current units. A prior same-source completed
unit may supply evidence; prior `deferred`, `blocked`, `out_of_scope`, or changed-source units become
current work and never suppress an accepted unit. Fail rather than merge a canonical ID collision. For
each legitimate `reassign_id` with live `blocked`, `covered`, or `candidate` evidence, append that
exact terminal record to the unit's `attempts` with the critic's source-backed
`reassignment_reason`. Preserve its owner, checks, fingerprints, and unresolved facts in that archive.
Increment the live `wave`; the next hunter must be a fresh owner and receives an `in_progress` unit
with empty live evidence. The hunter's terminal result writes only its new evidence into the live
fields. Never copy an archived owner's checks into the new live attempt. Sort IDs and validate the
ledger before another assignment. In `standard` and `deep`, when the post-wave critic reports no
accepted `missing_units` or legitimate `reassign_ids` and no `planned` units remain, spend the
separately reserved invocation on a distinct final-clean critic. Complete coverage only when that
critic also returns no accepted work. If it finds work, queue it and repeat the wave, post-wave
critic, and final-clean process. If time or resources force an early stop, mark every untouched unit
`deferred`, preserve the critic's reason, and disclose the gap in the report. Never use a silent wave
or subagent cap as evidence of complete coverage.

The run profile bounds this loop. A `quick` run has exactly one hunter wave followed by exactly one
final critic pass. Add each accepted `missing_unit` to the current ledger and mark it `deferred` with
reason `quick_profile_final_critic`. For each legitimate evidence-bearing `reassign_id`, archive the
live terminal state in `attempts`, increment `wave`, and set the live state to unassigned `deferred`
with empty evidence and reason `quick_profile_final_critic`. Do not launch a second hunter wave or
another critic. In a scoped run, the critic still reports out-of-scope gaps it notices, but the parent
records them as `out_of_scope` with the critic's reason instead of assigning them. The early-stop rule
above is the same mechanism: `quick` is a pre-declared early stop, not evidence of complete coverage.

A budget bounds it the same way. Before assigning each wave, compare remaining budget against its
hunter count, the validation reserve, the immediate post-wave critic, and the retained final-clean
critic (`quick` reserves only its single final post-wave critic). Shrink the hunter wave to fit, taking
units in priority order. If those mandatory reserves do not fit, launch no hunter from that wave and
mark its planned units `deferred` with reason `budget_cannot_reserve_critics_and_validation`.
Critic-proposed units enter the same ranked queue rather than extending the budget. If surviving
candidates exceed the validation reserve, follow the incomplete-run rule in `SKILL.md`: stop hunting,
validate in fingerprint order, retain unvalidated units as unresolved candidates, and never present
them as findings.
