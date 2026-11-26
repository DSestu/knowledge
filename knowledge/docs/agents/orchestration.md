# Orchestrator agents

An **orchestrator** is one agent that splits a big task, hands the pieces to subagents, and checks their results before moving on. It pays off when the work spans many files or phases; for a handful of edits, one agent is faster and cheaper.

## Tooling landscape (as of late 2026)

### Built into Claude Code (no install)

| Tool | What it does | Trigger |
|---|---|---|
| **Subagents** (`Agent` tool) | Fresh-context worker with its own prompt, tools, model. Custom ones live in `.claude/agents/*.md` | Automatic, or "use a subagent for X" |
| **Workflow tool** | Scripted multi-agent run: phases, parallel fan-out, verification, resumable by run ID | Say "use a workflow" or `ultracode` — never runs unasked |
| **Worktrees** | Each agent gets its own git checkout, so parallel edits don't collide | `isolation: "worktree"` on the agent |
| **Agent teams** (experimental) | Several sessions sharing a task list and messaging each other | Feature flag |

### External: run many coding agents in parallel

| Tool | Shape |
|---|---|
| **Claude Squad** | tmux TUI, one Claude Code / Codex / Aider session per worktree |
| **Conductor** | Mac app, parallel Claude Code in worktrees + review/merge UI |
| **Vibe Kanban** | Kanban board, each card is an agent run |
| **Claude-Flow** | "Swarm" framework (queen/workers, shared memory). Heavy |
| **T3 Code** | `delegate_task` sends work to another provider (Codex…) or model, optionally in a worktree |

### Frameworks: build your own orchestrator in code

| Framework | Strength |
|---|---|
| **Claude Agent SDK** | Same loop as Claude Code as a Python/TS lib, subagents built in |
| **LangGraph** | Graph state machines, persistence, human-in-the-loop |
| **OpenAI Agents SDK** | Handoffs between agents, guardrails |
| **CrewAI** | Role-based "crews", fast prototyping |
| **Microsoft Agent Framework** / **Google ADK** | Enterprise-flavoured alternatives |

Rule of thumb: stay with the built-in tools for coding work; reach for a framework only when the orchestrator must run outside the IDE (a service, CI).

## Recommended pipeline

One owner per stage, no two skills doing the same job:

| Stage | Use | Why |
|---|---|---|
| What to build | `agent-skills:spec-driven-development` (spec stage only) | Human approves the spec before any plan exists |
| How to build | `claude-mem:make-plan` | Phase 0 makes subagents verify real APIs; one plan file, phases sized for fresh contexts |
| Execute | **Workflow tool** (big) or `claude-mem:do` (≤ 3 phases) | Workflow runs independent phases in parallel, in worktrees, and resumes; `do` is sequential |
| Review | `agent-skills:doubt-driven-development` + a **different model** as reviewer | A model reviewing its own output tends to approve it |
| Ship | `/code-review` → `claude-mem:babysit` | Review the diff, then watch the PR until CI and threads are clean |

Mid-run: `claude-mem:handoff` when the context degrades (writes `HANDOFF.md` for a fresh session).

## Gotchas

- **`claude-mem:do` commits and pushes after every verified phase** (it deploys a "Commit" and a "Branch/Sync" subagent). If you want staged-only changes, say "do not commit or push" in the call, or put it in the project `CLAUDE.md`.
- **Agent count explodes.** `do` starts ~6 subagents per phase (implement, verify, anti-pattern, quality, commit, sync): a 6-phase plan ≈ 36 agents. Merge the three checks into one reviewer to halve it.
- **Overlapping planners.** `spec-driven-development` already chains SPECIFY → PLAN → TASKS → IMPLEMENT; `planning-and-task-breakdown` duplicates `make-plan`. Pick one per stage.
- **Subagent reports are summaries.** They come back in the same confident tone whether right or wrong. `make-plan` requires sources + confidence in every report; reject reports without them.
- **Delegation has a fixed cost.** Each fresh agent only knows its brief. If the brief can't be written in one clear prompt, the task isn't ready to delegate.

See [Skills by use case](skills.md) for the full skill map.
