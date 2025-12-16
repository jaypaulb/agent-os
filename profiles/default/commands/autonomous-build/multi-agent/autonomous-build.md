# Autonomous Build: Parallel Dev Team Manager (v2)

You are the autonomous build orchestrator running as a **dev team manager**. Your job is to coordinate up to 5 specialized agents working in parallel, dynamically dispatching work, validating outcomes, and learning from errors.

## Your Responsibilities

1. **Manage work queue** - Track organisms from ready → in_progress → completed
2. **Manage agent pool** - Maintain up to 5 concurrent agents in slots
3. **Dispatch dynamically** - Assign work immediately when slot opens (no batch waiting)
4. **Validate continuously** - Validate and commit as agents complete
5. **Learn from errors** - Extract patterns, improve future agents
6. **Handle merge conflicts** - Retry → serialize → escalate
7. **Track progress** - Use BV for intelligent work selection
8. **Run regression tests** - Test random organisms from completed phases

**Mode**: Fully autonomous - continuous parallel execution until all work done or user interrupts.

**Key Innovation**: No bottleneck from slowest agent. Agents continuously working while queue has organisms.

---

## INITIALIZATION

First, verify the project is ready and initialize/recover state:

```bash
echo "=== Autonomous Build v2 Initialization ==="
echo ""

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

# Check if state exists (recovering from interruption)
if [ -f ".beads/autonomous-state/work-queue.json" ]; then
  echo "📦 State detected - recovering from previous session"
  echo ""

  # Check for interrupted organisms (in_progress → ready)
  IN_PROGRESS_COUNT=$(jq '.in_progress | length' .beads/autonomous-state/work-queue.json)
  if [[ "$IN_PROGRESS_COUNT" -gt 0 ]]; then
    echo "Found $IN_PROGRESS_COUNT interrupted organisms - moving to ready queue"
    jq '.ready += .in_progress | .in_progress = []' .beads/autonomous-state/work-queue.json > /tmp/wq.json
    mv /tmp/wq.json .beads/autonomous-state/work-queue.json
  fi

  # Clean stale locks
  rm -f .beads/autonomous-state/locks/*.lock 2>/dev/null

  # Reset agent pool
  jq '.active = [] | .available_slots = [1, 2, 3, 4, 5]' .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  echo "✓ State recovered"
else
  # Initialize state from scratch
  echo "🆕 First run - initializing state"
  echo ""

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
  MAX_AGENTS=$(yq eval '.session.max_agents // 5' .beads/autonomous-config.yml 2>/dev/null || echo "5")
  cat > .beads/autonomous-state/agent-pool.json <<EOF
{
  "max_agents": $MAX_AGENTS,
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

  # Load ready work into queue
  echo ""
  echo "Loading ready organisms into work queue..."

  # Get ready organisms from Beads
  READY_JSON=$(bd ready --format json 2>/dev/null || echo "[]")

  # Transform to work-queue format and populate ready array
  echo "$READY_JSON" | jq '[.[] | {
    id: .id,
    title: .title,
    type: .type,
    assignee: "general-purpose",
    priority: 5,
    impact: 0,
    predicted_files: []
  }]' | jq '. as $ready | {
    ready: $ready,
    in_progress: [],
    completed: [],
    failed: [],
    blocked: []
  }' > .beads/autonomous-state/work-queue.json

  READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
  echo "✓ Loaded $READY_COUNT organisms"
fi

# Show current state
echo ""
echo "=== Current State ==="
READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
COMPLETED_COUNT=$(jq '.completed | length' .beads/autonomous-state/work-queue.json)
FAILED_COUNT=$(jq '.failed | length' .beads/autonomous-state/work-queue.json)
MAX_AGENTS=$(jq '.max_agents' .beads/autonomous-state/agent-pool.json)

echo "  Ready: $READY_COUNT organisms"
echo "  Completed: $COMPLETED_COUNT organisms"
echo "  Failed: $FAILED_COUNT organisms"
echo "  Agent slots: $MAX_AGENTS concurrent"
echo ""

if [[ "$READY_COUNT" -eq 0 ]]; then
  if [[ "$COMPLETED_COUNT" -eq 0 ]] && [[ "$FAILED_COUNT" -eq 0 ]]; then
    echo "❌ No work in queue. Run /orchestrate-tasks or add organisms to Beads."
    exit 1
  else
    echo "🎉 ALL WORK COMPLETE!"
    echo "  Total organisms: $(($COMPLETED_COUNT + $FAILED_COUNT))"
    echo "  Success rate: $(echo "scale=2; $COMPLETED_COUNT * 100 / ($COMPLETED_COUNT + $FAILED_COUNT)" | bc)%"
    exit 0
  fi
fi

echo "Starting parallel agent pool (max $MAX_AGENTS concurrent)..."
echo "Say 'stop' or 'pause' anytime to interrupt"
echo ""
```

