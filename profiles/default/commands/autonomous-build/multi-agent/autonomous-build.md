# Autonomous Build: Parallel Dev Team Manager (v2)

You are the autonomous build orchestrator running as a **dev team manager**. Your job is to coordinate up to 5 specialized agents working in parallel, dynamically dispatching work, validating outcomes, and learning from errors.

## Prerequisites & Dependencies

This command composes with existing orchestration infrastructure:

```bash
# Source shared BV helpers for consistent tooling
source agent-os/profiles/default/workflows/implementation/bv-helpers.md
```

**Key dependency**: Uses `/orchestrate-tasks --headless` for agent assignment.
Do NOT reimplement agent inference - defer to orchestrate-tasks.

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

First, verify the project is ready and initialize/recover state.

**IMPORTANT**: This command leverages `/orchestrate-tasks --headless` for proper agent assignment.
Do NOT duplicate agent inference logic here - orchestrate-tasks handles that.

```bash
echo "=== Autonomous Build v2 Initialization ==="
echo ""

# Verify we're at project root with Beads
if [ ! -d ".beads" ]; then
  echo "❌ No .beads/ directory found at project root"
  exit 1
fi
echo "✓ .beads directory found"

# Check for issues
ISSUE_COUNT=$(bd list --json 2>/dev/null | jq '. | length' || echo "0")
echo "Found $ISSUE_COUNT total issues in Beads"
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
  echo "🆕 First run - initializing state via /orchestrate-tasks --headless"
  echo ""

  # Create state directory structure
  mkdir -p .beads/autonomous-state/locks
  mkdir -p .beads/autonomous-state/learning
  mkdir -p .beads/autonomous-state/failures
```

**NOW CALL /orchestrate-tasks --headless** to create orchestration state with proper agent assignments.

This command will:
1. Discover all phases and organisms from Beads
2. Infer the correct specialized agent for each organism based on title/type:
   - Database/Model/Schema/Migration → `database-layer-builder`
   - API/Endpoint/Controller/Route → `api-layer-builder`
   - UI/Component/Page/View/Frontend → `ui-component-builder`
   - Integration/Wire/E2E → `integration-assembler`
   - Test Gap/Coverage → `test-gap-analyzer`
   - Default → `molecule-composer`
3. Write orchestration.yml to `.beads/autonomous-state/orchestration.yml`

After orchestrate-tasks completes, continue initialization:

