# Autonomous Build: Parallel Dev Team Manager (v2)

You are the autonomous build orchestrator running as a **dev team manager**. Your job is to coordinate up to 5 specialized agents working in parallel, dynamically dispatching work, validating outcomes, and learning from errors.

## Prerequisites & Dependencies

**CRITICAL**: This orchestrator uses `bd` and `bv` EXCLUSIVELY for issue tracking and selection.
- NO custom work-queue.json files
- NO custom orchestration.yml files
- ALL issue state comes from bd/bv commands

```bash
# Verify bd and bv are available
command -v bd &>/dev/null || { echo "❌ bd not found"; exit 1; }
command -v bv &>/dev/null || { echo "❌ bv not found"; exit 1; }
```

## Your Responsibilities

1. **Select work via bv** - Use `bv --robot-plan` for dependency-respecting work selection
2. **Track status via bd** - Use `bd update` to change issue status
3. **Manage agent pool** - Maintain up to 5 concurrent agents in slots
4. **Dispatch dynamically** - Assign work immediately when slot opens
5. **Validate continuously** - Validate and commit as agents complete
6. **Learn from errors** - Extract patterns, improve future agents
7. **Handle failures** - Use `bd label` to mark failed issues

**Mode**: Fully autonomous - continuous parallel execution until all work done or user interrupts.

---

## Agent Type Inference

Infer agent type from issue title at runtime (no pre-stored assignments):

```bash
infer_agent_type() {
  local TITLE="$1"

  case "$TITLE" in
    *[Dd]atabase*|*[Mm]odel*|*[Ss]chema*|*[Mm]igration*)
      echo "database-layer-builder"
      ;;
    *[Aa][Pp][Ii]*|*[Ee]ndpoint*|*[Cc]ontroller*|*[Rr]oute*)
      echo "api-layer-builder"
      ;;
    *[Uu][Ii]*|*[Cc]omponent*|*[Pp]age*|*[Vv]iew*|*[Ff]rontend*)
      echo "ui-component-builder"
      ;;
    *[Ii]ntegration*|*[Ww]ire*|*[Ee]2[Ee]*)
      echo "integration-assembler"
      ;;
    *[Tt]est*[Gg]ap*|*[Cc]overage*)
      echo "test-gap-analyzer"
      ;;
    *)
      echo "molecule-composer"
      ;;
  esac
}
```

---

## INITIALIZATION

First, verify the project is ready and initialize minimal state.

