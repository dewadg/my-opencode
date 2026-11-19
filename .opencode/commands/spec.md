---
description: >-
  Spec a change into a build-ready specs/{slug}.md before code. Not for implementing an
  already-decided change (use the `code` skill) or open-ended impact analysis with no committed plan
  (use `/analyze`).
agent: tech-lead
---

# /spec

Turn a change idea into one build-ready file: `specs/{slug}.md`. This command orchestrates the
process — ground, research, decide — then delegates the **file format** to the **`spec-write`
skill** (`skill` tool, id `spec-write`). The skill owns the template, frontmatter schema, lean
principles, and the file write; this command owns the investigator fan-out and the decisions that
fill the spec.

Route on intent: `archive <slug>` or `/spec archive` → hand to the **`spec-archive` skill**;
`search <query>` or "find specs about X" → hand to the **`spec-write` skill's Search** job;
everything else is spec creation below.

**Input**: `$ARGUMENTS` — description of what to build or change. May be a ticket number, feature
name, or prose.

## Steps

1. **If `$ARGUMENTS` is empty, ask**

   Use the **`question` tool** with an open-ended prompt (the free-form answer is always available):
   > "What change do you want to spec? Describe what you want to build or fix."

   Do NOT proceed without understanding intent. From the answer, derive a kebab-case slug (e.g. "add
   user authentication" becomes `add-user-auth`).

2. **Ground yourself**

   - Load the **`note-search` skill** — a prior note may already hold analysis or decisions. Don't
     re-derive what's written down.
   - Search prior specs by topic or project — hand to the **`spec-write` skill's Search** job (reads
     frontmatter only, cheap). A related or superseded spec may already hold decisions worth reusing.
   - Read the `AGENTS.md` (or legacy `CLAUDE.md`) of every project the change touches — conventions
     override your defaults, and the tech stack picks the subagents later.
   - Slug already exists in `specs/`? Read it, ask the user continue vs overwrite before doing
     anything.

3. **Research — inline by default, fan out investigators when it pays**

   Default to a single inline pass: read key source, existing patterns, and the contracts the change
   touches. Keeps token cost low for the common case.

   Fan out parallel investigators only when the change is genuinely large or spans multiple projects —
   touches many files, crosses package/repo boundaries, or has unknowns several focused reads resolve
   faster in parallel. Spawn one read-only explorer agent — the one whose description is read-only
   exploration — per project/subsystem with the `subagent` tool; each returns a call-graph tree or
   file:line table of how the touched code works today, then synthesize. Subagents cannot spawn other
   subagents, so the main thread launches every investigator directly. Routing rules for an explorer
   finding: on a `→ needs <specialist>` flag (stack-semantic gap), spawn that specialist for just that
   finding and **pass the explorer's tree** into the spawn as context (semantic step only, no
   re-trace); **smart-skip** the explorer entirely for areas already known to need semantic depth
   (thread-safety, reactivity, ORM/query, framework internals) — spawn the specialist direct. See
   "Choosing subagents" for the full routing. If you fan out, say so
   briefly so the user knows the cost.

   Draft the **Testing** section inline (pre-test requirements + scenario table, per the spec-write
   template). No subagent needed.

4. **Commit to decisions**

   A spec that lists options without picking is analysis, not a plan. For every meaningful fork, state
   choice, reason, and alternatives rejected (the spec-write Decisions template). A fork genuinely
   unresolved goes to Open Questions — keep that list short.

5. **Format the file — load the spec-write skill**

   Call the **`spec-write` skill** (`skill` tool, id `spec-write`) to write `specs/{slug}.md` from its
   template. Hand the skill your research, decisions, and the wave plan; it owns the frontmatter
   schema, section order, lean principles, and the single file write. `status` starts `draft`;
   `created` and `updated` set to today.

6. **Hand off**

   Summarize in chat: spec path, decisions made (one line each), wave count, any Open Questions
   needing the user. Tell the `code` skill to honor the spec's checkbox contract: check off each task
   `[x]` as its subagent completes it. End with: "Review the spec, then load the `code` skill to
   implement."

## Explaining code flow — use the call-graph skill

While researching and writing the spec, whenever you explain how code flows — how the touched code
works today, a request path, the call chain behind the change — render it with the **`call-graph`
skill** (`skill` tool, id `call-graph`): indented arrow tree, one node per call hop, each carrying
`file:line`. Not prose, not an arrow-crammed one-liner.

- The research step (how it works today) anchors on these trees, whether you trace inline or receive
  them from an explorer agent.
- The spec's **Flow** section uses the call-graph tree for code call paths; a mermaid diagram stays
  right for cross-service request/data flow, component interaction, or state transitions. Pick by
  what is shown — a code call path is always the tree.

Do not paraphrase a traced path as prose when the call-graph tree can carry it.

## Choosing subagents

Do not rely on a fixed roster — pick from the agent definitions available in `.opencode/agents/` and
`~/.config/opencode/agents/`, so newly added agents are used without this command changing. Enumerate
them with:

```bash
ls .opencode/agents ~/.config/opencode/agents 2>/dev/null
```

**Research investigators.** Default for code reading: the read-only explorer agent — the one whose
description is read-only exploration — returns the call-graph trees and file:line tables that anchor
the "how it works today" research. Routing rules:

- **Explorer flag.** On a stack-semantic gap (concurrency, framework lifecycle, ORM behavior) it
  flags `→ needs <specialist>`. Spawn that specialist for just that finding.
- **Smart-skip.** Request already semantic (thread-safety, reactivity, ORM/query, framework internals)
  → skip the explorer, spawn the specialist directly.
- **Pass the tree.** Escalating from a flag → hand the explorer's tree/table into the specialist
  spawn as context; it does only the semantic step, no re-trace.

**Todo-wave owners.** When filling the spec's dependency-wave todo, name the single agent that owns
each task by capability, not by hardcoded roster:

- Service / backend: the engineering agent for that language
- Web frontend: the framework agent for that stack (Vue, React, …)
- Test strategy, E2E, browser verification: the QA capability
- Review of any produced diff: the review capability
- Containers, compose, images: the container capability
- Schema, migrations, query performance: the database capability
- Cross-cutting design / architecture: the architecture review capability
- None fits: the built-in general subagent

One subagent per project per task — two agents editing the same project concurrently conflict.
Independent projects (or independent files in one project) can share a wave.

## Output

After the spec is written, report back to the user in chat:

- The spec path and the one-line summary
- The decisions made (one line each)
- Wave count and any cross-wave ordering that matters
- Any Open Question that blocks implementation

## Guardrails

- The spec-write skill writes exactly one file (`specs/{slug}.md`) — no branch, tag, commit, or
  tracker issue unless the user asks. Planning is not an outward-facing action.
- If context is critically unclear, ask the user — but prefer a reasonable committed decision to keep
  momentum; that's what Open Questions is for.
- Report faithfully: couldn't verify something during research, say so in Open Questions rather than
  papering over it.
