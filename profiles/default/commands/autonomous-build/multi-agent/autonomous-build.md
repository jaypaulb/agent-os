# Autonomous Build: Native Claude Code Orchestrator

You are the autonomous build orchestrator running continuous implementation sessions using native agent-os workflows.

This command:
1. Runs cross-phase regression tests when phases complete
2. Displays BV performance metrics between sessions
3. Spawns implementer agents via Task tool for each iteration
4. Analyzes session outcomes
5. Auto-continues until all work is complete or user interrupts

**Prerequisites**:
- Beads issues must be created (run `/autonomous-plan` or `/create-tasks` first)
- Project at root directory where `.beads/` exists
- BV available for graph intelligence (optional but recommended)

---

## PHASE 1: Initialization and Verification

Verify Beads is ready and work exists:

```bash
# Verify we're at project root with Beads
if [ ! -d ".beads" ]; then
  echo "❌ No .beads/ directory found at project root"
  echo "   Please run from project root after creating issues"
  echo ""
  echo "Run one of these first:"
  echo "  /autonomous-plan  - Create specs + tasks"
  echo "  /create-tasks     - Create tasks for existing specs"
  exit 1
fi

echo "✓ Beads initialized at project root"

# Check for issues
ISSUE_COUNT=$(bd list --format json 2>/dev/null | jq '. | length' || echo "0")
echo "Found $ISSUE_COUNT Beads issues"

if [ "$ISSUE_COUNT" -eq 0 ]; then
  echo ""
  echo "❌ No Beads issues found"
  echo "   Run /autonomous-plan or /create-tasks first"
  exit 1
fi

# Initialize session tracking
echo "0" > .beads/iteration-count 2>/dev/null || true
touch .beads/last-checked-phases 2>/dev/null || true

# Source BV helpers if available
if [ -f "agent-os/profiles/default/workflows/implementation/bv-helpers.md" ]; then
  source agent-os/profiles/default/workflows/implementation/bv-helpers.md
fi

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  Autonomous Build Ready"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Mode: Fully autonomous"
echo "Continues: Until all work complete or Ctrl+C"
echo "Regression testing: Cross-phase (when phase completes)"
echo ""
echo "Press Ctrl+C anytime to pause"
echo ""
echo "Starting in 3 seconds..."
sleep 3
```

---

## PHASE 2: Main Orchestration Loop

Run continuous sessions until work complete:

