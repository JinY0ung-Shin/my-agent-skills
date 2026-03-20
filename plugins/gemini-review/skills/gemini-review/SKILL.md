---
name: gemini-review
description: Get iterative code review from Gemini CLI until score >= 8.0 with 0 critical issues (tmux pane monitoring)
user_invocable: true
---

# Gemini Code Review Loop

Run Gemini to review code changes. If inside tmux, opens a separate pane for real-time monitoring.
Iterate — fix issues and re-request review until score >= 8.0 with 0 critical issues.

## Key Principle

Gemini runs in the **same working directory** as this project. There is no need to pass `git diff` output as an argument. Gemini can run `git diff` itself and freely read any source files it needs. The prompt only needs to tell Gemini which diff command to run.

## Steps

1. **Detect review target.** Check for uncommitted changes:

```bash
git diff --quiet HEAD 2>/dev/null; echo $?
```

- Exit code `1` → uncommitted changes exist → `DIFF_CMD="git diff HEAD"`
- Exit code `0` → no uncommitted changes → `DIFF_CMD="git diff HEAD~1"` (review the last commit)

2. **Prepare output file.**

```bash
REVIEW_OUT=$(mktemp /tmp/gemini-review-XXXXXX.txt)
```

3. **Write runner script.**

```bash
cat > /tmp/gemini-review-run.sh << 'SCRIPT'
#!/bin/bash
cd "$1"
GEMINI_SANDBOX=false gemini -p 'You are a senior code reviewer. Run `'"${DIFF_CMD}"'` to see the current changes in this repository, then perform a thorough code review.

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
SUGGESTION: N' --yolo | tee "$2"
SCRIPT
chmod +x /tmp/gemini-review-run.sh
```

4. **Launch Gemini.** Check if running inside tmux and branch accordingly:

```bash
echo $TMUX
```

### If inside tmux — run in a separate pane:

```bash
REVIEW_SIGNAL="gemini-review-$$"

# Append signal to the runner script
echo 'sleep 10; tmux wait-for -S "'"${REVIEW_SIGNAL}"'"' >> /tmp/gemini-review-run.sh

PANE_COUNT=$(tmux list-panes | wc -l)
if [ "$PANE_COUNT" -gt 1 ]; then
  LAST_PANE=$(tmux list-panes -F '#{pane_index}' | sort -n | tail -1)
  NEW_PANE=$(tmux split-window -v -t "$LAST_PANE" -c "$(pwd)" -d -P -F '#{pane_id}' "bash /tmp/gemini-review-run.sh '$(pwd)' '${REVIEW_OUT}'")
else
  NEW_PANE=$(tmux split-window -h -t 0 -c "$(pwd)" -d -P -F '#{pane_id}' "bash /tmp/gemini-review-run.sh '$(pwd)' '${REVIEW_OUT}'")
fi
tmux select-pane -t "$NEW_PANE" -T "Gemini Review"
```

Then wait for completion:

```bash
tmux wait-for ${REVIEW_SIGNAL}
cat ${REVIEW_OUT}
```

### If NOT inside tmux — run directly via Bash:

```bash
bash /tmp/gemini-review-run.sh "$(pwd)" "${REVIEW_OUT}"
cat ${REVIEW_OUT}
```

Run this Bash call with `run_in_background: true` so Claude doesn't block indefinitely.

5. **Parse the review result.** Extract the SCORE, CRITICAL count, and each individual finding from the output file.

6. **Check pass criteria:**
   - SCORE >= 8.0 **AND** CRITICAL == 0 → **PASS** — report the final result.
   - Otherwise → **proceed to step 7 (active judgment and response).**

