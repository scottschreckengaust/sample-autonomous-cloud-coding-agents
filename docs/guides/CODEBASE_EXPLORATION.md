# Codebase Exploration Notes

This document captures a structured walkthrough of the ABCA codebase, answering the question: **"Are you able to explore and answer questions about the code base?"**

**Answer: Yes.** Below is a comprehensive summary of what the codebase contains and how it is organized.

---

## Repository Structure

```
.
├── agent/          # Python agent runtime (runs inside MicroVM compute)
├── cdk/            # AWS CDK infrastructure (TypeScript)
├── cli/            # bgagent CLI (TypeScript/Node)
├── docs/           # Design docs, guides, Starlight docs site, Claude Code plugin
├── scripts/        # CI helpers
├── contracts/      # Shared contract files (e.g., memory-hash-vectors.json)
├── mise.toml       # Monorepo task orchestration
└── package.json    # Yarn workspace root
```

---

## Key Components

### 1. `agent/` — Python Agent Runtime

The agent runs inside an isolated MicroVM (AWS AgentCore). It executes the coding work: clone repo, create branch, write code, run tests, commit, open PR.

| File | Purpose |
|------|---------|
| `src/pipeline.py` | Main pipeline — wires all modules together |
| `src/runner.py` | Executes the Claude Code agent SDK session |
| `src/config.py` | Environment variable config + `TaskConfig` model |
| `src/hooks.py` | Pre/post agent hooks (e.g., nudge reader) |
| `src/post_hooks.py` | `ensure_pr`, `verify_build`, `verify_lint`, `ensure_committed` |
| `src/progress_writer.py` | Writes live progress events to DynamoDB TaskEventsTable |
| `src/prompt_builder.py` | Builds the system prompt from modular templates |
| `src/prompts/` | Prompt templates: `base.py`, `new_task.py`, `pr_iteration.py`, `pr_review.py` |
| `src/memory.py` | Read/write memory via AgentCore Memory |
| `src/context.py` | Fetch GitHub issue/PR data for context hydration |
| `src/telemetry.py` | Trajectory writing, trace upload to S3 |
| `src/observability.py` | Task span instrumentation |

**Tests:** `agent/tests/` — 527 tests, all passing.

---

### 2. `cdk/` — AWS CDK Infrastructure

All AWS resources defined as CDK constructs in TypeScript.

**Constructs** (`cdk/src/constructs/`):

| Construct | Purpose |
|-----------|---------|
| `task-orchestrator.ts` | Lambda Durable Function driving the task state machine |
| `task-api.ts` | API Gateway + Cognito authentication |
| `blueprint.ts` | Per-repo configuration construct |
| `ecs-agent-cluster.ts` | ECS cluster for compute environments |
| `agent-memory.ts` | AgentCore Memory integration |
| `agent-vpc.ts` | VPC with DNS firewall for egress control |
| `task-table.ts` | DynamoDB task records table |
| `task-events-table.ts` | DynamoDB task progress events table |
| `slack-integration.ts` | Slack channel integration |
| `linear-integration.ts` | Linear issue tracker integration |

**Lambda Handlers** (`cdk/src/handlers/`): One file per REST operation — `create-task.ts`, `orchestrate-task.ts`, `list-tasks.ts`, `get-task.ts`, `cancel-task.ts`, `nudge-task.ts`, `get-task-events.ts`, and Slack/Linear webhook handlers.

**Shared modules** (`cdk/src/handlers/shared/`): `types.ts` (task record schema and API types), `validation.ts`, `preflight.ts`, `create-task-core.ts`, `context-hydration.ts`, `memory.ts`, `repo-config.ts`, `orchestrator.ts`, `response.ts`, `error-classifier.ts`.

---

### 3. `cli/` — `bgagent` CLI

TypeScript CLI (`@backgroundagent/cli`) for submitting and managing tasks via the REST API.

| File | Purpose |
|------|---------|
| `src/bin/bgagent.ts` | Entry point |
| `src/commands/` | `configure`, `login`, `submit`, `list`, `status`, `cancel`, `events` |
| `src/api-client.ts` | HTTP client with Cognito auth header injection |
| `src/auth.ts` | Cognito token management and caching (`~/.bgagent/credentials.json`) |
| `src/types.ts` | API types (must be kept in sync with `cdk/src/handlers/shared/types.ts`) |
| `src/format.ts` | Output formatting (table, detail view, JSON) |

---

### 4. `docs/` — Documentation

- `docs/design/` — Architecture, Orchestrator, Compute, Memory, Repo Onboarding, Cost Model, API Contract
- `docs/guides/` — Developer Guide, User Guide, Deployment Guide, Prompt Guide, Quick Start, Roadmap, Slack/Linear setup
- `docs/abca-plugin/` — Claude Code plugin (skills: `setup`, `deploy`, `onboard-repo`, `submit-task`, `troubleshoot`, `status`; agents: `cdk-expert`, `agent-debugger`)
- `docs/src/content/docs/` — **Generated** Starlight mirrors (never edit directly; regenerate with `mise //docs:sync`)

---

## Task Lifecycle (Blueprint)

Every task follows a 5-step blueprint mixing deterministic and agentic steps:

```
Admission → Context Hydration → Pre-flight → Agent Execution → Finalization
```

1. **Admission** — Validate request, check concurrency limits, load Blueprint config
2. **Context Hydration** — Fetch GitHub issue/PR, query memory, assemble full prompt
3. **Pre-flight** — Verify GitHub API reachability + repo access (fail fast before compute)
4. **Agent Execution** — Isolated MicroVM: clone → branch → code → test → commit → PR
5. **Finalization** — Infer result, write memory learnings, update task status, release concurrency

---

## Task Types

| Type | Description |
|------|------------|
| `new_task` | New feature or bug fix starting from scratch |
| `pr_iteration` | Respond to code review comments on an existing PR |
| `pr_review` | Read-only analysis and review of an existing PR |

---

## Task State Machine

```
SUBMITTED → HYDRATING → RUNNING → FINALIZING → COMPLETED
                                              → FAILED
                                              → CANCELLED
                                              → TIMED_OUT
```

---

## Key Conventions

- **`MISE_EXPERIMENTAL=1`** is required for namespaced mise tasks (`mise //cdk:build`, `mise //agent:quality`, etc.)
- **Docs sync**: Always run `mise //docs:sync` (or `cd docs && node scripts/sync-starlight.mjs`) after editing `docs/guides/` or `docs/design/` — CI rejects stale Starlight mirrors
- **Types sync**: `cdk/src/handlers/shared/types.ts` and `cli/src/types.ts` must be kept manually in sync
- **Worktrees**: Always `git fetch origin main` before creating a new worktree; run `mise run install` per worktree
- **Commit style**: Conventional commits — `<type>(<module>): description`
