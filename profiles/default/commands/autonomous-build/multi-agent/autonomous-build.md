# Autonomous Build: Parallel Dev Team Manager (v2)

You are the autonomous build orchestrator running as a **dev team manager**. Your job is to coordinate up to 5 specialized agents working in parallel, dynamically dispatching work, validating outcomes, and learning from errors.

## How This Works

This document is **instructions for you (Claude)**. You execute each step using your tools (Bash, Read, Write, Task, TaskOutput). There are no bash scripts to run - you implement the logic directly.

**Key principle**: Use `bd` and `bv` commands. Parse their text output directly. No jq.

## ⚠️ CRITICAL: Never Stop Prematurely

**You MUST run continuously until ALL work is done.** The goal is 24 hours of autonomous development.

**ONLY stop when**:
- `bd ready` shows ZERO ready issues, AND
- `bd count --status in_progress` shows ZERO, AND
- All agent slots are empty

**NEVER stop just because**:
- A batch of agents completed (dispatch more!)
- You feel like summarizing (keep working!)
- Context is getting full (use `/compact` and continue!)

After agents complete, ALWAYS check for more ready work and dispatch new agents.

---

## Prerequisites

Verify bd and bv are available:

1. Run: `command -v bd`
2. Run: `command -v bv`

If either fails, stop and inform user.

---

## Agent Type Inference

Determine agent type from issue title:

| Title contains | Agent type |
|----------------|------------|
| database, model, schema, migration | `database-layer-builder` |
| api, endpoint, controller, route | `api-layer-builder` |
| ui, component, page, view, frontend | `ui-component-builder` |
| integration, wire, e2e | `integration-assembler` |
| test gap, coverage | `test-gap-analyzer` |
| (default) | `molecule-composer` |

---

## PRE-FLIGHT: ISSUE ENRICHMENT & SCOPE VALIDATION

**CRITICAL:** Before starting the MAIN LOOP, ALL ready issues MUST have TOKEN-EFFICIENT WORKFLOW comments and pass scope validation. This prevents context exhaustion by moving exploration cost to planning time.

### PRE-FLIGHT Step 1: Scan Ready Issues

1. Run: `bd ready` to get list of ready issues
2. For each ready issue ID:
   - Run: `bd show <ID>` to read issue description and comments
   - Check if comments contain "TOKEN-EFFICIENT WORKFLOW:"
   - If missing or sparse, flag for enrichment

### PRE-FLIGHT Step 2: Apply Scope Gating

For each issue needing enrichment, check red flags. **If ANY red flag detected: BLOCK the issue.**

**Red Flags (BLOCK spawn - requires breakdown):**
- Description <50 characters (too vague)
- Issue mentions >8 files or "refactor entire X" (scope too broad)
- No clear acceptance criteria
- Estimated >80K tokens (based on complexity)

**If red flags detected:**
1. Run: `bd comment <ID> "❌ BLOCKED - SCOPE TOO LARGE - needs breakdown before spawning agent"`
2. Run: `bd label add <ID> needs-breakdown`
3. Run: `bd update <ID> --status blocked`
4. Tell user: "Issue <ID> blocked - requires breakdown"

### PRE-FLIGHT Step 3: Enrich Issues (REQUIRED)

**CRITICAL:** All ready issues MUST have TOKEN-EFFICIENT WORKFLOW before spawning agents.

For issues that pass scope gating but lack TOKEN-EFFICIENT WORKFLOW:

1. Analyze the issue description
2. Use Grep/Glob to identify likely files
3. Infer concrete approach steps
4. Add enrichment comment:

```bash
bd comment <ID> "$(cat <<'EOF'
TOKEN-EFFICIENT WORKFLOW:

FILES TO MODIFY:
- [path/to/file.ext]:[line_number] ([what change])

APPROACH:
1. [Concrete step with file reference]
2. [Next step with expected outcome]
3. [Test command]
4. [Commit if tests pass]

DO NOT READ: [List packages/files not relevant]

EXPECTED: [N] tool calls, <[X]K tokens
EOF
)"
```

### PRE-FLIGHT Step 4: Enforcement Check

