# Autonomous Build v2: Testing Checklist

## Pre-Testing Setup

### Prerequisites

- [ ] Beads initialized at project root (`.beads/` directory exists)
- [ ] BV available (optional but recommended for parallel analysis)
- [ ] At least 30+ organism issues created via `/autonomous-plan`
- [ ] Git repository initialized with clean working directory

### State Initialization

- [ ] Create state directory: `.beads/autonomous-state/`
- [ ] State templates exist:
  - [ ] `profiles/default/commands/autonomous-build/state-templates/work-queue.json`
  - [ ] `profiles/default/commands/autonomous-build/state-templates/agent-pool.json`
  - [ ] `profiles/default/commands/autonomous-build/state-templates/improvements.json`
  - [ ] `profiles/default/commands/autonomous-build/state-templates/orchestration.yml`

---

## Phase 1: Headless Mode Testing

### Test /orchestrate-tasks --headless

**Objective:** Verify orchestrate-tasks creates orchestration.yml and exits without delegation.

#### Test 1.1: Basic headless execution

- [ ] Run: `/orchestrate-tasks --headless`
- [ ] Verify: Headless mode banner displayed
- [ ] Verify: Organisms discovered from Beads
- [ ] Verify: BV analysis runs (if available)
- [ ] Verify: Agents assigned based on organism type
- [ ] Verify: `.beads/autonomous-state/orchestration.yml` created
- [ ] Verify: Process exits with success message
- [ ] Verify: No agents spawned

**Expected output:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  HEADLESS MODE: Autonomous Build Integration
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ Headless Mode Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Orchestration state prepared for autonomous-build:
  • File: .beads/autonomous-state/orchestration.yml
  • Organisms: 47
  • Agents assigned: Yes

You can now run /autonomous-build to begin parallel implementation
```

#### Test 1.2: Orchestration file format

- [ ] Open `.beads/autonomous-state/orchestration.yml`
- [ ] Verify: YAML format valid
- [ ] Verify: Contains `beads:` array
- [ ] Verify: Each organism has:
  - [ ] `id` field (e.g., `bd-org-123`)
  - [ ] `title` field
  - [ ] `assignee` field (agent type)
  - [ ] `standards` array

**Example:**
```yaml
beads:
  - id: bd-org-123
    title: User validation atom
    assignee: molecule-composer
    standards:
      - global/atomic-design.md
```

#### Test 1.3: Agent assignment correctness

- [ ] Verify database organisms → `database-layer-builder`
- [ ] Verify API organisms → `api-layer-builder`
- [ ] Verify UI organisms → `ui-component-builder`
- [ ] Verify integration organisms → `integration-assembler`
- [ ] Verify test gaps → `test-gap-analyzer`
- [ ] Verify default fallback → `molecule-composer`

---

## Phase 2: Agent Pool Manager Testing

### Test autonomous-build initialization

**Objective:** Verify autonomous-build loads state and initializes correctly.

#### Test 2.1: State initialization

- [ ] Run: `/autonomous-build`
- [ ] Verify: Loads `.beads/autonomous-state/orchestration.yml`
- [ ] Verify: Initializes `work-queue.json` from orchestration.yml
- [ ] Verify: Initializes `agent-pool.json` with 5 slots
- [ ] Verify: Initializes `improvements.json` if not exists
- [ ] Verify: Creates `locks/` directory

**Check work-queue.json:**
```json
{
  "ready": ["bd-org-123", "bd-org-124", ...],
  "in_progress": [],
  "completed": [],
  "failed": [],
  "blocked": []
}
```

**Check agent-pool.json:**
```json
{
  "max_agents": 5,
  "active": [],
  "available_slots": [1, 2, 3, 4, 5]
}
```

#### Test 2.2: Dynamic dispatch

- [ ] Verify: STEP 1 spawns up to 5 agents immediately
- [ ] Verify: Each agent claimed from `ready` queue
- [ ] Verify: Each agent moved to `in_progress` queue
- [ ] Verify: Lock file created for each organism
- [ ] Verify: Agent pool updated with active agents

**Expected output:**
```
STEP 1: Dispatch New Work
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Spawned agent for bd-org-123 (User validation atom) [Slot 1]
✓ Spawned agent for bd-org-124 (Database migration) [Slot 2]
✓ Spawned agent for bd-org-125 (Login API endpoint) [Slot 3]
✓ Spawned agent for bd-org-126 (Login form component) [Slot 4]
✓ Spawned agent for bd-org-127 (Auth middleware) [Slot 5]