---

## MAIN LOOP

**Parallel Agent Pool Pattern**: You will continuously monitor the agent pool and work queue, dispatching new work immediately when slots become available. No waiting for ALL agents - each agent completes independently and frees its slot for the next organism.

**Loop until**: work queue empty AND all agents complete (or user interrupts).

Each loop iteration (heartbeat):

### STEP 1: Dispatch New Work

Check if we have free agent slots and ready organisms. If yes, spawn new agents:

```bash
# Get current state
ACTIVE_COUNT=$(jq '.active | length' .beads/autonomous-state/agent-pool.json)
MAX_AGENTS=$(jq '.max_agents' .beads/autonomous-state/agent-pool.json)
READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
AVAILABLE_SLOTS=$(jq -r '.available_slots[]' .beads/autonomous-state/agent-pool.json)

# Spawn new agents while slots available and work exists
while [[ "$ACTIVE_COUNT" -lt "$MAX_AGENTS" ]] && [[ "$READY_COUNT" -gt 0 ]]; do
  # Get free slot
  SLOT=$(echo "$AVAILABLE_SLOTS" | head -1)

  if [ -z "$SLOT" ]; then
    break
  fi

  # Select next organism (BV-aware if available)
  if command -v bv &> /dev/null; then
    # Use BV for intelligent selection (bottlenecks, keystones)
    ORGANISM_ID=$(bv --robot-plan --format json 2>/dev/null | jq -r '.tracks[0].items[0].id // empty')

    # Verify organism is in ready queue
    IS_READY=$(jq ".ready[] | select(.id == \"$ORGANISM_ID\") | .id" .beads/autonomous-state/work-queue.json)

    if [ -z "$IS_READY" ]; then
      # Fallback: first ready organism
      ORGANISM_ID=$(jq -r '.ready[0].id' .beads/autonomous-state/work-queue.json)
    fi
  else
    # Fallback: first ready organism
    ORGANISM_ID=$(jq -r '.ready[0].id // empty' .beads/autonomous-state/work-queue.json)
  fi

  if [ -z "$ORGANISM_ID" ]; then
    break
  fi

  # Get organism details
  ORGANISM_DATA=$(jq ".ready[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
  ORGANISM_TITLE=$(echo "$ORGANISM_DATA" | jq -r '.title')
  ORGANISM_TYPE=$(echo "$ORGANISM_DATA" | jq -r '.type')
  AGENT_TYPE=$(echo "$ORGANISM_DATA" | jq -r '.assignee')

  # Check for file conflicts with active work
  HAS_CONFLICT=false
  PREDICTED_FILES=$(echo "$ORGANISM_DATA" | jq -r '.predicted_files[]? // empty')

  if [ -n "$PREDICTED_FILES" ]; then
    for ACTIVE_AGENT in $(jq -c '.active[]' .beads/autonomous-state/agent-pool.json); do
      ACTIVE_ORGANISM=$(echo "$ACTIVE_AGENT" | jq -r '.organism_id')
      ACTIVE_FILES=$(jq ".in_progress[] | select(.id == \"$ACTIVE_ORGANISM\") | .predicted_files[]? // empty" .beads/autonomous-state/work-queue.json)

      # Check for file overlap
      for PREDICTED in $PREDICTED_FILES; do
        if echo "$ACTIVE_FILES" | grep -q "^$PREDICTED$"; then
          HAS_CONFLICT=true
          echo "⚠️  Skipping $ORGANISM_ID - file conflict with $ACTIVE_ORGANISM"
          break 2
        fi
      done
    done
  fi

  if [ "$HAS_CONFLICT" = true ]; then
    # Try next organism
    continue
  fi

  # Create lock file
  LOCK_FILE=".beads/autonomous-state/locks/${ORGANISM_ID}.lock"
  if [ -f "$LOCK_FILE" ]; then
    echo "⚠️  Organism $ORGANISM_ID already locked - skipping"
    continue
  fi

  echo "$SLOT|$(date -Iseconds)" > "$LOCK_FILE"

  # Load improvements for agent prompt
  IMPROVEMENTS=$(cat .beads/autonomous-state/learning/improvements.json)
  LEARNING_CONTEXT=""

  if [ "$(echo "$IMPROVEMENTS" | jq '.common_errors | length')" -gt 0 ]; then
    LEARNING_CONTEXT=$(echo "$IMPROVEMENTS" | jq -r '
      "IMPORTANT: Learn from previous sessions to avoid common mistakes:\n\n" +
      (.common_errors |
       group_by(.category) |
       map(
         "\(.[] | .category) issues:\n" +
         (.[0:3] | map("  ❌ \(.pattern)\n  ✅ \(.fix)") | join("\n"))
       ) | join("\n\n"))
    ')
  fi

  echo "🚀 Spawning agent in slot $SLOT: $ORGANISM_ID ($ORGANISM_TITLE)"

  # Spawn agent via Task tool (background=true)
  TASK_PROMPT="You are the implementer agent working on organism $ORGANISM_ID.

$LEARNING_CONTEXT

Your task: Implement this organism, test it, and close it.

## Process:

1. **Read organism details**:
   \`\`\`bash
   bd show $ORGANISM_ID
   \`\`\`

2. **Claim the organism**:
   \`\`\`bash
   bd update $ORGANISM_ID --status in_progress
   \`\`\`

3. **Implement following atomic design principles**:
   - Read spec context (if spec label exists)
   - Implement the feature
   - Write tests
   - Run tests and verify they pass

4. **Follow the implement-tasks workflow**:
   {{@agent-os/workflows/implementation/implement-tasks.md}}

   **IMPORTANT**: Monitor your context! If approaching limits:
   - Commit partial work
   - Update organism with progress note
   - Return control to orchestrator

5. **Close organism when complete**:
   \`\`\`bash
   bd update $ORGANISM_ID --status closed
   bd comment $ORGANISM_ID \"Implementation complete. Tests passing.\"
   \`\`\`

6. **Commit and push** using the git push fallback in implement-tasks.md

7. **Return control**: You're done. Orchestrator will continue.

Project root: $(pwd)
Organism: $ORGANISM_ID
Slot: $SLOT"

  # Spawn via Task tool (this is where you call the Task tool with run_in_background=true)
  # The orchestrator (Claude) will make this tool call

  echo ""
  echo "**NOW SPAWN AGENT VIA TASK TOOL**:"
  echo "  subagent_type: \"$AGENT_TYPE\""
  echo "  description: \"Implement $ORGANISM_ID (slot $SLOT)\""
  echo "  run_in_background: true"
  echo "  prompt: [see TASK_PROMPT above]"
  echo ""

  # NOTE: After spawning via Task tool, you'll receive a task_id
  # Use that task_id in the next steps to update state

  # Update work queue (move ready → in_progress)
  CLAIMED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
    claimed_at: \"$(date -Iseconds)\",
    slot: $SLOT,
    task_id: \"PLACEHOLDER_REPLACE_WITH_ACTUAL_TASK_ID\",
    attempt: 1
  }")

  jq "
    .in_progress += [$CLAIMED_DATA] |
    .ready = [.ready[] | select(.id != \"$ORGANISM_ID\")]
  " .beads/autonomous-state/work-queue.json > /tmp/wq.json
  mv /tmp/wq.json .beads/autonomous-state/work-queue.json

  # Update agent pool
  AGENT_DATA=$(jq -n "{
    slot: $SLOT,
    agent_type: \"$AGENT_TYPE\",
    organism_id: \"$ORGANISM_ID\",
    task_id: \"PLACEHOLDER_REPLACE_WITH_ACTUAL_TASK_ID\",
    started_at: \"$(date -Iseconds)\",
    last_checked: \"$(date -Iseconds)\",
    status: \"running\"
  }")

  jq "
    .active += [$AGENT_DATA] |
    .available_slots = [.available_slots[] | select(. != $SLOT)]
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  # Update counts for next iteration
  ACTIVE_COUNT=$((ACTIVE_COUNT + 1))
  READY_COUNT=$((READY_COUNT - 1))
  AVAILABLE_SLOTS=$(jq -r '.available_slots[]' .beads/autonomous-state/agent-pool.json)

  echo "✓ Agent spawned in slot $SLOT"
  echo ""
done
```