```bash
echo "=== Autonomous Build v2 Initialization ==="
echo ""

# Verify we're at project root with Beads
if [ ! -d ".beads" ]; then
  echo "❌ No .beads/ directory found at project root"
  exit 1
fi
echo "✓ .beads directory found"

# Check for issues using bd
ISSUE_COUNT=$(bd list --json 2>/dev/null | jq '. | length' || echo "0")
echo "Found $ISSUE_COUNT total issues in Beads"

if [ "$ISSUE_COUNT" -eq 0 ]; then
  echo "❌ No Beads issues found"
  echo "Run /autonomous-plan or /create-tasks first"
  exit 1
fi

# Check ready work using bd
READY_COUNT=$(bd ready --json 2>/dev/null | jq '. | length' || echo "0")
echo "Ready to work: $READY_COUNT issues"

if [ "$READY_COUNT" -eq 0 ]; then
  # Check if everything is closed
  CLOSED_COUNT=$(bd list --status closed --json 2>/dev/null | jq '. | length' || echo "0")
  if [ "$CLOSED_COUNT" -eq "$ISSUE_COUNT" ]; then
    echo "🎉 All issues are already closed!"
    exit 0
  fi

  # Check blocked
  BLOCKED_INFO=$(bd blocked 2>/dev/null || echo "")
  echo "⚠️  No ready work. Check blocked issues:"
  echo "$BLOCKED_INFO"
  exit 1
fi

# Initialize minimal state directory (only for agent pool and learning)
mkdir -p .beads/autonomous-state/locks
mkdir -p .beads/autonomous-state/learning

# Initialize or reset agent pool (slot tracking only)
MAX_AGENTS=5
cat > .beads/autonomous-state/agent-pool.json <<EOF
{
  "max_agents": $MAX_AGENTS,
  "slots": {},
  "heartbeat_interval": 10
}
EOF

# Initialize learning system (if not exists)
if [ ! -f ".beads/autonomous-state/learning/improvements.json" ]; then
  cat > .beads/autonomous-state/learning/improvements.json <<'EOF'
{
  "common_errors": [],
  "best_practices": [],
  "conflict_patterns": []
}
EOF
fi

# Clean any stale locks
rm -f .beads/autonomous-state/locks/*.lock 2>/dev/null

echo "✓ State initialized"

# Show current state from bd/bv
echo ""
echo "=== Current State (from bd/bv) ==="

READY=$(bd ready --json 2>/dev/null | jq '. | length')
IN_PROGRESS=$(bd list --status in_progress --json 2>/dev/null | jq '. | length')
CLOSED=$(bd list --status closed --json 2>/dev/null | jq '. | length')
FAILED=$(bd list --label failed --json 2>/dev/null | jq '. | length' || echo "0")

echo "  Ready: $READY"
echo "  In Progress: $IN_PROGRESS"
echo "  Closed: $CLOSED"
echo "  Failed: $FAILED"
echo "  Agent slots: $MAX_AGENTS concurrent"
echo ""

# Get execution plan from bv
echo "=== Execution Plan (from bv --robot-plan) ==="
bv --robot-plan 2>/dev/null | jq -r '.plan.tracks[:3][] | "Track \(.track_id): \(.items | length) items - \(.reason)"'
echo ""

echo "Starting parallel agent pool (max $MAX_AGENTS concurrent)..."
echo "Say 'stop' or 'pause' anytime to interrupt"
echo ""
```

---

## MAIN LOOP

**Parallel Agent Pool Pattern**: Continuously monitor and dispatch work using bd/bv.

Each loop iteration (heartbeat):

### STEP 1: Dispatch New Work

Check for free slots and ready organisms using bd/bv:

```bash
# Get current slot usage
ACTIVE_SLOTS=$(jq '.slots | keys | length' .beads/autonomous-state/agent-pool.json)
MAX_AGENTS=$(jq '.max_agents' .beads/autonomous-state/agent-pool.json)

# Get ready work from bv (dependency-respecting)
READY_WORK=$(bv --robot-plan 2>/dev/null)

# Spawn new agents while slots available
while [[ "$ACTIVE_SLOTS" -lt "$MAX_AGENTS" ]]; do
  # Find next available slot (1-5)
  SLOT=""
  for i in 1 2 3 4 5; do
    if ! jq -e ".slots[\"$i\"]" .beads/autonomous-state/agent-pool.json &>/dev/null; then
      SLOT=$i
      break
    fi
  done

  if [ -z "$SLOT" ]; then
    break  # No free slots
  fi

  # Select next organism from bv --robot-plan
  # Iterate through tracks to find first available (not locked, not in_progress)
  ORGANISM_ID=""
  for TRACK in $(echo "$READY_WORK" | jq -r '.plan.tracks[].track_id'); do
    CANDIDATE=$(echo "$READY_WORK" | jq -r ".plan.tracks[] | select(.track_id == \"$TRACK\") | .items[0].id // empty")

    if [ -z "$CANDIDATE" ]; then
      continue
    fi

    # Check if already locked by another slot
    if [ -f ".beads/autonomous-state/locks/${CANDIDATE}.lock" ]; then
      continue
    fi

    # Check current status via bd
    STATUS=$(bd show "$CANDIDATE" --json 2>/dev/null | jq -r '.status')
    if [ "$STATUS" = "open" ]; then
      ORGANISM_ID="$CANDIDATE"
      break
    fi
  done

  if [ -z "$ORGANISM_ID" ]; then
    break  # No ready work
  fi

  # Get organism details from bd
  ORGANISM_JSON=$(bd show "$ORGANISM_ID" --json 2>/dev/null)
  ORGANISM_TITLE=$(echo "$ORGANISM_JSON" | jq -r '.title')

  # Infer agent type from title
  AGENT_TYPE=$(infer_agent_type "$ORGANISM_TITLE")

  # Create lock
  echo "$SLOT|$(date -Iseconds)" > ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

  # Update bd status to in_progress
  bd update "$ORGANISM_ID" --status in_progress 2>/dev/null

  echo "🚀 Spawning agent in slot $SLOT: $ORGANISM_ID"
  echo "   Title: $ORGANISM_TITLE"
  echo "   Agent: $AGENT_TYPE"

  # Load learning context
  LEARNING_CONTEXT=""
  IMPROVEMENTS=$(cat .beads/autonomous-state/learning/improvements.json 2>/dev/null || echo "{}")
  if [ "$(echo "$IMPROVEMENTS" | jq '.common_errors | length')" -gt 0 ]; then
    LEARNING_CONTEXT=$(echo "$IMPROVEMENTS" | jq -r '
      "LEARN FROM PREVIOUS ERRORS:\n" +
      (.common_errors[:3] | map("  ❌ \(.pattern)\n  ✅ \(.fix)") | join("\n"))
    ')
  fi

  # Build agent prompt
  TASK_PROMPT="You are implementing organism $ORGANISM_ID.

$LEARNING_CONTEXT

## Your Task

Implement this organism, test it, and close it.

## Checkpoint Protocol

Log progress via bd comment after EACH step:
\`\`\`bash
bd comment $ORGANISM_ID \"CHECKPOINT: step N - description\"
\`\`\`

After each commit:
\`\`\`bash
bd comment $ORGANISM_ID \"COMMIT: \$(git rev-parse --short HEAD) - description\"
\`\`\`

## Process

1. **Read organism details**:
   \`\`\`bash
   bd show $ORGANISM_ID
   bd comment $ORGANISM_ID \"CHECKPOINT: step 1 - read details\"
   \`\`\`

2. **Implement**:
   - Read any linked specs
   - Implement the feature
   - **COMMIT after implementation**
   \`\`\`bash
   git add -A && git commit -m \"impl($ORGANISM_ID): description\"
   bd comment $ORGANISM_ID \"COMMIT: \$(git rev-parse --short HEAD) - implementation\"
   \`\`\`

3. **Write tests** (if applicable):
   - Write tests for the implementation
   - **COMMIT after writing tests**
   \`\`\`bash
   git add -A && git commit -m \"test($ORGANISM_ID): description\"
   bd comment $ORGANISM_ID \"COMMIT: \$(git rev-parse --short HEAD) - tests\"
   \`\`\`

4. **Run tests**:
   - Verify tests pass
   \`\`\`bash
   bd comment $ORGANISM_ID \"CHECKPOINT: step 4 - tests passing\"
   \`\`\`

5. **Close organism**:
   \`\`\`bash
   bd close $ORGANISM_ID
   bd comment $ORGANISM_ID \"CHECKPOINT: COMPLETE\"
   \`\`\`

**IMPORTANT**: Commit early, commit often. Each commit is recoverable.

Project root: $(pwd)
Organism: $ORGANISM_ID"

  echo ""
  echo "**NOW SPAWN AGENT VIA TASK TOOL**:"
  echo "  subagent_type: \"$AGENT_TYPE\""
  echo "  description: \"Implement $ORGANISM_ID (slot $SLOT)\""
  echo "  run_in_background: true"
  echo "  prompt: [see TASK_PROMPT above]"
  echo ""

  # After spawning, update agent pool with task_id
  # NOTE: You (Claude) will call Task tool and get task_id back
  # Then update: jq ".slots[\"$SLOT\"] = {organism_id: \"$ORGANISM_ID\", task_id: \"TASK_ID\"}"

  ACTIVE_SLOTS=$((ACTIVE_SLOTS + 1))
done
```

**IMPORTANT**: You (Claude orchestrator) call the Task tool to spawn agents. After receiving task_id, update the agent pool:

