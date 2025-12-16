# Autonomous Build State Management

This directory contains schema templates for the autonomous build state files.

## Overview

When `/autonomous-build` runs in a project, it creates a `.beads/autonomous-state/` directory with these files to track the build's progress. The orchestrator uses these files to:

1. **Manage the work queue** (ready → in_progress → completed)
2. **Track active agents** (slots 1-5)
3. **Accumulate learnings** (error patterns, best practices)
4. **Recover from interruptions** (state persists between sessions)

## State Files

### `work-queue.json`

Tracks organisms through their lifecycle:

```
ready → in_progress → completed
          ↓
        failed / blocked
```

**Sections:**
- `ready`: Organisms available to be claimed
- `in_progress`: Currently being worked on by agents
- `completed`: Successfully implemented and committed
- `failed`: Failed after N attempts (analysis issue created)
- `blocked`: Temporarily blocked (conflict, dependency)

### `agent-pool.json`

Tracks active agents in slots 1-5:

- **max_agents**: 5 (configurable via autonomous-config.yml)
- **active**: Array of currently running agents
- **available_slots**: Slot numbers currently free (1-5)
- **heartbeat_interval**: Seconds between status checks

### `improvements.json`

Learning system knowledge base:

- **common_errors**: Patterns extracted from failures
- **best_practices**: Successful patterns discovered
- **conflict_patterns**: Merge conflict scenarios and resolutions
- **organism_specific**: Type-specific learnings (database, api, ui)
- **metrics**: Success rate trends over iterations

### `orchestration.yml`

Work assignments from `/orchestrate-tasks` (headless mode):

- **beads**: All organisms with assignees and dependencies
- **tracks**: BV execution plan (parallel work streams)
- **insights**: Bottlenecks, keystones, influencers

This file is **created by /orchestrate-tasks**, not by autonomous-build.

## Locks Directory

`.beads/autonomous-state/locks/` contains per-organism lock files:

```
locks/
├── bd-org-123.lock
├── bd-org-124.lock
└── bd-org-125.lock
```

Lock files prevent double-claiming when multiple agent slots are available.

## Failures Directory

`.beads/autonomous-state/failures/` contains analysis files:

```
failures/
└── bd-org-125-analysis.md
```

Created when organism fails N times (default: 3). Contains:
- Error history
- Possible causes
- Recommended actions

## Initialization

The orchestrator creates these files on first run:

```bash
# In user's project directory
.beads/autonomous-state/
├── work-queue.json       # Initialized with ready organisms from orchestration.yml
├── agent-pool.json       # Initialized with max_agents=5, active=[], available_slots=[1,2,3,4,5]
├── orchestration.yml     # Created by /orchestrate-tasks --headless
├── locks/                # Created as needed
├── learning/
│   └── improvements.json # Initialized with empty arrays
└── failures/             # Created as needed
```

## State Recovery

If interrupted (Ctrl+C, crash, etc.), state is preserved:

```bash
# On next run of /autonomous-build
1. Read work-queue.json
2. Detect in_progress organisms (agents were killed)
3. Move in_progress → ready (retry)
4. Clean up stale locks
5. Resume from where we left off
```

## Configuration

Some settings come from `.beads/autonomous-config.yml`:

```yaml
# Max concurrent agents (default: 5)
session:
  max_agents: 5

# Heartbeat interval (default: 10 seconds)
session:
  heartbeat_interval: 10
```

## See Also

- `/profiles/default/commands/autonomous-build/multi-agent/autonomous-build.md` - Orchestrator implementation
- `/profiles/default/workflows/implementation/autonomous-config-template.yml` - Configuration template
- `/home/jaypaulb/.claude/plans/vast-pondering-iverson.md` - Architecture plan
