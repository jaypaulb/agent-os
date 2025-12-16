# Process for Orchestrating Product Implementation

Orchestrate implementation across ALL specs/phases in parallel from project root.

**Run this from PROJECT ROOT, not from a spec folder.**

## Execution Modes

This workflow supports two modes:

1. **Interactive Mode** (default): Full orchestration with agent delegation
2. **Headless Mode** (`--headless` flag): Prepare orchestration state and exit (for autonomous-build)

To run in headless mode for autonomous-build integration:
```bash
/orchestrate-tasks --headless
```

## Multi-Phase Process

### FIRST: Check for Headless Mode

```bash
# Parse command-line arguments
HEADLESS_MODE=false

for arg in "$@"; do
    if [ "$arg" = "--headless" ]; then
        HEADLESS_MODE=true
        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
        echo "  HEADLESS MODE: Autonomous Build Integration"
        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
        echo ""
        echo "Creating orchestration state for autonomous-build..."
        echo ""
        break
    fi
done
```

### FIRST: Verify Beads and Get All Phases

{{IF tracking_mode_beads}}
Check Beads at project root and discover all phases:

```bash
# Ensure we're at project root
if [ ! -d ".beads" ]; then
    echo "❌ No .beads/ directory found at project root"
    echo "   Please run from project root after running /autonomous-plan"
    exit 1
fi

echo "✓ Beads initialized at project root"

# Source BV helpers
source agent-os/profiles/default/workflows/implementation/bv-helpers.md

# Get all phase epics
if bv_available; then
    echo "Using BV to discover phases..."
    PHASE_EPICS=$(bv --filter "type:epic" --format json 2>/dev/null || \
                  bd list --type epic --json)
else
    echo "Using bd to discover phases..."
    PHASE_EPICS=$(bd list --type epic --json)
fi

PHASE_COUNT=$(echo "$PHASE_EPICS" | jq '. | length')

if [ "$PHASE_COUNT" -eq 0 ]; then
    echo "❌ No phase epics found"
    echo "   Run /autonomous-plan to create specs and tasks first"
    exit 1
fi

echo "✓ Found $PHASE_COUNT phase(s) to orchestrate"
echo ""
echo "Phases:"
echo "$PHASE_EPICS" | jq -r '.[] | "  • Phase: \(.title) (\(.id))"'
```

If phases exist, proceed to analyze parallel execution opportunities.

{{ELSE}}
IF you already know which spec we're working on and IF that spec folder has a `tasks.md` file, then use that and skip to the NEXT phase.

IF you don't already know which spec we're working on and IF that spec folder doesn't yet have a `tasks.md` THEN output the following request to the user:

```
Please point me to a spec's `tasks.md` that you want to orchestrate implementation for.

If you don't have one yet, then run any of these commands first:
/shape-spec
/write-spec
/create-tasks
```
{{ENDIF tracking_mode_beads}}

### NEXT: Analyze Parallel Execution Opportunities

Use BV to identify which work can run in parallel across ALL phases:

{{IF tracking_mode_beads}}
```bash
# Already at project root with BV helpers sourced

if bv_available; then
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Analyzing Parallel Execution Opportunities"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""

    # Get execution plan (BV analyzes ALL phases and finds parallel tracks)
    EXEC_PLAN=$(get_execution_plan)

    # How many independent tracks can run in parallel?
    TRACK_COUNT=$(echo "$EXEC_PLAN" | jq -r '.tracks | length')

    echo "BV found $TRACK_COUNT independent work stream(s):"
    echo ""

    # Display each track
    echo "$EXEC_PLAN" | jq -r '
        .tracks[] |
        "Track \(.track_id): \(.reason)",
        "  Priority: P\(.priority_range.min)-P\(.priority_range.max)",
        "  Items: \(.items | length) issues",
        "  Phases involved: \([.items[].labels[] | select(startswith("phase-"))] | unique | join(", "))",
        "  Top issues:",
        (.items[:3] | .[] | "    • \(.id): \(.title) (P\(.priority))"),
        ""
    '

    if [ "$TRACK_COUNT" -gt 1 ]; then
        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
        echo "✨ Parallel Execution Available!"
        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
        echo ""
        echo "You can run $TRACK_COUNT agents in parallel, each working on a different track."
        echo "This will significantly speed up implementation."
        echo ""
    else
        echo "Note: Only 1 work stream found - sequential execution recommended"
        echo ""
    fi
else
    echo "⚠️  BV unavailable - parallel analysis not available"
    echo "   Will proceed with sequential execution"
    TRACK_COUNT=1
fi
```

