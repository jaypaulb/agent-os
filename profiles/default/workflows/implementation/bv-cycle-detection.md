# BV Cycle Detection Workflow

Detect and prevent circular dependency creation in beads issue graphs.

## Overview

A **cycle** (circular dependency) occurs when Issue A depends on Issue B, which depends on Issue C, which depends back on Issue A:

```
A → B → C → A  (cycle!)
```

This creates a **deadlock** - none of these issues can be completed because they all wait on each other. Cycles are bugs in the dependency graph that must be fixed before work can proceed.

## Why Cycles are Problematic

### Deadlock

No issue in a cycle can be marked "ready" because they all have blocking dependencies:

```
bd-101 blocks bd-102
bd-102 blocks bd-103
bd-103 blocks bd-101  ← Creates cycle

Result: bd ready shows 0 items (all blocked)
```

### Execution Planning Failure

BV's execution planner cannot generate valid work tracks when cycles exist:

```
bv --robot-plan
Error: Cycle detected - cannot generate deterministic execution plan
```

### Ambiguous Priority

Which issue should be worked on first? There's no valid answer:

```
Start with bd-101? Blocked by bd-103
Start with bd-102? Blocked by bd-101
Start with bd-103? Blocked by bd-102
```

## What are Cycles?

### Direct Cycle (2-node)

```
bd-101 blocks bd-102
bd-102 blocks bd-101
```

Simple loop between two issues.

### Indirect Cycle (3+ nodes)

```
bd-101 blocks bd-102
bd-102 blocks bd-103
bd-103 blocks bd-104
bd-104 blocks bd-101
```

Longer chain that eventually loops back.

### Common Causes

1. **Bidirectional dependencies**
   - Issue A needs data from B
   - Issue B needs interface from A
   - **Solution:** Extract shared interface to separate issue

2. **Layering violations**
   - Database layer depends on API layer
   - API layer depends on database layer
   - **Solution:** Follow atomic design hierarchy strictly

3. **Discovered work creating back-edges**
   - Agent discovers Issue D while working on Issue C
   - Creates: C blocks D, D blocks B, B blocks C (cycle!)
   - **Solution:** Check for transitive dependencies before adding

4. **Manual dependency errors**
   - Typo in dependency creation
   - Misunderstanding of "blocks" direction
   - **Solution:** Use `bd dep tree` to visualize before adding

## Detection

### Check for Existing Cycles

```bash
# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

if bv_available; then
    if ! check_cycles; then
        echo "Circular dependencies detected - see output above"
        exit 1
    else
        echo "✓ No cycles detected - graph is valid"
    fi
fi
```

### Display Cycle Details

```bash
if bv_available; then
    INSIGHTS=$(get_graph_insights)
    CYCLES=$(echo "$INSIGHTS" | jq -r '.cycles')
    CYCLE_COUNT=$(echo "$CYCLES" | jq '. | length')

    if [[ "$CYCLE_COUNT" -gt 0 ]]; then
        echo "❌ ERROR: $CYCLE_COUNT circular dependencies detected!"
        echo ""
        echo "$CYCLES" | jq -r '.[] | "  Cycle: " + (. | join(" → "))'
        echo ""
        echo "Fix these cycles before proceeding."
        exit 1
    fi
fi
```

### Before Adding Dependencies

**CRITICAL:** Check if adding a dependency would create a cycle BEFORE running `bd dep add`:

```bash
# Want to add: bd dep add $CHILD $PARENT --type blocks
# (Meaning: $PARENT blocks $CHILD)

# Check if $CHILD already transitively blocks $PARENT
echo "Checking if adding '$PARENT blocks $CHILD' would create cycle..."

# Get dependency tree for $CHILD (what does $CHILD block?)
CHILD_BLOCKS=$(bd dep tree $CHILD | grep -o 'bd-[a-z0-9]*')

if echo "$CHILD_BLOCKS" | grep -q "$PARENT"; then
    echo "❌ CYCLE DETECTED: $CHILD already blocks $PARENT transitively!"
    echo "   Cannot add $PARENT blocks $CHILD - this would create a cycle."
    echo ""
    echo "   Path: $CHILD → ... → $PARENT (and you want to add $PARENT → $CHILD)"
    exit 1
else
    echo "✓ No cycle would be created"
    bd dep add $CHILD $PARENT --type blocks
fi
```