```bash
  # Verify orchestration.yml was created
  if [ ! -f ".beads/autonomous-state/orchestration.yml" ]; then
    echo "❌ orchestrate-tasks did not create orchestration.yml"
    exit 1
  fi
  echo "✓ Orchestration state created with agent assignments"

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

  # Initialize improvements.json (learning system)
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

  echo "✓ Agent pool and learning system initialized"

  # Load work queue FROM orchestration.yml (with proper agent assignments)
  echo ""
  echo "Loading organisms from orchestration.yml into work queue..."

  # Transform orchestration.yml to work-queue.json format
  # This preserves the agent assignments from orchestrate-tasks
  yq eval '.beads' .beads/autonomous-state/orchestration.yml -o json | jq '{
    ready: [.[] | {
      id: .id,
      title: .title,
      type: (.type // "organism"),
      assignee: .assignee,
      standards: (.standards // []),
      priority: (.priority // 5),
      impact: (.impact // 0),
      predicted_files: (.predicted_files // [])
    }],
    in_progress: [],
    completed: [],
    failed: [],
    blocked: []
  }' > .beads/autonomous-state/work-queue.json

  READY_COUNT=$(jq '.ready | length' .beads/autonomous-state/work-queue.json)
  echo "✓ Loaded $READY_COUNT organisms with specialized agent assignments"

  # Show agent distribution
  echo ""
  echo "Agent distribution:"
  jq -r '.ready | group_by(.assignee) | .[] | "  \(.[0].assignee): \(length) organisms"' .beads/autonomous-state/work-queue.json
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

  # Check for conflict resolution context
  CONFLICT_CONTEXT=""
  CONFLICT_ATTEMPT=$(echo "$ORGANISM_DATA" | jq -r '.conflict_attempt // 0')

  if [ "$CONFLICT_ATTEMPT" -gt 0 ]; then
    CONFLICT_DETAILS=$(echo "$ORGANISM_DATA" | jq -r '.conflict_details // "No details available"')
    CONFLICTED_FILES=$(echo "$ORGANISM_DATA" | jq -r '.conflicted_files // "Unknown"')

    CONFLICT_CONTEXT="
⚠️  CONFLICT RESOLUTION MODE (Attempt $CONFLICT_ATTEMPT)

This organism previously resulted in merge conflicts. You MUST resolve these conflicts as part of your implementation.

**Conflicted files:**
$CONFLICTED_FILES

**Conflict details:**
$CONFLICT_DETAILS

**How to resolve:**
1. Read the conflict diff above carefully
2. Understand what changes are in conflict
3. Implement your organism in a way that INCORPORATES BOTH sets of changes
4. For type/interface conflicts: merge both additions, avoid duplicates
5. For barrel file conflicts: merge exports in alphabetical order
6. For config conflicts: carefully merge both configurations
7. Test thoroughly to ensure both features work together

**Critical:** Your implementation MUST resolve the conflict, not recreate it.
"
  fi

  # Check for context exhaustion recovery mode
  RECOVERY_CONTEXT=""
  RECOVERY_MODE=$(echo "$ORGANISM_DATA" | jq -r '.recovery_mode // false')

  if [ "$RECOVERY_MODE" = "true" ]; then
    CONTINUE_FROM=$(echo "$ORGANISM_DATA" | jq -r '.continue_from_step // 0')
    PRIOR_STEPS=$(echo "$ORGANISM_DATA" | jq -r '.prior_steps | join(", ") // "unknown"')
    PRIOR_COMMITS=$(echo "$ORGANISM_DATA" | jq -r '.prior_commits | join(", ") // "none"')

    RECOVERY_CONTEXT="
🔄 RECOVERY MODE: Continuing from previous agent

A previous agent ran out of context while implementing this organism.
You are continuing their work. DO NOT start from scratch.

**Progress from previous agent:**
- Steps completed: $PRIOR_STEPS
- Continue from step: $CONTINUE_FROM
- Prior commits: $PRIOR_COMMITS

**Your task:**
1. First, verify the prior commits exist: \`git log --oneline -5\`
2. Check what was already implemented by reading the modified files
3. Continue from step $CONTINUE_FROM (do NOT redo completed steps)
4. Complete the remaining steps
5. Close the organism when done

**IMPORTANT:** The prior work is committed. Build on it, don't replace it.
"
  fi

  echo "🚀 Spawning agent in slot $SLOT: $ORGANISM_ID ($ORGANISM_TITLE)"

  if [ "$CONFLICT_ATTEMPT" -gt 0 ]; then
    echo "   ⚠️  Conflict resolution mode (attempt $CONFLICT_ATTEMPT)"
  fi

  if [ "$RECOVERY_MODE" = "true" ]; then
    echo "   🔄 Recovery mode (continue from step $CONTINUE_FROM)"
  fi

  # Spawn agent via Task tool (background=true)
  TASK_PROMPT="You are the implementer agent working on organism $ORGANISM_ID.

$LEARNING_CONTEXT

$CONFLICT_CONTEXT

$RECOVERY_CONTEXT

Your task: Implement this organism, test it, and close it.

## CRITICAL: Checkpoint Protocol

You MUST log progress via \`bd comment\` after EACH step. This allows recovery if you run out of context.

**After completing each step:**
\`\`\`bash
bd comment $ORGANISM_ID \"CHECKPOINT: step N - description\"
\`\`\`

**After each commit:**
\`\`\`bash
bd comment $ORGANISM_ID \"COMMIT: \$(git rev-parse --short HEAD) - description\"
\`\`\`

## Process:

1. **Read organism details**:
   \`\`\`bash
   bd show $ORGANISM_ID
   bd comment $ORGANISM_ID \"CHECKPOINT: step 1 - read details\"
   \`\`\`

2. **Claim the organism**:
   \`\`\`bash
   bd update $ORGANISM_ID --status in_progress
   bd comment $ORGANISM_ID \"CHECKPOINT: step 2 - claimed\"
   \`\`\`

3. **Implement**:
   - Read spec context (if spec label exists)
   - Implement the feature
   - **COMMIT after implementation** (before tests)
   \`\`\`bash
   git add -A && git commit -m \"impl($ORGANISM_ID): description\"
   bd comment $ORGANISM_ID \"COMMIT: \$(git rev-parse --short HEAD) - implementation\"
   bd comment $ORGANISM_ID \"CHECKPOINT: step 3 - implemented\"
   \`\`\`

4. **Write tests**:
   - Write tests for the implementation
   - **COMMIT after writing tests**
   \`\`\`bash
   git add -A && git commit -m \"test($ORGANISM_ID): description\"
   bd comment $ORGANISM_ID \"COMMIT: \$(git rev-parse --short HEAD) - tests\"
   bd comment $ORGANISM_ID \"CHECKPOINT: step 4 - tests written\"
   \`\`\`

5. **Run tests**:
   - Run tests and verify they pass
   \`\`\`bash
   bd comment $ORGANISM_ID \"CHECKPOINT: step 5 - tests passing\"
   \`\`\`

6. **Close organism** (FINAL):
   \`\`\`bash
   bd update $ORGANISM_ID --status closed
   bd comment $ORGANISM_ID \"CHECKPOINT: step 6 - COMPLETE\"
   \`\`\`

7. **Return control**: You're done. Orchestrator will continue.

**IMPORTANT**: Commit early, commit often. Each commit is recoverable if you run out of context.

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

### STEP 3: Handle Agent Completions (with Context Recovery)

For each completed agent from STEP 2, check for context exhaustion and recover:

```bash
# For completed agent with TASK_ID, ORGANISM_ID, SLOT:

echo "✅ Agent in slot $SLOT completed: $ORGANISM_ID"
echo ""

# Get agent output
echo "**NOW GET AGENT OUTPUT VIA TaskOutput TOOL**:"
echo "  task_id: \"$TASK_ID\""
echo "  block: true  # Get full output"
echo ""

# NOTE: TaskOutput may return error if agent died from context exhaustion
# In that case, we recover using the checkpoint file

# ═══════════════════════════════════════════════════════════════════
# CHECKPOINT RECOVERY: Check if agent died mid-task
# ═══════════════════════════════════════════════════════════════════

CHECKPOINT_FILE=".beads/autonomous-state/checkpoints/${ORGANISM_ID}.json"

if [ -f "$CHECKPOINT_FILE" ]; then
  CHECKPOINT_STATUS=$(jq -r '.status' "$CHECKPOINT_FILE")
  CHECKPOINT_STEP=$(jq -r '.current_step' "$CHECKPOINT_FILE")
  CHECKPOINT_COMMITS=$(jq -r '.commits | length' "$CHECKPOINT_FILE")

  echo "📋 Checkpoint found: step $CHECKPOINT_STEP, status: $CHECKPOINT_STATUS, commits: $CHECKPOINT_COMMITS"

  if [ "$CHECKPOINT_STATUS" != "completed" ]; then
    # Agent died before completing - check what was saved
    echo "⚠️  Agent died before completing (likely context exhaustion)"

    if [ "$CHECKPOINT_COMMITS" -gt 0 ]; then
      # Partial work was committed - can recover
      echo "✓ Found $CHECKPOINT_COMMITS commits - partial work recoverable"

      LAST_COMMIT=$(jq -r '.commits[-1]' "$CHECKPOINT_FILE")
      STEPS_DONE=$(jq -r '.steps_completed | join(", ")' "$CHECKPOINT_FILE")

      echo "  Last commit: $LAST_COMMIT"
      echo "  Steps completed: $STEPS_DONE"

      # Add recovery context for retry
      ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
      RECOVERY_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
        recovery_mode: true,
        continue_from_step: $CHECKPOINT_STEP,
        prior_commits: $(jq '.commits' "$CHECKPOINT_FILE"),
        prior_steps: $(jq '.steps_completed' "$CHECKPOINT_FILE"),
        context_death_at: \"$(date -Iseconds)\"
      } | del(.claimed_at, .slot, .task_id)")

      # Re-queue with recovery context
      jq "
        .ready = [$RECOVERY_DATA] + .ready |
        .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
      " .beads/autonomous-state/work-queue.json > /tmp/wq.json
      mv /tmp/wq.json .beads/autonomous-state/work-queue.json

      echo "✓ Re-queued $ORGANISM_ID for recovery (continue from step $CHECKPOINT_STEP)"

      # Free slot and continue
      # (slot freeing code follows in the next section)
      CONTEXT_DEATH_RECOVERY=true
    else
      # No commits - nothing to recover, will retry from scratch
      echo "❌ No commits found - retry from scratch"
      CONTEXT_DEATH_RECOVERY=false
    fi
  fi
