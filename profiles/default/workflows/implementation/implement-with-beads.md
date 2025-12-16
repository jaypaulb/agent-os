# Implement Tasks Using Beads

This workflow guides agents through implementation using beads for issue tracking, ready work discovery, and cross-session context recovery.

## Prerequisites

- Beads initialized in spec folder
- Issues created via `create-beads-issues.md` workflow
- `bd ready` shows unblocked work

## Overview

Beads-based implementation differs from tasks.md in key ways:

| Aspect | Beads Mode | Tasks.md Mode |
|--------|------------|---------------|
| **Work Discovery** | `bd ready` (shows unblocked issues) | Read task groups sequentially |
| **Progress Tracking** | `bd update [id] --status [status]` | Mark `[x]` in tasks.md |
| **Context Recovery** | `bd list --json` query history | Re-read tasks.md |
| **New Work Discovery** | `bd create` + `discovered-from` link | Add to tasks.md manually |
| **Dependencies** | Automatic via `blocks` relationships | Manual coordination |

---

## Step 1: Agent Startup - Context Recovery

**When an agent starts work** (especially across sessions), recover context first.

### Query Current State

```bash
# From project root

# Find work assigned to you that's in progress
bd list --json | jq -r '.[] | select(.assignee=="[your-agent-name]" and .status=="in_progress") | "\(.id): \(.title)"'

# Find ready work for your role
bd ready --assignee [your-agent-name]

# View full context for a specific issue
bd show [issue-id]

# Check what was discovered from previous work
bd list --json | jq -r '.[] | select(.discovered_from != null) | "\(.id): \(.title) (discovered from: \(.discovered_from))"'
```

### Understand Dependency State

```bash
# View dependency tree to see where you are in the workflow
bd dep tree [epic-id]

# Find what's blocking your assigned issues
bd list --json | jq -r '.[] | select(.assignee=="[your-agent-name]" and .blocked_by != null)'
```

**Expected State:**
- Atoms should be unblocked initially
- Molecules blocked by atoms
- Organisms blocked by molecules
- Tests blocked by organisms
- Integration blocked by everything

---

## Step 2: Select Work from Ready Queue

### Smart Task Selection with Execution Planning

```bash
# Source BV helpers for intelligent task selection
source "$(dirname "$0")/bv-helpers.md"

# Determine agent context
AGENT_NAME="[your-agent-name]"  # e.g., atom-writer, database-layer-builder
ATOMIC_LEVEL="[your-level-tag]"  # e.g., atom, molecule, database, api, ui

# Get execution plan (uses BV if available, falls back to bd ready)
EXEC_PLAN=$(get_execution_plan)

if bv_available; then
    # Use graph-based selection for maximum impact
    echo "Using BV execution planning for smart task selection..."

    # Extract highest-impact actionable issue for this agent's level
    NEXT_ISSUE=$(echo "$EXEC_PLAN" | jq -r \
        --arg agent "$AGENT_NAME" \
        --arg level "$ATOMIC_LEVEL" \
        '.tracks[] | .items[] |
         select(.tags[]? == $level) |
         select(.assignee == $agent or .assignee == null) |
         {id: .id, impact: .unblocks_count, priority: .priority} |
         @json' | \
        jq -s 'sort_by(-.impact, .priority) | .[0]')

    CURRENT_ISSUE_ID=$(echo "$NEXT_ISSUE" | jq -r '.id')
    IMPACT=$(echo "$NEXT_ISSUE" | jq -r '.impact')

    if [[ -n "$CURRENT_ISSUE_ID" && "$CURRENT_ISSUE_ID" != "null" ]]; then
        echo "Selected issue: $CURRENT_ISSUE_ID (unblocks $IMPACT downstream tasks)"

        # Check for structural significance
        INSIGHTS=$(get_graph_insights)

        # Check if this issue is a bottleneck
        IS_BOTTLENECK=$(echo "$INSIGHTS" | jq -r --arg id "$CURRENT_ISSUE_ID" \
            '.bottlenecks[] | select(.id == $id) | .value')

        # Check if this issue is a keystone
        IS_KEYSTONE=$(echo "$INSIGHTS" | jq -r --arg id "$CURRENT_ISSUE_ID" \
            '.keystones[] | select(.id == $id) | .value')

        # Check if this issue is an influencer
        IS_INFLUENCER=$(echo "$INSIGHTS" | jq -r --arg id "$CURRENT_ISSUE_ID" \
            '.influencers[] | select(.id == $id) | .value')

        # Display structural warnings
        if [[ -n "$IS_BOTTLENECK" && "$IS_BOTTLENECK" != "null" ]]; then
            echo ""
            echo "⚠️  BOTTLENECK (betweenness: $IS_BOTTLENECK)"
            echo "   This issue bridges different parts of the graph."
            echo "   Completing it will unblock multiple independent work streams."
        fi

        if [[ -n "$IS_KEYSTONE" && "$IS_KEYSTONE" != "null" ]]; then
            echo ""
            echo "⚠️  CRITICAL PATH (path length: $IS_KEYSTONE)"
            echo "   This issue is on the longest dependency chain."
            echo "   Delays here will push back the entire project timeline."
        fi

        if [[ -n "$IS_INFLUENCER" && "$IS_INFLUENCER" != "null" ]]; then
            echo ""
            echo "⚠️  FOUNDATIONAL WORK (eigenvector: $IS_INFLUENCER)"
            echo "   This issue is connected to many important downstream tasks."
            echo "   Implement with extra care and comprehensive testing."
        fi
    fi
else
    # Fallback: Basic bd ready selection
    echo "BV unavailable - using basic task selection..."
    CURRENT_ISSUE_ID=$(bd ready --assignee "$AGENT_NAME" --tag "$ATOMIC_LEVEL" --limit 1 | head -1)
fi

# Verify we have work to do
if [[ -z "$CURRENT_ISSUE_ID" || "$CURRENT_ISSUE_ID" == "null" ]]; then
    echo "No actionable work found for $AGENT_NAME at $ATOMIC_LEVEL level"
    exit 0
fi

# View full details
bd show $CURRENT_ISSUE_ID
```

