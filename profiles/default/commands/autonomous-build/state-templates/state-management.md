# State Management

This document defines how the autonomous-build orchestrator manages state.

## Core Principle

**`bd` and `bv` ARE the source of truth.** We do NOT maintain parallel state files.

| Need | Command |
|------|---------|
| Total issues | `bd count` |
| Ready work | `bd count --status open` |
| In-progress work | `bd count --status in_progress` |
| Completed work | `bd count --status closed` |
| Failed work | `bd count --label failed` |
| Work selection | `bv --robot-plan` |
| Priority insights | `bv --robot-priority` |
| Blocked issues | `bd blocked` |

## Minimal Local State

We only track two things locally:

1. **Agent slot assignments** - Which Task is working on which issue
2. **Lock files** - Prevent double-assignment

### Agent Pool (agent-pool.txt)

Simple text format:

```
max_agents=5
heartbeat=10
slot1=
slot2=<issue-id>|<task-id>
slot3=
slot4=<issue-id>|<task-id>
slot5=
```

- Empty slot: `slotN=`
- Occupied slot: `slotN=<issue-id>|<task-id>`

**Read**: Use Read tool
**Update**: Use Edit tool or Write tool

### Lock Files

Location: `.beads/autonomous-state/locks/<issue-id>.lock`

Content: `<slot>|<timestamp>`

Purpose: Prevent same issue being claimed twice.

## State Operations

### Initialize State

1. `mkdir -p .beads/autonomous-state/locks`
2. `mkdir -p .beads/autonomous-state/learning`
3. Write empty agent-pool.txt
4. `rm -f .beads/autonomous-state/locks/*.lock`

### Query Ready Work

Use bd commands:
- `bd count --status open` - count of ready issues
- `bd ready` - list of ready issues
- `bv --robot-plan` - dependency-respecting work plan (JSON, Claude parses)

### Claim an Issue

1. Check not locked: `test -f .beads/autonomous-state/locks/<ID>.lock`
2. Check status: `bd show <ID>` → look for "Status: open"
3. Create lock: Write `<slot>|<timestamp>` to `.beads/autonomous-state/locks/<ID>.lock`
4. Update bd: `bd update <ID> --status in_progress`
5. Update agent-pool.txt: Set `slotN=<ID>|<task-id>`

### Release an Issue (Success)

1. Remove lock: `rm .beads/autonomous-state/locks/<ID>.lock`
2. Issue already closed by agent via `bd close <ID>`
3. Clear slot in agent-pool.txt: Set `slotN=`

### Release an Issue (Failure)

1. Remove lock: `rm .beads/autonomous-state/locks/<ID>.lock`
2. Mark failed: `bd label add <ID> failed`
3. Add comment: `bd comment <ID> "FAILED: reason"`
4. Clear slot in agent-pool.txt

### Track Attempts

Check labels in `bd show <ID>` output for `attempt-N`.

Increment:
1. `bd label add <ID> attempt-N` (where N = current + 1)
2. `bd update <ID> --status open`

### Query Current State

All from bd:
- `bd count --status open` → READY
- `bd count --status in_progress` → IN_PROGRESS
- `bd count --status closed` → CLOSED
- `bd count --label failed` → FAILED

Active agents: Count occupied slots in agent-pool.txt.

## State Recovery

If interrupted:

1. Clean stale locks: `rm -f .beads/autonomous-state/locks/*.lock`
2. Reset agent pool: Write fresh agent-pool.txt with empty slots
3. Recover orphaned issues: `bd list --status in_progress` → reset each to open

**Why this works:** bd status is the source of truth. Issues marked `in_progress` with no active agent are orphaned and get reset to `open`.

## Summary

| What we DON'T track locally | What we DO track locally |
|-----------------------------|-----------------------------|
| Issue status | Agent-pool.txt (slot→issue mapping) |
| Issue metadata | Lock files (temporary) |
| Priority | |
| Dependencies | |
| Progress history | |

**bd/bv = source of truth. Local state = minimal coordination.**