```bash
jq ".slots[\"$SLOT\"] = {\"organism_id\": \"$ORGANISM_ID\", \"task_id\": \"$TASK_ID\", \"started_at\": \"$(date -Iseconds)\"}" \
  .beads/autonomous-state/agent-pool.json > /tmp/ap.json && mv /tmp/ap.json .beads/autonomous-state/agent-pool.json
```

### STEP 2: Monitor Active Agents (Context-Aware)

**CRITICAL**: Do NOT read full agent output - it fills your context and causes exhaustion.

Check status of all active agents using **status-only** polling:

```bash
# For each active slot
for SLOT in $(jq -r '.slots | keys[]' .beads/autonomous-state/agent-pool.json 2>/dev/null); do
  TASK_ID=$(jq -r ".slots[\"$SLOT\"].task_id" .beads/autonomous-state/agent-pool.json)
  ORGANISM_ID=$(jq -r ".slots[\"$SLOT\"].organism_id" .beads/autonomous-state/agent-pool.json)

  echo "Checking slot $SLOT: $ORGANISM_ID"

  # IMPORTANT: Check bd status FIRST (doesn't fill context)
  BD_STATUS=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.status')

  if [ "$BD_STATUS" = "closed" ]; then
    echo "✅ Organism closed (bd confirms)"
    # Proceed to STEP 3 for this slot
  else
    # Only check TaskOutput if bd says not closed yet
    echo "  Status: $BD_STATUS (checking task...)"
    # TaskOutput(task_id, block=false) - status only, don't expand output
  fi
done
```

**Tool call pattern** (you Claude orchestrator):
```python
# Check task status ONLY - do NOT read output content
TaskOutput(task_id="xxx", block=False)
# Just note: "running" or "completed"
# Do NOT expand or read the output content
```

### STEP 3: Handle Agent Completions (Minimal Context)

**CRITICAL**: Validate via bd, NOT by reading agent output.

The agent's job was to close the organism via `bd close`. We check bd, not agent output.

```bash
# For completed task with ORGANISM_ID, SLOT:

echo "Agent in slot $SLOT finished: $ORGANISM_ID"

# Check organism status via bd (source of truth)
ORGANISM_STATUS=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.status')

if [ "$ORGANISM_STATUS" = "closed" ]; then
  # SUCCESS - agent closed it
  echo "🎉 $ORGANISM_ID completed"

  # Free slot
  jq "del(.slots[\"$SLOT\"])" .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json
  rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

else
  # FAILURE - agent didn't close
  echo "❌ $ORGANISM_ID not closed (status: $ORGANISM_STATUS)"

  # Check last checkpoint via bd (one line only)
  LAST_CHECKPOINT=$(bd comments "$ORGANISM_ID" --json 2>/dev/null | jq -r '.[-1].body // "none"' | head -c 100)
  echo "  Last checkpoint: $LAST_CHECKPOINT..."

  # Track attempts via bd labels
  ATTEMPT=$(bd show "$ORGANISM_ID" --json 2>/dev/null | jq -r '.labels | map(select(startswith("attempt-"))) | .[0] // "attempt-0"' | sed 's/attempt-//')
  ATTEMPT=$((ATTEMPT + 1))

  if [ "$ATTEMPT" -lt 3 ]; then
    # Retry
    echo "  Queuing retry $ATTEMPT/3"
    bd label remove "$ORGANISM_ID" "attempt-$((ATTEMPT - 1))" 2>/dev/null || true
    bd label add "$ORGANISM_ID" "attempt-$ATTEMPT"
    bd update "$ORGANISM_ID" --status open
  else
    # Failed
    echo "  Max attempts - marking failed"
    bd label add "$ORGANISM_ID" "failed"
    bd comment "$ORGANISM_ID" "FAILED after 3 attempts"
  fi

  # Free slot
  jq "del(.slots[\"$SLOT\"])" .beads/autonomous-state/agent-pool.json > /tmp/ap.json
  mv /tmp/ap.json .beads/autonomous-state/agent-pool.json
  rm -f ".beads/autonomous-state/locks/${ORGANISM_ID}.lock"
fi

# CONTEXT CHECK: If context is low, /compact NOW before next slot
```

