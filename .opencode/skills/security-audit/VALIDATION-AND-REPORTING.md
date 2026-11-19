# Validation, Structured Output, Verification, and Reporting

### Phase 3: Independently validate every candidate

After the clean coverage-critic pass or an explicitly recorded early stop, consolidate Phase 2
candidates and carried same-source prior confirmations by stable fingerprint and root cause. Give
every unique proposed `confirmed` and `needs_validation` candidate to a fresh verifier via the
`subagent` tool — a security-review agent that did not hunt it. A carried prior confirmation follows
the same current verification path even though hunters exclude that unchanged root cause. A verifier
may read the hunter's returned evidence but must re-read every cited current source location and
rebuild any decisive trace from source itself.

Assign each verifier a canonical lowercase unique handle. It receives only the candidate, its linked
coverage-unit checks, architecture facts needed to interpret the path, the exact relevant companion
validation blocks, the source-only evidence boundary, the `confirmed`, `needs_validation`, and
`rejected` branches of `report-schema.json` copied verbatim, and prior records with the same
fingerprint. It must not receive another verifier's conclusion.

#### Candidate-verifier prompt

```text
You did not write this candidate. Try to refute it from repository source. Read only: do not build,
run, test, fuzz, or probe the target, and do not contact deployed endpoints or external/shared
services. Anything that needs execution is a needs_validation blocker, not something you run.

1. Verify every trace and evidence file, positive line number, scope, and description.
   Confirm the first entry is a real lower-trust entrypoint and the last is the
   claimed sink or boundary effect.
2. Reconstruct the strongest source-visible validation, identity, authorization,
   normalization, lifecycle, framework, and containment controls on the path.
   Where the architecture summary names a comparable baseline, note whether it
   shares the pattern — as calibration, never as grounds to dismiss.
3. For a proposed confirmed candidate, confirm the source deterministically reaches the claimed
   boundary result: the accepted input, the checks on its path, and the outcome the code produces.
   Reject any claim of runtime output that the source does not establish. Do not infer a stronger
   result than the source shows.
4. Verify that likelihood, impact, confidence, and the proposed source fix match only
   what the evidence establishes.
5. For a proposed needs_validation candidate, decide whether the blocker is genuinely outside
   source. If source refutes the trace, reject it. If the missing fact remains decisive, keep
   needs_validation and make the owner-run plans exact, minimal, and non-destructive.
6. Preserve the fingerprint for the same source-derived root cause across every state.

Return exactly one JSON object and no surrounding prose:
{"decision": "confirmed|needs_validation|rejected", "record": { ... }}
where record exactly matches the decision's verdict branch of the schema included
in this prompt. A corrected record replaces the hunter's wording.
```

A verifier can promote `needs_validation` to `confirmed` only after independently establishing the
complete source path and the source-derivable boundary result. Demote proposed confirmation to
`needs_validation` when a specific deployment or runtime fact remains unknown, or when the claim needs
execution the audit may not perform. Use `rejected` when source, a visible control, missing meaningful
impact, or an impossible prerequisite refutes the claim. `needs_validation` is never a parking place
for a speculative idea.

The parent checks that each verifier returned the same fingerprint unless it identified a genuinely
different root cause. Merge corrections, record the decision in every linked coverage unit, and
ensure there is one final record per fingerprint. Discard a malformed or prose-wrapped verifier result
without repairing it; re-run that candidate with a fresh verifier when the budget permits, otherwise
it remains an unvalidated ledger candidate under the incomplete-run rule.

When verifier evidence updates a ledger check, set that check's `agent_id` to the verifier's handle
and list its nonempty repository-relative `reviewed_paths`. Keep the unit-level `reviewed_paths` equal
to the union across checks. Use `method: "source"` with `artifact: null` for every check. The unit
retains its original assignment owner, so independently owned hunter and verifier checks can coexist.
For a carried prior record's seeded `planned` unit there is no prior owner: the verifier that
re-checks it becomes the unit's assignment owner, and its re-check is the unit's first check, moving
the unit to `candidate` with the carried fingerprint.

If a strict total-subagent budget cannot cover every candidate, set the run status to incomplete and
follow the deterministic budget rule in `SKILL.md`. An unvalidated candidate remains only in the
ledger. It does not enter `findings.json` under any verdict.

### Phase 4: Write and validate `findings.json`

The parent writes all independently decided records to `<run-dir>/findings.json`, sorted by
fingerprint. Include:

- `confirmed`: source-grounded vulnerabilities with a complete trace, the source-derivable boundary
  result, conditions, specific remediation, likelihood/impact/overall severity, and confidence.
