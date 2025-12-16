# Error Analyzer

This workflow extracts error patterns from failed validations and adds them to the learning system's knowledge base.

## Overview

When organisms fail validation, this analyzer:
1. Parses error messages and logs
2. Categorizes errors by type
3. Identifies common patterns
4. Generates fixes/prevention strategies
5. Updates improvements.json
6. Tracks error trends over time

**Goal**: Improve success rate over iterations by learning from mistakes.

---

## Usage

Called by the orchestrator in STEP 3 when an organism fails validation (before retry).

**Input:**
- `ORGANISM_ID`: Failed organism
- `VALIDATION_EXIT_CODE`: Which gate failed (1-5)
- `ERROR_OUTPUT`: Validation error messages
- `ATTEMPT`: Current attempt number

**Output:**
- Updated `.beads/autonomous-state/learning/improvements.json`
- Error patterns added to knowledge base

---

## Error Pattern Extraction

### Step 1: Parse Error Output

```bash
extract_errors() {
  local ORGANISM_ID=$1
  local VALIDATION_EXIT_CODE=$2
  local ERROR_OUTPUT=$3

  # Initialize error categories
  declare -A ERROR_PATTERNS

  case $VALIDATION_EXIT_CODE in
    1)
      # Gate 1: Test failures
      echo "Analyzing test failures..."

      # Extract test error patterns
      if echo "$ERROR_OUTPUT" | grep -q "Cannot find module"; then
        ERROR_PATTERNS["imports"]="Missing import statement"
      fi

      if echo "$ERROR_OUTPUT" | grep -q "is not a function"; then
        ERROR_PATTERNS["types"]="Type mismatch - expected function"
      fi

      if echo "$ERROR_OUTPUT" | grep -q "AssertionError"; then
        ERROR_PATTERNS["tests"]="Test assertion failed"
      fi

      if echo "$ERROR_OUTPUT" | grep -q "ReferenceError"; then
        ERROR_PATTERNS["dependencies"]="Undefined variable or function"
      fi

      if echo "$ERROR_OUTPUT" | grep -q "SyntaxError"; then
        ERROR_PATTERNS["syntax"]="Syntax error in code"
      fi
      ;;

    2)
      # Gate 2: Integration failures
      echo "Analyzing integration failures..."

      if echo "$ERROR_OUTPUT" | grep -q "dependency"; then
        ERROR_PATTERNS["dependencies"]="Missing or incompatible dependency"
      fi

      if echo "$ERROR_OUTPUT" | grep -q "interface"; then
        ERROR_PATTERNS["types"]="Interface incompatibility"
      fi
      ;;

    3)
      # Gate 3: Merge conflicts (handled by conflict resolution)
      echo "Merge conflict - handled by Phase 4"
      return 0
      ;;

    4)
      # Gate 4: Regression
      echo "Analyzing regression..."

      ERROR_PATTERNS["regression"]="Implementation broke existing functionality"
      ;;

    5)
      # Gate 5: Quality checks
      echo "Analyzing quality issues..."

      if echo "$ERROR_OUTPUT" | grep -q "lint"; then
        ERROR_PATTERNS["quality"]="Linting errors"
      fi

      if echo "$ERROR_OUTPUT" | grep -q "type"; then
        ERROR_PATTERNS["types"]="Type checking errors"
      fi
      ;;
  esac

  # Output patterns as JSON array
  for category in "${!ERROR_PATTERNS[@]}"; do
    echo "{\"category\": \"$category\", \"pattern\": \"${ERROR_PATTERNS[$category]}\"}"
  done | jq -s '.'
}
```

### Step 2: Generate Fixes

```bash
generate_fix() {
  local PATTERN=$1
  local CATEGORY=$2

  case $CATEGORY in
    "imports")
      echo "Always verify imports before implementing. Check existing files for import patterns."
      ;;
    "types")
      echo "Check interface definitions before using types. Read type files in templates/interfaces/"
      ;;
    "tests")
      echo "Review test setup and assertions. Check mock configuration in existing tests."
      ;;
    "dependencies")
      echo "Verify all dependencies are implemented. Check organism dependency graph."
      ;;
    "syntax")
      echo "Review code syntax carefully. Use linter before committing."
      ;;
    "quality")
      echo "Run linter and type checker before completing organism."
      ;;
    "regression")
      echo "Test related organisms after implementation. Run broader test suite."
      ;;
    *)
      echo "Review error message and adjust implementation accordingly."
      ;;
  esac
}
```

