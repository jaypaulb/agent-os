# Beads Cross-Session Context Recovery

This workflow enables agents to recover full context when resuming work across multiple sessions, preventing agent amnesia.

## The Problem: Agent Amnesia

AI agents suffer from context window limitations. When a session ends or context is compacted:
- Prior discoveries are forgotten
- Work in progress is unclear
- Dependencies are unknown
- History is lost

**Beads solves this** by providing persistent, queryable state that survives across sessions.

---

## When to Use Context Recovery

**Use this workflow when:**
- Starting a new session after previous work
- Agent is resumed after context compaction
- Unclear what was previously done
- Need to understand current project state
- Multiple agents working asynchronously

**Recovery Goal:** Reconstruct enough context to continue work effectively without re-reading entire codebase.

---

## Step 1: Navigate to Spec Folder

```bash
cd agent-os/specs/[this-spec]/

# Verify beads is initialized
ls .beads/
# Should show: issues.jsonl, config.yaml, beads.db
```

---

## Step 2: Query Overall Project State

### Total Progress Overview

```bash
# Count issues by status
echo "Project Status:"
echo "==============="
bd list --json | jq -r 'group_by(.status) | map({status: .[0].status, count: length}) | .[]' | column -t

# Example output:
# status        count
# closed        15
# in_progress   3
# blocked       2
# todo          8
```

### View Recent Activity

```bash
# Show recently closed issues (last 24 hours)
echo ""
echo "Recently Completed:"
echo "==================="
bd list --json | jq -r '.[] | select(.closed_at > "'$(date -d '24 hours ago' -Iseconds)'") | "✓ \(.id): \(.title) (closed: \(.closed_at))"'

# Show recently updated issues
echo ""
echo "Recently Modified:"
echo "=================="
bd list --json | jq -r '.[] | select(.updated_at > "'$(date -d '24 hours ago' -Iseconds)'") | "→ \(.id): \(.title) (updated: \(.updated_at))"'
```

---

## Step 3: Identify Your Work

### Find Issues Assigned to You

```bash
YOUR_AGENT="[your-agent-name]"  # e.g., atom-writer, molecule-composer

echo "Your Assignments ($YOUR_AGENT):"
echo "==============================="

# All your issues
bd list --assignee $YOUR_AGENT --json | jq -r '.[] | "\(.status) | \(.id): \(.title)"' | column -t -s '|'

# Just your in-progress work
echo ""
echo "Your In-Progress Work:"
bd list --assignee $YOUR_AGENT --status in_progress --json | jq -r '.[] | "\(.id): \(.title)"'

# Your completed work
echo ""
echo "Your Completed Work:"
bd list --assignee $YOUR_AGENT --status closed --json | jq -r '.[] | "\(.id): \(.title)"'
```

---

## Step 4: Recover In-Progress Context

For each in-progress issue, recover full context.

### Get Issue Details

```bash
CURRENT_ISSUE="bd-xxxx"  # From Step 3

# Full issue details
bd show $CURRENT_ISSUE

# Key fields to examine:
# - title: What is this issue?
# - description: Full requirements
# - status: in_progress, blocked, todo?
# - assignee: Who owns it?
# - notes: Any notes from previous sessions?
# - parent: What is this a child of?
# - dependencies: What blocks this? What does this block?
```

### Read Issue Notes/History

```bash
# Extract notes (previous session may have left guidance)
bd show $CURRENT_ISSUE | grep -A 20 "notes:"

# Example note from previous session:
# "Session ended. Next steps: Implement password validation atom.
#  Progress: Email validator done, tests passing."
```

### Understand Dependencies

```bash
# What blocks this issue?
bd show $CURRENT_ISSUE | grep "blocked_by:"

# If blocked:
BLOCKER_ID=$(bd show $CURRENT_ISSUE | grep "blocked_by:" | awk '{print $2}')
bd show $BLOCKER_ID  # Understand what's blocking you

# What does this issue block?
bd list --json | jq -r '.[] | select(.blocked_by=="'$CURRENT_ISSUE'") | "\(.id): \(.title)"'

# If this blocks other work, prioritize completing it
```