[ready: 42] [in_progress: 5] [completed: 0] [failed: 0]
```

#### Test 2.3: Non-blocking monitoring

- [ ] Verify: STEP 2 polls active agents using `TaskOutput block=false`
- [ ] Verify: Shows status for each slot (running/completed)
- [ ] Verify: Shows elapsed time for each agent
- [ ] Verify: Does not block on incomplete agents

**Expected output:**
```
STEP 2: Monitor Active Agents (5 running)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Slot 1] bd-org-123: running (15 min elapsed)
[Slot 2] bd-org-124: running (12 min elapsed)
[Slot 3] bd-org-125: COMPLETED
[Slot 4] bd-org-126: running (8 min elapsed)
[Slot 5] bd-org-127: running (5 min elapsed)
```

#### Test 2.4: Continuous dispatch (no idle slots)

- [ ] Wait for first agent to complete
- [ ] Verify: STEP 3 handles completion (validation)
- [ ] Verify: Slot freed immediately after validation
- [ ] Verify: STEP 1 spawns new agent in freed slot
- [ ] Verify: No idle time between completion and next dispatch

**Expected pattern:**
```
T=0:   5 agents running (slots 1-5 occupied)
T=5:   Agent 3 completes → validate → slot 3 freed → spawn agent 6
T=10:  Agent 1 completes → validate → slot 1 freed → spawn agent 7
...continuous until ready queue empty
```

---

## Phase 3: Validation Pipeline Testing

### Test 5-gate validation

**Objective:** Verify each organism validated through all 5 gates.

#### Test 3.1: Gate 1 - Run Tests

- [ ] Verify: Organism-specific tests discovered
- [ ] Verify: Tests executed
- [ ] Verify: Exit code 1 on test failure
- [ ] Verify: Exit code 0 on test pass

**Test failure scenario:**
- [ ] Introduce failing test
- [ ] Verify: Gate 1 fails
- [ ] Verify: Error output captured
- [ ] Verify: Organism moves to retry (not completed)

#### Test 3.2: Gate 2 - Integration Test

- [ ] Verify: Organism dependencies checked
- [ ] Verify: Integration tests run (if exist)
- [ ] Verify: Exit code 2 on integration failure

#### Test 3.3: Gate 3 - Merge Detection

- [ ] Verify: Merge conflict detector runs
- [ ] Verify: Stashes uncommitted changes
- [ ] Verify: Attempts merge with main
- [ ] Verify: Detects conflicts in shared files (types, barrel files, config)
- [ ] Verify: Conflict details written to `/tmp/conflict-details.txt`
- [ ] Verify: Exit code 3 on conflict
- [ ] Verify: Restores stashed changes after check

**Test conflict scenario:**
- [ ] Create conflict (modify same type definition in main branch)
- [ ] Verify: Gate 3 detects conflict
- [ ] Verify: Conflict details categorized (type/barrel/config/code)
- [ ] Verify: Resolution hints generated

#### Test 3.4: Gate 4 - Regression Sample

- [ ] Verify: Random completed organism selected
- [ ] Verify: Sampled organism's tests run
- [ ] Verify: Exit code 4 on regression failure
- [ ] Verify: Regressed organism reopened in Beads

#### Test 3.5: Gate 5 - Quality Checks

- [ ] Verify: Linter runs (if available)
- [ ] Verify: Type checker runs (if available)
- [ ] Verify: Coverage check runs (if configured)
- [ ] Verify: Exit code 5 on quality failure (if blocking)

#### Test 3.6: All gates pass

- [ ] Create organism with passing tests, no conflicts, no regressions
- [ ] Verify: All 5 gates pass
- [ ] Verify: Organism committed immediately
- [ ] Verify: Organism moved to completed queue
- [ ] Verify: Lock released

**Expected output:**
```
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
[ready: 41] [in_progress: 4] [completed: 1] [failed: 0]
```

---

## Phase 4: Merge Conflict Resolution Testing

### Test 3-tier conflict resolution

**Objective:** Verify merge conflicts handled through 3-tier strategy.

#### Test 4.1: Attempt 1 - Conflict-aware retry

**Setup:**
- [ ] Create organism that modifies shared type definition
- [ ] Modify same type in main branch (create conflict)

**Test:**
- [ ] Verify: Gate 3 detects conflict
- [ ] Verify: Conflict details extracted
- [ ] Verify: Organism moved to retry with conflict context
- [ ] Verify: Agent respawned with conflict diff in prompt
- [ ] Verify: Conflict context includes:
  - [ ] Conflicted files list
  - [ ] Conflict diff
  - [ ] Resolution instructions

**Agent prompt should include:**
```
⚠️  CONFLICT RESOLUTION MODE (Attempt 1)
This organism previously resulted in merge conflicts.

