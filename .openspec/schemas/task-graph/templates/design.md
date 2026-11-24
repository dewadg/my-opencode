# Design

## Context

<!-- Current state, problem and constraints that shape the approach. Link the ticket for motivation - don't restate it -->

## Goals / Non-Goals

**Goals:**
<!-- What this design aims to achieve -->

**Non-Goals:**
<!-- What is explicitly out of scope -->

## Flow

<!-- Mermaid diagram when a flow, schema or state changes. Only flowchart, sequenceDiagram,
     stateDiagram-v2 or erDiagram; quote labels with punctuation; no `;`, `#` or `<...>` in
     sequence messages; never use `end` as an id. Delete this section when nothing flows. -->

## Decisions

<!-- Key design decisions with rationale and alternatives considered -->

## Shared contracts

<!-- Types, interfaces, migrations and API shapes this change adds or changes, and the packages
     each part touches. List every new or renamed foreign key column and every column, jsonb key or
     payload field holding another row's id, named by the project's foreign-key rule. -->

## Test scenarios

<!-- Requirement cites the spec requirement the row proves. Proof is one of unit | integration |
     api | ui | device | manual: unit and integration route to verify; api, ui, device and manual
     route to QA (ui is web, device is mobile). -->

| ID | Scenario | Given | When | Then | Requirement | Proof |
|---|---|---|---|---|---|---|
| TC-<!-- id --> | <!-- scenario --> | <!-- given --> | <!-- when --> | <!-- then --> | <!-- requirement name --> | <!-- unit --> |

## Verification

<!-- Per project: the migrate, build, typecheck, lint and suite commands the verify stage runs, or
     `none`. State which behavior the suite proves and which only QA proves. Examples: Go API,
     `make migrate`, `go build`, `go vet`, `golangci-lint`, `go test`; web, nothing, `tsc --noEmit`,
     eslint, `vitest run`, build; mobile, nothing, `flutter analyze`, test, build. -->

| Project | Migrate | Build | Typecheck | Lint | Suite |
|---|---|---|---|---|---|
| <!-- project --> | <!-- command or none --> | <!-- command --> | <!-- command or none --> | <!-- command or none --> | <!-- command or none --> |

## Observability

<!-- Log events with their levels and fields, trace spans, metric names and labels, and what must
     never be logged. `none` needs the reason. Review checks this section against the code. -->

## Risks / Trade-offs

<!-- Known risks and trade-offs -->

## Migration Plan

<!-- Steps to deploy, rollback strategy. Delete this section when there is no migration. -->

## Docs to update

<!-- Every doc the change must update, at the project's paths. Derive the list from the project's
     own conventions; never assume a docs layout. Write `none` when nothing changes. -->

## Open Questions

<!-- Deferrable unknowns only. Delete the section when there are none. -->