- `needs_validation`: source-grounded candidates with an exact unresolved blocker and at least one
  applicable owner-run `local` or owner-observed `deployment` plan.
- `rejected`: source-grounded candidates disproved during validation, retained so future runs do not
  repeat the unsupported claim without changed evidence.

Read `report-schema.json` immediately before writing. It uses `additionalProperties: false`; do not
carry hunter wrapper fields into a record. Keep these verdict contracts distinct:

- A `confirmed` record uses `root_cause`, `intended_behavior`, `conditions`, `execution`,
  `remediation`, `severity`, and `confidence`. It must not use `claimed_root_cause`, `blockers`,
  `validation_plan`, or `reason`. `execution` is target-neutral and uses the target's native
  interface: API/HTTP input, CLI call, library call, message, file fixture, browser action, rendered
  policy, or harness as applicable. `observed_result` is nonempty and factual — under this skill it
  records what the source deterministically establishes, never invented runtime output.
- A `needs_validation` record uses `claimed_root_cause`, `trace`, `evidence`, `blockers`, and at least
  one nonempty `validation_plan.local` or `validation_plan.deployment` field. Include both only when
  both contexts can resolve distinct facts. It must not use severity, execution, remediation, reason,
  or confirmed root cause.
- A `rejected` record uses `claimed_root_cause`, `trace`, `evidence`, and `reason`. It must not use
  severity, execution, remediation, blockers, validation plan, or confirmed root cause.

Every record has a stable fingerprint, title, description, and repository-relative source paths. A
multi-step trace begins with `entrypoint`, ends with `sink`, and uses `propagation` only between them.
One-entry traces use `entrypoint` or `sink`. Overall severity cannot exceed demonstrated impact.

Run:

```sh
node .opencode/skills/security-audit/validate-findings.cjs <run-dir>/findings.json
node .opencode/skills/security-audit/validate-coverage-ledger.cjs <run-dir>/coverage-ledger.json
```

Use the real skill directory when it lives elsewhere (for example under
`~/.config/opencode/skills/security-audit/`). Fix every structural and semantic error before
continuing. The findings validator rejects input beyond 5 MiB, 1,000 top-level findings, or 64 nesting
levels, and caps reported error output at 100 messages. Validator success proves format and ledger
consistency only.

### Phase 5: Verify the final records with fresh eyes

Launch one fresh verifier per final `confirmed` and `needs_validation` record, in parallel, with a
security-review agent that did not hunt or perform Phase 3 on that candidate. This verifier checks
the structured record, not the hunter write-up, and stays inside the source-only boundary.

In a `quick` run, Phase 3 and Phase 5 merge: the Phase 3 verifier also performs these record checks
and returns the final schema-shaped record, so each candidate gets one fresh independent reviewer
instead of two. Every other profile keeps the two passes separate. Never skip independent review of a
`confirmed` record in any profile.

For `confirmed`, require it to check:

1. Every repository-relative trace/evidence path, line, scope, and described operation.
2. Real entry interface and the input shape the source accepts.
3. Every condition, parser/policy step, source-visible preventing layer, and the source-derivable
   result.
4. Affected principal/resource and demonstrated impact.
5. Severity separation: realistic likelihood, demonstrated impact, overall no greater than impact.
6. Remediation strategy and any `code_changes`, including whether the fix enforces the invariant
   without merely moving trust.

For `needs_validation`, require it to check:

1. The source path is real and supports only the `claimed_root_cause` stated.
2. Every listed blocker is decisive and not already answerable from source.
3. The candidate names a boundary and a possible concrete result rather than a generic concern.
4. At least one validation-plan field is present and exact. `local` is an owner-run, bounded fixture;
   `deployment` asks an owner to observe a configuration, identity, route, policy, or runtime fact.
   Do not invent a plan for an inapplicable context, and never send audit traffic to a deployment.
5. The fingerprint matches prior/current records for the same root cause.

Each verifier returns exactly one JSON object: `{"decision":"verified","fingerprint":"..."}` or
`{"decision":"replace","reason":"...","record":{...}}`, with no surrounding prose. A replacement
record must match its `confirmed`, `needs_validation`, or `rejected` schema branch. Treat a malformed
or prose-wrapped Phase 5 result the same way as in Phase 3: discard it without repairing it and re-run
with a fresh verifier when the budget permits.

