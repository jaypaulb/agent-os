# Improvement Injector

This workflow formats learned improvements and injects them into agent prompts to prevent repeated errors.

## Overview

Before spawning an implementation agent, this injector:
1. Loads improvements from knowledge base
2. Filters relevant learnings for organism type
3. Formats context for agent prompt
4. Prioritizes recent/frequent errors
5. Includes type-specific best practices

**Goal**: Each agent learns from all previous failures without repeating mistakes.

---

## Usage

Called by the orchestrator in STEP 1 (Dispatch New Work) before spawning each agent.

**Input:**
- `ORGANISM_TYPE`: Type of organism (database-layer, api-layer, ui-component)
- `ATTEMPT`: Current attempt number (affects which learnings to emphasize)

**Output:**
- Formatted learning context string for agent prompt

---

## Load and Filter Improvements

```bash
load_improvements() {
  local ORGANISM_TYPE=$1
  local ATTEMPT=$2

  IMPROVEMENTS_FILE=".beads/autonomous-state/learning/improvements.json"

  if [ ! -f "$IMPROVEMENTS_FILE" ]; then
    echo ""
    return
  fi

  # Get all improvements
  IMPROVEMENTS=$(cat "$IMPROVEMENTS_FILE")

  # Check if there are any learnings
  ERROR_COUNT=$(echo "$IMPROVEMENTS" | jq '.common_errors | length')
  PRACTICE_COUNT=$(echo "$IMPROVEMENTS" | jq '.best_practices | length')

  if [ "$ERROR_COUNT" -eq 0 ] && [ "$PRACTICE_COUNT" -eq 0 ]; then
    echo ""
    return
  fi

  # Build learning context
  build_learning_context "$IMPROVEMENTS" "$ORGANISM_TYPE" "$ATTEMPT"
}
```

---

## Build Learning Context

```bash
build_learning_context() {
  local IMPROVEMENTS=$1
  local ORGANISM_TYPE=$2
  local ATTEMPT=$3

  # Start with header
  CONTEXT="IMPORTANT: Learn from previous sessions to avoid common mistakes:\n\n"

  # Add general errors (grouped by category)
  CATEGORIES=$(echo "$IMPROVEMENTS" | jq -r '.common_errors | group_by(.category) | keys[]')

  for CATEGORY in $CATEGORIES; do
    # Get top 3 errors in this category (sorted by seen_count)
    CATEGORY_ERRORS=$(echo "$IMPROVEMENTS" | jq -r "
      .common_errors |
      map(select(.category == \"$CATEGORY\")) |
      sort_by(.seen_count) |
      reverse |
      .[0:3] |
      map(\"  ❌ \\(.pattern)\\n  ✅ \\(.fix)\") |
      join(\"\\n\")
    ")

    if [ -n "$CATEGORY_ERRORS" ] && [ "$CATEGORY_ERRORS" != "null" ]; then
      CONTEXT="${CONTEXT}${CATEGORY} issues:\n${CATEGORY_ERRORS}\n\n"
    fi
  done

  # Add organism-specific learnings
  TYPE_SPECIFIC=$(echo "$IMPROVEMENTS" | jq -r ".organism_specific[\"$ORGANISM_TYPE\"][]? // empty" 2>/dev/null || echo "")

  if [ -n "$TYPE_SPECIFIC" ]; then
    CONTEXT="${CONTEXT}${ORGANISM_TYPE} specific learnings:\n"
    echo "$TYPE_SPECIFIC" | while read LEARNING; do
      CONTEXT="${CONTEXT}  • ${LEARNING}\n"
    done
    CONTEXT="${CONTEXT}\n"
  fi

  # Add best practices
  BEST_PRACTICES=$(echo "$IMPROVEMENTS" | jq -r '.best_practices[]? // empty' 2>/dev/null || echo "")

  if [ -n "$BEST_PRACTICES" ]; then
    CONTEXT="${CONTEXT}Best practices discovered:\n"
    echo "$BEST_PRACTICES" | head -5 | while read PRACTICE; do
      CONTEXT="${CONTEXT}  • ${PRACTICE}\n"
    done
    CONTEXT="${CONTEXT}\n"
  fi

  # If this is a retry (attempt > 1), emphasize error prevention
  if [ "$ATTEMPT" -gt 1 ]; then
    CONTEXT="${CONTEXT}⚠️  RETRY MODE (Attempt $ATTEMPT):\n"
    CONTEXT="${CONTEXT}This organism failed validation previously. Pay extra attention to:\n"
    CONTEXT="${CONTEXT}  • Test failures and error messages\n"
    CONTEXT="${CONTEXT}  • Import statements and dependencies\n"
    CONTEXT="${CONTEXT}  • Type definitions and interfaces\n"
    CONTEXT="${CONTEXT}  • Integration with existing code\n\n"
  fi

  echo -e "$CONTEXT"
}
```

---

## Prioritize Learnings

Prioritize learnings based on relevance:

```bash
prioritize_learnings() {
  local IMPROVEMENTS=$1
  local ORGANISM_TYPE=$2

  # Calculate priority scores
  # Priority = seen_count * recency_factor * relevance_factor

  echo "$IMPROVEMENTS" | jq "
    .common_errors |
    map(
      . + {
        priority: (
          .seen_count *
          (if .trending == \"up\" then 2 elif .trending == \"down\" then 0.5 else 1 end) *
          (if .category == \"$ORGANISM_TYPE\" then 2 else 1 end)
        )
      }
    ) |
    sort_by(.priority) |
    reverse
  "
}
```

---

## Format for Agent Prompt

Format improvements as clear instructions:

```bash
format_for_prompt() {
  local LEARNING_CONTEXT=$1

  # Add markdown formatting for readability
  cat <<EOF
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📚 LEARNING CONTEXT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

$LEARNING_CONTEXT

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
EOF
}
```