```bash
while true; do
  ITERATION=$(($(cat .beads/iteration-count 2>/dev/null || echo 0) + 1))
  echo "$ITERATION" > .beads/iteration-count

  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  SESSION $ITERATION"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""

  # ==================================================
  # STEP 1: Cross-Phase Regression Testing
  # ==================================================

  echo "=== Step 1: Regression Testing ==="
  echo ""

  # Check if any phase epic just completed
  CURRENT_PHASES=$(bd list --type epic --status closed --format json 2>/dev/null | jq -r '.[].id' | sort || echo "")
  LAST_CHECKED=$(cat .beads/last-checked-phases 2>/dev/null || echo "")

  # Find newly completed phases
  NEW_PHASES=$(comm -13 <(echo "$LAST_CHECKED") <(echo "$CURRENT_PHASES") 2>/dev/null || echo "")

  if [ -n "$NEW_PHASES" ] && [ "$NEW_PHASES" != "" ]; then
    echo "🎉 Phase(s) completed since last check!"
    echo ""
    echo "Running cross-phase regression tests..."
    echo ""

    # For each completed phase, test 1 random issue
    REGRESSION_FOUND=false

    for PHASE_ID in $CURRENT_PHASES; do
      if [ -z "$PHASE_ID" ]; then
        continue
      fi

      PHASE_TITLE=$(bd show "$PHASE_ID" --format json 2>/dev/null | jq -r '.title' || echo "Unknown")
      echo "Testing Phase: $PHASE_TITLE ($PHASE_ID)"

      # Get all closed issues in this phase
      PHASE_LABEL=$(bd show "$PHASE_ID" --format json 2>/dev/null | jq -r '.labels[]' | grep "^phase-" | head -1 || echo "")

      if [ -z "$PHASE_LABEL" ]; then
        echo "  ⚠️  No phase label found, skipping"
        continue
      fi

      CLOSED_IN_PHASE=$(bd list --status closed --label "$PHASE_LABEL" --format json 2>/dev/null | jq -r '.[].id' || echo "")

      if [ -z "$CLOSED_IN_PHASE" ]; then
        echo "  No closed issues yet in this phase"
        continue
      fi

      # Randomly select 1 issue
      TEST_ISSUE=$(echo "$CLOSED_IN_PHASE" | shuf -n 1)
      TEST_TITLE=$(bd show "$TEST_ISSUE" --format json 2>/dev/null | jq -r '.title' || echo "Unknown")

      echo "  Testing: $TEST_ISSUE - $TEST_TITLE"

      # TODO: Spawn regression-verifier agent via Task tool
      # For now, we'll skip actual verification and just log
      echo "  ⚠️  Regression verification not yet implemented (coming soon)"
      echo "  Would test: $TEST_ISSUE via Playwright"

      # Placeholder for regression detection
      # if regression detected:
      #   bd update "$TEST_ISSUE" --status open --note "Regression detected during autonomous build"
      #   REGRESSION_FOUND=true
      #   echo "  ❌ REGRESSION DETECTED"

      echo "  ✓ Verification passed (placeholder)"
      echo ""
    done

    # Update tracking
    echo "$CURRENT_PHASES" > .beads/last-checked-phases

    if [ "$REGRESSION_FOUND" = true ]; then
      echo ""
      echo "❌ Regressions detected in cross-phase testing"
      read -p "Continue build despite regressions? [y/N]: " continue_build
      if [[ ! "$continue_build" =~ ^[Yy]$ ]]; then
        echo "Build paused. Fix regressions first."
        exit 1
      fi
    fi
  else
    echo "No new phases completed. Skipping regression tests."
  fi

  echo ""

  # ==================================================
  # STEP 2: Performance Analysis (BV Metrics)
  # ==================================================

  echo "=== Step 2: Performance Analysis ==="
  echo ""

  if command -v bv &> /dev/null 2>&1; then
    INSIGHTS=$(bv --robot-insights --format json 2>/dev/null || echo "{}")

    # Bottlenecks
    BOTTLENECK_COUNT=$(echo "$INSIGHTS" | jq '.bottlenecks | length' 2>/dev/null || echo "0")
    if [[ "$BOTTLENECK_COUNT" -gt 0 ]]; then
      echo "Bottlenecks (bridge multiple work streams):"
      echo "$INSIGHTS" | jq -r '.bottlenecks[:3][] | "  • \(.id): \(.title) (betweenness: \(.value))"' 2>/dev/null || echo "  None"
      echo ""
    fi

    # Keystones
    KEYSTONE_COUNT=$(echo "$INSIGHTS" | jq '.keystones | length' 2>/dev/null || echo "0")
    if [[ "$KEYSTONE_COUNT" -gt 0 ]]; then
      echo "Keystones (critical path):"
      echo "$INSIGHTS" | jq -r '.keystones[:3][] | "  • \(.id): \(.title) (path length: \(.value))"' 2>/dev/null || echo "  None"
      echo ""
    fi

    # Influencers
    INFLUENCER_COUNT=$(echo "$INSIGHTS" | jq '.influencers | length' 2>/dev/null || echo "0")
    if [[ "$INFLUENCER_COUNT" -gt 0 ]]; then
      echo "Influencers (foundational work):"
      echo "$INSIGHTS" | jq -r '.influencers[:3][] | "  • \(.id): \(.title) (eigenvector: \(.value))"' 2>/dev/null || echo "  None"
      echo ""
    fi

    # Check for cycles
    CYCLES=$(echo "$INSIGHTS" | jq '.cycles' 2>/dev/null || echo "[]")
    CYCLE_COUNT=$(echo "$CYCLES" | jq 'length' 2>/dev/null || echo "0")
    if [[ "$CYCLE_COUNT" -gt 0 ]]; then
      echo "⚠️  WARNING: $CYCLE_COUNT circular dependencies detected"
      echo "$CYCLES" | jq -r '.[] | "  Cycle: " + (. | join(" → "))' 2>/dev/null || true
      echo ""
    fi
  else
    echo "BV not available - skipping graph analysis"
    echo "(Install BV for bottleneck/keystone/influencer insights)"
  fi

  echo ""

  # ==================================================
  # STEP 3: Spawn Implementation Session
  # ==================================================

  echo "=== Step 3: Spawning Implementation Agent ==="
  echo ""

  # Record session start for diff tracking
  git rev-parse HEAD > .beads/last-iteration-commit 2>/dev/null || echo "HEAD" > .beads/last-iteration-commit

  # Check ready work count
  READY_COUNT=$(bd ready --format json 2>/dev/null | jq '. | length' || echo "0")
  echo "Ready work: $READY_COUNT issues"

  if [[ "$READY_COUNT" -eq 0 ]]; then
    echo ""
    echo "No ready work found."

    # Check if all done or all blocked
    OPEN_COUNT=$(bd list --status todo --format json 2>/dev/null | jq '. | length' || echo "0")

    if [[ "$OPEN_COUNT" -eq 0 ]]; then
      echo ""
      echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
      echo "  🎉 ALL ISSUES COMPLETE!"
      echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
      echo ""

      TOTAL=$(bd list --format json 2>/dev/null | jq '. | length' || echo "0")
      CLOSED=$(bd list --status closed --format json 2>/dev/null | jq '. | length' || echo "0")

      echo "Final Stats:"
      echo "  Total issues: $TOTAL"
      echo "  Completed: $CLOSED"
      echo "  Sessions: $ITERATION"
      echo ""
      echo "Autonomous build complete!"
      exit 0
    else
      echo ""
      echo "⚠️  All remaining issues are blocked"
      echo ""
      echo "Blocked issues:"
      bd list --status blocked --format json 2>/dev/null | jq -r '.[] | "  • \(.id): \(.title)"' || echo "  (none)"
      echo ""
      echo "Manual intervention needed. Exiting autonomous build."
      exit 1
    fi
  fi

  echo ""
  echo "Spawning implementer agent..."
  echo ""

  # TODO: Spawn implementer agent via Task tool
  # This will be implemented using Task tool to spawn the implementer agent
  # The agent will:
  # 1. Follow implement-with-beads.md workflow
  # 2. Query bd ready for work
  # 3. Select highest-impact issue
  # 4. Implement, test, close
  # 5. Return to orchestrator

  echo "⚠️  Agent spawning not yet implemented (coming soon)"
  echo "Would spawn: implementer agent with iteration=$ITERATION"
  echo ""

  # Placeholder: Simulate session completion
  echo "Simulating session completion..."
  sleep 2

  # ==================================================
  # STEP 4: Session Analysis
  # ==================================================

  echo ""
  echo "=== Step 4: Session Analysis ==="
  echo ""

  START_COMMIT=$(cat .beads/last-iteration-commit 2>/dev/null || echo "HEAD")
  CURRENT_COMMIT=$(git rev-parse HEAD 2>/dev/null || echo "HEAD")

  if command -v bv &> /dev/null && [ "$START_COMMIT" != "$CURRENT_COMMIT" ]; then
    DIFF=$(bv --robot-diff --diff-since "$START_COMMIT" --format json 2>/dev/null || echo "{}")

    CLOSED=$(echo "$DIFF" | jq '.changes.closed_issues | length' 2>/dev/null || echo "0")
    NEW=$(echo "$DIFF" | jq '.changes.new_issues | length' 2>/dev/null || echo "0")
    MODIFIED=$(echo "$DIFF" | jq '.changes.modified_issues | length' 2>/dev/null || echo "0")

    echo "Accomplishments:"
    echo "  • $CLOSED issues closed"
    echo "  • $NEW new issues discovered"
    echo "  • $MODIFIED issues updated"
    echo ""

    if [[ "$CLOSED" -gt 0 ]]; then
      echo "Closed:"
      echo "$DIFF" | jq -r '.changes.closed_issues[] | "  ✓ \(.id): \(.title)"' 2>/dev/null || echo "  (details unavailable)"
    fi
  else
    echo "No changes detected (or BV unavailable for diff tracking)"
  fi

  echo ""

  # ==================================================
  # STEP 5: Continuation Decision
  # ==================================================

  echo "=== Step 5: Continuation Decision ==="
  echo ""

  # Recheck ready work
  READY_COUNT=$(bd ready --format json 2>/dev/null | jq '. | length' || echo "0")
  BLOCKED_COUNT=$(bd list --status blocked --format json 2>/dev/null | jq '. | length' || echo "0")
  OPEN_COUNT=$(bd list --status todo --format json 2>/dev/null | jq '. | length' || echo "0")

  echo "Work Remaining:"
  echo "  Ready: $READY_COUNT"
  echo "  Blocked: $BLOCKED_COUNT"
  echo "  Open: $OPEN_COUNT"
  echo ""

  if [[ "$READY_COUNT" -eq 0 ]]; then
    if [[ "$OPEN_COUNT" -eq 0 ]]; then
      echo "🎉 All issues complete!"
      exit 0
    else
      echo "⚠️  No ready work. All remaining issues blocked."
      exit 1
    fi
  fi

  # Auto-continue
  echo "Continuing in 3 seconds... (Press Ctrl+C to stop)"
  sleep 3

done
```

---

## Notes

**Current Status**: This orchestrator provides the framework for autonomous builds but requires:
1. Integration with regression-verifier agent (Task tool spawning)
2. Integration with implementer agent (Task tool spawning)
3. Playwright MCP tools configuration for browser testing

**Placeholders**:
- Regression verification currently logs intention but doesn't spawn agent
- Implementation session currently simulates without spawning real agent
- These will be implemented in subsequent phases

**Usage**:
```bash
cd /path/to/project
/autonomous-build
```

The orchestrator will run until:
- All issues are complete (exit 0)
- All issues are blocked (exit 1)
- User interrupts with Ctrl+C

State is preserved in Beads, so you can resume anytime by running `/autonomous-build` again.