### NEXT: Analyze Priority Recommendations (Optional)

Analyze priority across ALL phases:

```bash
# Already at project root with BV helpers sourced

if bv_available; then
    echo ""
    echo "Running priority analysis across all phases..."
    echo ""

    # Get priority recommendations for entire product
    PRIORITY_RECS=$(get_priority_recommendations)

    REC_COUNT=$(echo "$PRIORITY_RECS" | jq '.recommendations | length')

    if [ "$REC_COUNT" -gt 0 ]; then
        # Display summary
        echo "$PRIORITY_RECS" | jq -r '
            "=== Priority Analysis (All Phases) ===",
            "",
            "Total Recommendations: \(.summary.recommendations)",
            "High Confidence (>0.8): \(.summary.high_confidence // 0)",
            "",
            "Top Priority Misalignments:",
            (.recommendations[:10] | .[] |
             "  • \(.issue_id): P\(.current_priority) → P\(.suggested_priority) (\(.confidence*100 | round)% confidence)",
             "    Phase: \([.labels[] | select(startswith("phase-"))] | join(", "))",
             "    \(.reasoning)",
             "")
        ' | tee "agent-os/product/priority-analysis.txt"

        echo ""
        echo "Review agent-os/product/priority-analysis.txt for full details."
        echo ""

        # Ask user if they want to apply high-confidence updates
        read -p "Apply high-confidence priority updates (>0.8 confidence)? [y/N]: " apply_priorities

        if [[ "$apply_priorities" =~ ^[Yy]$ ]]; then
            # Extract and apply high-confidence increases
            HIGH_CONF=$(echo "$PRIORITY_RECS" | jq -r '
                .recommendations[] |
                select(.confidence > 0.8 and .direction == "increase") |
                "\(.issue_id) \(.suggested_priority)"
            ')

            if [[ -n "$HIGH_CONF" ]]; then
                echo "Applying priority updates across all phases..."
                while read -r issue_id new_priority; do
                    bd update "$issue_id" --priority "$new_priority" \
                        --note "Priority updated based on graph analysis"
                    echo "✓ Updated $issue_id to P$new_priority"
                done <<< "$HIGH_CONF"

                echo ""
                echo "✓ Priority updates applied - re-run execution plan for updated tracks"
                # Re-fetch execution plan with updated priorities
                EXEC_PLAN=$(get_execution_plan)
            else
                echo "No high-confidence priority increases found."
            fi
        fi
    else
        echo "No priority recommendations across phases."
    fi
else
    echo "BV unavailable - skipping priority analysis"
fi
```
{{ENDIF tracking_mode_beads}}

### NEXT: Create Orchestration Plan

{{IF tracking_mode_beads}}
Create orchestration.yml with ALL organism issues across ALL phases:

```bash
# Already at project root

# Get all organism issues across all phases
if bv_available; then
    ORGANISMS=$(bv --filter "type:organism" --format json 2>/dev/null || \
                bd list --type organism --json)
else
    ORGANISMS=$(bd list --type organism --json)
fi

# Choose output path based on mode
if [ "$HEADLESS_MODE" = true ]; then
    # Headless mode: Write to autonomous-build state directory
    ORCHESTRATION_FILE=".beads/autonomous-state/orchestration.yml"

    # Ensure directory exists
    mkdir -p .beads/autonomous-state
else
    # Interactive mode: Write to product directory
    ORCHESTRATION_FILE="agent-os/product/orchestration.yml"

    # Ensure directory exists
    mkdir -p agent-os/product
fi

# Create orchestration file
cat > "$ORCHESTRATION_FILE" <<EOF
# Multi-Phase Orchestration Plan
# Generated: $(date)
# Total Phases: $PHASE_COUNT
# Mode: $([ "$HEADLESS_MODE" = true ] && echo "headless" || echo "interactive")
beads:
EOF

# Add each organism with its phase and spec labels
echo "$ORGANISMS" | jq -r '.[] | "  - id: \(.id)\n    title: \(.title)"' >> "$ORCHESTRATION_FILE"

echo ""
echo "✓ Created $ORCHESTRATION_FILE with $(echo "$ORGANISMS" | jq '. | length') organism issues"
```

