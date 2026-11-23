# AGENTS.md

Guidance for OpenCode (V2) when working in this repository.

## What this repository is

A personal collection of OpenCode **configuration artifacts** — agent definitions, skills, slash
commands, and an MCP server config example. It is not an application and has no package manager: no
build or lint step, and the only runnable check is the security-audit validator suite,
`node --test .opencode/skills/security-audit/*.test.cjs`. The "code" here is markdown that OpenCode
itself consumes.

The `.opencode/` tree is live project config: OpenCode discovers it automatically while working in
this repo. To use the artifacts everywhere, copy them into the global config (plain copies, not
symlinks; there is no install script):

```bash
cp -R .opencode/agents/*   ~/.config/opencode/agents/
cp -R .opencode/commands/* ~/.config/opencode/commands/
cp -R .opencode/skills/*   ~/.config/opencode/skills/
```

`~/.config/opencode/` may also hold agents/skills that are not tracked here — this repo is a
curated subset, not a mirror.

## Layout

- `.opencode/agents/*.md` — one agent per file. Frontmatter + system prompt. Every agent is a
  subagent except `tech-lead.md`, the primary the orchestrators run in.
- `.opencode/commands/*.md` — one user-invoked slash command per file. Frontmatter (`description`,
  optional `agent`, `model`, `subagent`) + prompt body. `$ARGUMENTS` expands to the text after the
  command name, `$1`/`$2` to positional arguments, and `` !`shell` `` injects command output. The
  body reads as instructions to the current agent.
- `.opencode/skills/<name>/SKILL.md` — one skill per directory. The directory name is the skill ID.
  Supporting scripts and `references/` (long examples, checklists, per-seam briefs) live alongside
  (e.g. `.opencode/skills/telegram-notify/scripts/send_telegram.sh`).
- `opencode.example.json` — example `mcp.servers` block (`context7`, `gitlab`, `goland`,
  `chrome-devtools`, `figma`, `mobile-device`); merge it into a project's or the global
  `~/.config/opencode/opencode.json`. Secrets are `{env:VAR}` references, never literals. `goland`
  is main-conversation-only — no agent is restricted to it. The GitLab server defaults to
  `GITLAB_READ_ONLY_MODE: "true"`; flip it to `"false"` only when you want issue/MR writes.

## Agent frontmatter

```yaml
---
description: >                 # third person, trigger-oriented; this is what routes work here
  Use this agent when ...
mode: subagent                 # subagent | primary | all
model: provider/model#variant  # leave out — pinned in the global agents block
color: "#3b82f6"               # optional six-digit hex
permissions:                   # ordered rules; last match wins
- action: edit
  resource: "*"
  effect: deny
- action: subagent
  resource: "*"
  effect: deny
---
```

OpenCode names the agent from the filename stem. V2 has no agent fields for `tools`, `skills`,
`mcpServers`, `effort`, `temperature`, or `name` — do not add them. Express tool access as
`permissions` actions (`read`, `edit`, `shell`, `glob`, `grep`, `webfetch`, `websearch`, `subagent`,
`skill`, `question`, `execute`, or an MCP `<server>_<tool>` name). MCP servers are configured
globally under `mcp.servers`; per-agent server selection is not supported. Skills are advertised to
every agent; a skill's instructions can say which agent is expected to use it.

Subagent models are pinned centrally, not per file: every agent here omits `model` from its
frontmatter and takes one from the `agents` block of the global
`~/.config/opencode/opencode.json` (`#low` for producers and explorers, `#high` for reviewers,
auditors and advisors). The repo ships no project `opencode.json`, so a session in this directory
inherits the global config. Switch every subagent's model by editing that block — OpenCode reloads
config on save. Without a block, copied agents inherit the parent session model instead.

Body convention across agents: a role paragraph, then phased workflow sections — no standalone
"When invoked" list, which belongs inside the workflow. Each rule is stated once; restatements are
cut. Follow the shape of an existing agent when adding a new one.

## Skill frontmatter

```yaml
---
name: code                     # display name; the directory is the ID
description: >                 # what it does + explicit "use when" examples + "not for" cases
  ...
metadata:
  opencode/autoinvoke: false   # optional; hide from the model's list (still loadable by ID)
---
```

Descriptions carry the routing burden — they are how OpenCode decides to advertise a skill. Write
them with concrete trigger phrases and, where a sibling skill overlaps, an explicit exclusion (see
`note-write` vs `note-search`, which disambiguate each other in their own
descriptions). The same applies to command `description` frontmatter — `analyze` and
`investigate` exclude each other and `code`.

Claude Code skill fields (`allowed-tools`, `argument-hint`, `effort`) are not OpenCode V2 fields —
do not use them.

## Tool vocabulary

Artifacts in this repo use OpenCode V2 tool names, never Claude Code ones: `read`, `grep`, `glob`,
`edit`, `write`, `shell`, `webfetch`, `websearch`, `question`, `skill`, `subagent`, `execute`. Say
"the `subagent` tool" (not `Agent`/`Task`), "the `skill` tool with id `<id>`" (not `Skill`), "the
`question` tool" (not `AskUserQuestion`), "the `read` tool" (not `Read`). Subagents cannot spawn
subagents (nesting depth is one), so every orchestrator runs in the main thread and launches each
specialist itself.

