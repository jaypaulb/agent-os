# BV Parallel Execution Coordination

Use BV's track-based planning to coordinate multiple agents working simultaneously on independent work streams.

## Overview

BV's execution planner identifies **independent tracks** - work streams that can proceed in parallel without blocking each other. This enables:

- **Multi-agent coordination** - Multiple Claude Code instances working concurrently
- **Faster completion** - Parallelizable work completes in parallel time, not sequential
- **Optimal resource usage** - Utilize available agent capacity efficiently

## Track-Based Planning

BV analyzes the dependency graph to identify independent branches:

```bash
bv --robot-plan --format json | jq -r '
    "=== Parallel Tracks ===",
    "",
    (.tracks[] |
     "Track \(.track_id): \(.reason)",
     "  Items: \(.items | length)",
     "  Highest priority: \(.items[0].priority)",
     "  Total unblocks: \(.items | map(.unblocks_count) | add)",
     "")
'
```

**Example output:**
```
=== Parallel Tracks ===

Track 1: Independent branch - User authentication flow
  Items: 8
  Highest priority: 1
  Total unblocks: 12

Track 2: Independent branch - Payment processing flow
  Items: 6
  Highest priority: 1
  Total unblocks: 8

Track 3: Independent branch - Admin dashboard UI
  Items: 4
  Highest priority: 2
  Total unblocks: 3
```

These 3 tracks can all proceed simultaneously - they have no inter-dependencies.

## Multi-Agent Coordination

### Architecture

```
Orchestrator (main session)
  ├─> Agent 1 (background) → Track 1 (auth flow)
  ├─> Agent 2 (background) → Track 2 (payment flow)
  └─> Agent 3 (background) → Track 3 (admin UI)
```

Each agent:
1. Claims work from its assigned track
2. Implements issues sequentially within track
3. Updates beads as work completes
4. Reports progress to orchestrator

### Assign Agents to Tracks

```bash
# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

if ! bv_available; then
    echo "BV unavailable - parallel execution requires BV"
    echo "Falling back to sequential execution"
    exit 1
fi

# Get execution plan
PLAN=$(get_execution_plan)
TRACK_COUNT=$(echo "$PLAN" | jq -r '.tracks | length')

echo "Detected $TRACK_COUNT independent work streams"

if [[ "$TRACK_COUNT" -lt 2 ]]; then
    echo "Only one track available - parallel execution not beneficial"
    echo "Use sequential execution instead"
    exit 0
fi

# Assign one agent per track
for i in $(seq 0 $(($TRACK_COUNT - 1))); do
    TRACK=$(echo "$PLAN" | jq -r --argjson idx "$i" '.tracks[$idx]')
    TRACK_ID=$(echo "$TRACK" | jq -r '.track_id')
    TRACK_REASON=$(echo "$TRACK" | jq -r '.reason')
    ITEM_COUNT=$(echo "$TRACK" | jq -r '.items | length')

    echo ""
    echo "Track $TRACK_ID: $TRACK_REASON"
    echo "  Items: $ITEM_COUNT"
    echo "  Assign to: agent-$i"

    # List work items in this track
    echo "$TRACK" | jq -r '.items[] | "    • \(.id): \(.title)"'
done
```

### Lock Mechanism

Prevent agents from stepping on each other's work:

```bash
# Create lock directory if not exists
mkdir -p .beads/locks

# Agent claims work item
claim_work() {
    local issue_id=$1
    local agent_name=$2
    local lock_file=".beads/locks/$issue_id.lock"

    # Check if already claimed
    if [[ -f "$lock_file" ]]; then
        local current_owner=$(cat "$lock_file")
        if [[ "$current_owner" != "$agent_name" ]]; then
            echo "Issue $issue_id already claimed by $current_owner"
            return 1
        fi
    fi

    # Claim the work
    echo "$agent_name" > "$lock_file"
    bd update "$issue_id" --assignee "$agent_name" --status in_progress
    return 0
}

# Agent releases work item
release_work() {
    local issue_id=$1
    local lock_file=".beads/locks/$issue_id.lock"

    rm -f "$lock_file"
}
```