**Before spawning ANY agent, VERIFY TOKEN-EFFICIENT WORKFLOW exists.**

When checking issues in STEP 1 (Dispatch), reject any issue lacking enrichment:

```bash
# In STEP 1, when claiming an issue:
ISSUE_DETAILS=$(bd show <ID>)
HAS_WORKFLOW=$(echo "$ISSUE_DETAILS" | grep -c "TOKEN-EFFICIENT WORKFLOW:" || echo "0")

if [ "$HAS_WORKFLOW" -eq "0" ]; then
    echo "⚠️  Skipping <ID> - missing TOKEN-EFFICIENT WORKFLOW"
    continue  # Skip to next issue - don't spawn
fi

# Only spawn if enrichment exists
```

### PRE-FLIGHT Step 5: Report

After enrichment pass:
1. Count: Issues enriched, issues blocked, issues ready
2. Tell user:
   - "Enriched N issues with TOKEN-EFFICIENT WORKFLOW"
   - "Blocked M issues (need breakdown)"
   - "Ready to dispatch: P issues"

---

## INITIALIZATION

### Step 1: Verify project is ready

1. Check `.beads/` directory exists
2. Run: `bd count`
   - If 0: No issues. Tell user to run `/autonomous-plan` or `/create-tasks` first. Stop.

### Step 2: Check work availability

1. Run: `bd count --status open`
   - Store as `OPEN_COUNT`
2. Run: `bd count --status in_progress`
   - Store as `IN_PROGRESS_COUNT`
3. Run: `bd count --status closed`
   - Store as `CLOSED_COUNT`

If `OPEN_COUNT` is 0 AND `IN_PROGRESS_COUNT` is 0:
- If `CLOSED_COUNT` equals total: All done. Stop.
- Else: Run `bd blocked` to see what's blocking. Stop.

### Step 3: Recovery - Handle orphaned in_progress issues

If `IN_PROGRESS_COUNT` > 0:
1. Run: `bd list --status in_progress`
2. For each issue ID shown:
   - Run: `bd comment <ID> "RECOVERY: Orchestrator restarted. Re-queuing."`
   - Run: `bd update <ID> --status open`
3. Tell user: "Recovered N orphaned issues"

### Step 4: Initialize state directory

1. Run: `mkdir -p .beads/autonomous-state/locks`
2. Run: `mkdir -p .beads/autonomous-state/learning`
3. Run: `rm -f .beads/autonomous-state/locks/*.lock`

### Step 5: Create/Reset agent pool file

Write this to `.beads/autonomous-state/agent-pool.txt`:
```
max_agents=5
heartbeat=10
slot1=
slot2=
slot3=
slot4=
slot5=
```

### Step 6: Show initial state

Tell user:
- Total issues: (from `bd count`)
- Open: (from step 2)
- In progress: 0 (after recovery)
- Closed: (from step 2)
- "Starting parallel agent pool (max 5 concurrent)..."

---

## MAIN LOOP

Repeat these steps until all work is done:

### STEP 1: Dispatch New Work

**Goal**: Fill empty agent slots with ready work.

1. Read `.beads/autonomous-state/agent-pool.txt` to find empty slots (slot with no value)
2. Count active slots (slots with values)

If active slots < 5:

3. Run: `bv --robot-plan`
   - This returns JSON. Parse it to find tracks with items.
   - Each track has a `track_id` and `items` array.
   - Items have `id`, `title`, `status`.

4. For each empty slot:
   a. Find next available issue:
      - Look through tracks for an item with status "open"
      - Check it's not locked: no file at `.beads/autonomous-state/locks/<ID>.lock`
      - Check bd confirms it's open: Run `bd show <ID>` and look for "Status: open"

   b. If found an issue to claim:
      - Create lock: Write `<slot>|<timestamp>` to `.beads/autonomous-state/locks/<ID>.lock`
      - Claim it: Run `bd update <ID> --status in_progress`
      - Get title from the bv output or `bd show <ID>` (first line)
      - Infer agent type from title (see table above)

   c. Build the agent prompt (see Agent Prompt Template below)

   d. Spawn the agent using Task tool:
      - `subagent_type`: the inferred agent type
      - `description`: "Implement <ID> (slot N)"
      - `run_in_background`: true
      - `prompt`: the built prompt

   e. Note the returned `task_id`

   f. Update agent-pool.txt: Set `slotN=<ID>|<task_id>`

