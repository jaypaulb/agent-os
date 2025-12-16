# State Management Workflow

This workflow defines how the autonomous-build orchestrator initializes, updates, and recovers state.

## State Initialization

On first run of `/autonomous-build`, the orchestrator creates the state directory:

```bash
# Create state directory structure
mkdir -p .beads/autonomous-state/locks
mkdir -p .beads/autonomous-state/learning
mkdir -p .beads/autonomous-state/failures

# Initialize work-queue.json
cat > .beads/autonomous-state/work-queue.json <<'EOF'
{
  "ready": [],
  "in_progress": [],
  "completed": [],
  "failed": [],
  "blocked": []
}
EOF

# Initialize agent-pool.json
cat > .beads/autonomous-state/agent-pool.json <<'EOF'
{
  "max_agents": 5,
  "active": [],
  "available_slots": [1, 2, 3, 4, 5],
  "heartbeat_interval": 10
}
EOF

# Initialize improvements.json
cat > .beads/autonomous-state/learning/improvements.json <<'EOF'
{
  "common_errors": [],
  "best_practices": [],
  "conflict_patterns": [],
  "organism_specific": {
    "database-layer": [],
    "api-layer": [],
    "ui-component": []
  },
  "metrics": {
    "total_iterations": 0,
    "total_organisms": 0,
    "success_rate_trend": []
  }
}
EOF

echo "✓ State initialized"
```

## Loading Ready Work

After initialization, populate work queue from orchestration.yml or `bd ready`:

```bash
# Check if orchestration.yml exists
if [ -f ".beads/autonomous-state/orchestration.yml" ]; then
  # Use orchestration.yml (from /orchestrate-tasks --headless)
  READY_ORGANISMS=$(yq eval '.beads[] | select(.status == "todo") | .id' .beads/autonomous-state/orchestration.yml)

  # Populate ready queue
  for ORGANISM_ID in $READY_ORGANISMS; then
    ORGANISM_DATA=$(yq eval ".beads[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/orchestration.yml -o json)

    # Add to work-queue.json ready array
    jq ".ready += [$ORGANISM_DATA]" .beads/autonomous-state/work-queue.json > /tmp/wq.json
    mv /tmp/wq.json .beads/autonomous-state/work-queue.json
  done
else
  # Fallback: Use bd ready
  READY_JSON=$(bd ready --format json 2>/dev/null || echo "[]")

  # Transform to work-queue format
  echo "$READY_JSON" | jq '{
    ready: [.[] | {
      id: .id,
      title: .title,
      type: .type,
      assignee: "general-purpose",
      priority: 5,
      predicted_files: []
    }],
    in_progress: [],
    completed: [],
    failed: [],
    blocked: []
  }' > .beads/autonomous-state/work-queue.json
fi

READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
echo "✓ Loaded $READY_COUNT organisms into queue"
```

## State Recovery

If interrupted (Ctrl+C, crash), recover state on next run:

```bash
# Check if state exists
if [ -f ".beads/autonomous-state/work-queue.json" ]; then
  echo "=== State Recovery ==="
  echo ""

  # Check for in_progress organisms (agents were killed)
  IN_PROGRESS=$(jq '.in_progress' .beads/autonomous-state/work-queue.json)
  IN_PROGRESS_COUNT=$(echo "$IN_PROGRESS" | jq 'length')

  if [[ "$IN_PROGRESS_COUNT" -gt 0 ]]; then
    echo "Found $IN_PROGRESS_COUNT interrupted organisms:"
    echo "$IN_PROGRESS" | jq -r '.[] | "  • \(.id): \(.title) (slot \(.slot))"'
    echo ""

    # Move in_progress → ready (retry)
    jq '.ready += .in_progress | .in_progress = []' .beads/autonomous-state/work-queue.json > /tmp/wq.json
    mv /tmp/wq.json .beads/autonomous-state/work-queue.json

    echo "✓ Moved to ready queue for retry"
  fi

  # Clean up stale locks
  rm -f .beads/autonomous-state/locks/*.lock
  echo "✓ Cleaned stale locks"

  # Reset agent pool
  jq '.active = [] | .available_slots = [1, 2, 3, 4, 5]' .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json
  echo "✓ Reset agent pool"

  # Show current state
  READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
  COMPLETED_COUNT=$(jq '.completed | length' .beads/autonomous-state/work-queue.json)
  FAILED_COUNT=$(jq '.failed | length' .beads/autonomous-state/work-queue.json)

  echo ""
  echo "Current state:"
  echo "  Ready: $READY_COUNT"
  echo "  Completed: $COMPLETED_COUNT"
  echo "  Failed: $FAILED_COUNT"
  echo ""
fi
```