### Parallel Implementation Flow

```bash
#!/bin/bash
# parallel-orchestrator.sh

source bv-helpers.md

# Get execution plan
PLAN=$(get_execution_plan)
TRACK_COUNT=$(echo "$PLAN" | jq -r '.tracks | length')

echo "Launching $TRACK_COUNT parallel agents..."

# Store background PIDs
declare -a AGENT_PIDS

# Launch one agent per track (background processes)
for i in $(seq 0 $(($TRACK_COUNT - 1))); do
    (
        TRACK_ID=$((i + 1))
        AGENT_NAME="agent-$i"

        echo "[$AGENT_NAME] Starting on track $TRACK_ID"

        # Get items for this track
        TRACK_ITEMS=$(echo "$PLAN" | jq -r --argjson idx "$i" '.tracks[$idx].items[] | .id')

        # Process each item in track
        for item_id in $TRACK_ITEMS; do
            # Attempt to claim work
            if claim_work "$item_id" "$AGENT_NAME"; then
                echo "[$AGENT_NAME] Working on $item_id"

                # Implement the issue
                # (Actual implementation would call Claude Code agent here)
                # For now, simulate work
                sleep $((RANDOM % 10 + 5))

                # Mark complete
                bd close "$item_id" --reason "Implemented by $AGENT_NAME"
                release_work "$item_id"

                echo "[$AGENT_NAME] Completed $item_id"
            else
                echo "[$AGENT_NAME] Skipping $item_id (claimed by another agent)"
            fi
        done

        echo "[$AGENT_NAME] Track $TRACK_ID complete"
    ) &

    AGENT_PIDS[$i]=$!
done

# Wait for all agents to complete
echo ""
echo "Waiting for all agents to complete..."

for pid in "${AGENT_PIDS[@]}"; do
    wait "$pid"
done

echo ""
echo "✓ All parallel tracks completed"

# Verify completion
REMAINING=$(bd list --status open --json | jq -r '. | length')
echo "Remaining open issues: $REMAINING"
```

### Agent Script Template

Each background agent runs this script:

```bash
#!/bin/bash
# agent-worker.sh TRACK_ID AGENT_NAME

TRACK_ID=$1
AGENT_NAME=$2

source bv-helpers.md

# Get work items for this track
PLAN=$(get_execution_plan)
TRACK_ITEMS=$(echo "$PLAN" | jq -r --argjson idx "$((TRACK_ID - 1))" '.tracks[$idx].items[] | .id')

echo "[$AGENT_NAME] Assigned to track $TRACK_ID"
echo "[$AGENT_NAME] Work items: $(echo "$TRACK_ITEMS" | wc -l)"

# Process each item
for item_id in $TRACK_ITEMS; do
    # Try to claim work
    if ! claim_work "$item_id" "$AGENT_NAME"; then
        echo "[$AGENT_NAME] Skipping $item_id (claimed by another agent)"
        continue
    fi

    echo "[$AGENT_NAME] Implementing $item_id"

    # Get issue details
    ISSUE_DETAILS=$(bd show "$item_id" --json)
    ISSUE_TITLE=$(echo "$ISSUE_DETAILS" | jq -r '.title')
    ISSUE_DESCRIPTION=$(echo "$ISSUE_DETAILS" | jq -r '.description')

    # Call Claude Code agent to implement
    # (In real usage, this would be a Claude Code API call or subagent invocation)
    claude-code implement \
        --issue "$item_id" \
        --title "$ISSUE_TITLE" \
        --description "$ISSUE_DESCRIPTION" \
        --agent "$AGENT_NAME"

    # Verify tests pass
    if npm test; then
        echo "[$AGENT_NAME] Tests passed for $item_id"
        bd close "$item_id" --reason "Implemented and tested by $AGENT_NAME"
        release_work "$item_id"
    else
        echo "[$AGENT_NAME] Tests failed for $item_id - leaving in progress"
        # Keep lock - don't release until fixed
    fi
done

echo "[$AGENT_NAME] Track $TRACK_ID complete"
```

