---
tags: [code]
projects: [mcp-server]
created: 2026-11-21
updated: 2026-11-21
related: [tasks.md, design.md]
description: template-resources — implementation journal
---
# Template resources

Ship the shared HTML board templates and their MCP resources, so the authoring guides can render
them. · status: shipped

## Execution log

| Node | Agent | Start | End | Minutes |
|---|---|---|---|---|
| 1 | js-eng | 2026-11-21 09:12:04 WIB (+0700) | 2026-11-21 09:58:37 WIB (+0700) | 47 |
| 2 | golang-eng | 2026-11-21 10:04:19 WIB (+0700) | 2026-11-21 11:26:02 WIB (+0700) | 82 |
| 3 | api-documenter | 2026-11-21 11:30:41 WIB (+0700) | 2026-11-21 12:07:26 WIB (+0700) | 37 |
| 4 | golang-eng | 2026-11-21 12:10:33 WIB (+0700) | 2026-11-21 12:41:58 WIB (+0700) | 31 |
| 5 | code-reviewer | 2026-11-21 12:44:12 WIB (+0700) | 2026-11-21 13:22:47 WIB (+0700) | 39 |
| 6 | qa-eng | 2026-11-21 12:44:12 WIB (+0700) | 2026-11-21 13:40:05 WIB (+0700) | 56 |
| Total | — | 2026-11-21 09:12:04 WIB (+0700) | 2026-11-21 13:40:05 WIB (+0700) | 292 |

## Decisions

| # | Decision | Rejected | Why | Effect | Source |
|---|---|---|---|---|---|
| D1 | Ship every icon as a `<symbol>` in `icons.svg`, referenced from the templates. | One SVG file per icon — it multiplies relative references and makes the 512 KiB set cap harder to reason about. | A single sprite keeps the reference count flat and the cap measurable in one place. | guides/templates/icons.svg | node 1 |
| D2 | List all seven templates in `TemplateResources` in a fixed order. | `fs.Glob` over the embedded directory — discovery order would not be stable, and a missing file would silently change the resource list. | The resource list must be deterministic and fail loudly when a template goes missing. | guides/templates.go | node 2 |
| D3 | Declare MIME per template in the registry: `text/html` for pages, `image/svg+xml` for the sprite, `application/json` for `canvas.json`. | Extension sniffing in the read handler — the MCP resource contract already declares the MIME type, and two sources would drift. | One declaration per template removes the second source of truth. | guides/templates.go, internal/mcpserver/handler_templates.go | node 2 |
| D4 | Document templates by URI only: one line per URI in `guides/design.md`, a resources list in `docs/mcp.md`. | A usage example per template — it duplicates content the templates already carry and would drift from the registry. | The templates are the documentation of themselves; the docs only need to point at them. | docs/mcp.md | node 3 |

## Outcome

Shipped the six-template set, the registry and resource surface, and the guide list. All six plan
nodes are done, verify and review clean. Deferred: a template preview page, which needs a runtime
the guides do not have yet.