## Cross-cutting contracts

These span multiple files; changing one means changing the others.

**Notes.** `note-write` and `note-search` define a shared note format that the `analyze` and
`investigate` commands and the `code` skill all depend on: files at `.notes/` (or wherever the
project's `AGENTS.md` says), named `{YYYY}-{MM}-{DD}[-{TICKET}]-descriptive-slug.md`, with
frontmatter `tags`, `projects`, `created`, `updated`, `related`, `description`. `note-search` greps
those exact fields — if the frontmatter schema changes in `note-write`, `note-search`'s grep recipes
must change with it. The `rca-write` skill owns the RCA note BODY section template (root-cause
sections) on top of `note-write`'s frontmatter/filename format; the `investigate` command invokes
`rca-write` to compile the RCA note. The `design-write` skill owns the design-doc note BODY section
template (analysis sections) the same way; the `analyze` command invokes `design-write` to compile
the design-doc note. If either skill's section template changes, the command that references those
sections must change with it.

A `code` session with more than two tasks or more than one project also journals: the tech lead is its
single writer, and `tech-lead.md` owns the journal's path and skeleton —
`.notes/code/{YYYY}-{MM}-{DD}[-{TICKET}]-{slug}/` with the same-stem `-journal.md` inside, archived by
one `mv` to `.notes/code/archive/`. One writer per file; the report cap bounds the message back, never
the record.

**Orchestrators.** The `analyze` and `investigate` commands and the `code` skill share one
architecture: plan the work as a checklist, delegate every project-level unit of work to a subagent
via the `subagent` tool, compile into a note (`investigate` compiles into an RCA note whose format
`rca-write` owns; `analyze` compiles into a design-doc note whose format `design-write` owns; both
still depend on `note-write`/`note-search` for file mechanics). They deliberately do **not** hardcode
an agent roster — they route by capability ("a service backend → the engineering agent for that
language") so that adding an agent to `.opencode/agents/` makes it usable without editing any command
or skill. Preserve that indirection; do not introduce agent-name lists into commands or skills.
All three run under the primary `tech-lead` (set by `agent:` frontmatter on the commands, and by
`default_agent` in the global `~/.config/opencode/opencode.json`), which is why the delegation
contract below binds them.
`analyze` and `investigate` also share a code-flow contract: when they explain how code flows, they
render it with the `call-graph` skill (indented arrow tree), never prose — so changes to the
`call-graph` output format affect what those commands emit.

**Coordination.** `tech-lead.md` is the primary that does not produce the product: it plans,
briefs, verifies returned reports, and publishes repo state (git history, remotes, PR/MR) inline;
it never runs builds or tests, or drives browser/device. Its role is prompt-enforced, not
permission-enforced — it keeps `edit`, `shell`, and the browser MCP tools, so nothing structurally
stops it from taking a subagent's work; what binds it is the evidence-attribution rule (every claim
cites a subagent `sessionID`) and the `code` skill's verification contract. That split — reversible
git/PR actions inline, destructive ones only on the user's explicit ask, adds by explicit path and
never `git add -A` — is also prompt-enforced, and it binds the `code` skill's rule that publication
stops at the rung the user named for the run (unstated: commit on the branch), with the tech lead
owning publication; change them together. The workflow statements are layered: invariants (rung,
autonomy, attribution, one agent per project) bind everywhere, while stage sequences, suite commands,
review steps and environment checks are defaults applied only where a project defines no convention —
so project rituals are discovered from its own `AGENTS.md`, README and CI config during recon, never
assumed, and an undiscoverable convention is asked once and journaled as a decision. Verification
splits along the builder/QA line: the implementing subagent owns all build and test-script execution —
by default a narrow build/lint/unit self-check for the packages it touched, then the full project
build and suite **once** per project, run by resuming that project's last implementing subagent after
the others finish. The QA agent owns only behavioral verification on the running app (endpoints via
hurl, browser/mobile flows from the user's point of view), never runs unit-test suites, and by default
runs **once**, last, after the review loop closes — so fix rounds never re-trigger it, and QA defects
loop back to the implementing agents with only the failed scenarios re-verified. One full run per
project at a time — concurrent full builds collide on the test database, dev-server port, and build
output.
If `qa-eng`'s tier-2 section, the `code` skill's Verifying section, or `tech-lead.md`'s clauses
change, change the other two with them. Briefs carry scope, not payload — no pasted diffs — and
reports are capped and point at evidence paths, so the tech lead's context stays bounded across
rounds; `defect-report`'s size rules and `code`'s briefing cap move together — change one, change
the other. `code`'s Context discipline, `tech-lead.md`'s reading rule, and `qa-eng`'s context
discipline express one premise — tool output and turn count dominate cost — so change them together.

**Breakdown.** The `breakdown` skill orchestrates in the main thread (it must — it spawns
`code-diver`, and subagents cannot spawn subagents): requirements `R1…Rn` → `business-impact` skill →
code map → tickets per seam → estimate → prioritize → bodies via `ticket-description` → tracker via
`glab`/`gh`. `business-impact` owns the framing block and the per-requirement value tags
(`Core` / `Supporting` / `Speculative`); `breakdown/references/prioritization.md` maps those tags to
its Value score — renaming a tag means changing both.

**Design shots.** The `design-shot` skill renders Claude Design canvas boards (`project/*.dc.html`,
indexed by `project/canvas.json`) locally with the canvas's own `artifact-type/dc-runtime.js` and
screenshots them via `scripts/shot.mjs` (puppeteer-core in `~/.cache/design-shot`, checked by
`scripts/check.sh`). OpenCode has no equivalent of Claude Code's `Artifact` tool, so the canvas is
supplied as a local directory (or a URL whose files the agent can fetch); the skill lists and renders
from there. `ticket-description` owns the ticket's **Design** section table
(`# | Screen · state | Preview | Direction | AC`); design-shot's **Hand off** section emits draft
rows in that shape (first three cells + Source line) — change the columns in both together. GitLab
image upload (`scripts/gitlab-upload.sh`) happens only inside their confirmed tracker writes. If the
Design artifact format changes (runtime path, `canvas.json` keys, `<dc-import>` prop passing),
`shot.mjs` changes with it.

**Findings.** The `defect-report` skill owns the severity-tagged finding-report format (tier table,
finding-line shape, totals line, rules) shared by the `code-reviewer` and `qa-eng` agents; both
invoke it instead of restating the format. If the format changes there, both agents' reports change
with it.

**Security audit.** The `security-audit` skill (adapted from Cloudflare's MIT `security-audit-skill`)
is source-only: it reads the target and never builds, runs, or probes it, so every claim that needs
execution becomes a `needs_validation` record with an owner-run validation plan. It orchestrates in
the main thread because nesting depth is one — every hunter, coverage critic, and verifier is one
`subagent` call, and independence comes from a fresh session rather than the write isolation upstream
relied on. It owns its own `report-schema.json` plus two validators, deliberately separate from
`defect-report`'s tier format; its note body template lives in `VALIDATION-AND-REPORTING.md`, the way
`rca-write` and `design-write` own theirs. It writes the report as a note inside the run directory,
`.notes/security-audit/{YYYY}-{MM}-{DD}[-{TICKET}]-{slug}/{same-stem}-report.md`, beside the machine
artifacts; changing those paths means changing prior-run discovery with them. Like `code`, it follows
the same session-directory rule: one stem, sibling-relative `related`, archive by one `mv`.

**Skill/agent references.** An agent's `permissions` and a skill's directory name must match real
artifacts under `.opencode/`; MCP server names in prose must match keys in the template
`opencode.example.json` (`mcp.servers`), or in a user's own global/project `opencode.json` if they
defined the server there. Renaming a skill, agent, or server means grepping the whole repo for the
old name.

**Skill portability.** Dependencies point one way: an agent may reference a skill, a skill may
reference another skill, and a skill must never reference an agent file or require a specific agent —
a skill is invoked under whatever primary is active. Capability phrases are how a skill names a
collaborator.

**Edits.** A targeted file change goes through the `edit` tool — `write` to create, `edit` to patch,
rewrite the whole file when several regions move: exact match, loud failure, no escaping. A uniform
mechanical change across many files is the exception: `sed -i` or `awk` is cheaper than N `edit`
calls — single-quoted, no `/` or `&` in pattern or replacement (pick another delimiter), then
`git diff --stat` to confirm only intended files moved. **A Python script or `python3` heredoc is
never a way to patch files**: the count assert passes on the wrong anchor, the whole-file rewrite
drops encoding and line endings, and a leftover `patch.py` reaches commits. The rule is written in
`tech-lead.md`, where it stays `edit`-only, **and** in every builder agent's own prompt (`js-eng`,
`react-eng`, `vue-eng`, `golang-eng`, `docker-eng`, `db-admin`, `qa-eng`, `api-documenter`), because
a subagent loads only its own file — an instruction living solely in `tech-lead.md` never reaches it.
Read-only agents (`code-reviewer`, `code-diver`, `security-auditor`, `compliance-auditor`) deny
`edit` outright; that split stays.

**Instruction layering.** Doctrine that must apply to every run lives in this file and in the primary
agent's prompt. Per-task workflow lives in each skill's `SKILL.md`, which is self-contained: a skill
may reference another skill as an optional enrichment, never as a precondition, and never an agent
file or path. Depth — long examples, checklists, per-seam briefs — lives in the owning skill's own
`references/` directory, read when the skill directs it to. Skills are advertised by description only
and their bodies load on demand, so a rule split across two skills can silently go unloaded.

## Commits

Conventional Commits, scoped by area: `feat(agents):`, `refactor(agents):`, `feat(skills):`,
`feat(commands):`, `docs:`.
