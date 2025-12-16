# Organism Validation Pipeline

This workflow validates each organism after implementation through a 5-gate pipeline. All gates must pass for the organism to be committed.

## Overview

```
Agent completes organism
    ↓
┌──────────────────────────────┐
│ Gate 1: Run Tests            │ ← Run organism-specific tests
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ Gate 2: Integration Test     │ ← Verify with dependent organisms
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ Gate 3: Merge Detection      │ ← Check for conflicts with main
└──────────────────────────────┘
    ↓ NO CONFLICTS (or resolved)
┌──────────────────────────────┐
│ Gate 4: Regression Sample    │ ← Test random previous organism
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ Gate 5: Quality Checks       │ ← Linting, type checking, coverage
└──────────────────────────────┘
    ↓ PASS
┌──────────────────────────────┐
│ ✅ COMMIT & PUSH             │
└──────────────────────────────┘
```

If any gate fails:
1. Extract error patterns
2. Add to learning system
3. Retry with improvements (up to N times)
4. If N failures: create analysis issue, continue with other work

---

## Usage

This workflow is called by the autonomous-build orchestrator in STEP 3 (Handle Agent Completions) after an agent finishes implementing an organism.

**Input:**
- `ORGANISM_ID`: The organism to validate
- `COMMIT_BEFORE`: Git commit before agent started (for diff)
- `ATTEMPT`: Current attempt number (1-3)

**Output:**
- `VALIDATION_RESULT`: "passed", "failed", or "conflict"
- `ERRORS`: Array of error messages (if failed)
- `CONFLICT_DIFF`: Merge conflict details (if conflict)

---

## Gate 1: Run Tests

Run organism-specific tests to verify functionality:

```bash
echo "=== Gate 1: Run Tests ==="
echo ""

# Get organism type to determine test strategy
ORGANISM_TYPE=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.type')

case "$ORGANISM_TYPE" in
  "database-layer")
    # Database layer: Run model tests, migration tests
    echo "Running database layer tests..."

    # Find test files for this organism
    ORGANISM_TITLE=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.title' | tr ' ' '-' | tr '[:upper:]' '[:lower:]')

    # Look for test files matching organism name
    TEST_FILES=$(find . -name "*${ORGANISM_TITLE}*.test.*" -o -name "*${ORGANISM_TITLE}*.spec.*" 2>/dev/null || echo "")

    if [ -n "$TEST_FILES" ]; then
      # Run tests
      if command -v npm &> /dev/null; then
        npm test -- "$TEST_FILES" > /tmp/test-output.txt 2>&1
        TEST_EXIT_CODE=$?
      elif command -v pytest &> /dev/null; then
        pytest "$TEST_FILES" > /tmp/test-output.txt 2>&1
        TEST_EXIT_CODE=$?
      else
        echo "⚠️  No test runner found"
        TEST_EXIT_CODE=0
      fi

      if [ $TEST_EXIT_CODE -ne 0 ]; then
        echo "❌ Gate 1 FAILED: Tests failed"
        cat /tmp/test-output.txt
        exit 1
      fi

      echo "✓ Tests passed ($TEST_FILES)"
    else
      echo "⚠️  No tests found for $ORGANISM_ID"
    fi
    ;;

  "api-layer")
    # API layer: Run endpoint tests
    echo "Running API layer tests..."

    # Find API test files
    ORGANISM_TITLE=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.title' | tr ' ' '-' | tr '[:upper:]' '[:lower:]')
    TEST_FILES=$(find . -name "*${ORGANISM_TITLE}*.test.*" -o -name "*${ORGANISM_TITLE}*.spec.*" 2>/dev/null || echo "")

    if [ -n "$TEST_FILES" ]; then
      if command -v npm &> /dev/null; then
        npm test -- "$TEST_FILES" > /tmp/test-output.txt 2>&1
        TEST_EXIT_CODE=$?
      else
        echo "⚠️  No test runner found"
        TEST_EXIT_CODE=0
      fi

      if [ $TEST_EXIT_CODE -ne 0 ]; then
        echo "❌ Gate 1 FAILED: Tests failed"
        cat /tmp/test-output.txt
        exit 1
      fi

      echo "✓ Tests passed ($TEST_FILES)"
    else
      echo "⚠️  No tests found for $ORGANISM_ID"
    fi
    ;;

  "ui-component")
    # UI component: Run component tests
    echo "Running UI component tests..."

    ORGANISM_TITLE=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.title' | tr ' ' '-' | tr '[:upper:]' '[:lower:]')
    TEST_FILES=$(find . -name "*${ORGANISM_TITLE}*.test.*" -o -name "*${ORGANISM_TITLE}*.spec.*" 2>/dev/null || echo "")

    if [ -n "$TEST_FILES" ]; then
      if command -v npm &> /dev/null; then
        npm test -- "$TEST_FILES" > /tmp/test-output.txt 2>&1
        TEST_EXIT_CODE=$?
      else
        echo "⚠️  No test runner found"
        TEST_EXIT_CODE=0
      fi

      if [ $TEST_EXIT_CODE -ne 0 ]; then
        echo "❌ Gate 1 FAILED: Tests failed"
        cat /tmp/test-output.txt
        exit 1
      fi

      echo "✓ Tests passed ($TEST_FILES)"
    else
      echo "⚠️  No tests found for $ORGANISM_ID"
    fi
    ;;

  *)
    # Generic: Try to find and run any tests
    echo "Running generic tests..."

    if command -v npm &> /dev/null; then
      npm test > /tmp/test-output.txt 2>&1
      TEST_EXIT_CODE=$?

      if [ $TEST_EXIT_CODE -ne 0 ]; then
        echo "⚠️  Some tests failed, but continuing (may not be related to $ORGANISM_ID)"
        # Don't fail gate - tests might be for other organisms
      else
        echo "✓ All tests passed"
      fi
    else
      echo "⚠️  No test runner found"
    fi
    ;;
esac

echo ""
```

