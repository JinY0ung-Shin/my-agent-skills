# my-agent-skills

Claude Code plugin marketplace. A collection of agent skills for development workflows.

## Install

```bash
/plugin marketplace add JinY0ung-Shin/my-agent-skills
```

## Plugins

| Plugin | Description |
|--------|-------------|
| `codex-review` | Codex CLI integration for iterative code review, plan review, and AI-assisted implementation. Supports tmux pane monitoring. |

### codex-review

```bash
/plugin install codex-review@my-agent-skills
```

Runs Codex to review your code changes against 4 criteria:
- Bugs / Logic Errors
- Security
- Performance
- Code Quality

Automatically fixes CRITICAL and WARNING issues, then re-reviews until the code passes.

### codex-plan-review

```bash
/codex-plan-review [plan-file-path]
```

Runs Codex to review an implementation plan or design document against 4 criteria:
- Feasibility
- Completeness
- Risk & Impact
- Clarity & Correctness

Actively evaluates each finding (Accept/Partial/Disagree/Defer) instead of blindly accepting all suggestions. Iterates until the plan passes.

### codex-implement

```bash
/codex-implement [task description]
```

Delegates implementation to Codex CLI with a structured 4-phase workflow:

1. **Plan** — Claude analyzes the codebase and creates a detailed implementation plan
2. **Execute** — Codex implements the plan on a safety branch
3. **Verify** — Runs `/codex-review` for quality review, iterates fixes until passing
4. **Complete** — Presents results and lets you merge, keep, or discard

Claude plans, Codex executes. The plan file is the contract between them.