### After Creating/Modifying Issues

After agents discover and create new issues with dependencies, verify no cycles introduced:

```bash
# After bd create and bd dep add operations
if bv_available; then
    if ! check_cycles; then
        echo ""
        echo "❌ New cycles introduced! Investigate immediately."
        echo "   Likely cause: Last bd dep add command created a circular dependency"
        echo ""

        # Show the cycles
        get_graph_insights | jq -r '.cycles[] | "  " + (. | join(" → "))'

        echo ""
        echo "Fix: Remove the dependency that closed the loop"
        exit 1
    fi
fi
```

## Integration Points

### 1. Create-Beads-Issues Workflow

**File:** `/workflows/implementation/create-beads-issues.md`

**Location:** After all issues and dependencies created

```bash
# Final validation - check for cycles
if command -v bv &> /dev/null; then
    echo ""
    echo "Validating dependency graph..."

    if ! check_cycles; then
        echo ""
        echo "❌ ERROR: Circular dependencies detected in issue structure!"
        echo ""
        echo "This indicates a bug in the spec or issue creation logic."
        echo "Review the dependency structure and remove circular dependencies."
        exit 1
    else
        echo "✓ No cycles detected - issue structure is valid"
    fi
fi
```

### 2. Auto-Build Command

**File:** `/commands/auto-build/auto-build.md`

**Location:** At start of build loop (already added in Phase 4)

Checks for:
- Cycles introduced since last build (regression)
- Existing cycles that would block autonomous execution

### 3. Implement-With-Beads Workflow

**File:** `/workflows/implementation/implement-with-beads.md`

**Location:** When discovering new work

```bash
# Before creating discovered-from dependency
CHILD_ISSUE="bd-new-discovered"
PARENT_ISSUE="bd-current"

# Verify this won't create a cycle
PARENT_BLOCKS=$(bd dep tree $PARENT_ISSUE | grep -o 'bd-[a-z0-9]*')

if echo "$PARENT_BLOCKS" | grep -q "$CHILD_ISSUE"; then
    echo "⚠️  WARNING: Cannot create discovered-from link"
    echo "   $PARENT_ISSUE already transitively blocks $CHILD_ISSUE"
    echo "   Adding this dependency would create a cycle"
else
    bd dep add $CHILD_ISSUE $PARENT_ISSUE --type discovered-from
fi
```

## Fixing Cycles

### Strategy 1: Remove Unnecessary Dependencies

One dependency in the cycle may be incorrect or unnecessary:

```
Cycle: bd-101 → bd-102 → bd-103 → bd-101

Investigation:
- bd-101 blocks bd-102: NECESSARY (db schema before models)
- bd-102 blocks bd-103: NECESSARY (models before API)
- bd-103 blocks bd-101: WRONG! (API shouldn't block schema)

Fix: Remove bd-103 → bd-101 dependency
```

### Strategy 2: Extract Shared Component

Two issues with bidirectional dependencies need a shared interface:

```
Cycle: bd-auth → bd-user → bd-auth

Problem:
- bd-auth needs User model from bd-user
- bd-user needs auth middleware from bd-auth

Fix: Extract shared interface
  bd-interface (new)
    ├─ blocks bd-auth
    └─ blocks bd-user

Result: No cycle, linear dependency chain
```

### Strategy 3: Reorder Work

Cycle exists because of incorrect dependency direction:

```
Cycle: bd-frontend → bd-backend → bd-frontend

Problem:
- bd-frontend depends on API endpoints from bd-backend (correct)
- bd-backend depends on UI mockups from bd-frontend (wrong!)

Fix: Reverse bd-backend → bd-frontend
  API can be built without UI mockups (just needs spec)
  UI needs API to be functional

Result: bd-backend blocks bd-frontend (correct direction)
```

### Strategy 4: Split Issues

One issue is doing too much, creating circular dependencies:

```
Cycle: bd-user-service → bd-auth-service → bd-user-service

Investigation:
- bd-user-service contains: user CRUD + authentication logic
- bd-auth-service needs user data (from bd-user-service)
- bd-user-service needs auth (from bd-auth-service)

Fix: Split bd-user-service into two issues
  bd-user-data (models, CRUD only)
  bd-user-auth (authentication logic)

New structure:
  bd-user-data → bd-auth-service → bd-user-auth
  (no cycle)
```