**IMPORTANT**: The bash script above shows the pseudo-code logic. You (Claude orchestrator) will actually call the Task tool to spawn agents. Replace `"PLACEHOLDER_REPLACE_WITH_ACTUAL_TASK_ID"` with the real task_id returned from the Task tool.

### STEP 2: Monitor Active Agents

Check status of all active agents (non-blocking poll):

```bash
# Check each active agent
for AGENT_SLOT in $(jq -r '.active[].slot' .beads/autonomous-state/agent-pool.json 2>/dev/null || echo ""); do
  if [ -z "$AGENT_SLOT" ]; then
    continue
  fi

  # Get agent details
  AGENT_DATA=$(jq ".active[] | select(.slot == $AGENT_SLOT)" .beads/autonomous-state/agent-pool.json)
  TASK_ID=$(echo "$AGENT_DATA" | jq -r '.task_id')
  ORGANISM_ID=$(echo "$AGENT_DATA" | jq -r '.organism_id')

  # Update last_checked timestamp
  jq "
    .active = [.active[] |
      if .slot == $AGENT_SLOT then
        .last_checked = \"$(date -Iseconds)\"
      else
        .
      end
    ]
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  echo "**NOW CHECK AGENT STATUS VIA TaskOutput TOOL**:"
  echo "  task_id: \"$TASK_ID\""
  echo "  block: false  # Non-blocking check"
  echo ""

  # NOTE: You (Claude orchestrator) will call:
  # TaskOutput(task_id=TASK_ID, block=False)
  #
  # If status == "completed":
  #   Continue to STEP 3 (Handle Completion) for this agent
  # If status == "running":
  #   Continue monitoring
done
```

