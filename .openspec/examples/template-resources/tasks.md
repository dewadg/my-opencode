<!-- openspec-graph: 2 -->

# Tasks

### 1. Template content — js-eng
Depends on: none
Owns: guides/templates/tokens.css, guides/templates/icons.svg, guides/templates/board-dashboard.html, guides/templates/board-mobile-flow.html, guides/templates/components.html, guides/templates/readme.md, guides/templates/canvas.json
Gate: build

- [x] 1.1 `tokens.css`: spacing 4/8/12/16/24/32/48/64, one type scale, one accent plus neutrals, three radii, two shadows, a visible focus ring, one font family with a system fallback. Check: `grep -c -- '--space' guides/templates/tokens.css` is at least 8 and every variable group is present. Covers: TC-838
- [x] 1.2 `board-dashboard.html` at 1440×900: sidebar, header, three KPI cards, a data table; links `tokens.css` relatively; `?state=` renders default, empty, loading and error; real copy. Check: `grep -n 'tokens.css'` shows one relative link and `grep -c 'state'` shows all four. Covers: TC-830, TC-835, TC-838
- [x] 1.3 `board-mobile-flow.html` at 390×844: status area, list, primary action, bottom tabs, same four states; `components.html`: every control and empty state, labelled with token names. Check: both link `tokens.css` relatively and carry one `<h1>`. Covers: TC-835, TC-838
- [x] 1.4 `icons.svg` (inline `<symbol>` sprite, at most 24 icons) and `readme.md` (what the design covers, decisions, open questions, pages and boards). Check: `grep -c '<symbol' guides/templates/icons.svg` is between 12 and 24; the readme has the four headings. Covers: TC-834
- [x] 1.5 `canvas.json`: pages `desktop` and `mobile`, boards on the three HTML files, two notes, `launch` on `desktop`. Check: the index lists three boards and two pages, and every board `file` is one of the templates. Covers: TC-837
- [x] 1.6 Keep the set inside the rules: every reference relative, no external host except Google Fonts, each file at most 128 KiB, the set at most 512 KiB. Check: `du -sk guides/templates` is at most 512; the external-host grep prints nothing. Covers: TC-836

### 2. Registry, resources and tests — golang-eng
Depends on: 1
Owns: guides/templates.go, guides/templates_test.go, internal/mcpserver/handler_templates.go, internal/mcpserver/handler_templates_test.go, internal/mcpserver/server.go
Gate: build

- [x] 2.1 `guides/templates.go`: `//go:embed templates/*`, a `Template` struct (`URI`, `Name`, `Description`, `MIME`, `Path`) and `TemplateResources` listing all seven in a fixed order. Check: `go build ./guides/` exits 0 and `go test -run '^$' ./guides/...` compiles the package and its tests. Covers: TC-833
- [x] 2.2 Register every entry on the MCP server beside `guide://html`, and implement the read handler returning the embedded bytes with the declared MIME type. Check: `go build ./internal/mcpserver/` and `go test -run '^$' ./internal/mcpserver/...` exit 0; the listing itself is exercised at verify. Covers: TC-833, TC-834
- [x] 2.3 `guides/templates_test.go`: HTML rules (doctype, charset, viewport, `lang`, one `h1`, `alt`), relative references resolve inside the set, caps, external hosts, `tokens.css` linked by both boards, and the worked index through `validateCanvasIndex`. Check: `go test -run '^$' -race ./guides/...` compiles the tests; the verify node runs them. Covers: TC-834, TC-835, TC-836, TC-837, TC-838
- [x] 2.4 `internal/mcpserver` tests: listing includes the seven templates with description and MIME; reading returns non-empty bytes; the guide resources are unchanged; the tool count stays 12. Check: `go test -run '^$' -race ./internal/mcpserver/...` compiles the tests; the verify node runs them. Covers: TC-833, TC-834

### 3. Guide list and docs — api-documenter
Depends on: 2
Owns: guides/design.md, docs/mcp.md
Gate: build

- [x] 3.1 Add the template list to `guides/design.md`: one line per URI on when to use it; the checklist names `template://readme.md`. Check: every URI in `TemplateResources` appears in the guide. Covers: TC-839
- [x] 3.2 `docs/mcp.md`: a resources section listing both guides and the seven templates, with what each is for. Check: each URI is named once and matches the registry. Covers: TC-833, TC-839

### 4. Verify — golang-eng
Depends on: 1, 2, 3
Owns: none
Resource: db:test
Gate: verify
Evidence: evidence/verify-go.md

- [x] 4.1 Run the project's verification commands against the tree: `make migrate`, `make test`, `make vet`, `make lint`, `make web-test`, `make web-build`. Check: each exit code reported; mechanical failures fixed in this session, non-mechanical ones held for the batched fix round. Covers: all

### 5. Review — code-reviewer
Depends on: 4
Owns: none
Gate: review
Evidence: evidence/review-round-1.md

- [x] 5.1 Static read of the diff: registry ordering, MIME correctness, the embed path, the tests' independence from the working directory, the observability section, and the guide/registry comparison. Check: findings by file and severity; no `make`, `go test` or lint command run. Covers: all

### 6. Render and publish — qa-eng
Depends on: 4
Owns: none
Gate: qa
Evidence: evidence/qa-report.md

- [x] 6.1 On local dev, write the seven templates into a fresh project and publish; open the canvas and confirm each board at its intended size: dashboard 1440×900, mobile 390×844, components at 1440×900, no horizontal overflow, notes placed, `launch` on `desktop`. Check: screenshots at both sizes and a console log with no error. Covers: TC-835, TC-837
- [x] 6.2 Switch `?state=` through default, empty, loading and error on both boards and confirm each is distinct and labelled. Check: four screenshots per board, or a stated gap. Covers: TC-830
- [x] 6.3 Read every published resource through the MCP server's `resources/list` and `resources/read`; confirm the seven templates plus both guides, with the declared description and MIME. Check: scenario pass/fail, one line per URI. Covers: TC-833, TC-834