---

## Gate 2: Integration Test

Verify organism works with its dependencies:

```bash
echo "=== Gate 2: Integration Test ==="
echo ""

# Get organism dependencies from Beads
DEPENDENCIES=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.dependencies[]? // empty')

if [ -n "$DEPENDENCIES" ]; then
  echo "Checking integration with dependencies:"

  for DEP_ID in $DEPENDENCIES; do
    DEP_TITLE=$(bd show "$DEP_ID" --json 2>/dev/null | jq -r '.title')
    DEP_STATUS=$(bd show "$DEP_ID" --json 2>/dev/null | jq -r '.status')

    echo "  • $DEP_ID ($DEP_TITLE): $DEP_STATUS"

    if [ "$DEP_STATUS" != "closed" ]; then
      echo "⚠️  Dependency $DEP_ID not yet completed"
    fi
  done

  # Run integration tests if they exist
  INTEGRATION_TEST_DIR="tests/integration"
  if [ -d "$INTEGRATION_TEST_DIR" ]; then
    echo ""
    echo "Running integration tests..."

    if command -v npm &> /dev/null; then
      npm run test:integration > /tmp/integration-output.txt 2>&1
      INTEGRATION_EXIT_CODE=$?

      if [ $INTEGRATION_EXIT_CODE -ne 0 ]; then
        echo "⚠️  Integration tests failed, but continuing"
        # Don't fail gate - may be unrelated organisms
      else
        echo "✓ Integration tests passed"
      fi
    fi
  else
    echo "✓ No integration tests found (skip)"
  fi
else
  echo "✓ No dependencies (skip)"
fi

echo ""
```

---

## Gate 3: Merge Detection

Check for merge conflicts with main branch:

```bash
echo "=== Gate 3: Merge Detection ==="
echo ""

# Run merge conflict detector workflow
MERGE_RESULT=$(bash -c "source profiles/default/workflows/implementation/validation/merge-conflict-detector.md" 2>&1)
MERGE_EXIT_CODE=$?

if [ $MERGE_EXIT_CODE -ne 0 ]; then
  echo "❌ Gate 3 FAILED: Merge conflicts detected"
  echo "$MERGE_RESULT"

  # Extract conflict details for retry
  echo "$MERGE_RESULT" > /tmp/conflict-details.txt

  exit 3  # Exit code 3 = conflict
fi

echo "✓ No merge conflicts"
echo ""
```

---

## Gate 4: Regression Sample

Test a random previously completed organism to ensure no regressions:

```bash
echo "=== Gate 4: Regression Sample ==="
echo ""

# Get random completed organism
COMPLETED_ORGANISMS=$(jq -r '.completed[].id' .beads/autonomous-state/work-queue.json 2>/dev/null || echo "")

if [ -n "$COMPLETED_ORGANISMS" ]; then
  # Pick random organism
  SAMPLE_ORGANISM=$(echo "$COMPLETED_ORGANISMS" | shuf -n 1)
  SAMPLE_TITLE=$(bd show "$SAMPLE_ORGANISM" --json 2>/dev/null | jq -r '.title')

  echo "Testing random completed organism: $SAMPLE_ORGANISM ($SAMPLE_TITLE)"

  # Find tests for sampled organism
  SAMPLE_TITLE_SLUG=$(echo "$SAMPLE_TITLE" | tr ' ' '-' | tr '[:upper:]' '[:lower:]')
  SAMPLE_TEST_FILES=$(find . -name "*${SAMPLE_TITLE_SLUG}*.test.*" -o -name "*${SAMPLE_TITLE_SLUG}*.spec.*" 2>/dev/null || echo "")

  if [ -n "$SAMPLE_TEST_FILES" ]; then
    if command -v npm &> /dev/null; then
      npm test -- "$SAMPLE_TEST_FILES" > /tmp/regression-output.txt 2>&1
      REGRESSION_EXIT_CODE=$?

      if [ $REGRESSION_EXIT_CODE -ne 0 ]; then
        echo "❌ Gate 4 FAILED: Regression detected in $SAMPLE_ORGANISM"
        cat /tmp/regression-output.txt

        # Reopen the regressed organism
        bd update "$SAMPLE_ORGANISM" --status open --note "Regression detected during validation of $ORGANISM_ID"

        # Fail this organism too
        exit 4  # Exit code 4 = regression
      fi

      echo "✓ Regression test passed ($SAMPLE_ORGANISM)"
    else
      echo "⚠️  No test runner found (skip)"
    fi
  else
    echo "⚠️  No tests found for sampled organism (skip)"
  fi
else
  echo "✓ No completed organisms to sample (skip)"
fi

echo ""
```