## Claiming an Organism

When agent slot is available, claim next organism from ready queue:

```bash
claim_organism() {
  ORGANISM_ID=$1
  SLOT=$2
  TASK_ID=$3

  # Create lock file
  LOCK_FILE=".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

  # Check if already locked
  if [ -f "$LOCK_FILE" ]; then
    echo "❌ Organism $ORGANISM_ID already claimed"
    return 1
  fi

  # Create lock
  echo "$SLOT|$TASK_ID|$(date -Iseconds)" > "$LOCK_FILE"

  # Move organism from ready → in_progress
  ORGANISM_DATA=$(jq ".ready[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)

  # Add claim metadata
  CLAIMED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
    claimed_at: \"$(date -Iseconds)\",
    slot: $SLOT,
    task_id: \"$TASK_ID\",
    attempt: 1
  }")

  # Update work queue
  jq "
    .in_progress += [$CLAIMED_DATA] |
    .ready = [.ready[] | select(.id != \"$ORGANISM_ID\")]
  " .beads/autonomous-state/work-queue.json > /tmp/wq.json
  mv /tmp/wq.json .beads/autonomous-state/work-queue.json

  echo "✓ Claimed $ORGANISM_ID in slot $SLOT"
  return 0
}
```

## Releasing an Organism

When agent completes (success, fail, or blocked):

```bash
release_organism() {
  ORGANISM_ID=$1
  NEW_STATUS=$2  # completed, failed, blocked
  METADATA=$3     # JSON with status-specific data

  # Remove lock
  rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

  # Get organism from in_progress
  ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)

  # Add release metadata
  RELEASED_DATA=$(echo "$ORGANISM_DATA" | jq ". + $METADATA")

  # Update work queue
  jq "
    .$NEW_STATUS += [$RELEASED_DATA] |
    .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
  " .beads/autonomous-state/work-queue.json > /tmp/wq.json
  mv /tmp/wq.json .beads/autonomous-state/work-queue.json

  echo "✓ Released $ORGANISM_ID → $NEW_STATUS"
}

# Example: Mark completed
release_organism "bd-org-123" "completed" "{
  \"completed_at\": \"$(date -Iseconds)\",
  \"commit_sha\": \"$(git rev-parse HEAD)\"
}"

# Example: Mark failed
release_organism "bd-org-456" "failed" "{
  \"attempts\": 3,
  \"last_error\": \"Tests failed\",
  \"analysis_issue_id\": \"bd-789\",
  \"failed_at\": \"$(date -Iseconds)\"
}"

# Example: Mark blocked
release_organism "bd-org-789" "blocked" "{
  \"blocked_by\": \"bd-org-456\",
  \"reason\": \"Merge conflict\",
  \"blocked_at\": \"$(date -Iseconds)\"
}"
```

## Agent Pool Management

Track active agents in slots 1-5:

