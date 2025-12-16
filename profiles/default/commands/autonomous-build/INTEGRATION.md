# Autonomous Build v2: Integration Guide

## Overview

This guide explains how the autonomous-build v2 system integrates with orchestrate-tasks to provide parallel dev team management.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│     /orchestrate-tasks --headless (Orchestrator Setup)      │
│                                                              │
│  1. Discover phases and organisms                           │
│  2. Analyze parallel execution opportunities (BV)           │
│  3. Assign specialized agents to organisms                  │
│  4. Create .beads/autonomous-state/orchestration.yml        │
│  5. Exit (return control)                                   │
└─────────────────────────────────────────────────────────────┘
              ↓ (creates orchestration.yml)
┌─────────────────────────────────────────────────────────────┐
│     /autonomous-build (Parallel Team Manager)               │
│                                                              │
│  1. Load orchestration.yml                                  │
│  2. Initialize work queue and agent pool                    │
│  3. Manage up to 5 concurrent agents                        │
│  4. Validate organisms through 5-gate pipeline              │
│  5. Handle merge conflicts (3-tier resolution)              │
│  6. Learn from errors (improvements.json)                   │
│  7. Commit verified work immediately                        │
│  8. Loop until all organisms complete                       │
└─────────────────────────────────────────────────────────────┘
              ↓ (spawns up to 5 in parallel)
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ database-layer   │ │ api-layer        │ │ ui-component     │
│ (Slot 1)         │ │ (Slot 2)         │ │ (Slot 3)         │
│ organism A       │ │ organism B       │ │ organism C       │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

---

## Workflow

### Step 1: Run orchestrate-tasks in headless mode

```bash
/orchestrate-tasks --headless
```

**What it does:**
- Discovers all organism issues from Beads
- Uses BV to analyze parallel execution opportunities
- Assigns specialized agents based on organism type (database-layer, api-layer, ui-component, etc.)
- Creates `.beads/autonomous-state/orchestration.yml` with assignments
- Exits immediately (does not spawn agents)

**Output:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ Headless Mode Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Orchestration state prepared for autonomous-build:
  • File: .beads/autonomous-state/orchestration.yml
  • Organisms: 47
  • Agents assigned: Yes

You can now run /autonomous-build to begin parallel implementation
```

### Step 2: Run autonomous-build

```bash
/autonomous-build
```

**What it does:**
- Loads `.beads/autonomous-state/orchestration.yml`
- Initializes state files (work-queue.json, agent-pool.json, improvements.json)
- Spawns agents dynamically as slots become available (up to 5 concurrent)
- Validates each organism through 5-gate pipeline
- Handles merge conflicts with 3-tier resolution
- Learns from errors and improves subsequent agents
- Commits verified work immediately (no batch waiting)

**Main loop:**
```
1. DISPATCH NEW WORK: Spawn agents while slots available
2. MONITOR ACTIVE AGENTS: Non-blocking poll (TaskOutput block=false)
3. HANDLE COMPLETIONS: Validate → Commit OR Retry
4. CHECK COMPLETION: Exit when work queue empty
5. DISPLAY PROGRESS: BV-powered dashboard
6. HEARTBEAT SLEEP: Brief sleep (10 seconds)
```

**Output example:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  AUTONOMOUS BUILD v2: Parallel Dev Team Manager
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Loaded 47 organisms from orchestration.yml

STEP 1: Dispatch New Work
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Spawned agent for bd-org-123 (User validation atom) [Slot 1]
✓ Spawned agent for bd-org-124 (Database migration) [Slot 2]
✓ Spawned agent for bd-org-125 (Login API endpoint) [Slot 3]
✓ Spawned agent for bd-org-126 (Login form component) [Slot 4]
✓ Spawned agent for bd-org-127 (Auth middleware) [Slot 5]

STEP 2: Monitor Active Agents (5 running)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Slot 1] bd-org-123: running (15 min elapsed)
[Slot 2] bd-org-124: running (12 min elapsed)
[Slot 3] bd-org-125: running (8 min elapsed)
[Slot 4] bd-org-126: COMPLETED
[Slot 5] bd-org-127: running (5 min elapsed)

STEP 3: Handle Agent Completions
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Validating bd-org-126 (Login form component)...

Gate 1: Run Tests ✓
Gate 2: Integration Test ✓
Gate 3: Merge Detection ✓
Gate 4: Regression Sample ✓
Gate 5: Quality Checks ✓

✅ All validation gates passed - committing organism
📦 Committed: 7a3f2b1

Moving to completed queue...
[ready: 42] [in_progress: 4] [completed: 1] [failed: 0]

STEP 1: Dispatch New Work (Slot 4 available)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Spawned agent for bd-org-128 (Password reset page) [Slot 4]

...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ ALL ORGANISMS COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total organisms: 47
✓ Completed: 44
⚠️  Failed (escalated): 3

Success rate: 93.6%
Total time: 3.5 hours

Failed organisms escalated to manual review:
  • bd-org-143: Complex auth integration (3 validation failures)
  • bd-org-157: Real-time notifications (conflict unresolved)
  • bd-org-162: Payment processing (needs security review)
```

