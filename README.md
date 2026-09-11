# Multi-Agent Delivery Orchestrator

A file-based, editor-native multi-agent system for shipping data-platform changes (dbt models, orchestrator DAGs, ETL jobs) from ticket to merged PR — with mandatory human approval gates at every irreversible step.

Built as [Cursor](https://cursor.com) `.mdc` rule files. No runtime, no daemon, no database. All agent state lives in git-diffable YAML.

> Originally built and run in production against a dbt + Airflow + Glue data platform. The pattern generalizes to any pipeline with an ordering constraint between an infrastructure layer and a transformation layer — swap the MCP servers and it works for Databricks, Snowflake, Dagster, or any equivalent stack.

---

## Why this exists

Coding agents are good at writing a model and bad at knowing when it is *safe* to build it. In a data platform the ordering constraint is real: a transformation that reads a table cannot be built until the job that loads that table has actually run in QA.

This system encodes that constraint as an explicit workflow path chosen per ticket, and refuses to advance past it without a human saying so.

It is not an autonomous agent. It is a **governed pipeline** — five narrow-scoped agents with a human in the loop at every point where being wrong is expensive.

---

## Agent roster

Five roles, one `.mdc` file each. Boundaries are strict and enforced by prompt contract — an agent that oversteps is a bug.

| Agent | Owns | Never does |
|---|---|---|
| **Orchestrator** | Routing, work selection, commit + push + PR, closeout | Write feature code; merge a PR |
| **Developer** | Plan Mode, implementation, feature branch, QA build | Any documentation — no comments, docstrings, or logs |
| **Tester** | Warehouse validation, test suites, load verification | Fix the code it finds broken |
| **Documentor** | All narrative: inline comments, maintenance headers, wiki | Change behaviour |
| **Reviewer** | PR diff review, rule compliance, approval polling, merge gate | Merge without explicit user confirmation |

**Why documentation is a separate agent.** The Developer adds zero narrative — no comments, no docstrings, no maintenance-log headers. A dedicated Documentor pass writes all of it afterward. Two payoffs: the code diff stays reviewable (behaviour changes aren't buried in comment churn), and documentation gets written against finished code rather than intent that may have drifted during implementation.

**Why the Orchestrator owns delivery, not the Developer.** Commit, push, and PR creation belong to the Orchestrator. The Developer cannot ship its own work. This makes "code complete" and "code delivered" separately observable states — which is what makes resume-after-interruption reliable.

---

## MCP servers

Tool access is scoped **per agent**, not per session. Name your servers generically — this reference config avoids anything specific to one machine's workspace layout.

| Server | Scope | Used by | Purpose |
|---|---|---|---|
| `worktracker-mcp` | global | Orchestrator, Reviewer | Work items, PR create/update, reviewers, comments (Azure DevOps / Jira / Linear — any tracker with an MCP server) |
| `warehouse-mcp` | project | Tester only | `execute_select` — read-only validation queries |
| `object-store-mcp` | project | Tester | Latest-output lookup, run status |
| `job-runner-mcp` | project | Tester | ETL job run status and output inspection |
| `lakehouse-mcp` | project | Tester | Lake-side validation |
| `lineage-mcp` | project | Developer, Tester | Model/column lineage for impact analysis |
| `scheduler-mcp` | global | Tester | DAG/pipeline run status |

The warehouse server is Tester-only and read-only. The Developer never gets a query tool; the Tester never gets a write tool. This is enforced in each agent's `.mdc` contract rather than by server permissions — a convention, but one restated in every prompt that could violate it.

---

## Orchestration flow

### Entry: a thin router

The top-level file (`agent-orchestrator.mdc`) does nothing but decide which section file to load. Keep it under ~700 tokens — it is loaded on every turn.

| Trigger | Loads |
|---|---|
| `start my day` | `orchestrator-kickoff.mdc` |
| `resume {id}` | `handoff-{id}.yml` + `orchestrator-pipeline.mdc` |
| `plan` / `implement` / `continue` | `orchestrator-pipeline.mdc` |
| `commit` / `PR` / `closeout` | `orchestrator-delivery.mdc` |

### Work selection

Reads a shared `projects-registry.json`, presents the project menu, resolves exactly one playbook, then picks a ticket by one of three modes — current sprint, area path, or a curated priority queue. **The system never auto-selects a ticket.**

### Path selection

A technology router classifies changed file paths and sets the workflow path:

| Path | Trigger | Build step | First human stop |
|---|---|---|---|
| `dbt_only` | Transform layer alone, no upstream dependency | Before Tester | After tests pass |
| `mixed_upstream_dbt` | Transform + the loaders it depends on | After QA load | At first PR |
| `upstream_only` | Loader/job alone | n/a | At first PR |

`mixed_upstream_dbt` inverts the usual order: the PR opens *early*, so the human can deploy it to QA and run the loaders — only then does the transform build happen. This is why the Documentor runs twice on that path (inline docs before the PR, wiki after the tests).

### Handoff

Every agent-to-agent transition passes exactly three things: work item ID, handoff file path, and progress state. Procedural detail lives in the receiving agent's own file and is never duplicated into the prompt.

---

## Memory / state layer

File-based. No database, no memory server.

| Artifact | Format | Scope | In git |
|---|---|---|---|
| `handoff-{id}.yml` | YAML | One per ticket | Yes |
| `projects-registry.json` | JSON | Shared project menu | Yes |
| `playbooks/*.yml` | YAML | Per-project config | Yes |
| `rules-skills-roster.yml` | YAML | Standards manifest | Yes |
| `.last-kickoff.local.json` | JSON | Per-user default | **No — gitignored** |

**The handoff contract is the state machine.** One file per work item, holding: resolved author identity, workflow path, progress state, named quality gates with owners, changed files/models, PR identifiers, and the list of standards already loaded. See [`templates/handoff-contract-template.yml`](templates/handoff-contract-template.yml).

Rules that matter: one handoff per ticket, never reused; no pointer file — every prompt names its work item explicitly. Switching tickets means opening a second handoff, not mutating the first.

**Why files beat a database here.** Agent state is reviewable in a pull request. You can see in a diff that an agent marked a gate pass. Recovery from a bad run is `git checkout`. There is no service to run and no divergence between what the agent believes and what a reviewer can read.

**The honest tradeoff:** no cross-ticket learning. Decisions made on one ticket don't inform the next. Durable knowledge (ADRs, conventions, past decisions) has to be promoted by hand into a rule or skill file. A queryable memory server is the obvious extension — the roster is the natural place to index it — but it isn't part of this system today.

**Per-user state must never touch shared files.** An early version stored "last kickoff" in the shared registry; every developer's morning produced a one-line diff and a merge conflict. Per-user state lives in a gitignored `.local.json`.

---

## Human-in-the-loop gates

The system stops and waits at every point where being wrong is expensive.

| Gate | Who decides | Blocks |
|---|---|---|
| Project + ticket selection | User | Everything |
| Plan approval | User | Any implementation |
| QA deploy + loader run | User | Build (mixed) / testing (upstream) |
| Continue after QA load | User | Test phase |
| Continue after tests | User | Docs + delivery (`dbt_only`) |
| Wiki new vs. update vs. skip | User | Documentor pass 2 |
| Merge | User | Merge. Always. |
| Handoff deletion | User | Cleanup |

**Plan Mode is the load-bearing gate.** For any user story, the Developer must enter Plan Mode, ask clarifying questions, produce a plan, and stop. Nothing downstream runs until the user approves. It can be skipped only by explicit instruction, recorded on the handoff as `plan_approved: skip`.

**The workflow deliberately ends at a question, not an action.** The Reviewer polls for approvals and then asks. Nothing auto-merges.

**Cadence.** One kickoff per day; `resume {id}` for every subsequent session on the same ticket. Auto-advance is permitted only *between* gates — e.g. implement → build → test runs without prompting, because interrupting there adds no decision value.

---

## Repo structure

```
.
├── AGENTS.md                          # cross-tool entry point
├── agent-orchestrator.mdc             # router — decides what to load
├── orchestrator-kickoff.mdc           # phase: daily start, work selection
├── orchestrator-pipeline.mdc          # phase: plan → implement → test
├── orchestrator-delivery.mdc          # phase: commit → PR → closeout
├── master-developer.mdc               # agent contract
├── master-tester.mdc                  # agent contract
├── master-documentor.mdc              # agent contract
├── master-reviewer.mdc                # agent contract
├── technology-router.mdc              # changed paths → workflow path
├── rules-skills-roster.yml            # standards manifest
├── playbooks/
│   ├── _template.yml                  # seed for a new project
│   └── example-project.yml            # filled example
└── templates/
    └── handoff-contract-template.yml  # per-ticket state schema
```

Every `.mdc` carries frontmatter (`description`, `alwaysApply: false`) so the editor matches it by description instead of loading it unconditionally.

---

## Guardrails

Stated as explicit negatives, because agents comply with prohibitions far better than with implied ordering:

- Do not fetch sprint tickets before the user picks a working mode
- Do not auto-select a ticket
- Do not implement before plan approval
- Do not run a build before the user confirms the QA load
- Do not merge
- Do not delete a handoff before closeout and confirmation
- Do not post closeout to a different work item than the session started on

**Least privilege.** Warehouse access is one agent, read-only.
**Auditability.** Every gate has a named owner and lands in a git-diffable file. Closeout posts a summary comment back to the originating ticket — mandatory, before any cleanup.
**Context economy as a guardrail.** Agents are explicitly forbidden from globbing the standards corpus; they resolve a manifest instead. An unbounded scan is not just slow — it crowds out the ticket context the agent needs to be correct.

---

## Token economics

Context discipline was treated as a first-class design constraint, not an afterthought.

| Change | Effect |
|---|---|
| Split monolith into router + 3 phase files | Entry point 14,402 → 656 tokens |
| Manifest instead of corpus scan | Bounded ~2k lookup replaces a ~400k-token scan surface |
| Deleted ASCII flow diagrams | −47% of file bytes; tables say it more precisely |
| Collapsed 3× path duplication | One matrix + one parameterised prompt |
| `resume {id}` shortcut | Skips work selection entirely |

| Invocation | Before | After | Cut |
|---|---|---|---|
| Daily kickoff | 14,402 | 4,489 | −69% |
| Resume ticket | 14,402 | 2,254 | −84% |
| Delivery / closeout | 14,402 | 1,867 | −87% |

Two lessons worth carrying into any similar system:

1. **Box-drawing diagrams are expensive and redundant.** Flow diagrams were 47% of the original router file. The routing tables that replaced them are more precise — they carry "auto-advance?" and "user gate?" columns a picture can't express.
2. **Always-on rules dominate everything else.** A rule marked "always apply" is multiplied by every turn in every session. Scoping rules to the paths they actually govern saves more than any other single change, because the saving is per-request rather than per-invocation. Audit always-on rules first — it's the highest-leverage change available.

---

## Adapting this to your stack

This repo ships as a template, not a finished product. Before you point it at a real project:

- [ ] Rename the MCP servers to match your actual tools (warehouse, object store, scheduler, work tracker)
- [ ] Fill in `playbooks/_template.yml` with your repo, branch names, and reviewer roster
- [ ] Adjust `technology-router.mdc`'s path table to your own layer boundaries (it doesn't have to be dbt/Airflow — any "infra must run before transform" pipeline fits)
- [ ] Add an `AGENTS.md` if your editor doesn't already read `.mdc` files natively — it's now the cross-tool convention and the file most contributors look for first
- [ ] Confirm every wired MCP server is referenced by at least one agent's `.mdc` — an unreferenced server is dead weight and a security surface
- [ ] Keep `.last-kickoff.local.json` (or your equivalent) gitignored — per-user state in a shared file causes merge conflicts on every kickoff

---

## Status

Production-tested against a real dbt + Airflow + Glue data platform, across multiple concurrent project boards.

**Known gaps:**
- No cross-ticket memory — durable decisions must be promoted by hand into a rule file
- The standards manifest (`rules-skills-roster.yml`) is hand-maintained
- Gate enforcement is prompt-contract, not runtime-enforced — an adversarial or badly-prompted agent could theoretically bypass a gate; this system assumes a cooperative agent operating under human review, not a sandboxed adversarial one

## License

MIT — see [LICENSE](LICENSE).