### Step 3: Update Knowledge Base

```bash
update_improvements() {
  local ORGANISM_ID=$1
  local ERROR_PATTERNS=$2  # JSON array

  IMPROVEMENTS_FILE=".beads/autonomous-state/learning/improvements.json"

  # For each extracted error pattern
  echo "$ERROR_PATTERNS" | jq -c '.[]' | while read ERROR; do
    CATEGORY=$(echo "$ERROR" | jq -r '.category')
    PATTERN=$(echo "$ERROR" | jq -r '.pattern')
    FIX=$(generate_fix "$PATTERN" "$CATEGORY")

    # Check if pattern already exists
    EXISTING=$(jq ".common_errors[] | select(.pattern == \"$PATTERN\")" "$IMPROVEMENTS_FILE" 2>/dev/null || echo "")

    if [ -n "$EXISTING" ]; then
      # Increment seen_count and update last_seen
      jq "
        .common_errors = [.common_errors[] |
          if .pattern == \"$PATTERN\" then
            .seen_count += 1 |
            .last_seen = \"$(date -Iseconds)\" |
            .trending = (if .seen_count > 5 then \"up\" else \"stable\" end)
          else
            .
          end
        ]
      " "$IMPROVEMENTS_FILE" > /tmp/improvements.json
      mv /tmp/improvements.json "$IMPROVEMENTS_FILE"

      echo "✓ Updated existing error pattern: $PATTERN (seen $(jq ".common_errors[] | select(.pattern == \"$PATTERN\") | .seen_count" "$IMPROVEMENTS_FILE")x)"
    else
      # Add new pattern
      NEW_ERROR=$(jq -n "{
        pattern: \"$PATTERN\",
        fix: \"$FIX\",
        category: \"$CATEGORY\",
        seen_count: 1,
        first_seen: \"$(date -Iseconds)\",
        last_seen: \"$(date -Iseconds)\",
        trending: \"stable\"
      }")

      jq ".common_errors += [$NEW_ERROR]" "$IMPROVEMENTS_FILE" > /tmp/improvements.json
      mv /tmp/improvements.json "$IMPROVEMENTS_FILE"

      echo "✓ Added new error pattern: $PATTERN"
    fi
  done
}
```

---

## Organism-Specific Learning

Extract organism type-specific patterns:

```bash
add_organism_specific_learning() {
  local ORGANISM_ID=$1
  local ORGANISM_TYPE=$2
  local PATTERN=$3

  IMPROVEMENTS_FILE=".beads/autonomous-state/learning/improvements.json"

  # Add to organism-specific learnings
  case $ORGANISM_TYPE in
    "database-layer")
      jq ".organism_specific[\"database-layer\"] += [\"$PATTERN\"] | .organism_specific[\"database-layer\"] |= unique" "$IMPROVEMENTS_FILE" > /tmp/improvements.json
      mv /tmp/improvements.json "$IMPROVEMENTS_FILE"
      ;;
    "api-layer")
      jq ".organism_specific[\"api-layer\"] += [\"$PATTERN\"] | .organism_specific[\"api-layer\"] |= unique" "$IMPROVEMENTS_FILE" > /tmp/improvements.json
      mv /tmp/improvements.json "$IMPROVEMENTS_FILE"
      ;;
    "ui-component")
      jq ".organism_specific[\"ui-component\"] += [\"$PATTERN\"] | .organism_specific[\"ui-component\"] |= unique" "$IMPROVEMENTS_FILE" > /tmp/improvements.json
      mv /tmp/improvements.json "$IMPROVEMENTS_FILE"
      ;;
  esac

  echo "✓ Added organism-specific learning for $ORGANISM_TYPE"
}
```

---

## Metrics Tracking

Update success rate metrics:

