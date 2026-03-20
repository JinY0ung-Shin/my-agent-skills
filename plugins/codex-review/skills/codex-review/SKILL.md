---
name: codex-review
description: Get iterative code review from Codex CLI until score >= 8.0 with 0 critical issues (tmux pane monitoring)
user_invocable: true
---

# Codex Code Review Loop

Run Codex to review code changes. If inside tmux, opens a separate pane for real-time monitoring.
Iterate — fix issues and re-request review until score >= 8.0 with 0 critical issues.

## Steps

1. **Detect review target.** Check for uncommitted changes:

```bash
git diff --quiet HEAD 2>/dev/null; echo $?
```

- Exit code `1` → uncommitted changes exist → `DIFF_CMD="git diff HEAD"`
- Exit code `0` → no uncommitted changes → `DIFF_CMD="git diff HEAD~1"` (직전 커밋 리뷰)

2. **Prepare output file.**

```bash
REVIEW_OUT=$(mktemp /tmp/codex-review-XXXXXX.txt)
```

3. **Write runner script.**

```bash
cat > /tmp/codex-review-run.sh << 'SCRIPT'
#!/bin/bash
cd "$1"
codex exec --dangerously-bypass-approvals-and-sandbox -o "$2" 'You are a senior code reviewer. Run `'"${DIFF_CMD}"'` to see the current changes in this repository, then perform a thorough code review.

Review criteria:
1. Bugs / Logic Errors — incorrect logic, missing error handling, edge cases
2. Security — API key exposure, injection, unsafe patterns
3. Performance — unnecessary computation, N+1 problems, memory leaks
4. Code Quality — duplicate code, naming, type safety, separation of concerns

You may read any source files you need for full context.

Output format:
- Organize findings by criterion. Omit sections with no findings.
- Mark severity: CRITICAL, WARNING, SUGGESTION
- Include specific file names and line references.
- At the end, output a summary block in EXACTLY this format:

SCORE: X.X/10
CRITICAL: N
WARNING: N
SUGGESTION: N'
SCRIPT
chmod +x /tmp/codex-review-run.sh
```

4. **Launch Codex.** Check if running inside tmux and branch accordingly:

```bash
echo $TMUX
```

### If inside tmux — run in a separate pane:

```bash
REVIEW_SIGNAL="codex-review-$$"

# Append signal to the runner script
echo 'sleep 10; tmux wait-for -S "'"${REVIEW_SIGNAL}"'"' >> /tmp/codex-review-run.sh

tmux split-window -h -c "$(pwd)" -d "bash /tmp/codex-review-run.sh '$(pwd)' '${REVIEW_OUT}'"
```

Then wait for completion:

```bash
tmux wait-for ${REVIEW_SIGNAL}
cat ${REVIEW_OUT}
```

### If NOT inside tmux — run directly via Bash:

```bash
bash /tmp/codex-review-run.sh "$(pwd)" "${REVIEW_OUT}"
cat ${REVIEW_OUT}
```

Run this Bash call with `run_in_background: true` so Claude doesn't block indefinitely.

5. **Parse the review result.** Extract the SCORE and CRITICAL count from the output file.

6. **Check pass criteria:**
   - SCORE >= 8.0 **AND** CRITICAL == 0 → **PASS** — report the final result.
   - Otherwise → **proceed to fix and re-review.**

7. **Fix issues and re-review (loop):**
   - Fix CRITICAL issues immediately without asking.
   - Fix WARNING issues and report what was changed.
   - For SUGGESTION items involving large structural changes, ask the user before acting.
   - After fixes, run the project's test suite (if available) to verify no regressions.
   - Then go back to step 2 and run the review again.
   - Repeat until pass criteria are met. No round limit.

8. **Cleanup.** After the loop completes:
   - Remove the temp output file: `rm -f ${REVIEW_OUT}`

9. **Final report.** When the review passes, present:
   - Final score and issue counts
   - Summary of all fixes applied across rounds
   - Test results (if tests were run)