else
  echo "⚠️  No checkpoint file found for $ORGANISM_ID"
fi

# ═══════════════════════════════════════════════════════════════════
# Normal completion handling (if not recovering from context death)
# ═══════════════════════════════════════════════════════════════════

if [ "${CONTEXT_DEATH_RECOVERY:-false}" != "true" ]; then

# Check if organism was closed by agent
ORGANISM_STATUS=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.status')

if [ "$ORGANISM_STATUS" != "closed" ]; then
  # Agent didn't close organism - treat as failure
  echo "❌ Agent completed but organism status is: $ORGANISM_STATUS"
  echo "Expected: closed"
  # Skip validation, go to retry logic below
  VALIDATION_PASSED=false
else
  # Run 5-gate validation pipeline
  echo "Running validation pipeline..."
  echo ""

  # Export variables for validator
  export ORGANISM_ID
  export COMMIT_BEFORE=$(cat .beads/last-iteration-commit 2>/dev/null || git rev-parse HEAD^)
  export ATTEMPT=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\") | .attempt // 1" .beads/autonomous-state/work-queue.json)

  # Run validator
  bash profiles/default/workflows/implementation/validation/organism-validator.md
  VALIDATION_EXIT_CODE=$?

  case $VALIDATION_EXIT_CODE in
    0)
      # All gates passed
      VALIDATION_PASSED=true
      ;;
    3)
      # Merge conflict detected - 3-tier resolution strategy
      echo "⚠️  Merge conflict detected"
      CONFLICT_DETAILS=$(cat /tmp/conflict-details.txt 2>/dev/null || echo "No details available")

      ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
      CONFLICT_ATTEMPT=$(echo "$ORGANISM_DATA" | jq -r '.conflict_attempt // 0')

      case $CONFLICT_ATTEMPT in
        0)
          # ATTEMPT 1: Conflict-aware retry
          echo "📝 Conflict Resolution Attempt 1: Retry with conflict awareness"
          echo ""
          echo "Feeding conflict diff to agent for resolution..."

          # Extract conflicted files for learning
          CONFLICTED_FILES=$(echo "$CONFLICT_DETAILS" | grep -oP '===\s+\K[^\s]+' || echo "")

          # Add conflict info to organism for retry
          RETRY_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
            conflict_attempt: 1,
            conflict_details: \"$(echo "$CONFLICT_DETAILS" | jq -sR .)\",
            conflicted_files: \"$(echo "$CONFLICTED_FILES" | jq -sR .)\",
            last_conflict_at: \"$(date -Iseconds)\"
          } | del(.claimed_at, .slot, .task_id)")

          jq "
            .ready += [$RETRY_DATA] |
            .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
          " .beads/autonomous-state/work-queue.json > /tmp/wq.json
          mv /tmp/wq.json .beads/autonomous-state/work-queue.json

          echo "✓ Organism queued for conflict-aware retry"
          echo "Next agent will receive conflict diff and resolution hints"
          ;;

        1)
          # ATTEMPT 2: Serialize conflicting organisms
          echo "📝 Conflict Resolution Attempt 2: Serialize conflicting work"
          echo ""

          # Identify which organism(s) caused the conflict
          echo "Analyzing conflict to identify conflicting organism..."

          # Check if any active agents are working on organisms that touch same files
          CONFLICTED_FILES=$(echo "$CONFLICT_DETAILS" | grep -oP '===\s+\K[^\s]+' || echo "")
          BLOCKING_ORGANISM=""

          for ACTIVE_AGENT in $(jq -c '.active[]' .beads/autonomous-state/agent-pool.json 2>/dev/null || echo ""); do
            ACTIVE_ORGANISM=$(echo "$ACTIVE_AGENT" | jq -r '.organism_id')
            ACTIVE_FILES=$(jq ".in_progress[] | select(.id == \"$ACTIVE_ORGANISM\") | .predicted_files[]? // empty" .beads/autonomous-state/work-queue.json || echo "")

            # Check for overlap
            for CONF_FILE in $CONFLICTED_FILES; do
              if echo "$ACTIVE_FILES" | grep -q "$CONF_FILE"; then
                BLOCKING_ORGANISM="$ACTIVE_ORGANISM"
                break 2
              fi
            done
          done

          if [ -n "$BLOCKING_ORGANISM" ]; then
            echo "Found conflicting organism: $BLOCKING_ORGANISM (currently active)"
            echo "Blocking $ORGANISM_ID until $BLOCKING_ORGANISM completes"

            # Move to blocked queue with dependency
            BLOCKED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
              blocked_by: \"$BLOCKING_ORGANISM\",
              reason: \"Serialized due to file conflict\",
              blocked_at: \"$(date -Iseconds)\",
              conflict_attempt: 2,
              conflict_details: \"$(echo "$CONFLICT_DETAILS" | jq -sR .)\"
            } | del(.claimed_at, .slot, .task_id)")

            jq "
              .blocked += [$BLOCKED_DATA] |
              .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
            " .beads/autonomous-state/work-queue.json > /tmp/wq.json
            mv /tmp/wq.json .beads/autonomous-state/work-queue.json

            echo "✓ Organism blocked until $BLOCKING_ORGANISM completes"
          else
            # No active conflicting organism - conflict is with already committed work
            echo "Conflict is with committed work on main branch"
            echo "Escalating to manual review (Attempt 3)"
            CONFLICT_ATTEMPT=2  # Force escalation
          fi
          ;;

        *)
          # ATTEMPT 3: Manual review escalation
          echo "📝 Conflict Resolution Attempt 3: Manual review required"
          echo ""
          echo "Creating analysis issue for manual resolution..."

          # Create Beads issue for manual review
          CONFLICT_ISSUE_TITLE="[CONFLICT] Merge conflict in $ORGANISM_ID"
          CONFLICT_ISSUE_BODY="Merge conflict detected after multiple resolution attempts.