## Commands Reference

### Check for Cycles

```bash
# Using BV
bv --robot-insights --format json | jq -r '.cycles'

# Using helper function
check_cycles  # Returns 0 if no cycles, 1 if cycles exist
```

### Visualize Dependencies

```bash
# Show what an issue blocks
bd dep tree [issue-id]

# Show what blocks an issue
bd dep tree [issue-id] --reverse

# Show full graph (if beads supports it)
bd dep graph
```

### Remove Dependency

```bash
# Remove incorrect dependency
bd dep remove [child] [parent] --type blocks
```

## Prevention Best Practices

1. **Follow atomic design hierarchy strictly**
   - Atoms never depend on molecules or organisms
   - Molecules never depend on organisms
   - Organisms can depend on molecules and atoms only
   - This naturally prevents most cycles

2. **Check before adding dependencies**
   - Use `bd dep tree` to visualize transitive dependencies
   - Ask: "Does CHILD already block PARENT transitively?"
   - If yes, adding PARENT blocks CHILD creates a cycle

3. **Use BV validation after bulk changes**
   - After creating many issues, run cycle detection
   - Catch cycles early before work begins

4. **Document dependency reasoning**
   - Use `--note` flag when adding dependencies
   - Clear reasoning helps identify incorrect dependencies later

5. **Regular cycle audits**
   - Run `check_cycles` daily
   - Part of auto-build pre-flight checks

## Real-World Example

### Scenario: User Authentication Feature

**Initial dependencies (WRONG - has cycle):**

```
bd-101 (User model)       blocks bd-102 (Auth service)
bd-102 (Auth service)     blocks bd-103 (Session manager)
bd-103 (Session manager)  blocks bd-104 (Login API)
bd-104 (Login API)        blocks bd-101 (User model)  ← CYCLE!
```

**Problem:** Login API needs User model for authentication, but we said User model depends on Login API.

**BV detection:**

```bash
$ bv --robot-insights --format json | jq -r '.cycles'
[
  ["bd-101", "bd-102", "bd-103", "bd-104", "bd-101"]
]
```

**Analysis:**

- bd-101 blocks bd-102: ✓ Correct (models before services)
- bd-102 blocks bd-103: ✓ Correct (auth before sessions)
- bd-103 blocks bd-104: ✓ Correct (sessions before API)
- bd-104 blocks bd-101: ✗ WRONG (API doesn't block models!)

**Root cause:** Dependency direction reversed. Login API NEEDS User model (not the other way around).

**Fix:**

```bash
# Remove incorrect dependency
bd dep remove bd-101 bd-104 --type blocks

# Corrected structure (no cycle):
bd-101 (User model) → bd-102 (Auth service) → bd-103 (Session manager) → bd-104 (Login API)
```

**Verification:**

```bash
$ bv --robot-insights --format json | jq -r '.cycles'
[]

✓ No cycles - graph is valid
```

## Fallback Behavior

If BV unavailable, cycle detection is skipped (permissive mode):

```bash
check_cycles  # Always returns 0 (success)
```

Agents should still use `bd dep tree` manually to verify dependencies don't create obvious loops.

## Troubleshooting

### BV reports cycle but I don't see it

**Cause:** Long transitive path (5+ nodes)

**Solution:** Use BV's cycle output to trace the full path:

```bash
bv --robot-insights --format json | jq -r '.cycles[] | . | join(" → ")'
```

### Cycle exists but graph seems valid

**Cause:** Bidirectional dependencies at different levels

**Example:**
```
bd-101 blocks bd-102 (correct)
bd-102 blocks bd-103 (correct)
bd-103 has discovered-from bd-101 (creates cycle!)
```

**Solution:** `discovered-from` edges also create dependencies. Use `--type` carefully.

### Removed dependency but cycle persists

**Cause:** Multiple paths create the cycle

**Solution:** Use `bd dep tree` to find all paths between nodes in cycle

## Summary

- **Cycles = deadlocks** - No work can proceed if cycles exist
- **Detect early** - Run `check_cycles` after creating issues
- **Prevent proactively** - Check before adding dependencies
- **Fix systematically** - Use strategies above (remove, extract, reorder, split)
- **Validate always** - Part of auto-build and CI/CD pre-flight checks

Cycle detection is critical for autonomous agents - deadlocked graphs halt all progress.
