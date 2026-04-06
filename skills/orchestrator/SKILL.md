---
name: orchestrator
description: Automatically activates orchestrator mode when Claude detects a complex, multi-step task that would benefit from decomposition and parallel Codex execution. Triggers on large feature requests, multi-file changes, complex debugging, or any task involving 3+ distinct implementation steps.
user-invocable: false
---

# Orchestrator Mode (Auto-Trigger)

When you detect a complex task that meets ANY of these criteria, suggest using `/orchestrate` to the user:

- Task requires changes across 3+ files
- Task involves multiple distinct implementation steps
- Task combines exploration + implementation + testing
- Task is a large feature with unclear scope
- Task requires both research and implementation
- User explicitly asks for orchestrated or parallel execution

**How to suggest:**

Tell the user:
> This looks like a multi-step task that would benefit from orchestrated execution. I can decompose it and run subtasks in parallel via Codex. Want me to run `/orchestrate <task>`?

**Do NOT auto-trigger for:**
- Simple single-file edits
- Quick bug fixes with obvious solutions
- Pure questions or research
- Tasks the user is clearly handling step-by-step themselves