**IMPORTANT**: You (Claude) will call the TaskOutput tool with `block=false` to poll agent status without waiting.

### STEP 3: Handle Agent Completions

For each completed agent from STEP 2, validate and commit (or retry):

```bash
# For completed agent with TASK_ID, ORGANISM_ID, SLOT:

echo "✅ Agent in slot $SLOT completed: $ORGANISM_ID"
echo ""

# Get agent output
echo "**NOW GET AGENT OUTPUT VIA TaskOutput TOOL**:"
echo "  task_id: \"$TASK_ID\""
echo "  block: true  # Get full output"
echo ""

# NOTE: After getting output, analyze for success/failure

# Validation logic (simplified for now - Phase 3 will add full validation pipeline)
ORGANISM_STATUS=$(bd show "$ORGANISM_ID" --format json 2>/dev/null | jq -r '.status')

if [ "$ORGANISM_STATUS" = "closed" ]; then
  # SUCCESS - Commit and move to completed
  echo "🎉 Organism $ORGANISM_ID completed successfully"

  COMMIT_SHA=$(git rev-parse HEAD 2>/dev/null || echo "unknown")

  # Release organism to completed queue
  ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
  COMPLETED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
    completed_at: \"$(date -Iseconds)\",
    commit_sha: \"$COMMIT_SHA\"
  }")

  jq "
    .completed += [$COMPLETED_DATA] |
    .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
  " .beads/autonomous-state/work-queue.json > /tmp/wq.json
  mv /tmp/wq.json .beads/autonomous-state/work-queue.json

  # Remove lock
  rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

  # Free agent slot
  jq "
    .active = [.active[] | select(.slot != $SLOT)] |
    .available_slots += [$SLOT] |
    .available_slots |= sort
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  echo "✓ Slot $SLOT freed"
  echo ""

elif [ "$ORGANISM_STATUS" = "in_progress" ]; then
  # PARTIAL - Agent exited early (context limits), retry later
  echo "⚠️  Organism $ORGANISM_ID partially complete (context limits)"
  echo "Moving back to ready queue for next session"

  # Move in_progress → ready
  ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)

  jq "
    .ready += [$ORGANISM_DATA | del(.claimed_at, .slot, .task_id, .attempt)] |
    .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
  " .beads/autonomous-state/work-queue.json > /tmp/wq.json
  mv /tmp/wq.json .beads/autonomous-state/work-queue.json

  # Remove lock
  rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

  # Free agent slot
  jq "
    .active = [.active[] | select(.slot != $SLOT)] |
    .available_slots += [$SLOT] |
    .available_slots |= sort
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  echo "✓ Slot $SLOT freed"
  echo ""

else
  # FAILED - Check attempt count
  ATTEMPT=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\") | .attempt // 1" .beads/autonomous-state/work-queue.json)
  MAX_ATTEMPTS=3

  if [[ "$ATTEMPT" -lt "$MAX_ATTEMPTS" ]]; then
    # Retry
    echo "❌ Organism $ORGANISM_ID failed (attempt $ATTEMPT/$MAX_ATTEMPTS)"
    echo "Moving to ready queue for retry"

    # Increment attempt count and move to ready
    ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
    RETRY_DATA=$(echo "$ORGANISM_DATA" | jq ".attempt = $((ATTEMPT + 1)) | del(.claimed_at, .slot, .task_id)")

    jq "
      .ready += [$RETRY_DATA] |
      .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
    " .beads/autonomous-state/work-queue.json > /tmp/wq.json
    mv /tmp/wq.json .beads/autonomous-state/work-queue.json

    # TODO Phase 5: Extract error patterns and add to improvements.json

  else
    # Max attempts reached - create analysis issue
    echo "❌ Organism $ORGANISM_ID failed after $MAX_ATTEMPTS attempts"
    echo "Creating analysis issue..."

    # TODO Phase 5: Create analysis issue and assign to general-purpose or Jaypaul
    # For now, just move to failed queue

    ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
    FAILED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
      attempts: $MAX_ATTEMPTS,
      last_error: \"Status not closed after implementation\",
      failed_at: \"$(date -Iseconds)\"
    }")

    jq "
      .failed += [$FAILED_DATA] |
      .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
    " .beads/autonomous-state/work-queue.json > /tmp/wq.json
    mv /tmp/wq.json .beads/autonomous-state/work-queue.json
  fi

  # Remove lock
  rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

  # Free agent slot
  jq "
    .active = [.active[] | select(.slot != $SLOT)] |
    .available_slots += [$SLOT] |
    .available_slots |= sort
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  echo "✓ Slot $SLOT freed"
  echo ""
fi
```

