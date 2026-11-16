---
description: Scope a new feature or change across one or more projects before building it — break the analysis into delegated tasks and compile the results into a note.
agent: coordinator
---

# Analyze

Break a complex analysis into tasks delegated across subagents. Output is a note.

**Input**: `$ARGUMENTS` — the description of the feature or change to be developed. May be a ticket
number, and may name the related projects.

## Steps

1. **If `$ARGUMENTS` is empty, ask what the user wants**

   Use the **`question` tool** with an open-ended prompt (the free-form answer is always available):
   > "What do you want to analyze?"

   **IMPORTANT**: Do NOT proceed without understanding what the user wants to build.

2. **Create the analysis note**

   **IMPORTANT**: Load the **`note-search` skill** first (`skill` tool, id `note-search`) — a past
   note may already cover this area.

   Then load the **`design-write` skill** (`skill` tool, id `design-write`) to create the design-doc
   note. design-write owns and enforces the design-doc note format (section template, conditional
   sections, cross-cutting checks) and builds on the note-write skill's frontmatter and filename
   conventions. This creates the note skeleton — every core section header in place — that will hold
   the analysis results.

   If a ticket number is given and an issue-tracker MCP is connected (Jira, Linear, GitHub issues),
   fetch the ticket for context.

3. **List impacted projects**

   Take them from the description, or determine them yourself. The goal is to know each impacted
   project's tech stack (Go, React, JavaScript, infra), because that is what selects the subagent.

   Read the `AGENTS.md` (or legacy `CLAUDE.md`) of every project in scope — their conventions and
   constraints belong in the analysis.

4. **Analyze**

   Track the work as a short checklist in your reply or in the note — one line per project and task.

   Run the orchestration in the main thread: **OpenCode subagents cannot spawn other subagents**
   (nesting depth is one), so the main thread launches every specialist directly. Loop through tasks
   in dependency order:

   a. **Scale fan-out to scope — tokens are the budget.** Specialized subagents are expensive, so
      fan out only when the scope is genuinely broad: multiple projects, a large surface, or several
      distinct concerns that benefit from parallel investigation. For a single project, a small
      surface, or a linear analysis you can hold in one pass, investigate **inline** with the
      `read`/`grep`/`glob`/`shell` tools instead of spawning an agent. Prefer the smallest unit of
      work that still covers the scope; one well-scoped agent beats a roster for a narrow task.

   b. **Spawn with the `subagent` tool.** Supply the agent ID, a short description, and a complete
      prompt. Subagents always run in the background; their report arrives when they go idle.
      Continue until all tasks are done. Each subagent gives its findings and its opinion; the main
      thread compiles them into the note.

   c. **If a finding requires user input** (unclear context), use the **`question` tool** to
      clarify, then continue.

## Explaining code flow — use the call-graph skill

Whenever you explain how code flows — how a feature works today, a request's path, a call chain,
a struct's methods and what they call — render it with the **`call-graph` skill** (`skill` tool, id
`call-graph`), not prose. Emit its indented arrow tree (one node per call hop, each carrying
`file:line`). This applies whether you trace inline or hand off to `code-diver`:

- The note's **Current state** section anchors on these trees/tables.
- The note's **Flow diagram** uses the call-graph tree for code call paths. (A mermaid
  sequence/flowchart is still right for cross-service request/data flow, deploy order, or state
  transitions — pick by what is being shown, but a code call path is always the tree.)
- Any chat explanation of flow given back to the user is the tree too, never a paragraph or an
  arrow-crammed one-liner.

Do not paraphrase a traced path as prose when the call-graph tree can carry it. The tree is the
format for code flow.

## Choosing subagents

Do not rely on a fixed roster — pick from the agent definitions available in `.opencode/agents/` and
`~/.config/opencode/agents/`, so newly added agents are used without this command changing. Enumerate
them with:

```bash
ls .opencode/agents ~/.config/opencode/agents 2>/dev/null
```

**Default for code reading: spawn `code-diver`.** It returns call-graph trees and file:line tables
that anchor the **Current state** and **Impact map** sections — trace how a feature works today, map
a struct's methods, find call sites, scope touched files. Routing rules:

- **code-diver flag.** On a stack-semantic gap (concurrency, framework lifecycle, ORM behavior) it
  flags `→ needs <specialist>`. Spawn that specialist for just that finding.
- **Smart-skip.** Request already semantic (thread-safety, reactivity, ORM/query, framework
  internals) → skip code-diver, spawn the specialist directly.
- **Pass the tree.** Escalating from a flag → hand code-diver's tree/table into the specialist
  spawn as context; it does only the semantic step, no re-trace.

Route the rest by capability: schema and migrations to the database agent, cross-cutting design to
the architecture review agent, test strategy to the QA agent. Reach for a language engineering agent
only when the analysis needs stack-specific depth code-diver cannot provide (concurrency-safety
review, framework internals). If several fit, pick the most specific. If none fits, use the built-in
`general` subagent.

## Output

After all tasks are done, report back to the user in chat:
- The TLDR (what changes, in which projects, rough size and risk)
- The recommended approach and why it beat the alternatives
- Any open question that blocks implementation
- The path to the analysis note

## Guardrails

- If context is critically unclear, ask the user — but prefer reasonable decisions to keep momentum.
- Verify each artifact file exists after writing, before moving to the next.