---

## State Files

All state persists in `.beads/autonomous-state/`:

### orchestration.yml
Created by `orchestrate-tasks --headless`, consumed by `autonomous-build`.

```yaml
# Multi-Phase Orchestration Plan
# Generated: 2025-12-16T15:30:00Z
# Total Phases: 3
# Mode: headless
beads:
  - id: bd-org-123
    title: User validation atom
    assignee: molecule-composer
    standards:
      - global/atomic-design.md
  - id: bd-org-124
    title: Database migration for users table
    assignee: database-layer-builder
    standards:
      - backend/*
      - global/atomic-design.md
  # ... 45 more organisms
```

### work-queue.json
Managed by autonomous-build. Tracks organism lifecycle.

```json
{
  "ready": ["bd-org-123", "bd-org-124", ...],
  "in_progress": [
    {"id": "bd-org-125", "agent_id": "a1b2c3d4", "slot": 1, "started_at": "..."},
    {"id": "bd-org-126", "agent_id": "e5f6g7h8", "slot": 2, "started_at": "..."}
  ],
  "completed": ["bd-org-127", "bd-org-128", ...],
  "failed": [
    {"id": "bd-org-143", "attempts": 3, "escalated": true, "reason": "..."}
  ],
  "blocked": []
}
```

### agent-pool.json
Managed by autonomous-build. Tracks active agents in 5 slots.

```json
{
  "max_agents": 5,
  "active": [
    {"slot": 1, "organism_id": "bd-org-125", "agent_id": "a1b2c3d4", "started_at": "..."},
    {"slot": 2, "organism_id": "bd-org-126", "agent_id": "e5f6g7h8", "started_at": "..."}
  ],
  "available_slots": [3, 4, 5]
}
```

### learning/improvements.json
Updated by error-analyzer after validation failures.

```json
{
  "common_errors": [
    {
      "pattern": "Missing import statement",
      "fix": "Always verify imports before implementing. Check existing files for import patterns.",
      "category": "imports",
      "seen_count": 8,
      "first_seen": "2025-12-16T10:00:00Z",
      "last_seen": "2025-12-16T14:30:00Z",
      "trending": "down"
    }
  ],
  "best_practices": [
    "Write tests before implementing feature",
    "Run tests after each significant change"
  ],
  "conflict_patterns": ["Type definition files", "Barrel/index files"],
  "organism_specific": {
    "database-layer": ["Always check for existing migrations before creating new ones"]
  },
  "metrics": {
    "total_iterations": 15,
    "total_organisms": 47,
    "success_rate": 0.936
  }
}
```

---

## Key Features

### 1. Dynamic Dispatch (No Bottleneck)

**Problem:** orchestrate-tasks v1 waited for ALL agents to complete before continuing.

**Solution:** autonomous-build v2 spawns new work immediately when slot opens.

```
T=0:   Spawn 5 agents
T=5:   Agent 1 completes → validate → commit → spawn agent 6
T=10:  Agent 3 completes → validate → commit → spawn agent 7
...continuous until queue empty
```

**Result:** 3-4x throughput increase.

### 2. 5-Gate Validation Pipeline

Each organism validated through:
1. **Gate 1: Run Tests** - Organism-specific tests
2. **Gate 2: Integration Test** - Dependencies work together
3. **Gate 3: Merge Detection** - Check for conflicts with main
4. **Gate 4: Regression Sample** - Random previous organism still works
5. **Gate 5: Quality Checks** - Linting, type checking, coverage

**If any gate fails:** Extract errors → Learn → Retry (up to 3 attempts)

### 3. 3-Tier Merge Conflict Resolution

**Conflict rate:** 15-25% (expected with parallel agents)

**Resolution strategy:**

```
Attempt 1: Conflict-aware retry
  ↓ Feed conflict diff to agent
  ↓ Agent regenerates with conflict fix
  ↓ SUCCESS: commit | FAIL ↓

Attempt 2: Serialize conflicting work
  ↓ Block organism until conflicting organism completes
  ↓ Retry after blocker clears
  ↓ SUCCESS: commit | FAIL ↓

Attempt 3: Manual review escalation
  ↓ Create analysis issue
  ↓ Assign to Jaypaul
  ↓ Continue with other work
```

**Auto-resolution rate:** 70%+ (expected)

### 4. Learning System

**Error extraction:** After each validation failure, extract patterns.

Example:
```
Test failure: "Cannot find module './validators/email'"
  → Extract: "Missing import statement"
  → Generate fix: "Always verify imports before implementing"
  → Add to improvements.json with seen_count
```

**Improvement injection:** Before spawning agent, inject learnings into prompt.

```markdown
IMPORTANT: Learn from previous sessions to avoid common mistakes:

imports issues:
  ❌ Missing import statement
  ✅ Always verify imports before implementing. Check existing files for import patterns.

types issues:
  ❌ Type mismatch - expected function
  ✅ Check interface definitions before using types. Read type files in templates/interfaces/

⚠️  RETRY MODE (Attempt 2):
This organism failed validation previously. Pay extra attention to:
  • Test failures and error messages
  • Import statements and dependencies
  • Type definitions and interfaces
```

