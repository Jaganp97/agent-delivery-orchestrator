# AGENTS.md

This repo is driven by five narrow-scoped agents. If your tool reads `AGENTS.md` instead of `.mdc` rule files, start here — it's a summary, not a replacement; the `.mdc` files are the source of truth.

## Start here

- New session, no ticket in hand → say **"start my day"** (loads `orchestrator-kickoff.mdc`)
- Returning to a ticket already in progress → say **"resume {ticket-id}"**
- Ready to implement, test, or continue a loaded ticket → say **"plan"**, **"implement"**, or **"continue"**
- Ready to ship → say **"commit"**, **"PR"**, or **"closeout"**

Full router logic: [`agent-orchestrator.mdc`](agent-orchestrator.mdc)

## The five agents

| Agent | File | Owns |
|---|---|---|
| Orchestrator | `agent-orchestrator.mdc` + `orchestrator-*.mdc` | Routing, delivery, closeout |
| Developer | `master-developer.mdc` | Plan Mode, implementation |
| Tester | `master-tester.mdc` | Validation, no fixes |
| Documentor | `master-documentor.mdc` | All narrative, no behaviour changes |
| Reviewer | `master-reviewer.mdc` | PR review, merge gate |

## Non-negotiables

1. Feature branch from `main`; PR into `develop`. Never the reverse.
2. Author identity is resolved at runtime (ticket assignee → `git config user.name` → OS user) — never hardcoded.
3. Documentation is Documentor-only. The Developer writes zero comments, docstrings, or logs.
4. One handoff file per work item. No pointer files. Switching tickets means opening a second handoff.
5. Nothing merges without an explicit human confirmation. No exceptions.

## Where state lives

Everything is a git-diffable file — see [`README.md § Memory / state layer`](README.md#memory--state-layer). There is no database and no background process.

## Before you start

Read the [Adapting this to your stack](README.md#adapting-this-to-your-stack) checklist in the README. This template ships with placeholder MCP server names and an empty playbook — it will not run against a real project until you fill those in.
