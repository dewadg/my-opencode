# my-opencode

My collection of OpenCode (V2) subagents, skills, commands, and MCP server config.

Nothing here is an application — it's markdown that OpenCode itself consumes. The `.opencode/`
tree is live project config, so this repo works as-is when opened in OpenCode. Copy what you want
into `~/.config/opencode/` to use it everywhere.

## Install

```bash
mkdir -p ~/.config/opencode/{agents,commands,skills}
cp -R .opencode/agents/*   ~/.config/opencode/agents/
cp -R .opencode/commands/* ~/.config/opencode/commands/
cp -R .opencode/skills/*   ~/.config/opencode/skills/
```

`opencode.example.json` is a sample MCP config; merge its `mcp.servers` block into a project's
`opencode.json` or the global `~/.config/opencode/opencode.json`, then export the env vars it
references: `CONTEXT7_API_KEY`, `GITLAB_PAT`, `FIGMA_API_KEY`. Secrets are `{env:VAR}` references,
never literals. The repo ships no project `opencode.json`, so MCP servers stay opt-in: nothing is
enabled unless the global config or a project adds `mcp.servers`.

## Agents

Subagents OpenCode delegates to via the `subagent` tool. Each file is frontmatter (description,
mode, permissions, optional model/color) plus a system prompt. The filename is the agent ID.

| Agent | Use for |
|---|---|
| `api-documenter` | API docs, OpenAPI specs, doc portals, generated code examples |
| `architect-reviewer` | System design decisions, architectural patterns, technology choices |
| `code-reviewer` | Code quality, security vulnerabilities, best practices |
| `code-diver` | Read-only codebase exploration — trace call flow (call-graph tree), locate symbols (file:line table); spawned by the `analyze` and `investigate` commands for code reading, escalates to a specialist on a stack-semantic gap |
| `compliance-auditor` | GDPR / CCPA obligations, data practices, privacy gaps |
| `db-admin` | Query performance, HA architecture, disaster recovery |
| `docker-eng` | Designing, building, running, optimizing containers |
| `golang-eng` | Go services, concurrency, cloud-native systems |
| `js-eng` | Modern JavaScript — browser, Node.js, Bun, full-stack |
| `qa-eng` | Versatile QA — BE endpoints via hurl (hurl-write / hurl-run), FE/UI via chrome-devtools MCP, plus test strategy, automation, quality metrics |
| `react-eng` | React 18+ performance, state management, component architecture |
| `security-auditor` | Security audits, vulnerability analysis, compliance gaps |
| `vue-eng` | Vue 3 Composition API, reactivity, Nuxt 3 |

Claude Code's `tools:`, `skills:`, `mcpServers:`, and `model: sonnet|opus|haiku` fields have no V2
equivalent. Tool access is expressed as `permissions` (for example, `code-diver` denies `edit`);
models are inherited from the parent session unless you set `model: provider/model#variant` on an
agent or override it in the global `~/.config/opencode/opencode.json`; MCP servers are configured
globally, not per agent.

Current model mapping — Flash only, effort as the dial:

| Tier | Model | Agents |
|---|---|---|
| Main / important | `deepseek/deepseek-flash#high` | `architect-reviewer`, `code-reviewer`, `compliance-auditor`, `db-admin`, `security-auditor` |
| Common | `deepseek/deepseek-flash#low` | `api-documenter`, `code-diver`, `docker-eng`, `golang-eng`, `js-eng`, `qa-eng`, `react-eng`, `vue-eng` |

The default primary and the per-agent pins live in the global `~/.config/opencode/opencode.json`;
the repo ships no project config.

These agents are from [awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) and modified for my personal use.

## Commands

User-invoked slash commands. Run as `/analyze` or `/investigate`.

| Command | Use for | Not for |
|---|---|---|
| `analyze` | Scoping a feature or change across projects before building it | Diagnosing a bug (`/investigate`); writing the code (`code` skill) |
| `investigate` | Finding the root cause of something broken | Scoping a new feature (`/analyze`); writing the fix (`code` skill) |

Both delegate code reading to `code-diver` and, when explaining code flow, render it with the
`call-graph` skill (indented arrow tree, one node per call hop) rather than prose. Commands run in
the current session unless their frontmatter sets `subagent: true`; both keep orchestration in
the main thread because OpenCode subagents cannot spawn other subagents.

## Skills

### Orchestrators

`analyze` and `investigate` (commands, above) and `code` (skill, below) share one shape: plan the
work as a checklist, delegate every project-level unit to a subagent via the `subagent` tool,
compile the result into a note. They route to agents by capability rather than by name, so adding
an agent to `.opencode/agents/` makes it usable without touching the command or skill. For code
reading and tracing they spawn `code-diver` by default, escalating to a language specialist only
when `code-diver` flags a stack-semantic gap
(`→ needs <specialist>`) — `smart-skip` sends semantic-flavored requests straight to the
specialist, `pass-tree` hands `code-diver`'s output to it so the specialist does only the semantic
step.