**Result:** Success rate improves over iterations (60% → 85%+).

---

## Recovery and State Management

### Interruption Recovery

If autonomous-build is interrupted (Ctrl+C, crash, timeout), state persists:

```bash
# Resume from where you left off
/autonomous-build
```

**What happens:**
- Loads work-queue.json to see what's in_progress
- Checks which agents are still running (TaskOutput)
- Completes in-progress work
- Continues dispatching from ready queue

### Lock Mechanism

Per-organism locks prevent double-claiming:

```bash
.beads/autonomous-state/locks/
├── bd-org-123.lock  # Contains agent_id
├── bd-org-124.lock
└── bd-org-125.lock
```

**Claim logic:**
```bash
claim_organism() {
    local ORGANISM_ID=$1
    local LOCK_FILE=".beads/autonomous-state/locks/${ORGANISM_ID}.lock"

    if [ -f "$LOCK_FILE" ]; then
        # Already claimed
        return 1
    fi

    # Create lock
    echo "$AGENT_ID" > "$LOCK_FILE"
    return 0
}
```

---

## BV Integration

### Execution Plan

Before running, BV analyzes parallel opportunities:

```bash
bv --execution-plan
```

**Output:**
```json
{
  "tracks": [
    {
      "track_id": 1,
      "reason": "User authentication flow",
      "items": ["bd-org-123", "bd-org-124", "bd-org-125"],
      "parallel_safe": true
    },
    {
      "track_id": 2,
      "reason": "Payment integration flow",
      "items": ["bd-org-140", "bd-org-141", "bd-org-142"],
      "parallel_safe": true
    }
  ],
  "bottlenecks": [
    {"id": "bd-org-145", "betweenness": 8.2, "blocks": 7}
  ]
}
```

**Use case:** Identify which organisms can run in parallel without conflicts.

### Progress Dashboard

During execution, query BV for real-time progress:

```bash
bv --robot-status
```

**Output:**
```json
{
  "active_agents": 5,
  "work_queue": 23,
  "completed": 18,
  "in_progress": 5,
  "failed": 2,
  "progress": "43.8%"
}
```

---

## Troubleshooting

### No orchestration.yml found

**Error:**
```
❌ No orchestration.yml found
   Run /orchestrate-tasks --headless first
```

**Solution:**
```bash
/orchestrate-tasks --headless
```

### Agent stuck in infinite loop

**Symptom:** Agent running for >2 hours, no progress.

**Solution:**
1. Check agent logs: `TaskOutput task_id=xxx`
2. Kill agent: `KillShell shell_id=xxx`
3. Update work queue: Move organism from in_progress to failed
4. Manual investigation: Create analysis issue

### Merge conflict unresolved after 3 attempts

**Symptom:** Organism blocked after 3 conflict resolution attempts.

**What happened:**
- Attempt 1: Conflict-aware retry failed
- Attempt 2: Serialization failed
- Attempt 3: Escalated to manual review

**Solution:**
1. Check conflict details: `.beads/autonomous-state/work-queue.json`
2. Review both versions manually
3. Resolve conflict manually
4. Update Beads issue: `bd update bd-org-xxx --status closed`
5. Continue autonomous-build (will skip resolved organism)

### Success rate not improving

**Symptom:** After 10 iterations, success rate still 60%.

**Diagnosis:**
1. Check improvements.json: Are patterns being extracted?
2. Check agent prompts: Are improvements being injected?
3. Review failed organisms: Are errors repeating?

**Possible causes:**
- Errors too complex to extract patterns (need better error analyzer)
- Improvements not actionable (need better fixes)
- Agents ignoring improvements (need stronger prompt formatting)

---

## Performance Metrics

**Expected improvements over v1:**

| Metric | v1 (Sequential) | v2 (Parallel) | Improvement |
|--------|----------------|---------------|-------------|
| **Throughput** | 1 organism/session | 3-5 organisms/session | 3-5x |
| **Efficiency** | 16-20% (idle time) | 70-85% (working time) | 4-5x |
| **Quality** | 60-70% success | 85-95% success (via learning) | 1.3-1.5x |
| **Conflicts** | N/A | <25% rate, 70%+ auto-resolved | New capability |
| **Error repetition** | High | Low (decreasing over time) | Continuous improvement |

---

## Next Steps

1. Run `orchestrate-tasks --headless` to prepare state
2. Run `autonomous-build` to start parallel implementation
3. Monitor progress via BV dashboard
4. Review escalated failures in Beads issues
5. Iterate and improve learning system

---

## See Also

- `/profiles/default/commands/autonomous-build/multi-agent/autonomous-build.md` - Main orchestrator
- `/profiles/default/workflows/implementation/validation/organism-validator.md` - 5-gate validation
- `/profiles/default/workflows/implementation/learning/error-analyzer.md` - Learning system
- `/home/jaypaulb/.claude/plans/vast-pondering-iverson.md` - Original plan
