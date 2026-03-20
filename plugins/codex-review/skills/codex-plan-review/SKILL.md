---
name: codex-plan-review
description: Get iterative plan/design review from Codex CLI until score >= 8.0 with 0 critical issues (tmux pane monitoring)
user_invocable: true
---

# Codex Plan Review Loop

Run Codex to review an implementation plan or design document. If inside tmux, opens a separate pane for real-time monitoring.
Iterate — fix issues and re-request review until score >= 8.0 with 0 critical issues.

## Key Principle

Codex runs in the **same working directory** as this project. It can freely read any files it needs. The prompt only needs to tell Codex which plan file to review.

## Steps

1. **Detect review target.** Determine the plan to review:

   - If a plan file path was provided as an argument, use that.
   - Otherwise, search for common plan files:

```bash
ls -1 PLAN.md plan.md DESIGN.md design.md ARCHITECTURE.md architecture.md RFC.md rfc.md 2>/dev/null | head -1
```

   - If no file is found, check if Claude Code is currently in plan mode and has an active plan. If so, export the plan to a temp file.
   - If still no plan is found, ask the user to specify the plan file path.

   Store the result: `PLAN_FILE="<path>"`

2. **Prepare output file.**

```bash
REVIEW_OUT=$(mktemp /tmp/codex-plan-review-XXXXXX.txt)
```

3. **Write runner script.**

```bash
cat > /tmp/codex-plan-review-run.sh << 'SCRIPT'
#!/bin/bash
cd "$1"
codex exec --dangerously-bypass-approvals-and-sandbox -o "$2" 'You are a senior software architect and technical reviewer. Read the plan file at `'"${PLAN_FILE}"'` and perform a thorough review of the implementation plan.

You may also read any source files in the repository to understand the current codebase context.

Review criteria:
1. Feasibility — Are the proposed steps technically feasible? Are there unrealistic assumptions, missing prerequisites, or incompatible dependencies?
2. Completeness — Are there missing steps, unaddressed edge cases, overlooked failure modes, or gaps in the rollback strategy?
3. Risk & Impact — Are risks properly identified and mitigated? Is the blast radius understood? Are there unintended side effects on existing functionality?
4. Clarity & Correctness — Is the plan unambiguous and actionable? Could an engineer follow it without guessing? Are the technical details accurate?

Output format:
- Organize findings by criterion. Omit sections with no findings.
- Mark severity: CRITICAL, WARNING, SUGGESTION
- Include specific references to the plan sections and, where relevant, existing source files.
- For each finding, explain WHY it is a problem and provide a CONCRETE recommendation.
- At the end, output a summary block in EXACTLY this format:

SCORE: X.X/10
CRITICAL: N
WARNING: N
SUGGESTION: N'
SCRIPT
chmod +x /tmp/codex-plan-review-run.sh
```

4. **Launch Codex.** Check if running inside tmux and branch accordingly:

```bash
echo $TMUX
```

### If inside tmux — run in a separate pane:

```bash
REVIEW_SIGNAL="codex-plan-review-$$"

# Append signal to the runner script
echo 'sleep 10; tmux wait-for -S "'"${REVIEW_SIGNAL}"'"' >> /tmp/codex-plan-review-run.sh

PANE_COUNT=$(tmux list-panes | wc -l)
if [ "$PANE_COUNT" -gt 1 ]; then
  LAST_PANE=$(tmux list-panes -F '#{pane_index}' | sort -n | tail -1)
  NEW_PANE=$(tmux split-window -v -t "$LAST_PANE" -c "$(pwd)" -d -P -F '#{pane_id}' "bash /tmp/codex-plan-review-run.sh '$(pwd)' '${REVIEW_OUT}'")
else
  NEW_PANE=$(tmux split-window -h -t 0 -c "$(pwd)" -d -P -F '#{pane_id}' "bash /tmp/codex-plan-review-run.sh '$(pwd)' '${REVIEW_OUT}'")
fi
tmux select-pane -t "$NEW_PANE" -T "Codex Plan Review"
```

Then wait for completion:

```bash
tmux wait-for ${REVIEW_SIGNAL}
cat ${REVIEW_OUT}
```

