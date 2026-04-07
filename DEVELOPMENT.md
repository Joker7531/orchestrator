# Development Log

This document tracks the development history, design decisions, and planned improvements for the Orchestrator plugin.

---

## v1.0.0 — 2026-04-06 · Initial Release

### What's included
- 5-phase orchestration protocol (Analysis → Exploration → Execution → Verification → Synthesis)
- 6 specialized Codex subagents: `codex-explorer`, `codex-implementer`, `codex-tester`, `codex-reviewer`, `codex-researcher`, `codex-refactorer`
- `/orchestrate` slash command entry point
- Auto-trigger skill for complex multi-step tasks
- Pure declarative Markdown architecture — no runtime dependencies

### Design decisions
- **Claude as coordinator only**: Claude never writes code or reads files directly; all work is delegated to Codex subagents. This keeps the orchestration layer clean and auditable.
- **Parallel-first execution**: Independent subtasks are always launched in parallel to minimize wall-clock time.
- **Complexity shortcuts**: Simple tasks skip unnecessary phases (e.g., bug fix skips verification phase).
- **Declarative plugin**: No npm/Node dependencies. The entire plugin is Markdown instructions, making it portable and easy to audit.

---

## v1.0.1 — 2026-04-06 · Permission Optimization

### Changes
- Configured `~/.claude/settings.json` with broad `allow` rules (`Bash(*)`, `Write(*)`, `Edit(*)`, `Read(*)`, `Glob(*)`, `Grep(*)`) so Codex subagents no longer require manual permission approval mid-task
- Added `deny` rules for destructive operations (`rm -rf*`, `sudo *`) as a safety baseline
- Rejected `bypassPermissions` mode in favor of explicit allow/deny rules for better auditability

### Motivation
Subagents were interrupting the orchestration flow with per-tool permission prompts. The fix pre-approves all common tool categories while preserving a hard block on the most dangerous operations.

---

## v1.0.2 — 2026-04-06 · Cost & Context Optimization

### Changes

**Fix 1: Agent model `inherit` → `haiku`** (all 6 agents)
- `codex-explorer`, `codex-implementer`, `codex-tester`, `codex-reviewer`, `codex-researcher`, `codex-refactorer`
- Wrapper agents only shape prompts and run a single Bash call — haiku is fully sufficient
- Estimated cost reduction: ~20× per subagent wrapper call vs. opus

**Fix 2: Structured return format** (all 6 agents)
- Agents no longer return raw Codex stdout; they summarize into typed XML-like sections
- Per-agent schemas:
  - `codex-explorer`: `<summary>` + `<files>` + `<open_questions>`
  - `codex-implementer` / `codex-refactorer`: `<summary>` + `<files>` + `<verification>` + `<residual_risks>`
  - `codex-tester`: `<summary>` + `<files>` + `<status>` + `<issues>`
  - `codex-reviewer`: `<verdict>` + `<findings>` + `<files>`
  - `codex-researcher`: `<summary>` + `<recommendation>` + `<tradeoffs>` + `<open_questions>`
- Limits max bullets to 5 per section, bounding result size regardless of Codex output length
- Reduces per-subagent context pollution in the main agent's conversation history

### Motivation
Evaluation revealed two gaps against the plugin's original design goals:
1. **Goal 1 (clean context)**: Raw Codex stdout was accumulating unbounded in the main agent's conversation history
2. **Goal 2 (cost reduction)**: `model: inherit` caused wrapper agents to run on opus (~7k tokens each), negating cost savings from delegating to Codex

### Testing
- Cache sync verified: all 6 agent files in `cache/local/orchestrator/1.0.0/` reflect `model: haiku` and structured return format
- **Bug found during deploy**: first `cp -r` left a stale nested `orchestrator/` directory in cache, causing plugin load failure. Fixed by removing it.
- **Bug found during deploy**: `plugin.json` had `author` as a string instead of `{"name": "..."}` object, causing manifest validation error. Fixed.
- Live test confirmed both fixes (codex-explorer):
  - Return format: `<summary>/<files>/<open_questions>` structure ✓
  - Execution time: 38149ms → 6984ms (haiku, ~5× faster) ✓
  - Token count: ~7k (same input volume, ~20× cheaper at haiku pricing) ✓

---

## v1.1.0 — 2026-04-07 · Server Agent & Model Disambiguation

### Changes

**New agent: `codex-server`**
- Dedicated subagent for remote server operations via SSH
- Supports command execution, log monitoring, file inspection, and status checks
- Follows the same haiku wrapper → Codex executor pattern as other agents
- Includes safety rules: no destructive commands (rm -rf, shutdown, reboot) without explicit authorization
- Structured return format: `<summary>` + `<server_state>` + `<action_taken>` + `<issues>`

**Fix: Agent model disambiguation in README**
- Agent table previously listed "Model: haiku" which was ambiguous — haiku is the wrapper, not the executor
- Replaced single "Model" column with "Wrapper" (haiku) and "Executor" (Codex / GPT) columns
- Added explanatory note above the table in both EN and ZH READMEs

**Updated orchestrate command**
- Added `codex-server` to the agent type list in Phase 1 analysis

---

## Planned Improvements

### Agent enhancements
- [ ] Add `codex-debugger` agent specialized for runtime error diagnosis
- [ ] Add `codex-documenter` agent for generating inline docs and changelogs
- [ ] Support agent retry logic when a subagent returns incomplete results

### Orchestration protocol
- [ ] Add Phase 0: context detection (auto-detect language, framework, test runner)
- [ ] Support task dependency graph visualization in the plan output
- [ ] Add user preference memory: remember preferred agent patterns per project type
- [ ] Support partial plan execution (skip specific phases on user request)

### UX improvements
- [ ] Add progress indicator during long multi-agent runs
- [ ] Allow users to override agent type per subtask in the plan confirmation step
- [ ] Add `--dry-run` mode: show plan without executing

### Quality & reliability
- [ ] Add validation step to check agent results meet acceptance criteria before synthesis
- [ ] Add timeout handling for long-running Codex tasks
- [ ] Add structured output format for agent results (JSON schema)

### Distribution
- [ ] Submit to Claude Code official marketplace
- [ ] Add CI/CD workflow for automated validation on PR
- [ ] Add plugin version auto-update notification

---

## How to Contribute

1. Fork the repo and create a feature branch from `develop`
2. Make changes and test locally by installing the plugin from your fork
3. Open a PR against `develop` with a clear description of the change
4. Reference the relevant planned improvement item if applicable

See [CONTRIBUTING](README.md#contributing) in the README for more details.
