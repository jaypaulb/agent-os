# State Management Workflow

This workflow defines how the autonomous-build orchestrator manages state using `bd` and `bv` exclusively.

## Core Principle

**`bd` and `bv` ARE the source of truth.** We do NOT maintain parallel state files.

| Need | bd/bv Solution |
|------|----------------|
| Ready work | `bd ready --json` |
| In-progress work | `bd list --status in_progress --json` |
| Completed work | `bd list --status closed --json` |
| Failed work | `bd list --label failed --json` |
| Work selection | `bv --robot-plan` |
| Priority insights | `bv --robot-priority` |
| Blocked issues | `bd blocked` |

## Minimal Local State

We only track two things locally:

1. **Agent slot assignments** - Which Claude Code task is working on which issue
2. **Learning data** - Error patterns for improving future agents

### Agent Pool (agent-pool.json)

```json
{
  "max_agents": 5,
  "slots": {
    "1": {"organism_id": "bd-123", "task_id": "abc123", "started_at": "2024-01-01T00:00:00Z"},
    "2": {"organism_id": "bd-456", "task_id": "def456", "started_at": "2024-01-01T00:00:00Z"}
  },
  "heartbeat_interval": 10
}
```

### Learning System (improvements.json)

```json
{
  "common_errors": [
    {"pattern": "Missing import", "fix": "Check imports at top of file", "seen_count": 3}
  ],
  "best_practices": [],
  "conflict_patterns": []
}
```

## State Operations

### Initialize State

```bash
# Create minimal state directory
mkdir -p .beads/autonomous-state/locks
mkdir -p .beads/autonomous-state/learning

# Initialize agent pool
cat > .beads/autonomous-state/agent-pool.json <<'EOF'
{
  "max_agents": 5,
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
```

### Query Ready Work

```bash
# Use bd ready for issues with no blockers
bd ready --json | jq '. | length'

# Use bv --robot-plan for dependency-respecting parallel tracks
bv --robot-plan | jq '.plan.tracks'
```

### Claim an Issue

```bash
ISSUE_ID="bd-123"
SLOT=1

# Create lock file
echo "$SLOT|$(date -Iseconds)" > ".beads/autonomous-state/locks/${ISSUE_ID}.lock"

# Update status via bd
bd update "$ISSUE_ID" --status in_progress

# Update agent pool
jq ".slots[\"$SLOT\"] = {\"organism_id\": \"$ISSUE_ID\", \"task_id\": \"$TASK_ID\", \"started_at\": \"$(date -Iseconds)\"}" \
  .beads/autonomous-state/agent-pool.json > /tmp/ap.json && mv /tmp/ap.json .beads/autonomous-state/agent-pool.json
```

### Release an Issue (Success)

```bash
ISSUE_ID="bd-123"
SLOT=1

# Remove lock
rm -f ".beads/autonomous-state/locks/${ISSUE_ID}.lock"

# Issue is already closed by agent via: bd close $ISSUE_ID

# Free slot
jq "del(.slots[\"$SLOT\"])" .beads/autonomous-state/agent-pool.json > /tmp/ap.json
mv /tmp/ap.json .beads/autonomous-state/agent-pool.json
```

### Release an Issue (Failure)

```bash
ISSUE_ID="bd-123"
SLOT=1
ATTEMPT=3

# Remove lock
rm -f ".beads/autonomous-state/locks/${ISSUE_ID}.lock"

# Mark as failed via bd label
bd label add "$ISSUE_ID" "failed"
bd comment "$ISSUE_ID" "FAILED: Max attempts ($ATTEMPT) reached. Manual review required."

# Free slot
jq "del(.slots[\"$SLOT\"])" .beads/autonomous-state/agent-pool.json > /tmp/ap.json
mv /tmp/ap.json .beads/autonomous-state/agent-pool.json
```

### Track Attempts

```bash
ISSUE_ID="bd-123"

# Check current attempt via label
ATTEMPT=$(bd show "$ISSUE_ID" --json | jq -r '.labels | map(select(startswith("attempt-"))) | .[0] // "attempt-0"' | sed 's/attempt-//')

# Increment attempt
NEXT_ATTEMPT=$((ATTEMPT + 1))
bd label remove "$ISSUE_ID" "attempt-$ATTEMPT" 2>/dev/null || true
bd label add "$ISSUE_ID" "attempt-$NEXT_ATTEMPT"

# Reset for retry
bd update "$ISSUE_ID" --status open
```

### Query Current State

```bash
# Ready count
READY=$(bd ready --json | jq '. | length')

# In-progress count
IN_PROGRESS=$(bd list --status in_progress --json | jq '. | length')

# Closed count
CLOSED=$(bd list --status closed --json | jq '. | length')

# Failed count (via label)
FAILED=$(bd list --label failed --json | jq '. | length' || echo "0")

# Active agents
ACTIVE=$(jq '.slots | keys | length' .beads/autonomous-state/agent-pool.json)

echo "Ready: $READY | In Progress: $IN_PROGRESS | Closed: $CLOSED | Failed: $FAILED | Active: $ACTIVE"
```

## State Recovery

If interrupted (Ctrl+C, crash), recovery is simple:

```bash
# Clean stale locks
rm -f .beads/autonomous-state/locks/*.lock

# Reset agent pool (slots)
jq '.slots = {}' .beads/autonomous-state/agent-pool.json > /tmp/ap.json
mv /tmp/ap.json .beads/autonomous-state/agent-pool.json

# bd status is already correct - in_progress issues stay in_progress
# They'll be picked up on the next run
```

**Why this works:** bd status is the source of truth. Issues marked `in_progress` will be retried automatically when the next run queries `bd ready` (which includes in_progress issues).

## Learning System

Extract errors and improve future agents:

```bash
# Add error pattern
add_error_pattern() {
  local PATTERN="$1"
  local FIX="$2"

  # Check if pattern exists
  EXISTS=$(jq ".common_errors[] | select(.pattern == \"$PATTERN\")" .beads/autonomous-state/learning/improvements.json)

  if [ -n "$EXISTS" ]; then
    # Increment count
    jq "(.common_errors[] | select(.pattern == \"$PATTERN\")).seen_count += 1" \
      .beads/autonomous-state/learning/improvements.json > /tmp/imp.json
  else
    # Add new
    jq ".common_errors += [{\"pattern\": \"$PATTERN\", \"fix\": \"$FIX\", \"seen_count\": 1}]" \
      .beads/autonomous-state/learning/improvements.json > /tmp/imp.json
  fi

  mv /tmp/imp.json .beads/autonomous-state/learning/improvements.json
}

# Get improvements for agent prompt
get_improvements() {
  jq -r '.common_errors[:3] | map("❌ \(.pattern)\n✅ \(.fix)") | join("\n\n")' \
    .beads/autonomous-state/learning/improvements.json
}
```

## Summary

| What we DON'T track locally | What we DO track locally |
|-----------------------------|--------------------------|
| work-queue.json | agent-pool.json (slots only) |
| orchestration.yml | improvements.json (learning) |
| Issue status | Lock files (temp) |
| Issue metadata | |
| Priority | |
| Dependencies | |

**bd/bv = source of truth. Local state = minimal coordination.**
