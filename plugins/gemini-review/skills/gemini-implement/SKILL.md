---
name: gemini-implement
description: Delegate implementation to Gemini CLI with structured planning, execution, and iterative quality review
user_invocable: true
---

# Gemini Implementation

Delegate implementation to Gemini CLI. Claude creates a detailed plan, Gemini executes it, then the result is reviewed and refined until quality passes.

## Key Principle

**Claude plans, Gemini executes.** Claude's strength is analyzing context and designing solutions. Gemini's strength is executing precise instructions at scale. The plan file is the contract between them.

Gemini runs in the **same working directory** as this project. It can read any files and modify them directly. The plan only needs to specify what to implement — Gemini handles the file operations.

## Steps

### Phase 1: Plan

1. **Parse task input.**

   - If an argument is provided, use it as the task description: `TASK_DESC="<argument>"`
   - If no argument is provided, ask the user what they want to implement.

2. **Analyze the codebase.** Read relevant source files to understand:
   - Project structure and architecture patterns
   - Code conventions (naming, style, error handling)
   - Existing tests and their structure
   - Dependencies and configuration

   Focus only on files relevant to the task. Do not try to read the entire codebase.

3. **Create implementation plan.**

```bash
PLAN_FILE=$(mktemp /tmp/gemini-impl-plan-XXXXXX.md)
```

   Write a plan to `PLAN_FILE` with this structure:

   ```markdown
   # Implementation Plan

   ## Task
   [What to implement, why, and acceptance criteria]

   ## Files to Modify
   - `exact/path/to/file.ts` — [specific changes: what to add, remove, or change]

   ## Files to Create
   - `exact/path/to/new-file.ts` — [purpose and key contents]

   ## Step-by-step Instructions
   1. [Specific instruction with exact file path and what to change]
   2. [Next instruction...]

   ## Code Conventions
   - [Concrete patterns observed in the codebase, with file:line references as examples]

   ## Constraints
   - [What NOT to do — scope limits, anti-patterns to avoid]

   ## Verification
   - [Exact commands to run to verify correctness]
   ```

   **The plan must be specific enough that Gemini can execute without guessing.** Include exact file paths, function signatures, and reference existing code as examples of patterns to follow.

   For large tasks, break the plan into sequential phases. Each phase should be independently implementable. Mark phases clearly:

   ```markdown
   ## Phase 1: [description]
   ### Steps
   1. ...

   ## Phase 2: [description]
   ### Steps
   1. ...
   ```

4. **Show the plan to the user** and get confirmation before proceeding. The user may want to adjust scope, reorder steps, or add constraints.

### Phase 2: Execute

5. **Create a safety branch.**

```bash
ORIGINAL_BRANCH=$(git branch --show-current)
BRANCH_NAME="gemini/implement-$(date +%Y%m%d-%H%M%S)"
git stash --include-untracked -q 2>/dev/null; STASHED=$?
git checkout -b "${BRANCH_NAME}"
[ "$STASHED" -eq 0 ] && git stash pop -q
```

6. **Prepare output file.**

```bash
IMPL_OUT=$(mktemp /tmp/gemini-impl-XXXXXX.txt)
```

7. **Write runner script.**

```bash
cat > /tmp/gemini-impl-run.sh << 'SCRIPT'
#!/bin/bash
cd "$1"
GEMINI_SANDBOX=false gemini -p 'You are a senior software engineer executing an implementation plan.

Read the plan at `'"${PLAN_FILE}"'` and implement it precisely.

Rules:
1. Read the plan completely before writing any code.
2. Read existing source files referenced in the plan to understand current patterns.
3. Implement each step in order.
4. Match existing code style exactly — indentation, naming conventions, error handling patterns.
5. Do not add anything not specified in the plan (no extra comments, docstrings, or improvements).
6. If a step is unclear, implement your best interpretation and note the deviation.
7. After implementing, run any verification commands specified in the plan.

Output a summary in EXACTLY this format:

FILE_CHANGES:
- [path]: [created|modified] — [brief description]

DEVIATIONS:
- [any deviations from the plan and why, or "None"]

BLOCKERS:
- [any steps that could not be completed and why, or "None"]

VERIFICATION:
- [results of verification commands, or "No verification commands specified"]

STATUS: [COMPLETE|PARTIAL]' --yolo | tee "$2"
SCRIPT
chmod +x /tmp/gemini-impl-run.sh
```

   If the plan has multiple phases, repeat steps 7-8 for each phase. Update the Gemini prompt to specify which phase to execute: `'Execute Phase N of the plan at ...'`