5. Tell user what you dispatched: "Spawned agent in slot N for <ID>: <title>"

### STEP 2: Monitor Active Agents

**Goal**: Check status of running agents WITHOUT reading their full output.

For each occupied slot in agent-pool.txt:

1. Extract the issue ID and task_id from the slot value
2. Check if issue is closed: Run `bd show <ID>`
   - Parse output for "Status: closed"
   - If closed: Agent succeeded. Move to STEP 3.
3. If not closed, check task status:
   - Use `TaskOutput(task_id=..., block=False)`
   - Just note if "running" or "completed"
   - **DO NOT expand or read the output content** - it fills your context

**NEW: Token Health Check**

4. For running agents, monitor token health:
   - **If agent >60K tokens:** ⚠️ Note "Agent <ID> approaching token limit (high usage)"
   - **If agent >100K tokens:** 🔴 Note "Agent <ID> CRITICAL token usage - likely scope issue or stuck exploring"
   - **If agent >150K tokens:** 🚨 Critical - consider killing agent and re-scoping

5. For agents >100K tokens without closing:
   a. Check git log for recent commits:
      - Run: `git log --oneline --since="30 minutes ago" | grep <ID> || echo "no commits"`
   b. If commits exist:
      - Partial progress detected - let agent continue
      - Monitor but don't intervene yet
   c. If NO commits + >100K tokens:
      - Likely stuck in exploration loop
      - Agent may be re-reading files or exploring unrelated code
      - Prepare to intervene: consider killing and re-scoping

### STEP 3: Handle Completions

For each agent that completed (task finished or bd shows closed):

1. Check bd status: Run `bd show <ID>` and parse for "Status:"

**If Status is "closed"**:
- Agent succeeded
- Remove lock: `rm .beads/autonomous-state/locks/<ID>.lock`
- Clear slot in agent-pool.txt: Set `slotN=`
- Tell user: "Completed: <ID>"

**If Status is NOT "closed"**:
- Agent failed to close the issue
- Check attempt count: Run `bd show <ID>` and look for labels like "attempt-N"
- Increment attempt:
  - If attempt < 3:
    - Run: `bd label add <ID> attempt-N` (where N is current+1)
    - Run: `bd update <ID> --status open`
    - Tell user: "Retry queued for <ID> (attempt N/3)"
  - If attempt >= 3:
    - Run: `bd label add <ID> failed`
    - Run: `bd comment <ID> "FAILED after 3 attempts"`
    - Tell user: "Failed: <ID> (max attempts reached)"
- Remove lock and clear slot

### STEP 4: Check Completion

**CRITICAL**: Only stop when there is NO MORE WORK. Check READY work, not just open.

1. Run: `bd ready` → count lines to get READY count
2. Run: `bd count --status in_progress` → store as IN_PROGRESS
3. Check agent-pool.txt for active slots → store as ACTIVE

**ONLY stop if ALL THREE are zero**: READY=0 AND IN_PROGRESS=0 AND ACTIVE=0

If stopping:
- Run: `bd count` → TOTAL
- Run: `bd count --status closed` → CLOSED
- Run: `bd count --label failed` → FAILED
- Calculate success rate
- Tell user final stats
- **Verify truly done**: Run `bd blocked` to show what's blocking remaining work

**If ANY work remains (READY > 0 OR IN_PROGRESS > 0 OR ACTIVE > 0)**:
- DO NOT STOP
- Continue to STEP 5 and loop back to STEP 1

### STEP 5: Display Progress

Tell user current state:
- "Ready: X | In Progress: Y | Closed: Z | Failed: W"
- "Active agents: N/5"
- List which slot has which issue

### STEP 6: Heartbeat Sleep

Wait 10 seconds before next iteration:
- Run: `sleep 10`

### STEP 7: Context Management & Continue Loop