**Selection Criteria (BV Mode):**
1. Unblocked (appears in execution plan tracks)
2. Assigned to you (or unassigned)
3. Matches your atomic level tag
4. **Maximum impact** (unblocks the most downstream tasks)
5. Secondary sort by priority

**Selection Criteria (Fallback Mode):**
1. Unblocked (shows in `bd ready`)
2. Assigned to you
3. Matches your atomic level tag
4. First available (greedy selection)

---

## Step 3: Mark Work In Progress

```bash
# Update status to in_progress
bd update $CURRENT_ISSUE_ID --status in_progress

# Verify status updated
bd show $CURRENT_ISSUE_ID | grep "status:"
# Expected: status: in_progress
```

This signals to other agents and future sessions that you're actively working on this issue.

---

## Step 4: Implement the Work

Follow the **atomic-workflow.md** for implementation guidance specific to your atomic level:

- **Atoms:** Pure functions, zero dependencies, 1-3 tests
- **Molecules:** Compose 2-3 atoms, 2-5 tests
- **Organisms:** Use molecules + atoms, 2-8 tests per layer
- **Tests:** Write focused tests for assigned level
- **Integration:** Wire organisms, E2E verification

### Reference Relevant Content

```bash
# Read issue description for requirements
bd show $CURRENT_ISSUE_ID

# Check parent issue for context
PARENT_ID=$(bd show $CURRENT_ISSUE_ID | grep "parent:" | awk '{print $2}')
bd show $PARENT_ID

# Check related issues
bd list --related $CURRENT_ISSUE_ID
```

### Write Code

