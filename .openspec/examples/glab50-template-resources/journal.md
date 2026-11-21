<!-- openspec-journal: 1 -->
# Journal — glab50-template-resources

## D1 · node 1 · One inline SVG sprite instead of per-icon files
Date: 2026-11-21 · Owner: js-eng
Decided: ship every icon as a `<symbol>` in `icons.svg`, referenced from the templates.
Rejected: one SVG file per icon — it multiplies relative references and makes the 512 KiB set cap harder to reason about.
Effect: guides/templates/icons.svg.

## D2 · node 2 · Fixed registry slice instead of walking the embed FS
Date: 2026-11-21 · Owner: golang-eng
Decided: list all seven templates in `TemplateResources` in a fixed order.
Rejected: `fs.Glob` over the embedded directory — discovery order would not be stable, and a missing file would silently change the resource list.
Effect: guides/templates.go.

## D3 · node 2 · Declare MIME per template instead of sniffing it
Date: 2026-11-21 · Owner: golang-eng
Decided: `text/html` for pages, `image/svg+xml` for the sprite, `application/json` for `canvas.json`, declared in the registry.
Rejected: extension sniffing in the read handler — the MCP resource contract already declares the MIME type, and two sources would drift.
Effect: guides/templates.go, internal/mcpserver/handler_templates.go.

## D4 · node 3 · Docs list templates by URI only
Date: 2026-11-21 · Owner: api-documenter
Decided: one line per URI in `guides/design.md` and a resources list in `docs/mcp.md`.
Rejected: a usage example per template — it duplicates content the templates already carry and would drift from the registry.
Effect: docs/mcp.md.