---

## Step 5: Understand the Dependency Graph

### View Full Tree

```bash
# Find the epic (parent of all work)
EPIC_ID=$(bd list --json | jq -r '.[] | select(.type=="epic") | .id' | head -1)

# View full dependency tree
bd dep tree $EPIC_ID

# Shows hierarchical structure:
# Epic
#   ├─ Database Organism [closed]
#   │   └─ Database Molecule [closed]
#   │       ├─ Email Atom [closed]
#   │       ├─ Password Atom [in_progress] ← YOU ARE HERE
#   │       └─ Phone Atom [todo]
#   ├─ API Organism [blocked by: Password Atom]
#   └─ ...
```

### Identify Current Phase

Based on what's closed/open, determine where the project is in the atomic workflow:

```bash
# Check atom status
ATOMS_DONE=$(bd list --tag atom --status closed --json | jq -r 'length')
ATOMS_TOTAL=$(bd list --tag atom --json | jq -r 'length')
echo "Atoms: $ATOMS_DONE / $ATOMS_TOTAL complete"

# Check molecule status
MOLECULES_DONE=$(bd list --tag molecule --status closed --json | jq -r 'length')
MOLECULES_TOTAL=$(bd list --tag molecule --json | jq -r 'length')
echo "Molecules: $MOLECULES_DONE / $MOLECULES_TOTAL complete"

# Check organism status
ORGANISMS_DONE=$(bd list --tag organism --status closed --json | jq -r 'length')
ORGANISMS_TOTAL=$(bd list --tag organism --json | jq -r 'length')
echo "Organisms: $ORGANISMS_DONE / $ORGANISMS_TOTAL complete"

# Conclusion: Still in atom phase, molecule phase, or organism phase?
```

---

## Step 6: Discover Discovered Work

### Find Issues Created via discovered-from

```bash
# Show work that was discovered during previous implementation
echo "Discovered Work:"
echo "================"
bd list --json | jq -r '.[] | select(.discovered_from != null) | "\(.id): \(.title) (discovered from: \(.discovered_from))"'

# Understand WHY each was discovered
for ID in $(bd list --json | jq -r '.[] | select(.discovered_from != null) | .id'); do
  echo ""
  echo "Issue: $ID"
  bd show $ID | grep -A 5 "description:"
  echo "---"
done
```

This reveals unexpected work found during implementation—critical context that would be lost without beads.

---

## Step 7: Identify Ready Work

### What Can Be Done Now?

```bash
# View all unblocked work
bd ready

# Filter by your role
bd ready --assignee $YOUR_AGENT

# Filter by atomic level
bd ready --tag atom        # Atoms (no dependencies)
bd ready --tag molecule    # Molecules (atoms done)
bd ready --tag organism    # Organisms (molecules done)
```

**Decision Point:**
1. **If you have in-progress work** → Continue it (from Step 4)
2. **If in-progress work is blocked** → Work on the blocker instead
3. **If no in-progress work** → Pick from ready queue

---

## Step 8: Understand What Changed Since Last Session

### Find Modified Issues

```bash
# Issues updated in last session (e.g., last 6 hours)
SINCE=$(date -d '6 hours ago' -Iseconds)

echo "Changes Since $SINCE:"
echo "====================="
bd list --json | jq -r '.[] | select(.updated_at > "'$SINCE'") | "\(.updated_at) | \(.id): \(.title) | \(.status)"' | sort | column -t -s '|'

# Read details of each changed issue to understand what happened
```

### Identify New Issues

```bash
# Issues created recently
echo ""
echo "Recently Created Issues:"
echo "========================"
bd list --json | jq -r '.[] | select(.created_at > "'$SINCE'") | "\(.created_at) | \(.id): \(.title)"' | sort | column -t -s '|'
```

