---
name: requesting-code-review
description: Use when completing tasks, implementing major features, or before merging to verify work meets requirements - dispatches code-reviewer subagent, runs codex cross-review, handles retries and timeouts, manages review-fix loop until zero issues from both reviewers
user-invocable: false
---

# Requesting Code Review

Two-phase review: code-reviewer subagent validates against the plan, then codex catches deep technical bugs. Both must pass before proceeding.

**Core principle:** Review early, review often. Fix ALL issues before proceeding.

## When to Request Review

**Mandatory:**
- After each task in plan execution
- After completing major feature
- Before merge to main

**Optional but valuable:**
- When stuck (fresh perspective)
- Before refactoring (baseline check)
- After fixing complex bug

## The Review Loop

The review process is a two-phase loop: code-reviewer → codex → fix → re-review → until zero issues from both.

```
┌──────────────────────────────────────────────────┐
│                                                  │
│   Dispatch code-reviewer                         │
│         │                                        │
│         ▼                                        │
│   Issues found? ──Yes──►──┐                      │
│         │                 │                      │
│        No                 │                      │
│         │                 │                      │
│         ▼                 │                      │
│   Run codex cross-review  │                      │
│         │                 │                      │
│         ▼                 │                      │
│   Issues found? ──Yes──►──┤                      │
│         │                 │                      │
│        No                 ▼                      │
│         │        Dispatch bug-fixer              │
│         ▼                 │                      │
│   Done (proceed)          ▼                      │
│              Re-review with prior issues ◄───────┘
│                                                  │
└──────────────────────────────────────────────────┘
```

**Exit condition:** Zero issues from both code-reviewer and codex, or issues accepted per your workflow's policy.

## Step 1: Initial Review

**Get git SHAs:**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # or commit before task
HEAD_SHA=$(git rev-parse HEAD)
```

**Dispatch code-reviewer subagent:**

```
<invoke name="Task">
<parameter name="subagent_type">plan-and-execute:code-reviewer</parameter>
<parameter name="description">Reviewing [what was implemented]</parameter>
<parameter name="prompt">
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [summary of implementation]
  PLAN_OR_REQUIREMENTS: [task/requirements reference]
  BASE_SHA: [commit before work]
  HEAD_SHA: [current commit]
  DESCRIPTION: [brief summary]
</parameter>
</invoke>
```

**Code reviewer returns:** Strengths, Issues (Critical/Important/Minor), Assessment

## Step 2: Handle Reviewer Response

### If Zero Issues
All categories empty → proceed to Step 2a (codex cross-review).

### If Any Issues Found
Regardless of category (Critical, Important, or Minor), dispatch bug-fixer:

```
<invoke name="Task">
<parameter name="subagent_type">plan-and-execute:task-bug-fixer</parameter>
<parameter name="description">Fixing review issues</parameter>
<parameter name="prompt">
  Fix issues from code review.

  Code reviewer found these issues:
  [list all issues - Critical, Important, and Minor]

  Your job is to:
  1. Understand root cause of each issue
  2. Apply fixes systematically (Critical → Important → Minor)
  3. Verify with tests/build/lint
  4. Commit your fixes
  5. Report back with evidence

  Work from: [directory]

  Fix ALL issues — including every Minor issue. The goal is ZERO issues on re-review.
  Minor issues are not optional. Do not skip them.
</parameter>
</invoke>
```

After fixes, proceed to Step 3.

## Step 2a: Codex Cross-Review

After code-reviewer reports zero issues, run codex to catch deep technical bugs.

**Run codex review:**
```bash
BASE_SHA=<last commit that passed review>  # e.g. base of current phase
codex -c 'model="gpt-5.3-codex"' -c 'model_reasoning_effort="xhigh"' \
  exec review --json --base $BASE_SHA \
  | tee /tmp/codex.stdout \
  | jq -r 'select(.type=="item.completed" and .item.type=="agent_message") | .item.text'