Do not apply a Phase 5 replacement as final when it promotes a record to a stronger verdict, including
any promotion to `confirmed`, or materially changes the root cause, trace, payload or instructions,
the source-derivable result, demonstrated impact, or severity. Give that complete replacement to a new
independent verifier that did not hunt, perform Phase 3 validation, or propose the Phase 5
replacement. The new verifier re-reads the current source, then returns `verified` or another
replacement. Apply a material replacement only after this fresh verification. If another material
replacement results, repeat with a fresh verifier. If budget or independence is unavailable, remove
the disputed record from `findings.json`, keep its ledger unit as an unresolved candidate, and set
`run_status: "incomplete"` with an exact `incomplete_reason`. Non-material wording or repository-line
corrections may be applied directly when they do not change meaning or evidence.

After every applied replacement, rerun both validators and update linked ledger decisions. If a final
verifier identifies a separate root cause, assign a new fingerprint and send it through independent
candidate validation before inclusion. Set `run_status: "complete"` only when every ledger candidate
has an independent final disposition and every retained record passes Phase 5.

Do not verify only `confirmed` records. A misleading `needs_validation` handoff wastes owner time and
can preserve a false premise.

### Phase 6: Produce target-neutral reports from final records

Only after Phase 5 passes for every record retained in `findings.json`, derive prose from the final
records, the ledger, and the hunter `hardening` notes retained in ledger bookkeeping. An incomplete run
may report independently verified records, but it must identify each unresolved ledger candidate and
must not present it as a finding. The prose never changes a verdict, severity, blocker, or demonstrated
impact.

Write three artifacts:

| Artifact | Location | Purpose |
|---|---|---|
| Report note | `<run-dir>/{same stem}-report.md` | The report. Only report artifact. |
| `FINDINGS-DETAIL.md` | `<run-dir>/` | Full source trace and reproduction for confirmed medium+ records |
| `NEEDS-VALIDATION.md` | `<run-dir>/` | Prioritized unresolved leads with owner-run plans |

Do not also write a `REPORT.md`. The note is the report.

#### Report note

Write the note with the **`note-write` skill's** file mechanics and frontmatter schema, inside
`<run-dir>` beside the machine artifacts; point its `related` entries at those sibling files
(`run-metadata.json`, `architecture.md`, `coverage-ledger.json`, `findings.json`,
`FINDINGS-DETAIL.md`, `NEEDS-VALIDATION.md`) so moving the directory preserves every link. Its
frontmatter `tags` MUST include `security-audit`. The note body:

1. **Run** — profile, scope, budget (if set) with subagent invocations spent versus planned, source
   ref, the `execution_policy: "source-only"` statement, prior-run use, and explicit deferred and
   out-of-scope coverage. Name carried same-source confirmations and changed-source revalidations. A
   `quick`, scoped, budget-limited, or incomplete run states plainly that it is a partial pass. If
   candidate validation exhausted a strict budget, state that the run is incomplete and list every
   unvalidated fingerprint and linked unit; do not describe those candidates as findings. If the
   budget prevented a mandatory critic, state which critic did not run and make no clean-coverage
   claim.
2. **Posture** — one short security-posture summary.
3. **Confirmed findings** — a table: severity, title, affected boundary, one-line source-derived
   result.
4. **Finding detail** — for each confirmed finding: repository source location, lower-trust
   principal, target-native reproduction the owner can run, conditions, the result the source
   establishes, impact, priority rationale, and smallest source fix.
5. **Needs validation** — a separate table. Title, repository trace, exact blocker, owner-run next
   step, and safe owner-observed deployment check. No severity, never called a confirmed
   vulnerability.
6. **Hardening and positive patterns** — concrete non-findings and source patterns worth keeping.
7. **Coverage** — from the ledger: covered, candidate, blocked, and deferred counts, important
   exclusions, and the final critic result.

Do not describe rejected records as findings. Mention their fingerprints only when they explain a
prior disagreement or coverage decision.

#### `FINDINGS-DETAIL.md`

For each confirmed `medium`, `high`, or `critical` record, copy the complete source path and the
target-neutral reproduction:

- ordered repository-relative trace and evidence;
- dummy attacker/principal and affected dummy resource;
- native input, invocation, or fixture, with the exact steps an owner would run;
- the result the source establishes and the security invariant it proves;
- conditions and containment;
- source-level remediation and regression case.

#### `NEEDS-VALIDATION.md`

For every unresolved record, copy the source trace, verified evidence, exact blocker, affected
boundary, and each applicable owner-run or owner-observed resolution plan. Keep these as prioritized
leads without severity. Do not turn them into live test guidance or assume the missing deployment
fact.

HTTP is one possible native interface, not the default. A library finding may use a function call, a
parser a fixture, a CLI a command, a desktop app an IPC or file action, and infrastructure a rendered
policy. Do not require an endpoint, external account, or live environment that the target does not
have.

Keep the report proportional to the evidence. A clean run may have zero confirmed records. State that
result and the remaining coverage/validation limits without inventing LOW findings.
