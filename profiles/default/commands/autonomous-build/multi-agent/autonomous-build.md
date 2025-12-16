# Autonomous Build: Native Orchestrator

You are running the autonomous build orchestrator. Your job is to coordinate continuous implementation sessions by spawning implementer agents until all work is complete.

## Your Responsibilities

1. Run cross-phase regression tests when phases complete (spawn regression-verifier agents)
2. Display BV performance metrics between sessions
3. Spawn implementer agents for each session via Task tool
4. Analyze session outcomes
5. Auto-continue until work complete or user interrupts

**Mode**: Fully autonomous - you will loop continuously until all work is done or user stops you.

---

## INITIALIZATION

First, verify the project is ready:

```bash
# Verify we're at project root with Beads
if [ ! -d ".beads" ]; then
  echo "❌ No .beads/ directory found at project root"
  exit 1
fi

# Check for issues
ISSUE_COUNT=$(bd list --format json 2>/dev/null | jq '. | length' || echo "0")
if [ "$ISSUE_COUNT" -eq 0 ]; then
  echo "❌ No Beads issues found"
  echo "Run /autonomous-plan or /create-tasks first"
  exit 1
fi

# Initialize session tracking
mkdir -p .beads 2>/dev/null || true
echo "0" > .beads/iteration-count 2>/dev/null || true
touch .beads/last-checked-phases 2>/dev/null || true

echo "✓ Autonomous Build Ready"
echo "  Issues: $ISSUE_COUNT"
echo "  Mode: Fully autonomous"
echo ""
echo "Starting continuous build loop..."
echo "Say 'stop' or 'pause' anytime to interrupt"
```

---

## MAIN LOOP

**You will now loop through sessions continuously.** After each session completes, automatically continue to the next session after 3 seconds. Loop until all work is complete or user interrupts.

For each iteration:

### STEP 1: Check Session Number

```bash
ITERATION=$(($(cat .beads/iteration-count 2>/dev/null || echo 0) + 1))
echo "$ITERATION" > .beads/iteration-count

echo ""
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "  SESSION $ITERATION"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
```

### STEP 2: Cross-Phase Regression Testing

Check if any phase epic just completed. If so, spawn regression-verifier agents to test random issues from each completed phase.

```bash
# Check for newly completed phases
CURRENT_PHASES=$(bd list --type epic --status closed --format json 2>/dev/null | jq -r '.[].id' | sort || echo "")
LAST_CHECKED=$(cat .beads/last-checked-phases 2>/dev/null || echo "")
NEW_PHASES=$(comm -13 <(echo "$LAST_CHECKED") <(echo "$CURRENT_PHASES") 2>/dev/null || echo "")

if [ -n "$NEW_PHASES" ] && [ "$NEW_PHASES" != "" ]; then
  echo "=== Cross-Phase Regression Testing ==="
  echo "🎉 Phase(s) completed! Running regression tests..."
  echo ""
fi
```

**If new phases were completed**, for each completed phase in `$CURRENT_PHASES`:

1. Get phase label and closed issues:
   ```bash
   PHASE_ID="[phase-epic-id]"
   PHASE_LABEL=$(bd show "$PHASE_ID" --format json 2>/dev/null | jq -r '.labels[]' | grep "^phase-" | head -1)
   CLOSED_IN_PHASE=$(bd list --status closed --label "$PHASE_LABEL" --format json 2>/dev/null | jq -r '.[].id')
   TEST_ISSUE=$(echo "$CLOSED_IN_PHASE" | shuf -n 1)
   ```

2. **Spawn regression-verifier agent** via Task tool:
   ```
   Use Task tool with:
   - subagent_type: "Explore" (for now, until regression-verifier is registered)
   - description: "Verify issue $TEST_ISSUE"
   - prompt: "You are the regression-verifier agent. Verify that issue $TEST_ISSUE still works correctly.

   Follow the workflow at:
   {{@agent-os/workflows/implementation/verification/regression-verification.md}}

   Test this specific issue: $TEST_ISSUE
   Project root: $(pwd)

   Report PASS or FAIL at the end."
   ```

