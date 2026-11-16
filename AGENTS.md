# AGENTS.md

Guidance for OpenCode (V2) when working in this repository.

## What this repository is

A personal collection of OpenCode **configuration artifacts** — agent definitions, skills, slash
commands, and an MCP server config example. It is not an application: there is nothing to build,
lint, or test, and no package manager. The "code" here is markdown that OpenCode itself consumes.

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
  subagent except `coordinator.md`, the primary the orchestrators run in.
- `.opencode/commands/*.md` — one user-invoked slash command per file. Frontmatter (`description`,
  optional `agent`, `model`, `subagent`) + prompt body. `$ARGUMENTS` expands to the text after the
  command name, `$1`/`$2` to positional arguments, and `` !`shell` `` injects command output. The
  body reads as instructions to the current agent.
- `.opencode/skills/<name>/SKILL.md` — one skill per directory. The directory name is the skill ID.
  Supporting scripts and references live alongside (e.g.
  `.opencode/skills/telegram-notify/scripts/send_telegram.sh`).
- `opencode.json` — project config: schema, the session default `model`, and the `agents` block that
  pins every subagent's model (MCP servers stay opt-in).
- `opencode.example.json` — example `mcp.servers` block (`context7`, `gitlab`, `goland`,
  `chrome-devtools`, `figma`, `mobile-device`); merge it into a project's or the global
  `~/.config/opencode/opencode.json`. Secrets are `{env:VAR}` references, never literals. `goland`
  is main-conversation-only — no agent is restricted to it.

## Agent frontmatter

```yaml
---
description: >                 # third person, trigger-oriented; this is what routes work here
  Use this agent when ...
mode: subagent                 # subagent | primary | all
model: provider/model#variant  # leave out — subagent models are pinned in opencode.json's agents block
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

Subagent models are pinned centrally, not per file: every agent here omits `model` from its frontmatter
and takes one from the `agents` block in `opencode.json` (`#low` for explorers and language engineers,
`#high` for reviewers and auditors). Switch every subagent's model by editing that one file —
OpenCode reloads config on save. When copying agents into `~/.config/opencode/agents/`, merge the same
`agents` block into `~/.config/opencode/opencode.json`, or the copied agents inherit the parent session
model instead.

Body convention across agents: a role paragraph, a "When invoked" list, then phased workflow
sections. Follow the shape of an existing agent when adding a new one.

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
`spec-write` vs `code`, which disambiguate each other in their own descriptions). The same applies to
command `description` frontmatter — `analyze` and `investigate` exclude each other and `code`.

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

**Specs.** The `spec-write` skill (driven explicitly by the `/spec` command, auto-fired by its own
description) writes single-file specs at `specs/{slug}.md` (live); the `spec-archive` skill moves
shipped or abandoned ones to `specs/archives/{YYYY-MM-DD}-{slug}.md`. Each spec carries frontmatter
`status` (`draft` | `in-progress` | `shipped` | `abandoned`), `created`, `updated`, `tags`,
`projects`, `ticket`, `related`, `description` — `tags`/`projects`/`created`/`updated`/`related`/
`description` match the note schema deliberately. `spec-write`'s **Search** job greps these exact
fields and awk-extracts the frontmatter block only, so look-back never reads a spec body.
`spec-archive` finalizes only `status` and `updated` — the rest of the schema must stay intact so
Search still routes. If the spec frontmatter schema changes, those grep recipes and the template's
frontmatter block must change with it.

**Orchestrators.** The `analyze` and `investigate` commands and the `code` skill share one
architecture: plan the work as a checklist, delegate every project-level unit of work to a subagent
via the `subagent` tool, compile into a note (`investigate` compiles into an RCA note whose format
`rca-write` owns; `analyze` compiles into a design-doc note whose format `design-write` owns; both
still depend on `note-write`/`note-search` for file mechanics). They deliberately do **not** hardcode
an agent roster — they route by capability ("a service backend → the engineering agent for that
language") so that adding an agent to `.opencode/agents/` makes it usable without editing any command
or skill. Preserve that indirection; do not introduce agent-name lists into commands or skills.
All three run under `coordinator` (set by `agent:` frontmatter on the commands, and by
`default_agent` in `opencode.json`), which is why the delegation contract below binds them.
`analyze` and `investigate` also share a code-flow contract: when they explain how code flows, they
render it with the `call-graph` skill (indented arrow tree), never prose — so changes to the
`call-graph` output format affect what those commands emit.

**Coordination.** `coordinator.md` is the primary that does not build: it plans, briefs, and verifies
returned reports. Its role is prompt-enforced, not permission-enforced — it keeps `edit`, `shell`,
and the browser MCP tools, so nothing structurally stops it from taking a subagent's work; what
binds it is the evidence-attribution rule (every claim cites a subagent `sessionID`) and the
`code` skill's two-tier verification contract. Verification splits in two: the implementing subagent
self-checks with a narrow build/lint/unit run for the packages it touched, then the QA agent runs the
full project build, full suite, and browser/mobile flows **once**, per project, after all
implementation is in. One full run per project at a time — concurrent full builds collide on the test
database, dev-server port, and build output. If `qa-eng`'s tier-2 section, the `code` skill's
Verifying section, or `coordinator.md`'s clauses change, change the other two with them.

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

**Skill/agent references.** An agent's `permissions` and a skill's directory name must match real
artifacts under `.opencode/`; MCP server names in prose must match keys in `opencode.json`
(`opencode.example.json` is the template). Renaming a skill, agent, or server means grepping the
whole repo for the old name.

## Commits

Conventional Commits, scoped by area: `feat(agents):`, `refactor(agents):`, `feat(skills):`,
`feat(commands):`, `docs:`.