{{ELSE}}
Create `agent-os/product/orchestration.yml` by discovering all specs in agent-os/specs/:

```bash
# Find all spec folders with tasks.md
SPEC_FOLDERS=$(find agent-os/specs -name "tasks.md" -exec dirname {} \;)

# Create orchestration file
cat > agent-os/product/orchestration.yml <<EOF
# Multi-Phase Orchestration Plan
# Generated: $(date)
specs:
EOF

# For each spec folder, extract task groups
for spec_folder in $SPEC_FOLDERS; do
    spec_name=$(basename "$spec_folder")

    echo "  - spec: $spec_name" >> agent-os/product/orchestration.yml
    echo "    task_groups:" >> agent-os/product/orchestration.yml

    # Extract task group names from tasks.md
    grep "^####" "$spec_folder/tasks.md" | sed 's/#### //' | while read -r task_group; do
        echo "      - name: $task_group" >> agent-os/product/orchestration.yml
    done
done

echo ""
echo "✓ Created agent-os/product/orchestration.yml for all specs"
cat agent-os/product/orchestration.yml
```
{{ENDIF tracking_mode_beads}}

{{IF use_claude_code_subagents}}
### NEXT: Assign atomic design agents to work

{{IF tracking_mode_beads}}
For beads mode, agents are automatically assigned based on atomic design levels:
- Atoms → `atom-writer`
- Molecules → `molecule-composer`
- Database organisms → `database-layer-builder`
- API organisms → `api-layer-builder`
- UI organisms → `ui-component-builder`
- Tests (molecules) → `test-writer-molecule`
- Tests (organisms) → `test-writer-organism`
- Test gaps → `test-gap-analyzer`
- Integration → `integration-assembler`

Analyze each organism issue and assign the appropriate agent:

```bash
# Already at project root

# Read current orchestration.yml (using variable from earlier section)
ORGANISMS=$(yq eval '.beads[] | .id' "$ORCHESTRATION_FILE")

# For each organism, infer the correct agent based on title
while read -r organism_id; do
    TITLE=$(bd show "$organism_id" --json | jq -r '.title')

    # Infer agent based on keywords in title
    if [[ "$TITLE" =~ [Dd]atabase|[Mm]odel|[Ss]chema|[Mm]igration ]]; then
        AGENT="database-layer-builder"
        STANDARDS="backend/*, global/atomic-design.md"
    elif [[ "$TITLE" =~ API|[Ee]ndpoint|[Cc]ontroller|[Rr]oute ]]; then
        AGENT="api-layer-builder"
        STANDARDS="backend/*, global/atomic-design.md"
    elif [[ "$TITLE" =~ UI|[Cc]omponent|[Pp]age|[Vv]iew|[Ff]rontend ]]; then
        AGENT="ui-component-builder"
        STANDARDS="frontend/*, global/atomic-design.md"
    elif [[ "$TITLE" =~ [Ii]ntegration|[Ww]ire|E2E ]]; then
        AGENT="integration-assembler"
        STANDARDS="global/*"
    elif [[ "$TITLE" =~ [Tt]est.*[Gg]ap|[Cc]overage ]]; then
        AGENT="test-gap-analyzer"
        STANDARDS="global/test-writing.md"
    else
        AGENT="molecule-composer"  # Default fallback
        STANDARDS="global/atomic-design.md"
    fi

    # Update orchestration.yml with assignee
    yq eval -i "(.beads[] | select(.id == \"$organism_id\") | .assignee) = \"$AGENT\"" "$ORCHESTRATION_FILE"
    yq eval -i "(.beads[] | select(.id == \"$organism_id\") | .standards) = [\"$STANDARDS\"]" "$ORCHESTRATION_FILE"

    echo "✓ $organism_id ($TITLE) → $AGENT"
done <<< "$ORGANISMS"

echo ""
echo "✓ Agent assignments complete"
```