```bash
# Find free slot
find_free_slot() {
  jq -r '.available_slots[0]' .beads/autonomous-state/agent-pool.json
}

# Add agent to pool
add_agent() {
  SLOT=$1
  AGENT_TYPE=$2
  ORGANISM_ID=$3
  TASK_ID=$4

  AGENT_DATA=$(jq -n "{
    slot: $SLOT,
    agent_type: \"$AGENT_TYPE\",
    organism_id: \"$ORGANISM_ID\",
    task_id: \"$TASK_ID\",
    started_at: \"$(date -Iseconds)\",
    last_checked: \"$(date -Iseconds)\",
    status: \"running\"
  }")

  jq "
    .active += [$AGENT_DATA] |
    .available_slots = [.available_slots[] | select(. != $SLOT)]
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  echo "✓ Agent in slot $SLOT: $AGENT_TYPE → $ORGANISM_ID"
}

# Remove agent from pool
remove_agent() {
  SLOT=$1

  jq "
    .active = [.active[] | select(.slot != $SLOT)] |
    .available_slots += [$SLOT] |
    .available_slots |= sort
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  echo "✓ Freed slot $SLOT"
}

# Get active agent count
get_active_count() {
  jq '.active | length' .beads/autonomous-state/agent-pool.json
}

# Get max agents (from config or default)
get_max_agents() {
  if [ -f ".beads/autonomous-config.yml" ]; then
    yq eval '.session.max_agents // 5' .beads/autonomous-config.yml
  else
    echo "5"
  fi
}
```

## Learning System

Extract errors and update improvements:

```bash
# Add error pattern
add_error_pattern() {
  PATTERN=$1
  FIX=$2
  CATEGORY=$3

  # Check if pattern exists
  EXISTING=$(jq ".common_errors[] | select(.pattern == \"$PATTERN\")" .beads/autonomous-state/learning/improvements.json)

  if [ -n "$EXISTING" ]; then
    # Increment seen_count
    jq "
      .common_errors = [.common_errors[] |
        if .pattern == \"$PATTERN\" then
          .seen_count += 1 |
          .last_seen = \"$(date -Iseconds)\"
        else
          .
        end
      ]
    " .beads/autonomous-state/learning/improvements.json > /tmp/improvements.json
  else
    # Add new pattern
    NEW_PATTERN=$(jq -n "{
      pattern: \"$PATTERN\",
      fix: \"$FIX\",
      category: \"$CATEGORY\",
      seen_count: 1,
      first_seen: \"$(date -Iseconds)\",
      last_seen: \"$(date -Iseconds)\",
      trending: \"stable\"
    }")

    jq ".common_errors += [$NEW_PATTERN]" .beads/autonomous-state/learning/improvements.json > /tmp/improvements.json
  fi

  mv /tmp/improvements.json .beads/autonomous-state/learning/improvements.json
}

# Get improvements for agent prompt injection
get_improvements_for_prompt() {
  jq -r '
    .common_errors |
    group_by(.category) |
    map({
      category: .[0].category,
      patterns: [.[] | {pattern: .pattern, fix: .fix}] | .[0:3]
    }) |
    .[] |
    "\(.category) issues:",
    (.patterns[] | "  ❌ \(.pattern)\n  ✅ \(.fix)")
  ' .beads/autonomous-state/learning/improvements.json
}
```

## Usage in Orchestrator

The orchestrator uses these functions in its main loop:

```markdown
## INITIALIZATION
1. Check if state exists
2. If not: initialize state files
3. If yes: recover from interruption
4. Load ready work into queue

## MAIN LOOP
while work_queue or active_agents:
  1. Check for free slots
  2. If free slots and ready work:
     - claim_organism()
     - spawn agent (background)
     - add_agent() to pool
  3. For each active agent:
     - Check status (non-blocking)
     - If completed:
       - Validate work
       - If success: release_organism("completed")
       - If fail: extract errors, release_organism("failed" or "blocked")
       - remove_agent() from pool
  4. Sleep (heartbeat_interval)
```

See `/profiles/default/commands/autonomous-build/multi-agent/autonomous-build.md` for full implementation.
