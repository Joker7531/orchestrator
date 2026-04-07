# orchestrator

Task orchestration plugin for Claude Code — keeps your main agent context clean while delegating heavy lifting to cheaper models.

[中文](README.zh-CN.md)

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg) ![Version: 1.0.0](https://img.shields.io/badge/version-1.0.0-green.svg)

---

## Design Goals

This plugin was built around two constraints:

**1. Keep the main agent context clean**

In long engineering tasks, the main agent's context window fills up with raw tool outputs, file contents, and intermediate results — making it harder to reason clearly and increasing costs. The orchestrator keeps the main agent focused on decisions only. All exploration, implementation, and verification work is offloaded to subagents whose outputs never appear directly in the main context.

**2. Reduce cost by using cheaper models for execution**

Complex tasks don't need an expensive model end-to-end. The orchestrator uses a three-tier model hierarchy:

```
Main agent (opus)       — task decomposition, dependency management, synthesis
  └─ Wrapper (haiku)    — prompt shaping, result compression
       └─ Codex (GPT)   — actual file reads, code writes, test runs
```

Expensive reasoning is concentrated in the main agent (a small number of high-level decisions). Cheap execution is distributed across haiku wrappers and Codex — where the volume of work actually lives.

---

## Requirements

- [Claude Code](https://claude.ai/code) with the Codex plugin installed
- The following skills must be installed:
  - `codex:codex-cli-runtime`
  - `codex:gpt-5-4-prompting`

---

## Installation

```bash
claude plugin install https://github.com/Joker7531/orchestrator
```

---

## Usage

```
/orchestrate Add JWT authentication to this Express API
/orchestrate Refactor the database layer to use connection pooling
/orchestrate Find and fix all TypeScript type errors in src/
/orchestrate Write unit tests for the auth module
```

The orchestrator also triggers automatically when Claude detects a task that spans 3+ files, multiple implementation steps, or a combination of exploration + implementation + testing.

---

## How It Works

### The three-tier execution model

Each subagent is a lightweight **haiku wrapper** — its only job is to (1) shape the main agent's natural-language task into a precise, structured Codex prompt, and (2) compress Codex's raw output into a bounded structured summary before returning it to the main agent.

```
Main agent sends:
  "Explore the auth module architecture"

Wrapper shapes into:
  <task>Explore src/auth/ architecture</task>
  <research_mode>Separate facts from inferences</research_mode>
  <grounding_rules>Cite file paths for every claim</grounding_rules>
  <structured_output_contract>Return: key files, dependencies, open questions</structured_output_contract>

Codex executes, wrapper compresses and returns:
  <summary>
  - AuthService depends on JwtStrategy and UserRepository
  - Token refresh handled in src/auth/refresh.guard.ts
  </summary>
  <files>
  - src/auth/auth.service.ts — core service
  - src/auth/refresh.guard.ts — token refresh logic
  </files>
  <open_questions>
  - Token expiry config not found in this scope
  </open_questions>
```

The main agent only ever sees the compressed summary — not the raw Codex output.

### Contract Registry

The orchestrator maintains a **CONTRACTS.md** file at the project root as the single source of truth for all public interfaces between modules. This eliminates the need for the main agent to read code files when wiring subagents together.

The information flows in one direction:

```
codex-implementer finishes a batch
  └─ returns <interface_contract> block in its output

codex-architect reads the produced files
  └─ updates CONTRACTS.md
  └─ returns <contract_summary> to the orchestrator

orchestrator injects <contract_summary> into next batch as <upstream_contracts>
  └─ downstream implementers read interfaces from prompt, not from disk
```

The main agent reads code files **only when a critical error cannot be diagnosed from agent outputs alone**.

### Five-phase protocol

```
Phase 1 · Analysis      Claude decomposes the task, presents a plan, waits for confirmation
Phase 2 · Exploration   codex-explorer + codex-researcher run in parallel
Phase 3 · Execution     implementation batches, each followed by codex-architect
Phase 4 · Verification  codex-tester + codex-reviewer run in parallel
Phase 5 · Synthesis     Claude collects summaries, resolves issues, presents final result
```

Phase 3 uses a batch-architect pattern for multi-module tasks:

```
Batch A (parallel implementers)
  └─ Batch A.arch: codex-architect → updates CONTRACTS.md, returns contract_summary
Batch B (parallel implementers, receive Batch A contracts via upstream_contracts)
  └─ Batch B.arch: codex-architect → updates CONTRACTS.md, returns contract_summary
...
```

Not every task needs all five phases:

| Task type | Phases used |
|---|---|
| Simple single-module fix | implement (no architect needed) |
| Multi-module feature | full flow with architect between each batch |
| Pure research | researcher only |
| Refactoring | explore → refactor → architect → test → review |

### Agents

Each agent has two layers: a **wrapper** (haiku) that shapes prompts and compresses results, and an **executor** (Codex / GPT) that does the actual work. The "Wrapper" column shows the Claude model used for prompt shaping; the real execution always happens inside Codex.

| Agent | Wrapper | Executor | Mode | Role |
|---|---|---|---|---|
| `codex-explorer` | haiku | Codex (GPT) | read-only | Codebase exploration and architecture mapping |
| `codex-implementer` | haiku | Codex (GPT) | write | Feature implementation and bug fixes; returns `<interface_contract>` |
| `codex-architect` | haiku | Codex (GPT) | write\* | Reads produced files, updates CONTRACTS.md, returns `<contract_summary>` |
| `codex-tester` | haiku | Codex (GPT) | write | Write and run tests |
| `codex-reviewer` | haiku | Codex (GPT) | read-only | Code review and quality analysis |
| `codex-researcher` | haiku | Codex (GPT) | read-only | Web research and documentation lookup |
| `codex-refactorer` | haiku | Codex (GPT) | write | Restructure, rename, and consolidate code |
| `codex-server` | haiku | Codex (GPT) | — | Remote server command execution, log monitoring, status checks |

\* `codex-architect` writes only to CONTRACTS.md — never to production code files.

---

## Architecture

```
orchestrator/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest
├── agents/
│   ├── codex-architect.md   # Write wrapper: contract extraction, CONTRACTS.md maintenance
│   ├── codex-explorer.md    # Read-only wrapper: exploration and architecture mapping
│   ├── codex-implementer.md # Write wrapper: implementation and bug fixes
│   ├── codex-refactorer.md  # Write wrapper: restructure and consolidate
│   ├── codex-researcher.md  # Read-only wrapper: web research and docs
│   ├── codex-reviewer.md    # Read-only wrapper: code review
│   ├── codex-server.md      # Server wrapper: SSH command execution, monitoring
│   └── codex-tester.md      # Write wrapper: test authoring and execution
├── commands/
│   └── orchestrate.md       # /orchestrate slash command
└── skills/
    └── orchestrator/
        └── SKILL.md         # Auto-trigger skill
```

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first.

1. Fork the repository: https://github.com/Joker7531/orchestrator
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit your changes and open a pull request against `develop`

---

## License

[MIT](https://opensource.org/licenses/MIT) — Copyright (c) Joker7531