Note: Atom and molecule agents work on child issues automatically based on beads hierarchy.

{{ELSE}}
Next we must determine which subagents should be assigned to which task groups.

**Autonomous Agent Assignment Process:**

For each task group in `tasks.md`:

1. **Read suggested agent** from task group metadata (look for `**Suggested Agent:**` line)
2. **Analyze task content** to compute congruence score:
   - Check keywords in task group name and description
   - Validate against atomic design patterns
   - Score: 0-100% congruence between suggested agent and task content

3. **Agent assignment decision logic:**

```python
if congruence >= 80:
    # High congruence - use suggested agent
    assigned_agent = suggested_agent
    log(f"✓ Using suggested agent '{suggested_agent}' for '{task_group}' (congruence: {congruence}%)")

elif congruence >= 50:
    # Medium congruence - pick better agent autonomously
    better_agent = infer_best_agent(task_group_content)
    assigned_agent = better_agent
    log(f"⚠ Overriding suggested agent '{suggested_agent}' → '{better_agent}' for '{task_group}' (congruence: {congruence}%, better match found)")

else:
    # Low congruence - pick correct agent autonomously
    correct_agent = infer_best_agent(task_group_content)
    assigned_agent = correct_agent
    log(f"⚠ Rejecting suggested agent '{suggested_agent}' → using '{correct_agent}' for '{task_group}' (low congruence: {congruence}%)")
```

**Congruence Scoring Heuristics:**

```markdown
atom-writer:
  - Keywords: "atom", "pure function", "utility", "validator", "formatter", "constant", "calculation"
  - Anti-patterns: "database", "API", "component", "integration", "molecule"
  - Score boost: +30 if task name contains "atom", +20 if description mentions "zero dependencies"

molecule-composer:
  - Keywords: "molecule", "compose", "helper", "service", "2-3 atoms", "composition"
  - Anti-patterns: "database", "API endpoint", "UI component", "model", "controller"
  - Score boost: +30 if task name contains "molecule", +20 if description mentions "compose"

database-layer-builder:
  - Keywords: "database", "model", "migration", "schema", "association", "ORM", "SQL"
  - Anti-patterns: "API endpoint", "controller", "UI", "component", "frontend"
  - Score boost: +30 if task name contains "database" or "model", +20 if mentions "migration"

api-layer-builder:
  - Keywords: "API", "endpoint", "controller", "REST", "authentication", "authorization", "route"
  - Anti-patterns: "database model", "UI", "component", "frontend", "CSS"
  - Score boost: +30 if task name contains "API" or "endpoint", +20 if mentions "controller"

ui-component-builder:
  - Keywords: "UI", "component", "form", "page", "view", "CSS", "responsive", "frontend"
  - Anti-patterns: "database", "model", "API endpoint", "controller", "migration"
  - Score boost: +30 if task name contains "UI" or "component", +20 if mentions "frontend"

test-writer-molecule:
  - Keywords: "test", "molecule", "unit test", "composition test"
  - Anti-patterns: "integration", "organism", "E2E"
  - Score boost: +40 if both "test" AND "molecule" present

test-writer-organism:
  - Keywords: "test", "organism", "integration test", "layer test"
  - Anti-patterns: "unit test", "molecule", "atom"
  - Score boost: +40 if both "test" AND "organism" present

test-gap-analyzer:
  - Keywords: "test gap", "coverage", "analysis", "fill gaps", "additional tests"
  - Anti-patterns: "unit test", "write tests for X"
  - Score boost: +50 if "gap" or "coverage" mentioned

integration-assembler:
  - Keywords: "integration", "wire", "E2E", "assemble", "connect layers", "full system"
  - Anti-patterns: "unit test", "atom", "molecule", "single layer"
  - Score boost: +40 if "integration" or "wire" mentioned
```

**Agent Inference Algorithm:**

```python
def infer_best_agent(task_group_content):
    scores = {}
    for agent in ALL_AGENTS:
        score = 0

        # Keyword matching
        for keyword in agent.keywords:
            if keyword.lower() in task_group_content.lower():
                score += 10

        # Anti-pattern penalty
        for anti_pattern in agent.anti_patterns:
            if anti_pattern.lower() in task_group_content.lower():
                score -= 20

        # Apply score boosts
        score += agent.compute_boost(task_group_content)

        scores[agent.name] = max(0, score)  # Floor at 0

    # Return highest-scoring agent
    return max(scores, key=scores.get)
```