### STEP 4: Check Completion

Check if all work is done:

```bash
READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
IN_PROGRESS_COUNT=$(jq '.in_progress | length' .beads/autonomous-state/work-queue.json)
ACTIVE_COUNT=$(jq '.active | length' .beads/autonomous-state/agent-pool.json)

if [[ "$READY_COUNT" -eq 0 ]] && [[ "$IN_PROGRESS_COUNT" -eq 0 ]] && [[ "$ACTIVE_COUNT" -eq 0 ]]; then
  # All work complete!
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  🎉 ALL WORK COMPLETE!"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""

  COMPLETED_COUNT=$(jq '.completed | length' .beads/autonomous-state/work-queue.json)
  FAILED_COUNT=$(jq '.failed | length' .beads/autonomous-state/work-queue.json)
  TOTAL=$(($COMPLETED_COUNT + $FAILED_COUNT))

  if [[ "$TOTAL" -gt 0 ]]; then
    SUCCESS_RATE=$(echo "scale=1; $COMPLETED_COUNT * 100 / $TOTAL" | bc)
  else
    SUCCESS_RATE="0"
  fi

  echo "Final Statistics:"
  echo "  Total organisms: $TOTAL"
  echo "  Completed: $COMPLETED_COUNT"
  echo "  Failed: $FAILED_COUNT"
  echo "  Success rate: ${SUCCESS_RATE}%"
  echo ""

  if [[ "$FAILED_COUNT" -gt 0 ]]; then
    echo "Failed organisms requiring attention:"
    jq -r '.failed[] | "  ❌ \(.id): \(.title) (attempts: \(.attempts))"' .beads/autonomous-state/work-queue.json
    echo ""
  fi

  exit 0
fi
```

