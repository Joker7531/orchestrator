# orchestrator

Dynamic task orchestration for Claude Code — decompose complex requests, dispatch subtasks to Codex subagents, synthesize results.

[中文](README.zh-CN.md)

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg) ![Version: 1.0.0](https://img.shields.io/badge/version-1.0.0-green.svg)

---

## What It Does

The orchestrator plugin gives Claude Code a structured way to tackle large, multi-step engineering tasks. When you invoke `/orchestrate`, Claude decomposes your request into focused subtasks and dispatches them to six specialized Codex subagents — each optimized for a different role (exploration, implementation, testing, review, research, refactoring). Results are collected, critical issues are resolved, and a clean summary is presented back to you.

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

## What's Included

### Agents

| Agent | Mode | Description |
|---|---|---|
| `codex-explorer` | read-only | Codebase exploration and architecture mapping |
| `codex-implementer` | write | Code implementation, feature additions, bug fixes |
| `codex-tester` | write | Write and run tests |
| `codex-reviewer` | read-only | Code review and quality analysis |
| `codex-researcher` | read-only | Web research and documentation lookup |
| `codex-refactorer` | write | Restructure, rename, and consolidate code |

### Commands

| Command | Description |
|---|---|
| `/orchestrate <task>` | Entry point — describe any task and Claude will handle decomposition and dispatch |

### Skills

| File | Description |
|---|---|
| `skills/orchestrator/SKILL.md` | Auto-trigger skill — activates automatically when a task involves 3+ files, multiple implementation steps, or a combination of exploration + implementation + testing |

---

## Usage / Workflow Examples

**Add a new feature:**

```
/orchestrate Add JWT authentication to this Express API
```

**Refactor existing code:**

```
/orchestrate Refactor the database layer to use connection pooling
```

**Fix type errors across the codebase:**

```
/orchestrate Find and fix all TypeScript type errors in src/
```

**Write tests for a module:**

```
/orchestrate Write unit tests for the auth module
```

---

## How It Works

The orchestrator follows a 5-phase protocol:

```
┌─────────────────────────────────────────────────────────────┐
│  Phase 1 · Analysis                                         │
│  Claude parses the request, decomposes into subtasks,       │
│  and waits for your confirmation before proceeding.         │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  Phase 2 · Exploration                              ░░ ░░   │
│  codex-explorer + codex-researcher run in parallel          │
│  to map the codebase and gather background context.         │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  Phase 3 · Execution                                        │
│  codex-implementer + codex-refactorer run —                 │
│  parallel where independent, sequential where dependent.    │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  Phase 4 · Verification                             ░░ ░░   │
│  codex-tester + codex-reviewer run in parallel              │
│  to validate correctness and code quality.                  │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────┐
│  Phase 5 · Synthesis                                        │
│  Claude collects all results, resolves critical issues,     │
│  and presents a clean summary with next steps.              │
└─────────────────────────────────────────────────────────────┘
```

### Complexity Shortcuts

Not every task needs all five phases. The orchestrator picks the right path automatically:

| Task type | Phases used |
|---|---|
| Simple bug fix | explore → implement → test |
| Pure research | researcher only |
| Large feature | full 5-phase flow |
| Refactoring | explore → refactor → test → review |

---

## Architecture

```
orchestrator/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest (name, version, description)
├── agents/
│   ├── codex-explorer.md    # Read-only agent: codebase and architecture mapping
│   ├── codex-implementer.md # Write agent: feature implementation and bug fixes
│   ├── codex-refactorer.md  # Write agent: restructure and consolidate code
│   ├── codex-researcher.md  # Read-only agent: web research and docs lookup
│   ├── codex-reviewer.md    # Read-only agent: code review and quality analysis
│   └── codex-tester.md      # Write agent: test authoring and execution
├── commands/
│   └── orchestrate.md       # /orchestrate slash command definition
└── skills/
    └── orchestrator/
        └── SKILL.md         # Auto-trigger skill for multi-step tasks
```

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

1. Fork the repository: https://github.com/Joker7531/orchestrator
2. Create your feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -m 'Add my feature'`
4. Push to the branch: `git push origin feature/my-feature`
5. Open a pull request

---

## License

[MIT](https://opensource.org/licenses/MIT) — Copyright (c) Joker7531