**Implementation:**

```bash
# Read tasks.md and extract task groups with suggested agents
while IFS= read -r task_group; do
    # Parse task group name
    name=$(extract_task_group_name "$task_group")

    # Parse suggested agent (if present)
    suggested_agent=$(grep -A 5 "#### $name" tasks.md | grep "Suggested Agent:" | sed 's/.*`\(.*\)`.*/\1/')

    # If no suggestion, infer from content
    if [[ -z "$suggested_agent" ]]; then
        echo "⚠ No suggested agent for '$name' - inferring from content..."
        assigned_agent=$(infer_best_agent "$task_group")
        echo "→ Assigned: $assigned_agent"
    else
        # Calculate congruence
        congruence=$(compute_congruence "$task_group" "$suggested_agent")

        if [[ "$congruence" -ge 80 ]]; then
            # High congruence - use suggestion
            assigned_agent="$suggested_agent"
            echo "✓ Using suggested agent '$assigned_agent' for '$name' ($congruence% congruence)"
        elif [[ "$congruence" -ge 50 ]]; then
            # Medium congruence - pick better agent
            better_agent=$(infer_best_agent "$task_group")
            assigned_agent="$better_agent"
            echo "⚠ Overriding '$suggested_agent' → '$better_agent' for '$name' ($congruence% congruence, better match)"
        else
            # Low congruence - reject suggestion
            correct_agent=$(infer_best_agent "$task_group")
            assigned_agent="$correct_agent"
            echo "⚠ Rejecting '$suggested_agent' → using '$correct_agent' for '$name' (low congruence: $congruence%)"
        fi
    fi

    # Add to orchestration.yml
    echo "  - name: $name" >> orchestration.yml
    echo "    claude_code_subagent: $assigned_agent" >> orchestration.yml
done < <(extract_task_groups tasks.md)
```

**Only ask user if truly ambiguous:**

If multiple agents score equally (within 10 points) and no clear winner:

```
Multiple agents could handle task group '[task-group-name]':
- database-layer-builder (score: 45)
- api-layer-builder (score: 42)

Which agent should handle this? [database-layer-builder/api-layer-builder]:
```

**Result:** `orchestration.yml` populated with validated agent assignments, minimal user input.

```yaml
task_groups:
  - name: Core Utilities and Pure Functions
    claude_code_subagent: atom-writer
  - name: Composed Helpers and Services
    claude_code_subagent: molecule-composer
  - name: Data Models and Migrations
    claude_code_subagent: database-layer-builder
  - name: API Endpoints and Controllers
    claude_code_subagent: api-layer-builder
  - name: UI Components and Pages
    claude_code_subagent: ui-component-builder
  - name: Molecule-Level Testing
    claude_code_subagent: test-writer-molecule
  - name: Organism-Level Testing
    claude_code_subagent: test-writer-organism
  - name: Test Gap Analysis
    claude_code_subagent: test-gap-analyzer
  - name: System Integration and E2E Verification
    claude_code_subagent: integration-assembler
```

{{ENDIF tracking_mode_beads}}

### NEXT: Choose Execution Mode (Parallel or Sequential)

{{IF tracking_mode_beads}}
The parallel execution analysis was already done earlier. Present the final options to the user:

```bash
# Already at project root with TRACK_COUNT and EXEC_PLAN from earlier analysis

if [[ "$TRACK_COUNT" -gt 1 ]]; then
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Ready to Execute: Choose Mode"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""
    echo "Option 1: Parallel Execution ($TRACK_COUNT independent work streams)"
    echo "  - Requires $TRACK_COUNT Claude Code instances"
    echo "  - Fastest approach"
    echo "  - Each track works independently across all phases"
    echo ""
    echo "Option 2: Sequential Execution (one agent at a time)"
    echo "  - Single Claude Code instance"
    echo "  - Slower but simpler"
    echo ""
else
    echo ""
    echo "Only 1 work stream detected - sequential execution recommended"
    echo ""
fi
```