```bash
update_metrics() {
  local ORGANISM_ID=$1
  local SUCCESS=$2  # true or false

  IMPROVEMENTS_FILE=".beads/autonomous-state/learning/improvements.json"

  # Get current metrics
  CURRENT_ITERATION=$(jq '.metrics.total_iterations' "$IMPROVEMENTS_FILE")
  NEXT_ITERATION=$((CURRENT_ITERATION + 1))

  # Count errors in this iteration
  ERROR_COUNT=0
  if [ "$SUCCESS" = "false" ]; then
    ERROR_COUNT=1
  fi

  # Calculate success rate for this iteration
  TOTAL_ORGANISMS=$(jq '.metrics.total_organisms' "$IMPROVEMENTS_FILE")
  NEW_TOTAL=$((TOTAL_ORGANISMS + 1))

  # Simplified success rate (would need full iteration tracking for accuracy)
  # For now, just track per-organism

  # Update metrics
  jq "
    .metrics.total_iterations = $NEXT_ITERATION |
    .metrics.total_organisms = $NEW_TOTAL
  " "$IMPROVEMENTS_FILE" > /tmp/improvements.json
  mv /tmp/improvements.json "$IMPROVEMENTS_FILE"

  echo "✓ Metrics updated: iteration $NEXT_ITERATION, organism $NEW_TOTAL"
}
```

---

## Integration with Orchestrator

Called from autonomous-build.md STEP 3 (after validation failure):

```bash
# In autonomous-build.md STEP 3 (validation failed):

if [ "$VALIDATION_PASSED" = false ]; then
  # Extract errors and learn
  echo "Analyzing errors for learning system..."

  # Get error output from validation
  ERROR_OUTPUT=$(cat /tmp/test-output.txt /tmp/lint-output.txt /tmp/type-output.txt 2>/dev/null || echo "No error details")

  # Export variables for error analyzer
  export ORGANISM_ID
  export VALIDATION_EXIT_CODE
  export ERROR_OUTPUT
  export ATTEMPT

  # Run error analyzer
  bash profiles/default/workflows/implementation/learning/error-analyzer.md

  echo "✓ Errors extracted and added to learning system"
fi
```

---

## Example: Test Failure Analysis

**Input:**
```
Organism: bd-org-123 (User validation atom)
Gate: 1 (Tests)
Error output:
  ✗ should validate email format
    TypeError: validateEmail is not a function
    at test/validation.test.js:15:5

  ✗ should reject invalid emails
    ReferenceError: validateEmail is not defined
```

**Extracted patterns:**
```json
[
  {
    "category": "types",
    "pattern": "Type mismatch - expected function"
  },
  {
    "category": "dependencies",
    "pattern": "Undefined variable or function"
  }
]
```

**Generated fixes:**
- "Check interface definitions before using types. Read type files in templates/interfaces/"
- "Verify all dependencies are implemented. Check organism dependency graph."

**Updated improvements.json:**
```json
{
  "common_errors": [
    {
      "pattern": "Type mismatch - expected function",
      "fix": "Check interface definitions before using types...",
      "category": "types",
      "seen_count": 1,
      "first_seen": "2025-12-16T10:30:00Z",
      "last_seen": "2025-12-16T10:30:00Z",
      "trending": "stable"
    }
  ]
}
```

---

## Best Practices Learning

When organisms succeed consistently, extract best practices:

```bash
extract_best_practices() {
  local ORGANISM_ID=$1

  # Get successful patterns from git commit messages
  COMMIT_MSG=$(git log -1 --format=%B 2>/dev/null || echo "")

  # Simple pattern extraction (can be enhanced)
  if echo "$COMMIT_MSG" | grep -q "test"; then
    add_best_practice "Write tests before implementing feature"
  fi

  if echo "$COMMIT_MSG" | grep -q "refactor"; then
    add_best_practice "Refactor for clarity after implementation"
  fi
}

add_best_practice() {
  local PRACTICE=$1
  IMPROVEMENTS_FILE=".beads/autonomous-state/learning/improvements.json"

  # Add if not already present
  if ! jq -e ".best_practices | index(\"$PRACTICE\")" "$IMPROVEMENTS_FILE" >/dev/null 2>&1; then
    jq ".best_practices += [\"$PRACTICE\"]" "$IMPROVEMENTS_FILE" > /tmp/improvements.json
    mv /tmp/improvements.json "$IMPROVEMENTS_FILE"
    echo "✓ Added best practice: $PRACTICE"
  fi
}
```

---

## See Also

- `/profiles/default/workflows/implementation/learning/improvement-injector.md` - Injects learnings into agent prompts
- `/profiles/default/commands/autonomous-build/state-templates/improvements.json` - Knowledge base schema
- `/profiles/default/commands/autonomous-build/multi-agent/autonomous-build.md` - Orchestrator integration