### If NOT inside tmux — run directly via Bash:

```bash
bash /tmp/codex-plan-review-run.sh "$(pwd)" "${REVIEW_OUT}"
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
   | **Accept** | The finding is clearly valid and there is a real gap or flaw in the plan | Revise the plan immediately |
   | **Partial** | The direction of the finding is correct but the suggested revision is inappropriate or excessive | Revise in a better way and write a rebuttal explaining why a different approach was taken |
   | **Disagree** | The finding is based on a misunderstanding, ignores project constraints, or is unaware of intentional design decisions | Do not change the plan; write a rebuttal with supporting rationale |
   | **Defer** | Valid but out of scope, requires stakeholder input, or depends on information not yet available | Ask the user before deciding |

   ### Considerations
   - **Understand the project context and constraints first.** The reviewer may not know about team capabilities, timeline constraints, or business requirements.
   - **A SUGGESTION for "a better architecture" can be rebutted if the current approach is sound and practical.** Theoretical perfection is not a valid reason to change a working plan.
   - **A finding marked CRITICAL may not actually be critical.** Re-evaluate severity based on the actual risk to the project.
   - **A WARNING about a missing step that is actually covered elsewhere in the plan should be rebutted.**
   - **Feasibility concerns based on speculation without evidence of actual technical limitations are candidates for rebuttal.**
   - **"You should also consider X" is not a finding unless omitting X leads to a concrete failure mode.**

8. **Revise the plan and request re-review:**

   After compiling revisions and rebuttals, update the plan file with the accepted changes, then send the following to Codex for re-review:

   ```bash
   cat > /tmp/codex-plan-review-run.sh << 'SCRIPT'
   #!/bin/bash
   cd "$1"
   codex exec --dangerously-bypass-approvals-and-sandbox -o "$2" 'You are a senior software architect conducting a RE-REVIEW. Read the updated plan file at `'"${PLAN_FILE}"'` and review the changes.

   ## Previous Review Context
   The author has addressed your previous review. They made the following responses:

   '"${REVIEW_RESPONSE}"'

   ## Your Task
   1. Verify that accepted revisions are correctly incorporated into the plan.
   2. For items the author disagreed with, READ their reasoning carefully:
      - If their argument is valid and well-supported, ACCEPT their position and remove the finding.
      - If you still believe the issue is real, provide ADDITIONAL EVIDENCE or a CONCRETE SCENARIO of how the plan could fail. Do not simply repeat the same point.
      - If you realize your original assessment was wrong, explicitly acknowledge it.
   3. Check if the revisions introduced any NEW gaps or issues.
   4. Do NOT re-raise findings that were already resolved or where you accepted the author rebuttal.

   Review criteria:
   1. Feasibility — Are the proposed steps technically feasible? Are there unrealistic assumptions, missing prerequisites, or incompatible dependencies?
   2. Completeness — Are there missing steps, unaddressed edge cases, overlooked failure modes, or gaps in the rollback strategy?
   3. Risk & Impact — Are risks properly identified and mitigated? Is the blast radius understood? Are there unintended side effects on existing functionality?
   4. Clarity & Correctness — Is the plan unambiguous and actionable? Could an engineer follow it without guessing? Are the technical details accurate?

   Output format:
   - Organize findings by criterion. Omit sections with no findings.
   - For each finding, note if it is NEW, UNRESOLVED (with new evidence), or REOPENED.
   - Mark severity: CRITICAL, WARNING, SUGGESTION
   - Include specific references to the plan sections.
   - At the end, output a summary block in EXACTLY this format:

   SCORE: X.X/10
   CRITICAL: N
   WARNING: N
   SUGGESTION: N'
   SCRIPT
   chmod +x /tmp/codex-plan-review-run.sh
   ```

   `REVIEW_RESPONSE` contains the response for each finding from step 7. Format:

   ```
   ## Finding: [original finding summary]
   - Decision: Accept / Partial / Disagree / Defer
   - Action: [what was revised in the plan, or "No change"]
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
   - Summary of all plan revisions applied across rounds
   - List of rebuttals that were accepted by the reviewer (for documentation)
   - Any deferred items that need future attention
