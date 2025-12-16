# Organism Validation Pipeline

This workflow validates organisms after implementation. It's run by the orchestrator (Claude) to verify work before marking complete.

## Overview

```
Agent completes organism
    ↓
┌──────────────────────────────┐
│ Gate 1: Run Tests            │
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ Gate 2: Integration Test     │
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ Gate 3: Merge Detection      │
└──────────────────────────────┘
    ↓ NO CONFLICTS
┌──────────────────────────────┐
│ Gate 4: Regression Sample    │
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ Gate 5: Quality Checks       │
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ ✅ VALIDATION PASSED         │
└──────────────────────────────┘
```

## How to Use

The orchestrator (Claude) runs these validation gates by executing commands and checking results. No jq required - parse text output directly.

**Input:** An organism ID (e.g., `bd-123`)

**Output:**
- PASSED: All gates passed
- FAILED: Gate N failed (with reason)
- CONFLICT: Merge conflict detected

---

## Gate 1: Run Tests

**Goal**: Verify organism-specific tests pass.

1. Get organism details:
   - Run: `bd show <ID>`
   - Parse output for title and type

2. Find test files:
   - Convert title to slug: lowercase, replace spaces with dashes
   - Look for files matching `*<slug>*.test.*` or `*<slug>*.spec.*`

3. Run tests if found:
   - Node project: `npm test -- <test-files>`
   - Python project: `pytest <test-files>`
   - Check exit code

4. Result:
   - Exit 0: PASS
   - Exit non-zero: FAIL - report test output
   - No tests found: PASS (with warning)

---

## Gate 2: Integration Test

**Goal**: Verify organism works with dependencies.

1. Get dependencies:
   - Run: `bd dep list <ID>`
   - If dependencies listed, check each:
     - Run: `bd show <dep-id>` and check for "Status: closed"
     - Warn if dependency not complete

2. Run integration tests if they exist:
   - Check for `tests/integration/` directory
   - Run: `npm run test:integration` if available

3. Result:
   - Tests pass or don't exist: PASS
   - Tests fail: WARN (non-blocking)

---

## Gate 3: Merge Detection

**Goal**: Check for merge conflicts with main branch.

1. Check current branch status:
   - Run: `git status`
   - Ensure working directory is clean

2. Attempt merge check:
   - Run: `git fetch origin main 2>/dev/null || git fetch origin master`
   - Run: `git merge --no-commit --no-ff origin/main 2>&1`
   - Check exit code

3. If conflict:
   - Run: `git merge --abort`
   - Result: CONFLICT
   - List conflicted files

4. If no conflict:
   - Run: `git merge --abort` (we're just checking)
   - Result: PASS

---

## Gate 4: Regression Sample

**Goal**: Test a random completed organism to catch regressions.

1. Get completed organisms:
   - Run: `bd list --status closed --limit 10`
   - Pick one at random (or skip if none)

2. Find and run tests for sampled organism:
   - Same process as Gate 1

3. Result:
   - Tests pass: PASS
   - Tests fail: FAIL - regression detected
   - No tests: PASS (with warning)

---

## Gate 5: Quality Checks

**Goal**: Run linting and type checking.

1. Linting (if available):
   - Run: `npm run lint` or equivalent
   - Non-blocking: warn on failures

2. Type checking (if available):
   - TypeScript: `tsc --noEmit`
   - Python: `mypy .`
   - Non-blocking: warn on failures

3. Result:
   - Always PASS (quality checks are advisory)
   - Report any warnings found

---

## Running Validation

The orchestrator calls validation like this:

1. Tell user: "Validating <ID>..."

2. Run each gate in order:
   - If gate fails: Stop, report failure, return FAILED
   - If gate passes: Continue to next gate

3. All gates passed:
   - Tell user: "Validation PASSED for <ID>"
   - Return PASSED

---

## Exit Codes (for reference)

| Code | Meaning |
|------|---------|
| 0 | All gates passed |
| 1 | Gate 1 failed (tests) |
| 2 | Gate 2 failed (integration) |
| 3 | Gate 3 failed (merge conflict) |
| 4 | Gate 4 failed (regression) |
| 5 | Gate 5 failed (quality) |

---

## Commands Reference

| Need | Command |
|------|---------|
| Get issue details | `bd show <ID>` |
| List dependencies | `bd dep list <ID>` |
| List closed issues | `bd list --status closed` |
| Check git status | `git status` |
| Test merge | `git merge --no-commit --no-ff origin/main` |
| Abort merge | `git merge --abort` |
| Run tests | `npm test` or `pytest` |
| Run lint | `npm run lint` |
| Type check | `tsc --noEmit` |