**Conflicted files:**
  • src/templates/interfaces/User.ts

**Conflict details:**
<<<<<<< HEAD (recent change by database-layer)
export interface User {
  id: string;
  passwordHash: string;
}
=======
export interface User {
  id: string;
  token?: string;
}
>>>>>>> your-branch

Please regenerate your implementation incorporating BOTH changes.
```

**Success scenario:**
- [ ] Agent resolves conflict (includes both fields)
- [ ] Validation passes
- [ ] Organism committed

**Failure scenario (proceed to Attempt 2):**
- [ ] Agent fails to resolve conflict
- [ ] Verify: Organism retried with serialization

#### Test 4.2: Attempt 2 - Serialize conflicting work

**Test:**
- [ ] Verify: Conflicting organism identified
- [ ] Verify: Current organism blocked until conflicting organism completes
- [ ] Verify: Organism moved to blocked queue
- [ ] Verify: Other work continues (non-blocking)
- [ ] Wait for conflicting organism to complete
- [ ] Verify: Blocked organism unblocked
- [ ] Verify: Organism retried

**Success scenario:**
- [ ] Retry after serialization succeeds
- [ ] Organism committed

**Failure scenario (proceed to Attempt 3):**
- [ ] Still conflicts after serialization
- [ ] Verify: Escalated to manual review

#### Test 4.3: Attempt 3 - Manual review escalation

**Test:**
- [ ] Verify: Analysis issue created in Beads
- [ ] Verify: Issue title: `[ANALYSIS] Merge conflict: {organism.title}`
- [ ] Verify: Issue contains:
  - [ ] Conflict details
  - [ ] Both versions (main vs. branch)
  - [ ] Resolution options
- [ ] Verify: Issue assigned to Jaypaul (or general-purpose for first attempt)
- [ ] Verify: Original organism moved to blocked queue
- [ ] Verify: Autonomous-build continues with other work

---

## Phase 5: Learning System Testing

### Test error pattern extraction

**Objective:** Verify errors extracted and improvements generated.

#### Test 5.1: Extract error patterns

**Setup:**
- [ ] Create organism that fails validation with known error (e.g., missing import)

**Test:**
- [ ] Verify: Gate 1 fails with import error
- [ ] Verify: Error analyzer workflow runs
- [ ] Verify: Error pattern extracted:
  - [ ] Category: `imports`
  - [ ] Pattern: `Missing import statement`
  - [ ] Fix: `Always verify imports before implementing...`
- [ ] Verify: Pattern added to `improvements.json`
- [ ] Verify: `seen_count` incremented if pattern already exists

**Check improvements.json:**
```json
{
  "common_errors": [
    {
      "pattern": "Missing import statement",
      "fix": "Always verify imports before implementing. Check existing files for import patterns.",
      "category": "imports",
      "seen_count": 1,
      "first_seen": "2025-12-16T10:00:00Z",
      "last_seen": "2025-12-16T10:00:00Z",
      "trending": "stable"
    }
  ]
}
```

#### Test 5.2: Improvement injection

**Test:**
- [ ] Add error patterns to improvements.json (manually or via previous test)
- [ ] Spawn new agent
- [ ] Verify: Agent prompt includes learning context
- [ ] Verify: Learning context shows:
  - [ ] Common errors grouped by category
  - [ ] Top 3 errors per category (sorted by seen_count)
  - [ ] Fix for each error
  - [ ] Organism-specific learnings (if applicable)
  - [ ] Best practices (if any)
  - [ ] Retry mode warning (if attempt > 1)

**Agent prompt should include:**
```
IMPORTANT: Learn from previous sessions to avoid common mistakes:

imports issues:
  ❌ Missing import statement
  ✅ Always verify imports before implementing. Check existing files for import patterns.

types issues:
  ❌ Type mismatch - expected function
  ✅ Check interface definitions before using types. Read type files in templates/interfaces/
```

#### Test 5.3: Success rate improvement over iterations

**Long-term test:**
- [ ] Run autonomous-build on 30+ organisms
- [ ] Track success rate per iteration (iteration = set of organisms)
- [ ] Verify: Success rate improves over time
- [ ] Verify: `seen_count` for errors increases
- [ ] Verify: `trending` field updates (`up`/`stable`/`down`)

**Expected metrics:**
```
Iteration 1: 60% success (12 errors)
Iteration 2: 72% success (8 errors)
Iteration 3: 81% success (5 errors)
Iteration 10: 94% success (2 errors)
```

---

## Phase 6: End-to-End Testing

### Full autonomous build

**Objective:** Complete all organisms from orchestration.yml without manual intervention.

#### Test 6.1: Complete build (30+ organisms)

**Setup:**
- [ ] Create project with 30+ organism issues via `/autonomous-plan`
- [ ] Run `/orchestrate-tasks --headless`
- [ ] Run `/autonomous-build`

**Test:**
- [ ] Verify: All organisms processed
- [ ] Verify: No infinite loops (max 2 hour timeout per organism)
- [ ] Verify: Completed organisms committed to git
- [ ] Verify: Failed organisms escalated (analysis issues created)
- [ ] Verify: Final summary displayed

**Expected output:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ ALL ORGANISMS COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total organisms: 47
✓ Completed: 44
⚠️  Failed (escalated): 3

Success rate: 93.6%
Total time: 3.5 hours
```

#### Test 6.2: Interruption recovery

**Test:**
- [ ] Start autonomous-build
- [ ] Wait for 2-3 agents to spawn
- [ ] Interrupt (Ctrl+C)
- [ ] Verify: State persists (work-queue.json, agent-pool.json, locks/)
- [ ] Resume: `/autonomous-build`
- [ ] Verify: Loads previous state
- [ ] Verify: Completes in-progress organisms
- [ ] Verify: Continues with remaining organisms
- [ ] Verify: No duplicate work

#### Test 6.3: Performance metrics

**Measure:**
- [ ] Throughput: organisms completed per hour
- [ ] Efficiency: working time / total time (target: 70-85%)
- [ ] Quality: success rate (target: 85-95%)
- [ ] Conflict resolution: auto-resolution rate (target: 70%+)

**Compare to v1 (sequential):**
- [ ] Throughput increase: 3-4x expected
- [ ] Quality improvement: 1.3-1.5x expected (via learning)

---

## Edge Cases and Error Handling

### Test edge cases

#### Test 7.1: Empty orchestration.yml

- [ ] Create empty orchestration.yml
- [ ] Run autonomous-build
- [ ] Verify: Exits gracefully with message "No organisms to implement"

#### Test 7.2: All organisms already completed

- [ ] Mark all organisms as closed in Beads
- [ ] Run orchestrate-tasks --headless
- [ ] Verify: Orchestration.yml empty or skips closed organisms
- [ ] Run autonomous-build
- [ ] Verify: Exits with "All organisms already complete"

#### Test 7.3: Agent crashes

- [ ] Force agent crash (invalid prompt, timeout, etc.)
- [ ] Verify: TaskOutput detects failure
- [ ] Verify: Organism moved to failed queue
- [ ] Verify: Error logged
- [ ] Verify: Other agents continue

#### Test 7.4: Git push failure