---

## Integration with Orchestrator

Used in autonomous-build.md STEP 1:

```bash
# In autonomous-build.md STEP 1 (Dispatch New Work):

# Load improvements for agent prompt
IMPROVEMENTS=$(cat .beads/autonomous-state/learning/improvements.json)
LEARNING_CONTEXT=""

if [ "$(echo "$IMPROVEMENTS" | jq '.common_errors | length')" -gt 0 ]; then
  # Use improvement injector
  export ORGANISM_TYPE
  export ATTEMPT=$(echo "$ORGANISM_DATA" | jq -r '.attempt // 1')

  LEARNING_CONTEXT=$(bash profiles/default/workflows/implementation/learning/improvement-injector.md)
fi

# Inject into agent prompt
TASK_PROMPT="You are the implementer agent working on organism $ORGANISM_ID.

$LEARNING_CONTEXT

Your task: Implement this organism, test it, and close it.
...
"
```

---

## Example Output

**For database-layer organism on attempt 2:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📚 LEARNING CONTEXT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

IMPORTANT: Learn from previous sessions to avoid common mistakes:

imports issues:
  ❌ Missing import statement
  ✅ Always verify imports before implementing. Check existing files for import patterns.
  ❌ Import path incorrect
  ✅ Use relative imports from project root. Check other files for examples.

types issues:
  ❌ Type mismatch - expected function
  ✅ Check interface definitions before using types. Read type files in templates/interfaces/
  ❌ Interface incompatibility
  ✅ Verify interface matches existing definitions. Don't create duplicate types.

tests issues:
  ❌ Test assertion failed
  ✅ Review test setup and assertions. Check mock configuration in existing tests.

database-layer specific learnings:
  • Always check for existing migrations before creating new ones
  • Use consistent naming: {timestamp}_{action}_{table}.js
  • Verify foreign key relationships in schema
  • Test database constraints thoroughly

Best practices discovered:
  • Write tests before implementing feature
  • Run tests after each significant change
  • Read similar organism implementations before starting
  • Commit early and often with clear messages
  • Check for existing type definitions before creating new ones

⚠️  RETRY MODE (Attempt 2):
This organism failed validation previously. Pay extra attention to:
  • Test failures and error messages
  • Import statements and dependencies
  • Type definitions and interfaces
  • Integration with existing code

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Metrics Dashboard

Show learning effectiveness:

```bash
show_learning_metrics() {
  IMPROVEMENTS_FILE=".beads/autonomous-state/learning/improvements.json"

  if [ ! -f "$IMPROVEMENTS_FILE" ]; then
    echo "No learning data available yet"
    return
  fi

  echo "=== Learning System Metrics ==="
  echo ""

  # Total learnings
  ERROR_COUNT=$(jq '.common_errors | length' "$IMPROVEMENTS_FILE")
  PRACTICE_COUNT=$(jq '.best_practices | length' "$IMPROVEMENTS_FILE")
  TOTAL_ORGANISMS=$(jq '.metrics.total_organisms' "$IMPROVEMENTS_FILE")

  echo "Knowledge Base:"
  echo "  • Error patterns learned: $ERROR_COUNT"
  echo "  • Best practices: $PRACTICE_COUNT"
  echo "  • Total organisms processed: $TOTAL_ORGANISMS"
  echo ""

  # Top errors
  echo "Most common errors:"
  jq -r '.common_errors | sort_by(.seen_count) | reverse | .[0:5] | .[] | "  • \(.pattern) (seen \(.seen_count)x, trending: \(.trending))"' "$IMPROVEMENTS_FILE"
  echo ""

  # Trending
  TRENDING_UP=$(jq '.common_errors | map(select(.trending == "up")) | length' "$IMPROVEMENTS_FILE")
  TRENDING_DOWN=$(jq '.common_errors | map(select(.trending == "down")) | length' "$IMPROVEMENTS_FILE")

  echo "Trends:"
  echo "  • Errors increasing: $TRENDING_UP"
  echo "  • Errors decreasing: $TRENDING_DOWN"
  echo ""
}
```

---

## Advanced Features

### Context-Aware Injection

Adjust learnings based on context:

```bash
# If previous attempt failed with specific error, emphasize that fix
if [ "$ATTEMPT" -gt 1 ]; then
  LAST_FAILURE=$(echo "$ORGANISM_DATA" | jq -r '.last_failure // ""')

  if echo "$LAST_FAILURE" | grep -q "test"; then
    CONTEXT="${CONTEXT}🔴 CRITICAL: Previous attempt failed tests. Focus on:\n"
    CONTEXT="${CONTEXT}  • Writing correct test cases\n"
    CONTEXT="${CONTEXT}  • Mocking dependencies properly\n"
    CONTEXT="${CONTEXT}  • Checking assertion logic\n\n"
  fi
fi
```

### Success Pattern Recognition

Learn from successful patterns:

```bash
# When organism succeeds on first try
if [ "$ATTEMPT" -eq 1 ] && [ "$SUCCESS" = true ]; then
  # Extract success pattern
  COMMIT_MSG=$(git log -1 --format=%B)

  # Simple heuristic: if commit message mentions certain patterns
  if echo "$COMMIT_MSG" | grep -qE "test.*first|TDD|test-driven"; then
    add_best_practice "Write tests first (TDD approach)"
  fi
fi
```

---

## See Also

- `/profiles/default/workflows/implementation/learning/error-analyzer.md` - Extracts errors and updates knowledge base
- `/profiles/default/commands/autonomous-build/state-templates/improvements.json` - Knowledge base schema
- `/profiles/default/commands/autonomous-build/multi-agent/autonomous-build.md` - Orchestrator (STEP 1)