**Original organism:** $ORGANISM_ID

**Conflict details:**
\`\`\`
$CONFLICT_DETAILS
\`\`\`

**Resolution attempts:**
1. Conflict-aware retry - Failed
2. Serialization - Failed or N/A
3. Manual review - Current step

**Action required:**
- Review the conflict diff above
- Manually resolve the conflicting files
- Implement $ORGANISM_ID with conflict resolution
- Close this issue when complete

**Assign to:** Jaypaul (manual intervention required)"

          # Create the issue using bd
          # Note: This is placeholder - actual implementation depends on bd CLI capabilities
          echo "TODO: Create Beads issue with bd CLI"
          echo "  Title: $CONFLICT_ISSUE_TITLE"
          echo "  Assignee: Jaypaul"
          echo "  Labels: conflict, manual-review, urgent"

          # For now, move to failed queue with manual review flag
          FAILED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
            failed_at: \"$(date -Iseconds)\",
            failure_reason: \"Merge conflict - manual review required\",
            conflict_attempt: 3,
            conflict_details: \"$(echo "$CONFLICT_DETAILS" | jq -sR .)\",
            requires_manual_review: true,
            assigned_to: \"Jaypaul\"
          } | del(.claimed_at, .slot, .task_id)")

          jq "
            .failed += [$FAILED_DATA] |
            .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
          " .beads/autonomous-state/work-queue.json > /tmp/wq.json
          mv /tmp/wq.json .beads/autonomous-state/work-queue.json

          echo "✓ Organism moved to failed queue"
          echo "✓ Manual review required - assigned to Jaypaul"
          echo ""
          echo "Conflict resolution exhausted all automatic strategies"
          ;;
      esac

      # Remove lock
      rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

      # Free slot
      jq "
        .active = [.active[] | select(.slot != $SLOT)] |
        .available_slots += [$SLOT] |
        .available_slots |= sort
      " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
      mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

      echo "✓ Slot $SLOT freed"
      echo ""
      continue  # Skip to next agent
      ;;
    *)
      # Other validation failure (gate 1, 2, 4, or 5)
      echo "❌ Validation failed at gate $VALIDATION_EXIT_CODE"
      VALIDATION_PASSED=false
      # Will retry below
      ;;
  esac
fi

if [ "$VALIDATION_PASSED" = true ]; then
  # SUCCESS - Move to completed
  echo "🎉 Organism $ORGANISM_ID validated and completed successfully"

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

else
  # VALIDATION FAILED OR PARTIAL - Check attempt count and retry
  ATTEMPT=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\") | .attempt // 1" .beads/autonomous-state/work-queue.json)
  MAX_ATTEMPTS=3

  if [[ "$ATTEMPT" -lt "$MAX_ATTEMPTS" ]]; then
    # Retry with learnings
    echo "❌ Organism $ORGANISM_ID failed validation (attempt $ATTEMPT/$MAX_ATTEMPTS)"
    echo "Moving to ready queue for retry with error learnings"

    # Extract error context for learning system
    if [ "$ORGANISM_STATUS" = "in_progress" ]; then
      FAILURE_REASON="Partial implementation (context limits)"
    elif [ "$VALIDATION_PASSED" = false ]; then
      FAILURE_REASON="Validation failed at gate $VALIDATION_EXIT_CODE"
    else
      FAILURE_REASON="Unknown failure"
    fi

    echo "Failure reason: $FAILURE_REASON"

    # Extract error patterns for learning system
    echo "Analyzing errors for learning system..."

    # Get error output from validation logs
    ERROR_OUTPUT=""
    if [ -f "/tmp/test-output.txt" ]; then
      ERROR_OUTPUT="${ERROR_OUTPUT}\n=== Test Output ===\n$(cat /tmp/test-output.txt)\n"
    fi
    if [ -f "/tmp/lint-output.txt" ]; then
      ERROR_OUTPUT="${ERROR_OUTPUT}\n=== Lint Output ===\n$(cat /tmp/lint-output.txt)\n"
    fi
    if [ -f "/tmp/type-output.txt" ]; then
      ERROR_OUTPUT="${ERROR_OUTPUT}\n=== Type Output ===\n$(cat /tmp/type-output.txt)\n"
    fi
    if [ -f "/tmp/integration-output.txt" ]; then
      ERROR_OUTPUT="${ERROR_OUTPUT}\n=== Integration Output ===\n$(cat /tmp/integration-output.txt)\n"
    fi
    if [ -f "/tmp/regression-output.txt" ]; then
      ERROR_OUTPUT="${ERROR_OUTPUT}\n=== Regression Output ===\n$(cat /tmp/regression-output.txt)\n"
    fi

    # Export variables for error analyzer
    export ORGANISM_ID
    export VALIDATION_EXIT_CODE=${VALIDATION_EXIT_CODE:-0}
    export ERROR_OUTPUT
    export ATTEMPT

    # Run error analyzer (if validation failed with error output)
    if [ "$VALIDATION_PASSED" = false ] && [ -n "$ERROR_OUTPUT" ]; then
      bash profiles/default/workflows/implementation/learning/error-analyzer.md 2>/dev/null || echo "⚠️  Error analyzer failed"
      echo "✓ Errors extracted and added to learning system"
    fi

    # Increment attempt count and move to ready
    ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
    RETRY_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
      attempt: $((ATTEMPT + 1)),
      last_failure: \"$FAILURE_REASON\",
      last_failure_at: \"$(date -Iseconds)\"
    } | del(.claimed_at, .slot, .task_id)")

    jq "
      .ready += [$RETRY_DATA] |
      .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
    " .beads/autonomous-state/work-queue.json > /tmp/wq.json
    mv /tmp/wq.json .beads/autonomous-state/work-queue.json

  else
    # Max attempts reached - extract learnings and move to failed
    echo "❌ Organism $ORGANISM_ID failed after $MAX_ATTEMPTS attempts"
    echo "Extracting error patterns for analysis..."

    # Get failure history
    ORGANISM_DATA=$(jq ".in_progress[] | select(.id == \"$ORGANISM_ID\")" .beads/autonomous-state/work-queue.json)
    LAST_FAILURE=$(echo "$ORGANISM_DATA" | jq -r '.last_failure // "Unknown"')

    # Create failure analysis file
    mkdir -p .beads/autonomous-state/failures
    ANALYSIS_FILE=".beads/autonomous-state/failures/${ORGANISM_ID}-analysis.md"

    cat > "$ANALYSIS_FILE" <<EOF
# Failure Analysis: $ORGANISM_ID

## Organism Details

$(bd show "$ORGANISM_ID" 2>/dev/null || echo "Issue not found")

## Failure Summary

- **Attempts**: $MAX_ATTEMPTS
- **Last failure**: $LAST_FAILURE
- **Failed at**: $(date -Iseconds)

## Failure History

$(echo "$ORGANISM_DATA" | jq -r 'if .last_failure then "Attempt \(.attempt): \(.last_failure) at \(.last_failure_at)" else "No history available" end')

## Possible Causes

- Organism too complex (needs decomposition into smaller organisms)
- Missing dependencies (requires other organisms to be completed first)
- Incorrect implementation approach
- Test failures indicating spec misunderstanding
- Merge conflicts with concurrent work

## Recommended Actions

1. Review error messages and logs
2. Check if organism dependencies are met
3. Consider breaking into smaller sub-organisms
4. Verify spec requirements are clear
5. Manual implementation or escalation to Jaypaul

## Next Steps

TODO Phase 5:
- Create Beads analysis issue
- Assign to general-purpose agent for automated analysis
- If automated analysis fails, escalate to Jaypaul
EOF

    echo "✓ Analysis file created: $ANALYSIS_FILE"

    # Move to failed queue with analysis reference
    FAILED_DATA=$(echo "$ORGANISM_DATA" | jq ". + {
      attempts: $MAX_ATTEMPTS,
      last_error: \"$LAST_FAILURE\",
      failed_at: \"$(date -Iseconds)\",
      analysis_file: \"$ANALYSIS_FILE\"
    }")

    jq "
      .failed += [$FAILED_DATA] |
      .in_progress = [.in_progress[] | select(.id != \"$ORGANISM_ID\")]
    " .beads/autonomous-state/work-queue.json > /tmp/wq.json
    mv /tmp/wq.json .beads/autonomous-state/work-queue.json

    echo "Organism moved to failed queue"
    echo "Phase 5 will implement automated analysis and escalation"
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

fi  # End of CONTEXT_DEATH_RECOVERY check

# If we recovered from context death, also free the slot
if [ "${CONTEXT_DEATH_RECOVERY:-false}" = "true" ]; then
  # Remove lock
  rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

  # Free agent slot
  jq "
    .active = [.active[] | select(.slot != $SLOT)] |
    .available_slots += [$SLOT] |
    .available_slots |= sort
  " .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

  # Clean up checkpoint file (will be recreated on retry)
  rm -f "$CHECKPOINT_FILE"

  echo "✓ Slot $SLOT freed (context recovery)"
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
    subagent_type=AGENT_TYPE,  # From work-queue assignee (set by orchestrate-tasks)
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
✓ .beads directory found
Found 252 total issues in Beads

🆕 First run - initializing state via /orchestrate-tasks --headless
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  HEADLESS MODE: Autonomous Build Integration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Orchestration state created with agent assignments

Agent distribution:
  database-layer-builder: 8 organisms
  api-layer-builder: 12 organisms
  ui-component-builder: 15 organisms
  molecule-composer: 20 organisms
  integration-assembler: 5 organisms

✓ Loaded 60 organisms with specialized agent assignments

=== Current State ===
  Ready: 60 organisms
  Completed: 0 organisms
  Failed: 0 organisms
  Agent slots: 5 concurrent

Starting parallel agent pool (max 5 concurrent)...

🚀 Spawning agent in slot 1: bd-org-001 (User model) → database-layer-builder
🚀 Spawning agent in slot 2: bd-org-002 (Auth endpoints) → api-layer-builder
🚀 Spawning agent in slot 3: bd-org-003 (Login component) → ui-component-builder
🚀 Spawning agent in slot 4: bd-org-004 (Validation utils) → molecule-composer
🚀 Spawning agent in slot 5: bd-org-005 (Error handling) → molecule-composer

=== Progress Dashboard ===
Work Queue:
  Ready: 55 | In Progress: 5 | Completed: 0 | Failed: 0

Agent Pool:
  Slot 1: bd-org-001 (database-layer-builder) - running
  Slot 2: bd-org-002 (api-layer-builder) - running
  Slot 3: bd-org-003 (ui-component-builder) - running
  Slot 4: bd-org-004 (molecule-composer) - running
  Slot 5: bd-org-005 (molecule-composer) - running

[10 seconds later...]

✅ Agent in slot 4 completed: bd-org-004
🎉 Organism bd-org-004 completed successfully
✓ Slot 4 freed

🚀 Spawning agent in slot 4: bd-org-006 (Password hashing) → molecule-composer

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