---

## Step 9: Reconstruct Implementation Intent

### Read Parent Context

```bash
# If working on a molecule, read parent organism
PARENT_ID=$(bd show $CURRENT_ISSUE | grep "parent:" | awk '{print $2}')

echo "Parent Context:"
echo "==============="
bd show $PARENT_ID

# If working on an atom, read molecule (parent) and organism (grandparent)
GRANDPARENT_ID=$(bd show $PARENT_ID | grep "parent:" | awk '{print $2}')
echo ""
echo "Grandparent Context:"
echo "===================="
bd show $GRANDPARENT_ID
```

### Read Related Issues

```bash
# Issues related to current work
echo ""
echo "Related Work:"
echo "============="
bd list --related $CURRENT_ISSUE --json | jq -r '.[] | "\(.id): \(.title) (\(.status))"'
```

### Connect to Spec

```bash
# Read the epic description (links to spec)
SPEC_LINK=$(bd show $EPIC_ID | grep "spec.md")

echo ""
echo "Original Spec:"
echo "=============="
echo "$SPEC_LINK"
echo ""
echo "Review spec.md for full context if needed."
```

---

## Step 10: Build Execution Plan

Based on recovered context, create a plan for this session.

### Summary Template

```bash
echo "Context Recovery Complete"
echo "========================="
echo ""
echo "Project: $(bd show $EPIC_ID | grep title | cut -d: -f2)"
echo "Phase: [Atoms|Molecules|Organisms|Testing|Integration]"
echo ""
echo "Your current work:"
echo "- Issue: $CURRENT_ISSUE"
echo "- Title: $(bd show $CURRENT_ISSUE | grep title | cut -d: -f2)"
echo "- Status: $(bd show $CURRENT_ISSUE | grep status | cut -d: -f2)"
echo "- Blockers: $(bd show $CURRENT_ISSUE | grep blocked_by | cut -d: -f2)"
echo ""
echo "Ready work:"
bd ready --assignee $YOUR_AGENT --count 3
echo ""
echo "Next steps:"
echo "1. [Based on status: continue in-progress, start ready work, or unblock]"
echo "2. [Follow atomic-workflow.md for implementation guidance]"
echo "3. [Write tests, run tests, close issue when done]"
```

---

## Agent-Specific Recovery Patterns

### Atom Writer Recovery

```bash
# Find your atom work
bd list --assignee atom-writer --status in_progress

# Atoms should have minimal context—just implement pure function
# Read issue description, implement, test, close
```

### Molecule Composer Recovery

```bash
# Find your molecule work
bd list --assignee molecule-composer --status in_progress

# Check which atoms are available (closed)
bd list --tag atom --status closed

# Use those atoms in your molecule composition
```

### Organism Builder Recovery

```bash
# Find your organism work
bd list --assignee database-layer-builder --status in_progress  # or api-layer-builder, ui-component-builder

# Check which molecules are available
bd list --tag molecule --status closed

# Check parent organism for layer requirements
PARENT=$(bd show $CURRENT_ISSUE | grep parent | awk '{print $2}')
bd show $PARENT | grep -A 20 "description:"
```

### Integration Assembler Recovery

```bash
# Check if all organisms are complete
bd list --tag organism --json | jq -r '.[] | "\(.status) | \(.id): \(.title)"' | column -t -s '|'

# All should be "closed" before integration starts
# If any blocked: Work on blockers first
```

---

## Success Criteria

✅ Agent understands overall project status (closed/in-progress/blocked/todo counts)
✅ Agent finds their assigned work and its current state
✅ Agent reads notes from previous sessions
✅ Agent understands dependency graph and current phase
✅ Agent knows what's blocking their work (if anything)
✅ Agent identifies ready work in their queue
✅ Agent reconstructs parent/related context for current issue
✅ Agent has clear next steps without re-reading entire codebase

This workflow prevents agent amnesia and enables seamless continuation across sessions.