- [ ] Simulate git push failure (network issue, permission error, etc.)
- [ ] Verify: Organism validation passes but commit fails
- [ ] Verify: Error logged
- [ ] Verify: Organism retried or escalated

#### Test 7.5: Concurrent modifications to shared files

- [ ] Two agents modify same type definition file simultaneously
- [ ] Verify: First agent commits successfully
- [ ] Verify: Second agent detects conflict (Gate 3)
- [ ] Verify: Second agent resolves via 3-tier strategy

---

## BV Integration Testing

### Test BV features

#### Test 8.1: Execution plan

- [ ] Run orchestrate-tasks --headless with BV available
- [ ] Verify: BV analyzes parallel execution opportunities
- [ ] Verify: Multiple tracks identified (if applicable)
- [ ] Verify: Tracks displayed to user

**Expected output:**
```
BV found 2 independent work stream(s):

Track 1: User authentication flow
  Priority: P0-P2
  Items: 12 issues
  Phases involved: phase-1, phase-2

Track 2: Payment integration flow
  Priority: P1-P3
  Items: 8 issues
  Phases involved: phase-2, phase-3
```

#### Test 8.2: Priority recommendations

- [ ] Run priority analysis (if enabled in orchestrate-tasks)
- [ ] Verify: Priority misalignments identified
- [ ] Verify: High-confidence updates suggested
- [ ] Apply updates
- [ ] Verify: Priorities updated in Beads

#### Test 8.3: Robot status dashboard

- [ ] Run autonomous-build
- [ ] Query: `bv --robot-status`
- [ ] Verify: Real-time progress displayed
- [ ] Verify: Active agents count matches agent-pool.json
- [ ] Verify: Work queue count matches work-queue.json
- [ ] Verify: Progress percentage accurate

---

## Acceptance Criteria

### Phase 1-5 Complete
- [x] State management files created
- [x] Agent pool manager implemented
- [x] Validation pipeline created
- [x] Merge conflict resolution added
- [x] Learning system built

### Phase 6 Complete
- [ ] Headless mode works (orchestrate-tasks --headless)
- [ ] Integration documented (INTEGRATION.md)
- [ ] Testing checklist created (TESTING.md)
- [ ] All tests pass
- [ ] End-to-end autonomous build completes successfully
- [ ] Performance metrics meet targets:
  - [ ] Throughput: 3-4x improvement over v1
  - [ ] Efficiency: 70-85% working time
  - [ ] Quality: 85-95% success rate
  - [ ] Conflict resolution: 70%+ auto-resolution

### Production Ready
- [ ] No manual intervention needed (except escalated failures)
- [ ] State recoverable after interruption
- [ ] BV dashboard shows accurate progress
- [ ] Documentation complete
- [ ] User can run: `/orchestrate-tasks --headless && /autonomous-build`

---

## Test Results Log

**Test Run: [Date]**

| Phase | Test | Status | Notes |
|-------|------|--------|-------|
| 1.1 | Basic headless execution | ⏸️ Pending | |
| 1.2 | Orchestration file format | ⏸️ Pending | |
| 1.3 | Agent assignment | ⏸️ Pending | |
| 2.1 | State initialization | ⏸️ Pending | |
| 2.2 | Dynamic dispatch | ⏸️ Pending | |
| 2.3 | Non-blocking monitoring | ⏸️ Pending | |
| 2.4 | Continuous dispatch | ⏸️ Pending | |
| 3.1-3.6 | 5-gate validation | ⏸️ Pending | |
| 4.1-4.3 | 3-tier conflict resolution | ⏸️ Pending | |
| 5.1-5.3 | Learning system | ⏸️ Pending | |
| 6.1-6.3 | End-to-end build | ⏸️ Pending | |
| 7.1-7.5 | Edge cases | ⏸️ Pending | |
| 8.1-8.3 | BV integration | ⏸️ Pending | |

---

## Next Steps

1. Complete all Phase 1-3 tests (prerequisites)
2. Run Phase 4 tests (conflict resolution)
3. Run Phase 5 tests (learning system)
4. Run Phase 6 end-to-end test (full autonomous build)
5. Measure performance metrics
6. Document results
7. Deploy as default autonomous-build
