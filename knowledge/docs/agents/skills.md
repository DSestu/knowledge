# Skills by use case

A **skill** is a packaged instruction set (`SKILL.md`) Claude Code loads on demand. Call one with `/<name>` (plugin skills: `/<plugin>:<name>`), or let Claude pick it from its description.

Sources used below:

- **built-in** — ships with Claude Code
- **agent-skills** — `addyosmani/agent-skills` plugin
- **claude-mem** — `thedotmack/claude-mem` plugin (persistent memory across sessions)
- **ponytail** — "lazy senior dev" minimal-diff plugin

## Plan a big change

| Situation | Skill |
|---|---|
| Idea still fuzzy, need to be questioned | `agent-skills:interview-me`, `agent-skills:idea-refine` |
| Need a spec / PRD before coding | `agent-skills:spec-driven-development` |
| Need a phased plan subagents can execute | `claude-mem:make-plan` |
| Need tasks with acceptance criteria + dependency graph | `agent-skills:planning-and-task-breakdown` |

## Execute

| Situation | Skill |
|---|---|
| Run a `make-plan` plan with subagents | `claude-mem:do` (commits per phase, see [gotchas](orchestration.md#gotchas)) |
| Large multi-agent run, parallel + resumable | Workflow tool ("use a workflow"); `workflow-authoring` is its reference |
| Build in thin tested slices | `agent-skills:incremental-implementation` |
| Test-first | `agent-skills:test-driven-development` |
| Keep the diff minimal | `ponytail:ponytail` |
| Something broke, find the root cause | `agent-skills:debugging-and-error-recovery` |

## Review and verify

| Situation | Skill |
|---|---|
| Bugs in the current diff or a PR | `/code-review [low…max] [--fix\|--comment]` (built-in) |
| Clean-up / reuse / simplification of the diff | `/simplify` (built-in) |
| Cross-examine decisions while working | `agent-skills:doubt-driven-development` |
| Multi-axis review before merge | `agent-skills:code-review-and-quality` |
| Security pass | `/security-review` (built-in), `agent-skills:security-and-hardening` |
| Watch a PR until CI + review threads are clean | `claude-mem:babysit` |

## Sessions and memory

| Situation | Skill |
|---|---|
| Context is long or the agent loops | `claude-mem:handoff` → fresh session reads `HANDOFF.md` |
| "Did we already solve this?" | `claude-mem:mem-search` |
| Onboard a new repo into memory | `claude-mem:learn-codebase` |
| Recurring prompt / polling | `/loop` (local interval), `/schedule` (cloud cron) |

## Design agent systems (reference reading)

| Topic | Skill |
|---|---|
| Topologies, fan-out/fan-in, failure handling | `agent-orchestration-patterns` |
| Cost: caching, model routing, context budgets | `agent-token-optimization-patterns` |
| Memory types, retrieval, consolidation | `agent-memory-patterns` |
| Shared rules across agents (CLAUDE.md layering) | `agent-shared-values-patterns` |
| Self-evaluation, LLM-as-judge | `agent-self-evaluation-patterns` |

## Writing your own skill

- `anthropic-skills:skill-creator` scaffolds a skill and runs evals on its triggering.
- Personal skills: `~/.claude/skills/<name>/SKILL.md`; project skills: `.claude/skills/<name>/SKILL.md`.
- The `description` line decides when Claude auto-loads it: write it as "Use when …" triggers, not a summary.
