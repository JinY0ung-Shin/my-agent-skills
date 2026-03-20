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

5. **Parse the review result.** Extract the SCORE, CRITICAL count, and each individual finding from the output file.

6. **Check pass criteria:**
   - SCORE >= 8.0 **AND** CRITICAL == 0 → **PASS** — report the final result.
   - Otherwise → **proceed to step 7 (능동적 판단 및 대응).**

7. **능동적 판단 — 각 피드백을 분류하고 대응 결정:**

   리뷰 결과를 무조건 수용하지 않는다. 각 finding에 대해 다음 기준으로 **독립적으로 판단**한다:

   ### 판단 기준
   | 분류 | 조건 | 대응 |
   |------|------|------|
   | **수용 (Accept)** | 지적이 명확히 타당하고 코드에 실제 문제가 있음 | 즉시 수정 |
   | **부분 수용 (Partial)** | 지적의 방향은 맞지만 제안된 해결책이 부적절하거나 과도함 | 더 나은 방식으로 수정하고, 왜 다르게 수정했는지 반론 작성 |
   | **반론 (Disagree)** | 지적이 오해에 기반하거나, 컨텍스트를 무시하거나, 의도된 설계를 모름 | 수정하지 않고 반론 근거를 작성 |
   | **보류 (Defer)** | 타당하지만 현재 범위를 벗어나거나 대규모 구조 변경이 필요 | 사용자에게 확인 후 결정 |

   ### 판단 시 고려사항
   - **코드의 의도와 컨텍스트를 우선 파악한다.** 리뷰어가 코드의 목적이나 제약조건을 이해하지 못했을 수 있다.
   - **"더 좋은 방법이 있다"는 SUGGESTION은 현재 방식에 실제 문제가 없다면 반론할 수 있다.** 취향 차이는 수정 사유가 아니다.
   - **CRITICAL로 분류되었더라도 실제로 critical이 아닐 수 있다.** 심각도를 독자적으로 재평가한다.
   - **WARNING이라도 실제 버그 가능성이 높으면 CRITICAL로 격상하여 즉시 수정한다.**
   - **성능 지적은 실측 데이터 없이 추측에 기반한 경우 반론 대상이다.**

8. **반론 메시지 작성 및 재리뷰 요청:**

   수정/반론 결과를 정리한 후, 재리뷰 요청 시 Codex에게 다음을 포함하여 전달한다:

   ```bash
   cat > /tmp/codex-review-run.sh << 'SCRIPT'
   #!/bin/bash
   cd "$1"
   codex exec --dangerously-bypass-approvals-and-sandbox -o "$2" 'You are a senior code reviewer conducting a RE-REVIEW. Run `'"${DIFF_CMD}"'` to see the current changes.

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
   SUGGESTION: N'
   SCRIPT
   chmod +x /tmp/codex-review-run.sh
   ```

   `REVIEW_RESPONSE`는 step 7에서 작성한 각 finding에 대한 대응 내역이다. 형식:

   ```
   ## Finding: [원본 finding 요약]
   - Decision: Accept / Partial / Disagree / Defer
   - Action: [수정한 내용 또는 "No change"]
   - Reasoning: [판단 근거 — 반론인 경우 구체적 근거 포함]
   ```

9. **재리뷰 결과 재판단 (반복):**
   - 재리뷰 결과에서 다시 step 7의 판단 프로세스를 적용한다.
   - 리뷰어가 반론을 수용했으면 해당 항목은 종결.
   - 리뷰어가 새로운 근거와 함께 재지적하면, 그 근거를 검토하여 다시 수용/반론을 결정한다.
   - **동일한 지적이 새로운 근거 없이 3회 반복되면, 해당 항목은 "의견 차이"로 기록하고 사용자에게 최종 판단을 요청한다.**
   - 재리뷰 후에도 pass criteria 미충족 시 step 7부터 반복.
   - Repeat until pass criteria are met. No round limit.

10. **Cleanup.** After the loop completes:
   - Remove the temp output file: `rm -f ${REVIEW_OUT}`

11. **Final report.** When the review passes, present:
   - Final score and issue counts
   - Summary of all fixes applied across rounds
   - Test results (if tests were run)