```

The `jq` filter extracts the reviewer's findings from codex's JSON output stream. Raw output is preserved in `/tmp/codex.stdout` for debugging.

**If codex reports no issues:** Done — proceed to next task.

**If codex reports issues:** Treat identically to code-reviewer issues. Dispatch bug-fixer:

```
<invoke name="Task">
<parameter name="subagent_type">plan-and-execute:task-bug-fixer</parameter>
<parameter name="description">Fixing codex review issues</parameter>
<parameter name="prompt">
  Fix these issues found during code review:

  [paste codex output]

  Your job is to:
  1. Understand root cause of each issue
  2. Apply fixes systematically
  3. Verify with tests/build/lint
  4. Commit your fixes
  5. Report back with evidence

  Work from: [directory]

  Fix ALL issues. The goal is ZERO issues on re-review.
</parameter>
</invoke>
```

After fixes, proceed to Step 3. Re-review restarts the full loop from Step 1 (code-reviewer must pass again before codex re-runs).

## Step 3: Re-Review After Fixes

**CRITICAL:** Track prior issues across review cycles.

```
<invoke name="Task">
<parameter name="subagent_type">plan-and-execute:code-reviewer</parameter>
<parameter name="description">Re-reviewing after fixes (cycle N)</parameter>
<parameter name="prompt">
  Use template at requesting-code-review/code-reviewer.md

  WHAT_WAS_IMPLEMENTED: [from bug-fixer's report]
  PLAN_OR_REQUIREMENTS: [original task/requirements]
  BASE_SHA: [commit before this fix cycle]
  HEAD_SHA: [current commit after fixes]
  DESCRIPTION: Re-review after bug fixes (review cycle N)

  PRIOR_ISSUES_TO_VERIFY_FIXED:
  [list all outstanding issues from previous reviews]

  Verify:
  1. Each prior issue listed above is actually resolved
  2. No regressions introduced by the fixes
  3. Any new issues in the changed code

  Report which prior issues are now fixed and which (if any) remain.
</parameter>
</invoke>
```

**Tracking prior issues:**
- When re-reviewer explicitly confirms fixed → remove from list
- When re-reviewer doesn't mention an issue → keep on list (silence ≠ fixed)
- When re-reviewer finds new issues → add to list

Loop back to Step 2 if any issues remain. Step 2's zero-issues path includes codex cross-review (Step 2a), so the full two-phase cycle runs again.

## Handling Failures

### Operational Errors
If reviewer reports operational errors (can't run tests, missing scripts):
1. **STOP** - do not continue
2. Report to human
3. When told to continue, re-execute same review

### Timeouts / Empty Response
Usually means context limits. Retry with focused scope:

**First retry:** Narrow to changed files only:
```
FOCUSED REVIEW - Context was too large.

Review ONLY the diff between BASE_SHA and HEAD_SHA.
Focus on: [list only files actually modified]

Skip: broad architectural analysis, unchanged files, tangential concerns.

WHAT_WAS_IMPLEMENTED: [summary]
PLAN_OR_REQUIREMENTS: [reference]
BASE_SHA: [sha]
HEAD_SHA: [sha]
```

**Second retry:** Split into multiple smaller reviews (one per file or logical group).

**Third failure:** Stop and ask human for help.

### Codex Command Failures
If the codex command fails (not installed, network issues, unexpected output format):
1. Check `/tmp/codex.stdout` for raw output
2. Verify codex is installed and accessible
3. Retry the command once
4. If still failing, report to human — do not skip the cross-review

## Quick Reference

| Situation | Action |
|-----------|--------|
| Code-reviewer zero issues | Proceed to codex cross-review |
| Codex zero issues | Done (proceed to next task) |
| Any issues (either reviewer) | Fix, re-review from Step 1 |
| Operational error | Stop, report, wait |
| Codex command failure | Check installation, retry, ask human |
| Timeout | Retry with focused scope |
| 3 failed retries | Ask human |

## Red Flags

**Never:**
- Skip review because "it's simple"
- Skip codex cross-review because "code-reviewer already passed"
- Proceed with ANY unfixed issues (Critical, Important, OR Minor)
- Argue with valid technical feedback without evidence
- Rationalize skipping Minor issues ("they're just style", "we can fix later")

**Minor issues are NOT optional.** The code reviewer flagged them for a reason. Fix all of them. "Minor" means lower severity, not "ignorable."

**If reviewer wrong:**
- Push back with technical reasoning
- Show code/tests that prove it works
- Request clarification on unclear feedback

## Integration

**Called by:**
- executing-an-implementation-plan (after each task)
- finishing-a-development-branch (final review)
- Ad-hoc when you need a review

**Template location:** requesting-code-review/code-reviewer.md