3. **Check result**: If agent reports FAIL/regression:
   ```bash
   bd update "$TEST_ISSUE" --status open --note "Regression detected during autonomous build"

   read -p "Regression detected. Continue build? [y/N]: " continue_build
   if [[ ! "$continue_build" =~ ^[Yy]$ ]]; then
     exit 1
   fi
   ```

4. Update tracking:
   ```bash
   echo "$CURRENT_PHASES" > .beads/last-checked-phases
   ```

### STEP 3: Performance Analysis (BV Metrics)

Display graph intelligence if BV is available:

```bash
echo "=== Performance Analysis ==="
echo ""

if command -v bv &> /dev/null; then
  INSIGHTS=$(bv --robot-insights --format json 2>/dev/null || echo "{}")

  # Show bottlenecks
  echo "$INSIGHTS" | jq -r 'if .bottlenecks | length > 0 then "Bottlenecks:", (.bottlenecks[:3][] | "  • \(.id): \(.title) (betweenness: \(.value))") else "" end'

  # Show keystones
  echo "$INSIGHTS" | jq -r 'if .keystones | length > 0 then "", "Keystones:", (.keystones[:3][] | "  • \(.id): \(.title) (path length: \(.value))") else "" end'

  # Check cycles
  CYCLE_COUNT=$(echo "$INSIGHTS" | jq '.cycles | length' 2>/dev/null || echo "0")
  if [[ "$CYCLE_COUNT" -gt 0 ]]; then
    echo ""
    echo "⚠️  WARNING: $CYCLE_COUNT circular dependencies detected"
  fi
else
  echo "BV not available - skipping graph analysis"
fi

echo ""
```

### STEP 4: Check Ready Work

Before spawning implementer, verify work exists:

```bash
READY_COUNT=$(bd ready --format json 2>/dev/null | jq '. | length' || echo "0")
echo "Ready work: $READY_COUNT issues"

if [[ "$READY_COUNT" -eq 0 ]]; then
  OPEN_COUNT=$(bd list --status todo --format json 2>/dev/null | jq '. | length' || echo "0")

  if [[ "$OPEN_COUNT" -eq 0 ]]; then
    echo ""
    echo "🎉 ALL ISSUES COMPLETE!"
    TOTAL=$(bd list --format json 2>/dev/null | jq '. | length' || echo "0")
    CLOSED=$(bd list --status closed --format json 2>/dev/null | jq '. | length' || echo "0")
    echo "  Total: $TOTAL | Completed: $CLOSED | Sessions: $ITERATION"
    exit 0
  else
    echo "⚠️  All remaining issues are blocked"
    exit 1
  fi
fi
```

### STEP 5: Spawn Implementation Session

Record session start and spawn implementer agent:

```bash
# Record start for diff tracking
git rev-parse HEAD > .beads/last-iteration-commit 2>/dev/null

echo "=== Spawning Implementation Agent ==="
echo ""
```

**Now spawn implementer agent via Task tool:**

Use Task tool with:
- **subagent_type**: `"general-purpose"`
- **description**: `"Implement ready issues (iteration $ITERATION)"`
- **prompt**:

```
You are the implementer agent in an autonomous build (session $ITERATION).

Your task: Implement ONE issue from ready work, test it, and close it.

## Process:

1. **Find ready work**:
   ```bash
   bd ready --limit 5
   ```

2. **Select highest-priority issue** (or use BV if available):
   ```bash
   # If BV available:
   bv --robot-plan --format json | jq -r '.tracks[0].items[0].id'

   # Otherwise first ready issue:
   bd ready --limit 1 --format json | jq -r '.[0].id'
   ```

3. **Claim the issue**:
   ```bash
   bd update <issue-id> --status in_progress
   ```

4. **Implement following atomic design principles**:
   - Read issue details: `bd show <issue-id>`
   - Read spec context (if spec label exists)
   - Implement the feature
   - Write tests
   - Run tests and verify they pass

5. **Follow the implement-tasks workflow**:
   {{@agent-os/workflows/implementation/implement-tasks.md}}

   **IMPORTANT**: Monitor your context! If approaching limits:
   - Commit partial work
   - Update issue with progress note
   - Return control to orchestrator
   - (See "Context Management" section in implement-tasks.md)

6. **Close issue when complete**:
   ```bash
   bd update <issue-id> --status closed
   bd comment <issue-id> "Implementation complete. Tests passing."
   ```

7. **Commit and push**:
   ```bash
   git add [modified-files]
   git commit -m "Implement [feature-name]"
   git push
   ```

8. **Write session summary** to META issue:
   ```bash
   META_ID=$(cat .beads_project.json | jq -r '.meta_issue_id' 2>/dev/null || echo "")
   if [ -n "$META_ID" ]; then
     bd comment "$META_ID" "Session $ITERATION: Completed <issue-id> - [title]"
   fi
   ```

9. **Return control**: You're done. Orchestrator will continue.

Project root: $(pwd)
Target: 1 issue per session (quality over quantity)
```

**Wait for agent to complete.**

### STEP 6: Session Analysis

After agent returns, analyze what changed:

```bash
echo ""
echo "=== Session Analysis ==="
echo ""

START_COMMIT=$(cat .beads/last-iteration-commit 2>/dev/null || echo "HEAD")
CURRENT_COMMIT=$(git rev-parse HEAD 2>/dev/null || echo "HEAD")

# Check for in-progress issues (early exit)
IN_PROGRESS_COUNT=$(bd list --status in_progress --format json 2>/dev/null | jq '. | length' || echo "0")

if [[ "$IN_PROGRESS_COUNT" -gt 0 ]]; then
  echo "⚠️  Detected early session exit (context limits)"
  echo "In-progress issues:"
  bd list --status in_progress --format json 2>/dev/null | jq -r '.[] | "  ⏳ \(.id): \(.title)"'
  echo ""
  echo "This is normal. Next session will continue."
  echo ""
fi

# Show accomplishments
if command -v bv &> /dev/null && [ "$START_COMMIT" != "$CURRENT_COMMIT" ]; then
  DIFF=$(bv --robot-diff --diff-since "$START_COMMIT" --format json 2>/dev/null || echo "{}")

  CLOSED=$(echo "$DIFF" | jq '.changes.closed_issues | length' 2>/dev/null || echo "0")
  NEW=$(echo "$DIFF" | jq '.changes.new_issues | length' 2>/dev/null || echo "0")

  echo "Accomplishments:"
  echo "  • $CLOSED issues closed"
  echo "  • $NEW new issues discovered"

  if [[ "$CLOSED" -gt 0 ]]; then
    echo ""
    echo "Closed:"
    echo "$DIFF" | jq -r '.changes.closed_issues[] | "  ✓ \(.id): \(.title)"' 2>/dev/null
  fi
fi

echo ""
```

### STEP 7: Auto-Continue

Wait 3 seconds then loop to next session:

```bash
echo "=== Continuation ==="
echo ""
echo "Continuing to next session in 3 seconds..."
echo "(Say 'stop' or 'pause' to interrupt)"
echo ""
sleep 3
```

**Then loop back to STEP 1** - start next iteration.

---

## How to Run This

You (Claude) are the orchestrator. When user runs `/autonomous-build`:

1. Run the INITIALIZATION bash commands
2. Enter the MAIN LOOP
3. For each iteration:
   - Run the bash commands for state checking
   - Spawn agents via Task tool at the appropriate points
   - Wait for agents to complete
   - Analyze results
   - Auto-continue after 3 seconds
4. Loop until all work complete or user interrupts

**You manage the loop** - after each session, wait 3 seconds and start the next one automatically.

**User can interrupt** anytime by saying "stop", "pause", or pressing Ctrl+C in their terminal.

**State is preserved** in Beads, so if interrupted, user can resume by running `/autonomous-build` again.