7. **Active judgment — classify and decide on each finding:**

   Do not blindly accept all review findings. Evaluate each finding **independently** using the following criteria:

   ### Decision Criteria
   | Classification | Condition | Action |
   |----------------|-----------|--------|
   | **Accept** | The finding is clearly valid and there is a real problem in the code | Fix immediately |
   | **Partial** | The direction of the finding is correct but the suggested fix is inappropriate or excessive | Fix in a better way and write a rebuttal explaining why a different approach was taken |
   | **Disagree** | The finding is based on a misunderstanding, ignores context, or is unaware of intentional design | Do not change the code; write a rebuttal with supporting rationale |
   | **Defer** | Valid but out of scope or requires large-scale structural changes | Ask the user before deciding |

   ### Considerations
   - **Understand the intent and context of the code first.** The reviewer may not understand the purpose or constraints of the code.
   - **A SUGGESTION that "there's a better way" can be rebutted if the current approach has no actual problem.** Style preference is not a valid reason to change code.
   - **A finding marked CRITICAL may not actually be critical.** Re-evaluate severity independently.
   - **A WARNING with high actual bug probability should be escalated to CRITICAL and fixed immediately.**
   - **Performance findings based on speculation without measured data are candidates for rebuttal.**

8. **Write rebuttal and request re-review:**

   After compiling fixes and rebuttals, send the following to Gemini for re-review:

   ```bash
   cat > /tmp/gemini-review-run.sh << 'SCRIPT'
   #!/bin/bash
   cd "$1"
   GEMINI_SANDBOX=false gemini -p 'You are a senior code reviewer conducting a RE-REVIEW. Run `'"${DIFF_CMD}"'` to see the current changes.

   ## Previous Review Context
   The developer has addressed your previous review. They made the following responses:

   '"${REVIEW_RESPONSE}"'

   ## Your Task
   1. Verify that accepted fixes are correctly implemented.
   2. For items the developer disagreed with, READ their reasoning carefully:
      - If their argument is valid and well-supported, ACCEPT their position and remove the finding.
      - If you still believe the issue is real, provide ADDITIONAL EVIDENCE or a CONCRETE EXAMPLE of how it could fail. Do not simply repeat the same point.
      - If you realize your original assessment was wrong, explicitly acknowledge it.
   3. Check if the fixes introduced any NEW issues.
   4. Do NOT re-raise findings that were already resolved or where you accepted the developer rebuttal.

   Review criteria:
   1. Bugs / Logic Errors — incorrect logic, missing error handling, edge cases
   2. Security — API key exposure, injection, unsafe patterns
   3. Performance — unnecessary computation, N+1 problems, memory leaks
   4. Code Quality — duplicate code, naming, type safety, separation of concerns

   Output format:
   - Organize findings by criterion. Omit sections with no findings.
   - For each finding, note if it is NEW, UNRESOLVED (with new evidence), or REOPENED.
   - Mark severity: CRITICAL, WARNING, SUGGESTION
   - Include specific file names and line references.
   - At the end, output a summary block in EXACTLY this format:

   SCORE: X.X/10
   CRITICAL: N
   WARNING: N
   SUGGESTION: N' --yolo | tee "$2"
   SCRIPT
   chmod +x /tmp/gemini-review-run.sh
   ```

   `REVIEW_RESPONSE` contains the response for each finding from step 7. Format:

   ```
   ## Finding: [original finding summary]
   - Decision: Accept / Partial / Disagree / Defer
   - Action: [what was fixed, or "No change"]
   - Reasoning: [rationale — include specific supporting evidence for rebuttals]
   ```

9. **Re-evaluate re-review results (iterate):**
   - Apply the same judgment process from step 7 to the re-review results.
   - If the reviewer accepted a rebuttal, that item is closed.
   - If the reviewer re-raised a finding with new evidence, review that evidence and decide to accept or rebut again.
   - **If the same finding is repeated 3 times without new evidence, mark it as "difference of opinion" and ask the user for a final decision.**
   - If pass criteria are still not met after re-review, repeat from step 7.
   - Repeat until pass criteria are met. No round limit.

10. **Cleanup.** After the loop completes:
   - Remove the temp output file: `rm -f ${REVIEW_OUT}`

11. **Final report.** When the review passes, present:
   - Final score and issue counts
   - Summary of all fixes applied across rounds
   - Test results (if tests were run)