## Conflict Resolution

If two agents attempt to claim the same issue:

```bash
# Agent 1 claims first
claim_work "bd-101" "agent-1"  # Success
# Creates .beads/locks/bd-101.lock with "agent-1"

# Agent 2 attempts claim
claim_work "bd-101" "agent-2"  # Fails
# Reads .beads/locks/bd-101.lock, sees "agent-1", returns 1

# Agent 2 skips to next item
```

### Detecting Conflicts

```bash
# Check for lock/status mismatches
LOCKS=$(ls .beads/locks/*.lock 2>/dev/null | wc -l)
IN_PROGRESS=$(bd list --status in_progress --json | jq -r '. | length')

if [[ "$LOCKS" -ne "$IN_PROGRESS" ]]; then
    echo "⚠️  Lock mismatch detected - possible conflict!"
    echo "   Locks: $LOCKS, In-progress issues: $IN_PROGRESS"

    # List discrepancies
    for lock_file in .beads/locks/*.lock; do
        issue_id=$(basename "$lock_file" .lock)
        lock_owner=$(cat "$lock_file")

        bd_assignee=$(bd show "$issue_id" --json | jq -r '.assignee')

        if [[ "$lock_owner" != "$bd_assignee" ]]; then
            echo "   Conflict: $issue_id lock says $lock_owner, beads says $bd_assignee"
        fi
    done
fi
```

## Orchestration Integration

**File:** `/commands/orchestrate-tasks/orchestrate-tasks.md`

Add parallel execution option after creating orchestration.yml:

```bash
# After creating orchestration.yml

# Check if parallel execution is beneficial
if bv_available; then
    PLAN=$(get_execution_plan)
    TRACK_COUNT=$(echo "$PLAN" | jq -r '.tracks | length')

    if [[ "$TRACK_COUNT" -gt 1 ]]; then
        echo ""
        echo "Parallel execution opportunity detected:"
        echo "  $TRACK_COUNT independent work streams available"
        echo ""

        read -p "Run agents in parallel? [y/N]: " parallel_choice

        if [[ "$parallel_choice" =~ ^[Yy]$ ]]; then
            {{workflows/implementation/bv-parallel-execution}}
        else
            echo "Running agents sequentially..."
            # Continue with sequential orchestration
        fi
    else
        echo "Only one work stream - using sequential execution"
    fi
fi
```

## Performance Considerations

### When to Use Parallel Execution

✅ **Use when:**
- Multiple independent tracks (BV reports ≥2 tracks)
- Sufficient agent capacity (multiple Claude Code instances available)
- Long-running work (>2 hours estimated total)
- No shared resources (database, files)

❌ **Don't use when:**
- Single track (everything sequential)
- Limited resources (one agent available)
- Quick work (<30 minutes total)
- Heavy integration points (frequent merge conflicts)

### Speedup Calculation

```bash
# Sequential time
TOTAL_ITEMS=20
AVG_ITEM_TIME=15  # minutes
SEQUENTIAL_TIME=$((TOTAL_ITEMS * AVG_ITEM_TIME))  # 300 minutes (5 hours)

# Parallel time (3 tracks)
TRACK1_ITEMS=8
TRACK2_ITEMS=7
TRACK3_ITEMS=5

PARALLEL_TIME=$((TRACK1_ITEMS * AVG_ITEM_TIME))  # 120 minutes (2 hours)
# (Longest track determines total time)

SPEEDUP=$(echo "scale=2; $SEQUENTIAL_TIME / $PARALLEL_TIME" | bc)
echo "Estimated speedup: ${SPEEDUP}x"
```

### Overhead

Parallel execution adds overhead:
- Lock checking (~1 second per issue)
- Coordination between agents (~5% time)
- Context switching if running on same machine

**Rule of thumb:** Parallel is worth it if speedup > 1.5x