### STEP 5: Display Progress Dashboard

Show current state of agent pool and work queue:

```bash
echo ""
echo "=== Progress Dashboard ==="
READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
IN_PROGRESS_COUNT=$(jq '.in_progress | length' .beads/autonomous-state/work-queue.json)
COMPLETED_COUNT=$(jq '.completed | length' .beads/autonomous-state/work-queue.json)
FAILED_COUNT=$(jq '.failed | length' .beads/autonomous-state/work-queue.json)
ACTIVE_COUNT=$(jq '.active | length' .beads/autonomous-state/agent-pool.json)

echo "Work Queue:"
echo "  Ready: $READY_COUNT | In Progress: $IN_PROGRESS_COUNT | Completed: $COMPLETED_COUNT | Failed: $FAILED_COUNT"

echo ""
echo "Agent Pool:"
if [[ "$ACTIVE_COUNT" -gt 0 ]]; then
  jq -r '.active[] | "  Slot \(.slot): \(.organism_id) (\(.agent_type)) - \(.status)"' .beads/autonomous-state/agent-pool.json
else
  echo "  (no active agents)"
fi

# BV insights if available
if command -v bv &> /dev/null; then
  echo ""
  echo "BV Insights:"
  INSIGHTS=$(bv --robot-insights --format json 2>/dev/null || echo "{}")
  echo "$INSIGHTS" | jq -r 'if .bottlenecks | length > 0 then "  Bottlenecks: " + (.bottlenecks[:2][] | "\(.id)") else "" end'
fi

echo ""
```

### STEP 6: Heartbeat Sleep

Wait before next iteration:

```bash
# Get heartbeat interval from config
HEARTBEAT=$(jq '.heartbeat_interval' .beads/autonomous-state/agent-pool.json)
sleep $HEARTBEAT
```

**Then loop back to STEP 1** - dispatch new work, monitor agents, handle completions, repeat.

---

## How to Run This (v2 Parallel Mode)

You (Claude) are the orchestrator acting as a **dev team manager**. When user runs `/autonomous-build`:

### 1. INITIALIZATION

- Run INITIALIZATION bash commands to set up or recover state
- Verify `.beads/autonomous-state/` exists with work-queue.json, agent-pool.json, improvements.json

### 2. ENTER MAIN LOOP

Loop continuously (heartbeat pattern):

**STEP 1: Dispatch**
- Check free agent slots
- While slots available AND ready work exists:
  - Select next organism (BV-aware)
  - Check for file conflicts
  - Create lock
  - Load learnings
  - **Spawn agent via Task tool** (run_in_background=true)
  - Update work queue (ready → in_progress)
  - Update agent pool (add to active)

**STEP 2: Monitor**
- For each active agent:
  - **Call TaskOutput(task_id, block=false)** to check status
  - Update last_checked timestamp

**STEP 3: Handle Completions**
- For each completed agent:
  - **Call TaskOutput(task_id, block=true)** to get output
  - Validate organism (check if closed)
  - If success: move to completed, commit
  - If partial: move to ready (retry)
  - If failed: retry (up to 3 attempts) or move to failed
  - Free agent slot
  - Remove lock

**STEP 4: Check Completion**
- If ready=0 AND in_progress=0 AND active=0: EXIT (done)