### Code reading

| Skill | Use for |
|---|---|
| `call-graph` | Render how code flows as an indented call-graph tree — one node per call hop with `file:line`. Fires on explicit invoke or "explain how X works" |

| Skill | Use for | Not for |
|---|---|---|
| `code` | Writing or changing code across one or more repos | Diagnosing an unexplained bug (`/investigate`); scoping (`/analyze`) |

### Notes

| Skill | Use for |
|---|---|
| `note-write` | Persist knowledge as a dated, tagged markdown note |
| `note-search` | Retrieve past notes by project, topic, ticket, or date |

Notes live in `.notes/` (or wherever the project's `AGENTS.md` says), named
`{YYYY}-{MM}-{DD}[-{TICKET}]-slug.md`, with `tags` / `projects` / `created` / `updated` / `related` /
`description` frontmatter. The orchestrator skills read and write this format, so the two note skills
must stay in sync.

### API & testing

| Skill | Use for |
|---|---|
| `golang-unit-test` | Go table-driven tests with before/after hooks |
| `golang-project-structure` | Scaffold, extend, audit, or migrate a Go service onto the fx + cobra + caarlos0/env + goose + OTel + slog layout; ships a runnable scaffold and `scripts/audit.sh` |
| `hurl-write` | Author `.hurl` HTTP test scenario files — chained requests, captures, asserts, variables/secrets |
| `hurl-run` | Execute `.hurl` files or one-off stdin requests with hurl; exit-code verdict, debug + report flags |
| `test-scenario` | Given/When/Then test scenarios as a markdown table — happy, alternate, edge, error paths |
| `defect-report` | Shared severity-tagged findings format (tier table, finding line, totals) — `code-reviewer` and `qa-eng` both emit through it |
| `hoppscotch-import` | Generate a Hoppscotch collection JSON from an OpenAPI spec, route code, Postman, HAR, or plain list |

### Writing

| Skill | Use for |
|---|---|
| `ticket-description` | Ticket/issue description bodies in a fixed section structure, any tracker |
| `pr-description` | PR/MR description bodies from the branch diff, emoji-sectioned template |

Both produce description text only — they never create the ticket or open the PR.

### Release & ops

| Skill | Use for |
|---|---|
| `glab-release` | Cut a GitLab release: pick prev tag, build a conventional-commit changelog, tag |
| `gitlab-security-scan` | Add Free-tier security scans to GitLab CI: per-package Semgrep SAST, osv-scanner dependency scan, JUnit scan report in the Tests tab, Renovate MRs |
| `design-shot` | Screenshot Claude Design canvas screens (whole or cropped, any state). OpenCode has no `Artifact` tool, so supply the canvas as a local export or fetchable files |
| `breakdown` | Turn a requirements block into prioritized, human-sized tickets (orchestrates the planning skills above) |
| `business-impact` | Frame the business value of a feature, requirement, or backlog item |
| `telegram-notify` | Ping me on Telegram when a long-running job or watch condition resolves |

### Security

| Skill | Use for | Not for |
|---|---|---|
| `security-audit` | Source-only security audit — recon, coverage ledger, hunting waves, independent verification, and a report note; also threat models and focused vulnerability review | Adding CI scans (`gitlab-security-scan`); reviewing a diff (`code-reviewer`); compliance/controls assessment (`compliance-auditor`); writing the fix (`code` skill); formatting findings as a table (`defect-report`) |

Adapted from Cloudflare's [`security-audit-skill`](https://github.com/cloudflare/security-audit-skill)
(MIT, [license](.opencode/skills/security-audit/LICENSE)); rewritten source-only for OpenCode, with
the upstream `report-schema.json` and validators kept verbatim.

## MCP servers

`opencode.example.json` wires up:

- **context7** — up-to-date library/framework docs
- **gitlab** — issues, merge requests, pipelines
- **goland** — IDE integration (local, `127.0.0.1:64422`); main conversation only
- **chrome-devtools** — browser driving for UI testing
- **figma** — Figma design metadata (figma-context-mcp) for design-vs-implementation checks
- **mobile-device** — Android/iOS app driving for mobile QA

MCP servers are global to a project; OpenCode has no per-agent `mcpServers` list. Restrict access
with `permissions` using the normalized `<server>_<tool>` action when an agent should not reach a
server. Tool names here must match the keys under `mcp.servers`.

## License

MIT — see [LICENSE](LICENSE).
