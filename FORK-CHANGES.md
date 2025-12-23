# Fork Changes - Atomic Design + Beads Integration

This document details all changes made in this fork compared to the upstream [buildermethods/agent-os](https://github.com/buildermethods/agent-os).

## Overview

This fork enhances agent-os with:
1. **10 specialized atomic design agents** - Decomposed from the monolithic implementer
2. **Beads issue tracking integration** - Distributed, git-backed issue tracking for AI agents
3. **Dual tracking mode** - Per-spec choice between tasks.md and beads
4. **Enhanced atomic design standard** - Updated with beads patterns and agent mappings
5. **New atomic workflow** - Step-by-step guidance for atomic design development

**Total Changes:** 28 files (15 created, 13 modified)

---

## 1. Atomic Design Agents (10 New Agents)

### Tier 1: Foundation Builders

**`profiles/default/agents/atom-writer.md`** (NEW)
- Writes pure, single-responsibility atoms with zero peer dependencies
- Tools: Write, Read, Bash, WebFetch
- Focus: Pure functions, constants, utilities
- Testing: Pure input/output unit tests

**`profiles/default/agents/molecule-composer.md`** (NEW)
- Composes 2-3 atoms into simple helpers/services
- Single clear purpose, downward-only dependencies
- Testing: 2-5 focused tests per molecule

### Tier 2: Domain Builders (Organisms)

**`profiles/default/agents/database-layer-builder.md`** (NEW)
- Database models, migrations, associations
- Uses atoms for validations, molecules for complex logic
- Testing: 2-8 integration tests for database layer
- Standards: backend/*, global/atomic-design.md

**`profiles/default/agents/api-layer-builder.md`** (NEW)
- API endpoints, controllers, authentication
- Depends on database layer completion
- Uses molecules for business logic, atoms for formatting
- Testing: 2-8 integration tests for API layer
- Standards: backend/*, global/atomic-design.md

**`profiles/default/agents/ui-component-builder.md`** (NEW)
- UI components, forms, pages, styles
- Has Playwright tool for visual verification
- Depends on API layer completion
- Testing: 2-8 integration tests for UI layer
- Standards: frontend/*, global/atomic-design.md

### Tier 3: Test Specialists

**`profiles/default/agents/test-writer-molecule.md`** (NEW)
- Unit tests for molecule-level code
- 2-5 focused tests per molecule
- Tests composition logic, not atom internals

**`profiles/default/agents/test-writer-organism.md`** (NEW)
- Integration tests for organism-level code
- 2-8 tests per organism layer
- Layer-specific testing (database, API, UI)

**`profiles/default/agents/test-gap-analyzer.md`** (NEW)
- Reviews all tests, identifies gaps
- Writes up to 10 additional strategic tests
- Focuses on integration gaps, security, critical errors

### Tier 4: Integration

**`profiles/default/agents/template-designer.md`** (NEW)
- Defines structural contracts (interfaces, schemas, protocols)
- Runs early in spec analysis, before atoms
- No implementation, only structure

**`profiles/default/agents/integration-assembler.md`** (NEW)
- Wires all organisms together
- E2E verification with Playwright
- Runs full project test suite
- Creates verification report

---

## 2. Beads Integration (4 New Workflows)

### Workflow Files

**`profiles/default/workflows/implementation/atomic-workflow.md`** (NEW)
- Comprehensive 5-phase workflow for atomic design implementation
- Phase 1: Atoms, Phase 2: Molecules, Phase 3: Organisms, Phase 4: Tests, Phase 5: Integration
- Includes handoff protocols between agents
- Conditional support for beads/tasks.md

**`profiles/default/workflows/implementation/create-beads-issues.md`** (NEW)
- Auto-generates beads issues from spec.md
- Creates hierarchical structure: Epic → Organisms → Molecules → Atoms
- Sets blocking dependencies (atoms block molecules, molecules block organisms)
- Example commands for each step

**`profiles/default/workflows/implementation/implement-with-beads.md`** (NEW)
- Agent workflow for using beads during implementation
- Context recovery, ready work discovery, progress tracking
- Agent-specific workflows for each atomic level

**`profiles/default/workflows/implementation/beads-context-recovery.md`** (NEW)
- Cross-session context recovery to prevent agent amnesia
- Query commands for project state, discovered work, dependency graphs
- Integration with `bd list --json` for full context

### Automatic Beads Installation

When a user chooses beads mode during spec initialization:

1. **Check if beads is installed** (`command -v bd`)
2. **If not installed**, prompt user:
   - **(i)nstall beads now** → Runs beads install script automatically
   - **(f)all back to tasks.md** → Updates spec-config.yml to use tasks_md instead
3. **Verify installation** - If auto-install fails, automatically falls back to tasks.md
4. **Initialize beads** - If installed successfully, runs `bd init --stealth`

This provides a seamless UX where users don't need to pre-install beads.

### Beads Commands Used

- `bd init --stealth` - Initialize beads in spec folder
- `bd create "title" -p priority` - Create new issue
- `bd ready` - Show unblocked work
- `bd update [id] --status in_progress` - Mark work started
- `bd close [id] --reason "reason"` - Mark work complete
- `bd dep add [child] [parent] --type discovered-from` - Link discovered work
- `bd list --json` - Get full issue state for context recovery

---

## 2.5. BV (Beads Viewer) Robot Protocol Integration (7 New Workflows, 9 Modified Files)

### Overview

Integrated `bv` (beads viewer) robot protocol to provide deterministic graph intelligence for AI agents. BV offloads complex graph algorithms from LLMs to specialized tooling, enabling smarter task selection and dependency management.

**Architecture:** Hybrid approach
- **BV for intelligence** - Execution planning, priority recommendations, graph insights (read-only)
- **BD for mutations** - Creating/updating/closing issues (source of truth)
- **Graceful fallback** - If BV unavailable, falls back to basic `bd` commands

### New Workflow Files (7 Created)

**`profiles/default/workflows/implementation/bv-helpers.md`** (NEW)
- Reusable bash helper functions for BV detection and fallback
- Functions: `bv_available()`, `get_execution_plan()`, `get_priority_recommendations()`, `get_session_diff()`, `get_graph_insights()`, `check_cycles()`, `get_recipe()`, `list_recipes()`
- All functions gracefully degrade if BV unavailable

**`profiles/default/workflows/implementation/bv-priority-analysis.md`** (NEW)
- Graph-based priority recommendation workflow
- Detects priority misalignments (low priority but high importance)
- Confidence-scored suggestions (>0.8 = high confidence)
- Integration with task selection and orchestration

**`profiles/default/workflows/implementation/bv-diff-tracking.md`** (NEW)
- Time-travel diff tracking for session-to-session changes
- Session summaries (what was accomplished?)
- Regression detection (new cycles introduced?)
- Prepared for auto-build harness integration

**`profiles/default/workflows/implementation/bv-graph-insights.md`** (NEW)
- Structural graph intelligence workflow
- Metrics explained:
  - **Bottlenecks (Betweenness)**: Issues bridging different graph parts
  - **Keystones (Critical path)**: Longest dependency chains
  - **Influencers (Eigenvector)**: Foundational work connected to important tasks
  - **Hubs/Authorities (HITS)**: Aggregators and prerequisites
- Agent decision examples and real-world scenarios

**`profiles/default/workflows/implementation/bv-cycle-detection.md`** (NEW)
- Circular dependency detection and prevention
- Pre-flight checks before adding dependencies
- Post-creation validation in create-beads-issues workflow
- Cycle fixing strategies (remove, extract, reorder, split)

**`profiles/default/workflows/implementation/bv-recipes.md`** (NEW)
- Pre-built filtered views for common queries
- Built-in recipes: `actionable`, `high-impact`, `blocked`, `stale`, `recent`
- Integration with execution planning
- Custom recipe support

**`profiles/default/workflows/implementation/bv-parallel-execution.md`** (NEW)
- Multi-agent coordination using track-based planning
- Lock mechanism to prevent work conflicts
- Track assignment and parallel orchestration
- Performance considerations and speedup calculations

### Modified Workflow Files (9 Updated)

**1. `profiles/default/workflows/specification/initialize-spec.md`**
- Added BV installation prompt after beads installation (line 163)
- Prompts: "(i)nstall bv now or (s)kip?"
- Updates `spec-config.yml` with `bv_enabled: true/false`
- Automatic installation via `cargo install bv` (placeholder command)

**2. `profiles/default/workflows/implementation/implement-with-beads.md`**
- Replaced greedy task selection with execution plan logic (lines 67-131)
- Smart selection: highest-impact actionable issue (unblocks most downstream tasks)
- Added structural warnings for bottlenecks, keystones, influencers (lines 103-138)
- Added session summary at end with diff tracking (lines 518-583)

**3. `profiles/default/workflows/implementation/create-beads-issues.md`**
- Added cycle detection validation after all dependencies created (Step 9.5, lines 424-460)
- Validates dependency graph has no circular dependencies
- Exits with error if cycles detected

**4. `profiles/default/commands/implement-tasks/single-agent/1-determine-tasks.md`**
- Enhanced with BV execution plan display (lines 7-84)
- Shows parallel tracks available
- Displays recommended next task with reasoning (impact = unblocks count)
- Recipe quick checks (actionable, high-impact, blocked counts)

**5. `profiles/default/commands/implement-tasks/single-agent/2-implement-tasks.md`**
- Added BV session summary before completion check (lines 10-46)
- Displays closed/created/modified issue counts
- Warns if cycles introduced during session

**6. `profiles/default/commands/orchestrate-tasks/orchestrate-tasks.md`**
- Added priority analysis step (lines 52-114)
- Displays priority recommendations with confidence scores
- Optional auto-application of high-confidence priority updates
- Added parallel execution option (lines 210-265)
- Detects independent tracks, offers parallel vs sequential choice

**7. `profiles/default/commands/auto-build/auto-build.md`**
- Added Phase 1.5: Check for Changes and Cycles (lines 73-145)
- Diff tracking since last build
- Cycle regression detection (warns if new cycles introduced)
- Pre-flight cycle check before build starts
- Stores commit reference for next build comparison

**8. `README.md`**
- Added BV section with feature overview (lines 93-146)
- Installation instructions
- Example: greedy vs impact-aware selection
- Graph metrics explained

**9. `FORK-CHANGES.md`**
- This section! Technical documentation of BV integration

### BV Commands Used

- `bv --robot-plan --format json` - Execution planning with track-based parallelization
- `bv --robot-priority --format json` - Priority recommendations based on graph metrics
- `bv --robot-diff --diff-since <ref> --format json` - Time-travel diffs between commits/sessions
- `bv --robot-insights --format json` - Graph metrics (bottlenecks, keystones, influencers, cycles)
- `bv --recipe <name>` - Quick filtered views (actionable, high-impact, blocked, stale, recent)
- `bv --robot-recipes --format json` - List available recipes

### Spec Configuration Update

**`.agent-os/specs/[spec-name]/spec-config.yml`** - Added field:
```yaml
tracking_mode: beads
created: 2025-12-15T10:30:00Z
beads_initialized: true
bv_enabled: true  # NEW: Whether bv is available for graph intelligence
```

### Key Integration Points

**1. Smart Task Selection (Phase 2 - Highest Priority)**

Agents use `bv --robot-plan` instead of greedy `bd ready`:

```bash
# Before (greedy)
bd ready | head -1  # First ready item

# After (impact-aware)
bv --robot-plan | jq '.tracks[0].items[0]'  # Highest-impact ready item
```

Selects work that unblocks the most downstream tasks.

**2. Priority Intelligence (Phase 3 - High Priority)**

Detects priority misalignments during orchestration:

```bash
# Detect: Low priority but high structural importance
bv --robot-priority | jq '.recommendations[] | select(.direction == "increase")'

# Example output:
# bd-101: P3 → P1 (95% confidence)
# Reason: Critical path item - blocks 8 downstream tasks
```

**3. Cycle Detection (Phase 6 - High Priority)**

Prevents deadlocks from circular dependencies:

```bash
# After creating beads issues
if ! check_cycles; then
    echo "❌ Circular dependencies detected!"
    bv --robot-insights | jq '.cycles'  # Show cycle paths
    exit 1
fi
```

**4. Time-Travel Diffs (Phase 4 - Medium Priority)**

Session summaries show what changed:

```bash
# At end of session
SESSION_START=$(cat .beads/session-start-commit)
bv --robot-diff --diff-since "$SESSION_START" | jq '.changes.closed_issues'
# Shows all issues closed this session
```

**5. Graph Insights (Phase 5 - High Priority)**

Structural warnings for critical work:

```bash
# If selected issue is a bottleneck
IS_BOTTLENECK=$(bv --robot-insights | jq '.bottlenecks[] | select(.id == $CURRENT_ISSUE_ID)')

if [[ -n "$IS_BOTTLENECK" ]]; then
    echo "⚠️  BOTTLENECK - This issue bridges multiple work streams"
fi
```

**6. Recipe System (Phase 7 - Medium Priority)**

Quick filtered views:

```bash
bv --recipe actionable  # Unblocked ready work
bv --recipe high-impact  # Top PageRank/betweenness scores
bv --recipe blocked     # Issues waiting on dependencies
```

**7. Parallel Execution (Phase 8 - Medium Priority)**

Multi-agent coordination:

```bash
TRACK_COUNT=$(bv --robot-plan | jq '.tracks | length')
# If ≥2 tracks, offer parallel execution option
# Launch one agent per track
```

### Graceful Fallback

All BV features degrade gracefully:

| Feature | BV Available | BV Unavailable |
|---------|--------------|----------------|
| Task selection | Impact-aware planning | Basic `bd ready` (greedy) |
| Priority recommendations | Graph-based suggestions | Empty recommendations |
| Cycle detection | Pre-flight validation | Permissive (skip check) |
| Session diffs | Full change tracking | Empty diff object |
| Graph insights | Bottleneck/keystone analysis | Empty insights |
| Recipes | Filtered views | Empty arrays |
| Parallel execution | Track-based coordination | Sequential execution |

No hard failures - agents continue working with reduced intelligence if BV is unavailable.

### Testing Phases

1. **BV Installation** - Auto-install prompt, fallback to tasks.md
2. **Smart Task Selection** - Verify impact-aware selection vs greedy
3. **Priority Intelligence** - Test with misaligned priorities
4. **Time-Travel Diffs** - Track changes across sessions
5. **Graph Insights** - Complex spec with shared dependencies
6. **Cycle Detection** - Intentional cycle creation, verify detection
7. **Recipe System** - All built-in recipes functional
8. **Parallel Execution** - Independent tracks, multi-agent coordination

---

## 3. Configuration System (3 Modified Files)

### `config.yml` (MODIFIED)

Added new configuration section:

```yaml
# Issue tracking mode (affects all new specs)
tracking_mode: tasks_md  # Default: tasks_md | beads

# Whether beads CLI is available as tracking option
beads_enabled: true      # Default: true
```

### `scripts/common-functions.sh` (MODIFIED)

**Updated `process_conditionals()` function:**
- Added 2 new parameters: `tracking_mode` (param 5), `beads_enabled` (param 6)
- Added 3 new conditional cases:
  - `tracking_mode_beads` - true when tracking_mode=beads
  - `tracking_mode_tasks_md` - true when tracking_mode=tasks_md
  - `beads_enabled` - true when beads available

**Updated function calls:**
- Both `process_conditionals()` calls now pass the new parameters

### `scripts/project-install.sh` (MODIFIED)

**Updated `load_configuration()` function:**
- Added EFFECTIVE_TRACKING_MODE and EFFECTIVE_BEADS_ENABLED variables
- Added verbose output for new config values
- These values are passed to `compile_agent()` which processes conditionals

**Updated `compile_agent()` function (line 987):**
```bash
content=$(process_conditionals "$content" \
  "${EFFECTIVE_USE_CLAUDE_CODE_SUBAGENTS:-true}" \
  "${EFFECTIVE_STANDARDS_AS_CLAUDE_CODE_SKILLS:-true}" \
  "false" \
  "${EFFECTIVE_TRACKING_MODE:-tasks_md}" \
  "${EFFECTIVE_BEADS_ENABLED:-true}")
```

---

## 4. Spec Initialization (3 Modified Files)

### `profiles/default/workflows/specification/initialize-spec.md` (MODIFIED)

Added Step 4: "Choose Issue Tracking Mode"

Conditional prompt:
```markdown
{{IF beads_enabled}}
Which issue tracking mode would you like to use for this spec?

**Option 1: tasks.md (Recommended for simpler features)**
- Hierarchical task groups with checkboxes
- Sequential task execution
- Manual progress tracking

**Option 2: beads (Recommended for complex features)**
- Distributed issue tracking
- Auto-generated atomic design hierarchy
- Cross-session context recovery
- Ready work queue

{{ENDIF beads_enabled}}
```

Creates `spec-config.yml` with user's choice:
```yaml
tracking_mode: beads  # or tasks_md
created: 2025-12-15T10:30:00Z
beads_initialized: true
```

Initializes beads if chosen: `bd init --stealth`

### `profiles/default/commands/write-spec/single-agent/write-spec.md` (MODIFIED)

Updated next-step output with conditional:
```markdown
{{IF tracking_mode_beads}}
Next step: Create beads issues from the spec
- Run `/create-tasks` to auto-generate beads issues with atomic design hierarchy
{{ELSE}}
Next step: Create task breakdown
- Run `/create-tasks` to generate hierarchical task groups in tasks.md
{{ENDIF tracking_mode_beads}}
```

### `profiles/default/commands/write-spec/multi-agent/write-spec.md` (MODIFIED)

Similar conditional next-step output as single-agent version.

---

## 5. Task Commands with Conditionals (6 Modified Files)

### `profiles/default/commands/create-tasks/single-agent/create-tasks.md` (MODIFIED)

Added conditional wrapper for Phase 2:
```markdown
{{IF tracking_mode_beads}}
  {{workflows/implementation/create-beads-issues}}
{{ELSE}}
  {{workflows/implementation/create-tasks-list}}
{{ENDIF tracking_mode_beads}}
```

### `profiles/default/commands/create-tasks/multi-agent/create-tasks.md` (MODIFIED)

- Updated Phase 2 with conditional for beads vs tasks.md creation
- Updated Phase 3 output with different messages for each tracking mode

### `profiles/default/commands/implement-tasks/single-agent/1-determine-tasks.md` (MODIFIED)

Added conditional to query beads for ready work:
```markdown
{{IF tracking_mode_beads}}
Query beads to find ready work:
```bash
cd .agent-os/specs/[this-spec]/
bd ready
```
{{ELSE}}
Read `.agent-os/specs/[this-spec]/tasks.md`...
{{ENDIF tracking_mode_beads}}
```

### `profiles/default/commands/implement-tasks/single-agent/2-implement-tasks.md` (MODIFIED)

Added conditional for checking completion status:
- Beads mode: `bd list --json | jq -r '.[] | select(.status!="closed")'`
- Tasks.md mode: Check for `[x]` markers

### `profiles/default/commands/implement-tasks/multi-agent/implement-tasks.md` (MODIFIED)

- Updated Phase 1 with conditional for beads ready work query
- Updated Phase 3 with conditional completion check

### `profiles/default/commands/orchestrate-tasks/orchestrate-tasks.md` (MODIFIED)

Added conditional for orchestration structure:

**Beads mode:**
```yaml
beads:
  - id: bd-a1b2
    title: Database Layer
    assignee: database-layer-builder
    standards: [backend/*, global/atomic-design.md]
```

**Tasks.md mode:**
```yaml
task_groups:
  - name: database-layer
    claude_code_subagent: database-layer-builder
    standards: [backend/*]
```

Automatic agent assignment for beads mode based on atomic design levels.

---

## 6. Standards Enhancement (1 Modified File)

### `profiles/default/standards/global/atomic-design.md` (MODIFIED)

**Added sections:**

1. **Agent Responsibility Mappings** - Table showing which agent handles which atomic level
2. **Dependency Flow Enforcement** - Rules for ensuring proper dependency direction
3. **Beads Integration Patterns** - Examples of beads issue hierarchy for atomic design
4. **Testing Strategy per Atomic Level** - What to test at each level

**Example beads hierarchy:**
```
bd-a1b2e9         [epic] User Authentication Feature
bd-a1b2e9.1       [organism] Database Layer
bd-a1b2e9.1.1     [molecule] User validation helpers
bd-a1b2e9.1.1.1   [atom] Email validator function
```

---

## 7. Implementer Update (1 Modified File)

### `profiles/default/agents/implementer.md` (MODIFIED)

**Transformed from monolithic to dual-role:**

**Mode 1: Direct Implementation (Simple Features)**
- For straightforward features
- Implements tasks directly following spec and tasks.md
- When: task groups < 10 tasks, no complex atomic design needed

**Mode 2: Coordination (Complex Features)**
- For complex features using atomic design principles
- Coordinates specialized agents
- When: multiple layers, atomic design agents needed, long-running implementation

**Added coordination guidelines:**
- Identify atomic level of current task group
- Respect dependencies (atoms → molecules → organisms)
- Pass context to agents (spec.md, requirements.md, standards)
- Track progress in tasks.md or beads

**Maintains backward compatibility** with existing workflows.

---

## Template Conditional Syntax

All conditional processing uses this syntax:

```markdown
{{IF tracking_mode_beads}}
  Content for beads mode
{{ELSE}}
  Content for tasks.md mode
{{ENDIF tracking_mode_beads}}
```

Available conditionals:
- `use_claude_code_subagents` - When using Claude Code subagents
- `standards_as_claude_code_skills` - When using skills for standards
- `tracking_mode_beads` - When spec uses beads tracking
- `tracking_mode_tasks_md` - When spec uses tasks.md tracking
- `beads_enabled` - When beads is available as option

---

## Per-Spec Configuration

Each spec can have its own tracking mode via `.agent-os/specs/[spec-name]/spec-config.yml`:

```yaml
tracking_mode: beads  # or tasks_md
created: 2025-12-15T10:30:00Z
beads_initialized: true
```

This file is created during spec initialization and read by subsequent commands.

---

## Backward Compatibility

All changes maintain backward compatibility:

✅ Existing specs using tasks.md continue working
✅ Default tracking mode is tasks_md
✅ Beads is optional (beads_enabled flag)
✅ Original implementer agent preserved as coordinator
✅ All existing commands work with both tracking modes

---

## Dependencies

### Required
- Bash 4.0+
- Git
- jq (for JSON processing)

### Optional
- [Beads CLI](https://github.com/steveyegge/beads) - Required only for beads tracking mode
- Playwright - Used by ui-component-builder and integration-assembler agents

---

## Installation

See [README.md](README.md) for installation instructions.

---

## Contributing

To contribute improvements back to this fork:

1. Fork this repo (jaypaulb/agent-os)
2. Create a feature branch
3. Make your changes
4. Submit a pull request

To sync with upstream buildermethods/agent-os:

```bash
git fetch upstream
git merge upstream/master
```

---

## License

Same as upstream: [LICENSE](LICENSE)

---

## Credits

**Fork maintainer:** jaypaulb
**Original creator:** Brian Casel @ [Builder Methods](https://buildermethods.com)
**Beads integration:** Inspired by [steveyegge/beads](https://github.com/steveyegge/beads)

This fork builds upon the excellent foundation of agent-os with enhancements for atomic design principles and distributed issue tracking.