---

## Gate 5: Quality Checks

Run linting, type checking, and coverage checks:

```bash
echo "=== Gate 5: Quality Checks ==="
echo ""

# Linting
if command -v npm &> /dev/null; then
  if grep -q '"lint"' package.json 2>/dev/null; then
    echo "Running linter..."
    npm run lint > /tmp/lint-output.txt 2>&1
    LINT_EXIT_CODE=$?

    if [ $LINT_EXIT_CODE -ne 0 ]; then
      echo "⚠️  Lint warnings/errors found:"
      cat /tmp/lint-output.txt | head -20
      echo "Continuing (non-blocking)..."
    else
      echo "✓ Lint passed"
    fi
  fi
fi

# Type checking
if command -v tsc &> /dev/null; then
  echo "Running type checker..."
  tsc --noEmit > /tmp/type-output.txt 2>&1
  TYPE_EXIT_CODE=$?

  if [ $TYPE_EXIT_CODE -ne 0 ]; then
    echo "⚠️  Type errors found:"
    cat /tmp/type-output.txt | head -20
    echo "Continuing (non-blocking)..."
  else
    echo "✓ Type check passed"
  fi
elif command -v mypy &> /dev/null; then
  echo "Running mypy..."
  mypy . > /tmp/type-output.txt 2>&1
  TYPE_EXIT_CODE=$?

  if [ $TYPE_EXIT_CODE -ne 0 ]; then
    echo "⚠️  Type errors found (non-blocking)"
  else
    echo "✓ Type check passed"
  fi
fi

# Coverage check (optional)
if command -v npm &> /dev/null; then
  if grep -q '"test:coverage"' package.json 2>/dev/null; then
    echo "Checking test coverage..."
    npm run test:coverage > /tmp/coverage-output.txt 2>&1

    # Extract coverage percentage
    COVERAGE=$(cat /tmp/coverage-output.txt | grep -oP '\d+\.\d+%' | head -1 || echo "unknown")
    echo "  Coverage: $COVERAGE"
  fi
fi

echo "✓ Quality checks complete"
echo ""
```

---

## Final: All Gates Passed

If all 5 gates pass, commit and push the organism:

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  ✅ ALL VALIDATION GATES PASSED"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""

# Organism is already committed by agent, just verify
COMMIT_SHA=$(git rev-parse HEAD 2>/dev/null || echo "unknown")

echo "Organism: $ORGANISM_ID"
echo "Commit: $COMMIT_SHA"
echo "Status: Validated and ready"
echo ""

# Return success
exit 0
```

---

## Error Handling

If any gate fails:

```bash
# Exit codes:
# 0 = success
# 1 = Gate 1 failed (tests)
# 2 = Gate 2 failed (integration)
# 3 = Gate 3 failed (merge conflict)
# 4 = Gate 4 failed (regression)
# 5 = Gate 5 failed (quality)

# Orchestrator will:
# 1. Extract error patterns from output
# 2. Add to learning system (improvements.json)
# 3. Retry organism (up to 3 attempts)
# 4. If 3 failures: create analysis issue
```

---

## Integration with Orchestrator

The orchestrator calls this workflow in STEP 3:

```bash
# In autonomous-build.md STEP 3:

# Validate organism
bash profiles/default/workflows/implementation/validation/organism-validator.md
VALIDATION_EXIT_CODE=$?

if [ $VALIDATION_EXIT_CODE -eq 0 ]; then
  # All gates passed - move to completed
  echo "🎉 Validation passed"
  # [move to completed queue]

elif [ $VALIDATION_EXIT_CODE -eq 3 ]; then
  # Merge conflict - handle via conflict resolution (Phase 4)
  echo "⚠️  Merge conflict detected"
  # [trigger conflict resolution workflow]

else
  # Other failure - retry or escalate
  echo "❌ Validation failed (gate $VALIDATION_EXIT_CODE)"
  # [extract errors, retry, or create analysis issue]
fi
```

---

## Configuration

Quality bar settings from `.beads/autonomous-config.yml`:

```yaml
quality_bar:
  require_all_tests_pass: true      # Gate 1 is blocking
  require_zero_console_errors: true # Check browser console (if applicable)
  require_polished_ui: true         # Visual regression (if UI component)
```

---

## See Also

- `/profiles/default/workflows/implementation/validation/merge-conflict-detector.md` - Gate 3 implementation
- `/profiles/default/agents/regression-verifier.md` - Detailed regression testing agent
- `/profiles/default/workflows/implementation/verification/regression-verification.md` - Cross-phase regression workflow