**DO NOT**:
- Read full agent output via TaskOutput(block=True)
- Expand collapsed outputs
- Store agent outputs in variables

**DO**:
- Check bd status (external, doesn't fill context)
- Check task completion status only (running/completed)
- Run `/compact` if context warning appears

### STEP 4: Check Completion

Check if all work is done using bd:

```bash
# Query bd for current state
READY=$(bd ready --json 2>/dev/null | jq '. | length')
IN_PROGRESS=$(bd list --status in_progress --json 2>/dev/null | jq '. | length')
ACTIVE_SLOTS=$(jq '.slots | keys | length' .beads/autonomous-state/agent-pool.json)

if [[ "$READY" -eq 0 ]] && [[ "$IN_PROGRESS" -eq 0 ]] && [[ "$ACTIVE_SLOTS" -eq 0 ]]; then
  echo ""
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo "  🎉 ALL WORK COMPLETE!"
  echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  echo ""

  # Get final stats from bd
  TOTAL=$(bd list --json 2>/dev/null | jq '. | length')
  CLOSED=$(bd list --status closed --json 2>/dev/null | jq '. | length')
  FAILED=$(bd list --label failed --json 2>/dev/null | jq '. | length' || echo "0")

  SUCCESS_RATE=$(echo "scale=1; ($CLOSED - $FAILED) * 100 / $TOTAL" | bc 2>/dev/null || echo "N/A")

  echo "Final Statistics (from bd):"
  echo "  Total organisms: $TOTAL"
  echo "  Closed: $CLOSED"
  echo "  Failed: $FAILED"
  echo "  Success rate: ${SUCCESS_RATE}%"
  echo ""

  if [ "$FAILED" -gt 0 ]; then
    echo "Failed organisms requiring attention:"
    bd list --label failed --json 2>/dev/null | jq -r '.[] | "  ❌ \(.id): \(.title)"'
  fi

  exit 0
fi
```

### STEP 5: Display Progress Dashboard

Show current state from bd/bv:

```bash
echo ""
echo "=== Progress Dashboard ==="

# All data from bd/bv
READY=$(bd ready --json 2>/dev/null | jq '. | length')
IN_PROGRESS=$(bd list --status in_progress --json 2>/dev/null | jq '. | length')
CLOSED=$(bd list --status closed --json 2>/dev/null | jq '. | length')
FAILED=$(bd list --label failed --json 2>/dev/null | jq '. | length' || echo "0")
ACTIVE_SLOTS=$(jq '.slots | keys | length' .beads/autonomous-state/agent-pool.json)

echo "Work Queue (from bd):"
echo "  Ready: $READY | In Progress: $IN_PROGRESS | Closed: $CLOSED | Failed: $FAILED"

echo ""
echo "Agent Pool:"
if [ "$ACTIVE_SLOTS" -gt 0 ]; then
  jq -r '.slots | to_entries[] | "  Slot \(.key): \(.value.organism_id)"' .beads/autonomous-state/agent-pool.json
else
  echo "  (no active agents)"
fi

# BV insights
echo ""
echo "BV Insights:"
bv --robot-priority 2>/dev/null | jq -r '.recommendations[:2][] | "  \(.issue_id): \(.reasoning | join(", "))"' || echo "  (none)"

echo ""
```

### STEP 6: Heartbeat Sleep

Wait before next iteration:

```bash
HEARTBEAT=$(jq '.heartbeat_interval' .beads/autonomous-state/agent-pool.json)
sleep $HEARTBEAT
```

### STEP 7: Clear Context Before Next Iteration

**CRITICAL**: Before starting the next loop iteration, clear context to prevent accumulation.

```bash
echo ""
echo "=== End of Iteration ==="
echo ""

# Summarize this iteration
COMPLETED_THIS_ITER=$(bd list --status closed --json 2>/dev/null | jq '. | length')
FAILED_THIS_ITER=$(bd list --label failed --json 2>/dev/null | jq '. | length' || echo "0")
READY_REMAINING=$(bd ready --json 2>/dev/null | jq '. | length')

echo "Iteration summary:"
echo "  Completed: $COMPLETED_THIS_ITER total"
echo "  Failed: $FAILED_THIS_ITER total"
echo "  Remaining: $READY_REMAINING ready"
echo ""

# Learnings already persisted to improvements.json in STEP 3
echo "Learnings persisted to improvements.json"
echo ""
```

**NOW CLEAR CONTEXT**: You (Claude orchestrator) MUST use `/compact` or equivalent to clear your context before starting the next iteration. This prevents context exhaustion.

**Why this matters:**
- Each iteration accumulates agent outputs, validation results, error logs
- Without clearing, context fills up and orchestrator dies mid-loop
- Learnings are persisted to `improvements.json`, so nothing is lost
- bd/bv state is external, so nothing is lost

**After clearing context, loop back to STEP 1.**

---

## How to Run This

You (Claude) are the orchestrator. When user runs `/autonomous-build`:

### 1. INITIALIZATION
- Verify .beads directory exists
- Check issue count via `bd list --json | jq '. | length'`
- Check ready work via `bd ready --json | jq '. | length'`
- Initialize minimal agent-pool.json (slot tracking only)

### 2. ENTER MAIN LOOP

Loop continuously:

**STEP 1: Dispatch**
- Check free slots in agent-pool.json
- Get work plan from `bv --robot-plan`
- For each free slot:
  - Select next organism from plan
  - Get details via `bd show <id> --json`
  - Infer agent type from title
  - Create lock file
  - Update status via `bd update <id> --status in_progress`
  - **Spawn agent via Task tool** (run_in_background=true)
  - Update agent-pool.json with task_id

**STEP 2: Monitor (Context-Aware)**
- For each active slot:
  - Check bd status FIRST: `bd show <id> --json | jq '.status'`
  - Only poll TaskOutput if bd says not closed
  - **DO NOT read/expand agent output** - fills context
  - If context low: `/compact` immediately

**STEP 3: Handle Completions (Minimal Context)**
- Check organism status via bd (source of truth)
- If closed: success, free slot
- If not closed: check attempt label, retry or mark failed
- **DO NOT read agent output** - just check bd

**STEP 4: Check Completion**
- Query `bd ready`, `bd list --status in_progress`
- If all done: exit

**STEP 5: Display Progress**
- Show stats from bd queries

**STEP 6: Sleep**
- Wait heartbeat_interval

**STEP 7: Clear Context**
- Summarize iteration (completed, failed, remaining)
- Learnings already persisted to improvements.json
- **Use `/compact` to clear context**
- Loop back to STEP 1

### 3. Key Principles

| Principle | Implementation |
|-----------|----------------|
| **Issue tracking** | bd exclusively |
| **Work selection** | bv --robot-plan |
| **Status updates** | bd update, bd close |
| **Failure tracking** | bd label add failed |
| **Progress tracking** | bd comment |
| **Slot management** | Minimal agent-pool.json |
| **Context management** | NEVER read agent output; check bd status; `/compact` proactively |

### 4. Tool Calls

```python
# STEP 1: Spawn agents
Task(
    subagent_type=AGENT_TYPE,  # Inferred from title
    description="Implement bd-123 (slot 1)",
    prompt="[implementation prompt]",
    run_in_background=True
)

# STEP 2: Monitor (non-blocking)
TaskOutput(task_id="abc123", block=False)

# STEP 3: Get output (blocking)
TaskOutput(task_id="abc123", block=True)
```

### 5. State Recovery

If interrupted:
1. User runs `/autonomous-build` again
2. Locks are cleaned
3. Agent pool is reset
4. bd status reflects true state (in_progress organisms are already marked)
5. Loop resumes with bd/bv queries

**No custom state files to recover** - bd/bv ARE the source of truth.
