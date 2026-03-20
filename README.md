# my-agent-skills

Claude Code plugin marketplace. A collection of agent skills for development workflows.

## Install

```bash
/plugin marketplace add JinY0ung-Shin/my-agent-skills
```

## Plugins

| Plugin | Description |
|--------|-------------|
| `codex-review` | Iterative code review with Codex CLI. Repeats review-fix cycle until score >= 8.0 with 0 critical issues. Supports tmux pane monitoring. |

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