8. **Launch Gemini.** Check if running inside tmux and branch accordingly:

```bash
echo $TMUX
```

#### If inside tmux — run in a separate pane:

```bash
IMPL_SIGNAL="gemini-impl-$$"

# Append signal to the runner script
echo 'sleep 10; tmux wait-for -S "'"${IMPL_SIGNAL}"'"' >> /tmp/gemini-impl-run.sh

tmux split-window -h -c "$(pwd)" -d "bash /tmp/gemini-impl-run.sh '$(pwd)' '${IMPL_OUT}'"
```

Then wait for completion:

```bash
tmux wait-for ${IMPL_SIGNAL}
cat ${IMPL_OUT}
```

#### If NOT inside tmux — run directly via Bash:

```bash
bash /tmp/gemini-impl-run.sh "$(pwd)" "${IMPL_OUT}"
cat ${IMPL_OUT}
```

Run this Bash call with `run_in_background: true` so Claude doesn't block indefinitely.

### Phase 3: Verify and Refine

9. **Check implementation status.** Read `${IMPL_OUT}` and parse:
   - STATUS: COMPLETE → proceed to step 10.
   - STATUS: PARTIAL or BLOCKERS present → report to user and decide:
     - Provide additional context and retry blocked steps
     - Skip blocked steps and review what was completed
     - Abort: switch back to `${ORIGINAL_BRANCH}` and delete the branch

10. **Review changes.** Examine what Gemini actually did:

```bash
git diff
git status
```

   Compare against the plan:
   - Were all planned steps completed?
   - Are the changes consistent with the plan?
   - Do the changes look correct at a glance?

   If there are obvious issues that don't require a full review (e.g., missed a step entirely), fix them directly before proceeding.

11. **Run quality review.** Invoke `/gemini-review` to get an independent code review of the implementation. Follow the full gemini-review process including active judgment and iteration.

12. **Handle review failures.** If gemini-review identifies accepted issues that need code changes:

   Compile the findings classified as Accept or Partial into fix instructions (`FIX_INSTRUCTIONS`), then run Gemini:

   ```bash
   cat > /tmp/gemini-impl-run.sh << 'SCRIPT'
   #!/bin/bash
   cd "$1"
   GEMINI_SANDBOX=false gemini -p 'You are fixing code review issues in this repository.

   ## Issues to Fix
   '"${FIX_INSTRUCTIONS}"'

   Rules:
   1. Fix only what is listed above.
   2. Do not refactor or modify unrelated code.
   3. Match existing code style.

   Output:
   FIXES_APPLIED:
   - [file]: [what was fixed]

   STATUS: [COMPLETE|PARTIAL]' --yolo | tee "$2"
   SCRIPT
   chmod +x /tmp/gemini-impl-run.sh
   ```

   `FIX_INSTRUCTIONS` contains each accepted finding with specific fix guidance. Format:

   ```
   ### Issue: [finding summary]
   - Severity: [CRITICAL|WARNING]
   - File: [path]
   - Fix: [specific instruction on what to change]
   ```

   Launch using the same tmux/non-tmux pattern from step 8. After fixes, re-run `/gemini-review`. Repeat until review passes (score >= 8.0, 0 critical issues).

13. **Run verification.** If the plan specified verification commands (tests, build, lint), run them now.

   If verification fails, compile error details into `FIX_INSTRUCTIONS` and run Gemini again using the same pattern as step 12. Re-verify after fixes. Repeat until verification passes.

### Phase 4: Complete

14. **Present results to user.** Show:
    - `git diff --stat ${ORIGINAL_BRANCH}...HEAD` — summary of all changes
    - Final review score and issue counts
    - Verification/test results
    - Branch name: `${BRANCH_NAME}`

15. **Ask user for decision:**
    - **Merge**: merge `${BRANCH_NAME}` into `${ORIGINAL_BRANCH}` and delete the implementation branch
    - **Keep**: leave changes on `${BRANCH_NAME}` for further review
    - **Discard**: switch back to `${ORIGINAL_BRANCH}` and delete `${BRANCH_NAME}`

16. **Cleanup.**
    - Remove temp files: `rm -f ${IMPL_OUT} ${PLAN_FILE} /tmp/gemini-impl-run.sh`
    - Execute the user's branch decision from step 15