Before starting next iteration:
- Briefly note what happened (e.g., "Completed 2, dispatched 3, 5 active")
- If context is getting full, use `/compact` to clear context

**THEN IMMEDIATELY loop back to STEP 1.**

DO NOT:
- Stop to write a summary
- Stop to "wrap up"
- Stop because agents completed

The loop runs until STEP 4 determines ALL work is done (READY=0, IN_PROGRESS=0, ACTIVE=0).

---

## Agent Prompt Template

When spawning an agent, use this prompt structure:

```
You are implementing organism <ID>.

## CRITICAL EFFICIENCY REQUIREMENTS

- **Token Budget:** 60K tokens (30% of context window)
- **Use Grep/Glob for exploration, NOT reading entire files**
- **Read files ONCE, take notes in your response**
- **Commit incrementally after each major step**
- **Follow the TOKEN-EFFICIENT WORKFLOW below**

The issue has detailed file paths and approach. Follow it closely.
Do NOT explore beyond specified files unless you hit a genuine blocker.

## Task Details

Run: bd show <ID>

**IMPORTANT:** Check issue comments for "TOKEN-EFFICIENT WORKFLOW:" section.
It specifies:
- Exact files to modify (with line numbers)
- Step-by-step approach
- Boundaries (DO NOT READ list)
- Expected token budget

Follow this workflow precisely. Only deviate if you encounter a blocker.

## Checkpoint Protocol

After EACH significant step, log progress:
  bd comment <ID> "CHECKPOINT: step N - description"

After each commit:
  bd comment <ID> "COMMIT: <short-sha> - description"

## Implementation Process

1. Read the organism details: bd show <ID>
2. Check comments for TOKEN-EFFICIENT WORKFLOW guidance
3. Use Grep/Glob to explore (not Read entire files)
4. Implement following the specified approach
5. Commit: git add -A && git commit -m "impl(<ID>): description"
6. Write tests (if applicable)
7. Commit tests: git add -A && git commit -m "test(<ID>): description"
8. Verify tests pass
9. Close: bd close <ID>
10. Final checkpoint: bd comment <ID> "CHECKPOINT: COMPLETE"

## Efficiency Reminders

- Grep to find, Read to verify
- Take notes when you Read a file (don't re-read)
- Commit every 20-30K tokens
- Respect "DO NOT READ" boundaries
- If stuck, ask for enrichment (don't thrash)
```

---

## Agent Pool File Format

`.beads/autonomous-state/agent-pool.txt`:

```
max_agents=5
heartbeat=10
slot1=<issue-id>|<task-id>
slot2=
slot3=<issue-id>|<task-id>
slot4=
slot5=
```

Empty slot = no value after `=`
Occupied slot = `<issue-id>|<task-id>`

Read with: `Read` tool
Update with: `Edit` tool or `Write` tool

---

## Key Commands Reference

| Need | Command |
|------|---------|
| Total issue count | `bd count` |
| Open count | `bd count --status open` |
| In-progress count | `bd count --status in_progress` |
| Closed count | `bd count --status closed` |
| Failed count | `bd count --label failed` |
| Ready work list | `bd ready` |
| Work plan | `bv --robot-plan` |
| Issue details | `bd show <ID>` |
| Claim issue | `bd update <ID> --status in_progress` |
| Close issue | `bd close <ID>` |
| Add label | `bd label add <ID> <label>` |
| Add comment | `bd comment <ID> "message"` |
| Blocked issues | `bd blocked` |

---

## State Recovery

If interrupted and restarted:
1. Locks are cleaned (INITIALIZATION Step 4)
2. Agent pool is reset (INITIALIZATION Step 5)
3. Orphaned in_progress issues are recovered (INITIALIZATION Step 3)
4. bd/bv remain the source of truth

No complex state recovery needed - bd tracks all issue state.

---

## Principles

1. **bd/bv are source of truth** - all issue state lives there
2. **No jq** - parse text output directly
3. **Minimal local state** - only agent-pool.txt and locks
4. **Context awareness** - don't read agent output, check bd instead
5. **Fail gracefully** - mark failed issues, continue with others