**If user chooses parallel execution:**
Follow the parallel execution workflow:
{{workflows/implementation/bv-parallel-execution}}

**If user chooses sequential or only 1 track exists:**
Continue with delegation section below...

{{ENDIF tracking_mode_beads}}

{{ENDIF use_claude_code_subagents}}

{{UNLESS standards_as_claude_code_skills}}
### NEXT: Ask user to assign standards to each task group

Next we must determine which standards should guide the implementation of each task group.  Ask the user to provide this info using the following request to user and WAIT for user's response:

```
Please specify the standard(s) that should be used to guide the implementation of each task group:

1. [task-group-name]
2. [task-group-name]
3. [task-group-name]
[repeat for each task-group you've added to orchestration.yml]

For each task group number, you can specify any combination of the following:

"all" to include all of your standards
"global/*" to include all of the files inside of standards/global
"frontend/css.md" to include the css.md standard file
"none" to include no standards for this task group.
```

Using the user's responses, update `orchestration.yml` to specify those standards for each task group.  `orchestration.yml` should end up having AT LEAST the following information added to it:

```yaml
task_groups:
  - name: [task-group-name]
    standards:
      - [users' 1st response for this task group]
      - [users' 2nd response for this task group]
      - [users' 3rd response for this task group]
      # Repeat for all standards that the user specified for this task group
  - name: [task-group-name]
    standards:
      - [users' 1st response for this task group]
      - [users' 2nd response for this task group]
      # Repeat for all standards that the user specified for this task group
  # Repeat for each task group found in tasks.md
```

For example, after this step, the `orchestration.yml` file might look like this (exact names will vary):

```yaml
task_groups:
  - name: authentication-system
    standards:
      - all
  - name: user-dashboard
    standards:
      - global/*
      - frontend/components.md
      - frontend/css.md
  - name: task-group-with-no-standards
  - name: api-endpoints
    standards:
      - backend/*
      - global/error-handling.md
```

Note: If the `use_claude_code_subagents` flag is enabled, the final `orchestration.yml` would include BOTH `claude_code_subagent` assignments AND `standards` for each task group.
{{ENDUNLESS standards_as_claude_code_skills}}

### NEXT: Headless Mode Exit (if applicable)

```bash
# If running in headless mode, exit here and return control
if [ "$HEADLESS_MODE" = true ]; then
    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "  ✅ Headless Mode Complete"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo ""
    echo "Orchestration state prepared for autonomous-build:"
    echo "  • File: $ORCHESTRATION_FILE"
    echo "  • Organisms: $(yq eval '.beads | length' "$ORCHESTRATION_FILE")"
    echo "  • Agents assigned: Yes"
    echo ""
    echo "You can now run /autonomous-build to begin parallel implementation"
    echo ""
    exit 0
fi
```

**Interactive mode continues below** (only if not in headless mode):

{{IF use_claude_code_subagents}}
### NEXT: Delegate implementations to assigned subagents

{{IF tracking_mode_beads}}
**For Beads mode:** Delegate organism issues from `orchestration.yml` to their assigned agents.

Loop through each organism in orchestration.yml:

```bash
# Already at project root

# Read orchestration.yml and delegate each organism (using variable from earlier)
yq eval '.beads[]' "$ORCHESTRATION_FILE" -o json | jq -c '.' | while read -r organism; do
    ORGANISM_ID=$(echo "$organism" | jq -r '.id')
    ORGANISM_TITLE=$(echo "$organism" | jq -r '.title')
    ASSIGNEE=$(echo "$organism" | jq -r '.assignee')
    STANDARDS=$(echo "$organism" | jq -r '.standards | join(", ")')

    # Get organism details from beads
    ORGANISM_DATA=$(bd show "$ORGANISM_ID" --json)
    SPEC_LABEL=$(echo "$ORGANISM_DATA" | jq -r '.labels[] | select(startswith("phase-") | not)')
    SPEC_PATH="agent-os/specs/$SPEC_LABEL"

    echo ""
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "Delegating: $ORGANISM_TITLE"
    echo "  Issue: $ORGANISM_ID"
    echo "  Agent: $ASSIGNEE"
    echo "  Spec: $SPEC_PATH"
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

    # Delegate to the assigned agent
    # (This would call the appropriate subagent with context)
    # For now, just mark for tracking
done
```