Follow standards and atomic design principles:
- Reference `{{standards/global/atomic-design.md}}`
- Reference domain-specific standards (backend/*, frontend/*)
- Respect dependency flow (downward only)
- Write tests at appropriate level

---

## Step 5: Handle Discovered Work

**When you discover new work** while implementing (e.g., realize you need another atom):

### Create Discovered Issue

```bash
# Create new issue
bd create "[Atom] New utility function needed" \
  -t atom \
  -p 3 \
  --tag atom \
  --assignee atom-writer \
  --description "Pure function for X. Discovered while implementing $CURRENT_ISSUE_ID."

NEW_ISSUE_ID=$(bd list --format=json | jq -r '.[0].id')

# Link to current work using discovered-from
bd dep add $NEW_ISSUE_ID $CURRENT_ISSUE_ID --type discovered-from

# If new issue blocks current work, set dependency
bd dep add $CURRENT_ISSUE_ID $NEW_ISSUE_ID --type blocks
```

**Decision Point:**
- **If new work is small:** Implement it now (you're unblocked)
- **If new work is large:** Mark current issue as blocked, switch to new issue
- **If new work is for another agent:** Create issue, let them handle it

### Update Current Issue if Blocked

```bash
# If discovered work blocks you
bd update $CURRENT_ISSUE_ID --status blocked --note "Blocked by $NEW_ISSUE_ID"

# Switch to new issue
CURRENT_ISSUE_ID=$NEW_ISSUE_ID
bd update $CURRENT_ISSUE_ID --status in_progress
```

---

## Step 6: Run Tests

### Execute Tests for Your Atomic Level

```bash
# Run only tests for this specific work
# (Example - adjust based on your test framework)
npm test -- --grep "your-component-name"
pytest tests/test_your_module.py
cargo test your_module

# Capture results
TEST_RESULT=$?  # 0 = pass, non-zero = fail
```

### Verify Test Count

Ensure you're writing focused tests, not comprehensive suites:

| Atomic Level | Expected Test Count |
|--------------|---------------------|
| Atom | 1-3 tests |
| Molecule | 2-5 tests |
| Organism | 2-8 tests |
| Integration | Critical paths only |

---

## Step 7: Complete the Issue

### If Tests Pass

```bash
# Close issue with completion reason
bd close $CURRENT_ISSUE_ID --reason "Implemented with $TEST_COUNT tests passing. Files: [list files created/modified]"

# Verify closure
bd list --json | jq -r '.[] | select(.id=="'$CURRENT_ISSUE_ID'") | .status'
# Expected: closed
```

### If Tests Fail

**DO NOT close the issue.**

```bash
# Add note about failure
bd update $CURRENT_ISSUE_ID --note "Tests failing: [error summary]. Investigating."

# Keep status as in_progress or set to blocked
bd update $CURRENT_ISSUE_ID --status blocked

# Create issue for blocker if needed
bd create "[Bug] Test failure in $CURRENT_ISSUE_ID" \
  -t bug \
  -p 1 \
  --description "Tests failing with error: [error]. Related to $CURRENT_ISSUE_ID"
```

**Fix the tests** or understand why they're failing before marking complete.

---

## Step 8: Unblock Dependent Work

When you close an issue, beads automatically unblocks dependent work.

### Verify Unblocking

```bash
# Check what became unblocked
bd ready

# Should now show issues that were blocked by $CURRENT_ISSUE_ID
```

**Example:**
- You close atom issue `bd-a1b2`
- Molecule issue `bd-a1b2.1` was blocked by it
- `bd ready` now shows `bd-a1b2.1`

This creates natural bottom-up flow.

---

## Step 9: Continue with Next Ready Issue

### Find Next Work

```bash
# Check ready queue again
bd ready --assignee [your-agent-name]

# Select next issue
CURRENT_ISSUE_ID="bd-yyyy"  # Next from queue

# Repeat from Step 3
```

### When No More Ready Work

```bash
# Check if everything is done
bd list --json | jq -r '.[] | select(.status!="closed")'

# If empty: All work complete!
# If not empty: Work is blocked, investigate

# Find blocked work
bd list --json | jq -r '.[] | select(.status=="blocked") | "\(.id): \(.title) - blocked by: \(.blocked_by)"'
```

---

## Step 10: Session Handoff

When ending a work session (context limit, stopping for the day, etc.):

### Document Current State

```bash
# Review in-progress work
bd list --json | jq -r '.[] | select(.status=="in_progress") | "\(.id): \(.title)"'

# Add notes to any in-progress issues
bd update [in-progress-id] --note "Session ended. Next steps: [what to do next]. Progress: [what's done]."

# Optionally set back to todo if not truly in progress
bd update [in-progress-id] --status todo
```

### Summary Output

```bash
echo "Session Summary:"
echo "==============="
echo ""
echo "Completed this session:"
bd list --json | jq -r '.[] | select(.closed_at > "'$(date -d '2 hours ago' -Iseconds)'") | "✓ \(.id): \(.title)"'

echo ""
echo "Still in progress:"
bd list --json | jq -r '.[] | select(.status=="in_progress") | "→ \(.id): \(.title)"'

echo ""
echo "Ready for next session:"
bd ready --count 5

echo ""
echo "To resume: cd agent-os/specs/[this-spec]/ && bd ready"
```

---

## Agent-Specific Workflows

### Atom Writer

```bash
# Find atom work
bd ready --tag atom --assignee atom-writer

# Atoms have no dependencies (should always be unblocked initially)
# Implement pure functions
# 1-3 tests each
# Close atoms → unblocks molecules
```

### Molecule Composer

```bash
# Find molecule work (requires atoms complete)
bd ready --tag molecule --assignee molecule-composer

# Wait for atoms to close
# Compose 2-3 atoms per molecule
# 2-5 tests per molecule
# Close molecules → unblocks organisms
```

### Database Layer Builder

```bash
# Find database organism work
bd ready --tag database --assignee database-layer-builder

# Requires molecules complete
# Use atoms + molecules for validations
# 2-8 focused tests
# Close database → unblocks API layer
```

### API Layer Builder

```bash
# Find API organism work
bd ready --tag api --assignee api-layer-builder

# Requires database layer complete
# Use database models + molecules
# 2-8 focused tests
# Close API → unblocks UI layer
```

### UI Component Builder

```bash
# Find UI organism work
bd ready --tag ui --assignee ui-component-builder

# Requires API layer complete
# Use molecules for component composition
# Call APIs from API layer
# 2-8 focused tests
# Visual verification with Playwright
```

### Test Gap Analyzer

```bash
# Find test gap work
bd ready --tag test --assignee test-gap-analyzer

# Requires all organisms complete
# Review all existing tests
# Identify gaps
# Write up to 10 additional tests
# Close → unblocks integration
```

### Integration Assembler

```bash
# Find integration work
bd ready --assignee integration-assembler

# Requires all organisms + tests complete
# Wire composition roots
# E2E verification
# Full test suite run
# This is the final issue
```

---

## Troubleshooting

### Issue: `bd ready` shows nothing

**Possible causes:**
1. Everything is blocked (check dependency tree: `bd dep tree [epic-id]`)
2. Everything is complete (check: `bd list --status closed`)
3. Blocking issue needs attention

**Solution:**
```bash
# Find blocked issues
bd list --json | jq -r '.[] | select(.blocked_by != null) | "\(.id): blocked by \(.blocked_by)"'

# Work on the blocker first
```

### Issue: Can't find your assigned work

```bash
# List all your assignments
bd list --assignee [your-agent-name]

# Check if already complete
bd list --assignee [your-agent-name] --status closed

# Check if blocked
bd list --assignee [your-agent-name] --status blocked
```

### Issue: Need to change issue details

```bash
# Update description
bd update [id] --description "New description"

# Change assignee
bd update [id] --assignee different-agent

# Change priority
bd update [id] -p 1  # Higher priority

# Add tags
bd update [id] --tag additional-tag
```

---

## Step 8: Session Summary (End of Work Session)

At the end of your work session, generate a summary of what was accomplished:

```bash
# Source BV helpers
source "$(dirname "$0")/bv-helpers.md"

if bv_available; then
    # Get session start reference (should be set at session start)
    SESSION_START=$(cat .beads/session-start-commit 2>/dev/null || echo "HEAD~1")

    echo ""
    echo "=== Session Summary ==="
    echo ""

    # Get diff since session start
    DIFF=$(get_session_diff "$SESSION_START")

    # Display accomplishments
    echo "$DIFF" | jq -r '
        "Work completed:",
        "  • \(.changes.closed_issues | length) issues closed",
        "  • \(.changes.new_issues | length) new issues discovered",
        "  • \(.changes.modified_issues | length) issues updated",
        "",
        "Closed issues:",
        (.changes.closed_issues[] | "  ✓ \(.id): \(.title)"),
        "",
        (.changes.new_issues | if length > 0 then
            "Discovered issues:",
            (.[] | "  + \(.id): \(.title)")
        else "" end)
    '

    # Warn if cycles introduced
    NEW_CYCLES=$(echo "$DIFF" | jq -r '.graph_changes.new_cycles | length')
    if [[ "$NEW_CYCLES" -gt 0 ]]; then
        echo ""
        echo "⚠️  WARNING: $NEW_CYCLES new cycles introduced during this session"
        echo "$DIFF" | jq -r '.graph_changes.new_cycles[] | "  " + (. | join(" → "))'
    fi

    # Update session start for next session
    git rev-parse HEAD > .beads/session-start-commit
else
    # Fallback: Basic summary without BV
    echo ""
    echo "=== Session Summary ==="
    echo ""
    bd list --status closed | tail -5 | while read -r line; do
        echo "  ✓ $line"
    done
fi
```

**Session Start Tracking:**

At the beginning of a work session, record the starting point:

```bash
# At session start (once per session)
git rev-parse HEAD > .beads/session-start-commit
```

This enables accurate session-to-session tracking.

---

## Success Criteria

✅ Agent queries beads on startup for context recovery
✅ Agent uses `bd ready` to find unblocked work
✅ Agent marks issues `in_progress` before starting
✅ Agent creates `discovered-from` links for new work
✅ Agent only closes issues when tests pass
✅ Agent adds notes for session handoffs
✅ Bottom-up flow naturally emerges (atoms → molecules → organisms)
✅ All issues eventually reach `closed` status

This workflow ensures agents maintain context across sessions, discover work naturally, and coordinate through the dependency graph.