## Progress Monitoring

Monitor all agents in real-time:

```bash
#!/bin/bash
# monitor-parallel.sh

while true; do
    clear
    echo "=== Parallel Execution Monitor ==="
    echo ""

    # Show in-progress work
    echo "In Progress:"
    bd list --status in_progress --json | jq -r '.[] | "  • \(.id) (\(.assignee))"'

    # Show completed work
    CLOSED_COUNT=$(bd list --status closed --json | jq -r '. | length')
    echo ""
    echo "Completed: $CLOSED_COUNT issues"

    # Show remaining work
    OPEN_COUNT=$(bd list --status open --json | jq -r '. | length')
    echo "Remaining: $OPEN_COUNT issues"

    # Show lock status
    LOCK_COUNT=$(ls .beads/locks/*.lock 2>/dev/null | wc -l)
    echo "Active locks: $LOCK_COUNT"

    sleep 5
done
```

## Error Handling

### Agent Failure

If an agent crashes mid-work:

```bash
# Lock remains but agent is gone
# Detect: Lock file exists, but no process for that agent

# Cleanup stale locks
for lock_file in .beads/locks/*.lock; do
    issue_id=$(basename "$lock_file" .lock)
    lock_owner=$(cat "$lock_file")

    # Check if agent process still running
    if ! pgrep -f "$lock_owner" > /dev/null; then
        echo "Stale lock detected: $issue_id (agent $lock_owner not running)"

        # Release lock
        rm -f "$lock_file"

        # Reset issue status
        bd update "$issue_id" --status open --assignee ""
        echo "✓ Reset $issue_id to available"
    fi
done
```

### Deadlock Prevention

Ensure agents don't create circular waits:

- **Rule:** Agents only claim work from their assigned track
- **Rule:** Work within a track is ordered (no cycles within track)
- **Rule:** Tracks are independent (no cross-track dependencies)

BV's track planning ensures these rules hold.

## Real-World Example

### Scenario: E-commerce Feature Implementation

**Tracks identified:**
```
Track 1: User authentication (8 issues)
  bd-101 → bd-102 → bd-103 → bd-201 → bd-202 → bd-301 → bd-302 → bd-401

Track 2: Product catalog (6 issues)
  bd-104 → bd-105 → bd-203 → bd-303 → bd-304 → bd-402

Track 3: Payment processing (7 issues)
  bd-106 → bd-107 → bd-108 → bd-204 → bd-305 → bd-306 → bd-403
```

**Sequential time:**
- 21 issues × 20 min/issue = 420 minutes (7 hours)

**Parallel time:**
- Track 1: 8 issues × 20 min = 160 minutes (2h 40m)
- Track 2: 6 issues × 20 min = 120 minutes (2h)
- Track 3: 7 issues × 20 min = 140 minutes (2h 20m)
- **Total: 160 minutes (2h 40m)** (longest track)

**Speedup: 2.6x**

## Fallback

If BV unavailable, parallel execution is not possible:

```bash
if ! bv_available; then
    echo "BV unavailable - cannot identify parallel tracks"
    echo "Falling back to sequential execution"
    # Continue with sequential agent orchestration
fi
```

Sequential execution remains fully functional.

## Best Practices

1. **Verify independence** - Run `bv --robot-plan` first to confirm tracks are truly independent
2. **Assign by expertise** - Match agent capabilities to track content (frontend agent → UI track)
3. **Monitor progress** - Use monitoring script to catch stuck agents early
4. **Clean up locks** - Run stale lock cleanup after parallel execution completes
5. **Limit concurrency** - Don't launch more agents than CPU cores (typically 4-8)

## Summary

Parallel execution with BV enables:

- **Multi-agent coordination** - Use track assignments
- **Conflict prevention** - Use lock mechanism
- **Progress tracking** - Monitor all agents centrally
- **Graceful degradation** - Falls back to sequential if BV unavailable

Best for large features (≥20 issues) with clear independent tracks (≥2 tracks).
