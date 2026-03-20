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