For each organism delegation, provide the subagent with:
- The organism issue ID from Beads
- The spec file: `agent-os/specs/[spec-label]/spec.md`
- The requirements: `agent-os/specs/[spec-label]/planning/requirements.md`
- Instruct subagent to:
  - Implement the organism and its child molecules/atoms
  - Update Beads issue status as work progresses
  - Close the organism issue when complete

{{ELSE}}
**For tasks.md mode:** Loop through each spec's task groups.

For EACH spec in `agent-os/product/orchestration.yml`:
  - Read that spec's `tasks.md` file
  - Loop through each task group
  - Delegate to the assigned subagent specified in orchestration.yml

For each delegation, provide the subagent with:
- The task group (including the parent task and all sub-tasks)
- The spec file: `agent-os/specs/[spec-slug]/spec.md`
- Instruct subagent to:
  - Perform their implementation
  - Check off the task and sub-task(s) in `agent-os/specs/[spec-slug]/tasks.md`
{{ENDIF tracking_mode_beads}}

{{UNLESS standards_as_claude_code_skills}}
In addition to the above items, also instruct the subagent to closely adhere to the user's standards & preferences as specified in the following files.  To build the list of file references to give to the subagent, follow these instructions:

{{workflows/implementation/compile-implementation-standards}}

Provide all of the above to the subagent when delegating tasks for it to implement.
{{ENDUNLESS standards_as_claude_code_skills}}
{{ENDIF use_claude_code_subagents}}

{{UNLESS use_claude_code_subagents}}
### NEXT: Generate prompts

Now we must generate an ordered series of prompt texts, which will be used to direct the implementation of each task group listed in `orchestration.yml`.

Follow these steps to generate this spec's ordered series of prompts texts, each in its own .md file located in `agent-os/specs/[this-spec]/implementation/prompts/`.

LOOP through EACH task group in `agent-os/specs/[this-spec]/tasks.md` and for each, use the following workflow to generate a markdown file with prompt text for each task group:

#### Step 1. Create the prompt markdown file

Create the prompt markdown file using this naming convention:
`agent-os/specs/[this-spec]/implementation/prompts/[task-group-number]-[task-group-title].md`.

For example, if the 3rd task group in tasks.md is named "Comment System" then create `3-comment-system.md`.

#### Step 2. Populate the prompt file

Populate the prompt markdown file using the following Prompt file content template.

##### Bracket content replacements

In the content template below, replace "[spec-title]" and "[this-spec]" with the current spec's title, and "[task-group-number]" with the current task group's number.

{{UNLESS standards_as_claude_code_skills}}
To replace "[orchestrated-standards]", use the following workflow:

{{workflows/implementation/compile-implementation-standards}}
{{ENDUNLESS standards_as_claude_code_skills}}

#### Prompt file content template:

```markdown
We're continuing our implementation of [spec-title] by implementing task group number [task-group-number]:

## Implement this task and its sub-tasks:

[paste entire task group including parent task, all of its' sub-tasks, and sub-bullet points]

## Understand the context

Read @agent-os/specs/[this-spec]/spec.md to understand the context for this spec and where the current task fits into it.

Also read these further context and reference:
- @agent-os/specs/[this-spec/]/planning/requirements.md
- @agent-os/specs/[this-spec/]/planning/visuals

## Perform the implementation

{{workflows/implementation/implement-tasks}}

{{UNLESS standards_as_claude_code_skills}}
## User Standards & Preferences Compliance

IMPORTANT: Ensure that your implementation work is ALIGNED and DOES NOT CONFLICT with the user's preferences and standards as detailed in the following files:

[orchestrated-standards]
{{ENDUNLESS standards_as_claude_code_skills}}
```

### Step 3: Output the list of created prompt files

Output to user the following:

```
Ready to begin implementation of [spec-title]!

Use the following list of prompts to direct the implementation of each task group:

[list prompt files in order]

Input those prompts into this chat one-by-one or queue them to run in order.

Progress will be tracked in `agent-os/specs/[this-spec]/tasks.md`
```
{{ENDUNLESS use_claude_code_subagents}}