**STEP 5: Display Progress**
- Show work queue counts
- Show active agents
- Show BV insights

**STEP 6: Sleep**
- Sleep for heartbeat_interval (default 10 seconds)
- Loop back to STEP 1

### 3. Key Differences from v1

| Aspect | v1 (Sequential) | v2 (Parallel) |
|--------|----------------|---------------|
| **Agents** | 1 at a time | Up to 5 concurrent |
| **Dispatch** | Batch (wait for all) | Dynamic (dispatch on slot open) |
| **Bottleneck** | Slowest agent blocks all | Each agent independent |
| **State** | Beads issues only | work-queue.json + agent-pool.json |
| **Monitoring** | Blocking wait | Non-blocking poll (TaskOutput block=false) |
| **Efficiency** | 16-20% (idle time) | 70-85% (working time) |
| **Throughput** | 1 organism/session | 3-5 organisms/session |

### 4. Tool Calls You'll Make

You (Claude orchestrator) will make these tool calls during the loop:

```python
# STEP 1: Spawn agents
Task(
    subagent_type="general-purpose",  # or specialized agent
    description="Implement bd-org-123 (slot 1)",
    prompt="[full implementation prompt with learnings]",
    run_in_background=True  # CRITICAL: Non-blocking
)
# Returns: task_id

# STEP 2: Monitor (non-blocking)
TaskOutput(
    task_id="a1b2c3d4",
    block=False  # CRITICAL: Don't wait, just check status
)
# Returns: status ("running" or "completed")

# STEP 3: Get output (blocking)
TaskOutput(
    task_id="a1b2c3d4",
    block=True  # Wait for full output
)
# Returns: agent output
```

### 5. State Recovery

If interrupted (Ctrl+C, crash):

1. User runs `/autonomous-build` again
2. INITIALIZATION detects existing `.beads/autonomous-state/`
3. Recovery logic:
   - Move in_progress → ready (retry interrupted organisms)
   - Clean stale locks
   - Reset agent pool (all slots available)
4. Resume from where you left off

**User can interrupt** anytime by saying "stop", "pause", or Ctrl+C.

**State is preserved** in `.beads/autonomous-state/`, so recovery is seamless.

### 6. Configuration

Users can customize via `.beads/autonomous-config.yml`:

```yaml
session:
  max_agents: 5           # Max concurrent agents (default: 5)
  heartbeat_interval: 10  # Seconds between loops (default: 10)
```

### 7. Example Session

```
=== Autonomous Build v2 Initialization ===
✓ State initialized
✓ Loaded 15 organisms

=== Current State ===
  Ready: 15 organisms
  Completed: 0 organisms
  Failed: 0 organisms
  Agent slots: 5 concurrent

Starting parallel agent pool (max 5 concurrent)...

🚀 Spawning agent in slot 1: bd-org-001 (User model)
🚀 Spawning agent in slot 2: bd-org-002 (Auth endpoints)
🚀 Spawning agent in slot 3: bd-org-003 (Login component)
🚀 Spawning agent in slot 4: bd-org-004 (Validation utils)
🚀 Spawning agent in slot 5: bd-org-005 (Error handling)

=== Progress Dashboard ===
Work Queue:
  Ready: 10 | In Progress: 5 | Completed: 0 | Failed: 0

Agent Pool:
  Slot 1: bd-org-001 (database-layer-builder) - running
  Slot 2: bd-org-002 (api-layer-builder) - running
  Slot 3: bd-org-003 (ui-component-builder) - running
  Slot 4: bd-org-004 (general-purpose) - running
  Slot 5: bd-org-005 (general-purpose) - running

[10 seconds later...]

✅ Agent in slot 4 completed: bd-org-004
🎉 Organism bd-org-004 completed successfully
✓ Slot 4 freed

🚀 Spawning agent in slot 4: bd-org-006 (Password hashing)

[continues until all organisms complete...]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  🎉 ALL WORK COMPLETE!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Final Statistics:
  Total organisms: 15
  Completed: 14
  Failed: 1
  Success rate: 93.3%
```
