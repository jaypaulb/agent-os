# BV Time-Travel Diff Tracking

Track changes in beads issues across sessions, commits, or time periods using BV's time-travel diff capabilities.

## Overview

BV's time-travel diff feature allows agents and users to answer questions like:
- "What changed during this work session?"
- "Were new cycles introduced since last commit?"
- "How many issues closed since yesterday?"
- "What's the progress since last build?"

This is particularly valuable for:
- Session summaries (show what was accomplished)
- Regression detection (did we introduce cycles?)
- Progress tracking (velocity metrics)
- Auto-build harness integration (track changes between builds)

## Use Cases

### 1. Session Summary

Show what changed during a work session:

```bash
# At start of session, record starting point
git rev-parse HEAD > .beads/session-start-commit

# ... work happens ...

# At end of session, generate summary
SESSION_START=$(cat .beads/session-start-commit)
bv --robot-diff --diff-since "$SESSION_START" --format json | jq -r '
    "=== Work Session Summary ===",
    "",
    "Completed:",
    "  • \(.changes.closed_issues | length) issues closed",
    "",
    (.changes.closed_issues[] | "  ✓ \(.id): \(.title)"),
    "",
    "New work discovered:",
    "  • \(.changes.new_issues | length) issues created",
    "",
    "Modified:",
    "  • \(.changes.modified_issues | length) issues updated"
'
```

### 2. Regression Detection

Check if new cycles were introduced:

```bash
# Before risky operation
CYCLES_BEFORE=$(bv --robot-insights --format json | jq -r '.cycles | length')

# ... operation that modifies dependencies ...

# After operation
CYCLES_AFTER=$(bv --robot-insights --format json | jq -r '.cycles | length')

if [[ "$CYCLES_AFTER" -gt "$CYCLES_BEFORE" ]]; then
    echo "❌ New cycles introduced! Rolling back..."

    # Show the new cycles
    bv --robot-diff --diff-since HEAD~1 --format json | jq -r '
        .graph_changes.new_cycles[] | "  Cycle: " + (. | join(" → "))
    '

    exit 1
fi
```

### 3. Progress Tracking

Track progress over time periods:

```bash
# Changes in last 24 hours
bv --robot-diff --diff-since "24h" --format json | jq -r '
    "Progress (last 24 hours):",
    "  Closed: \(.changes.closed_issues | length)",
    "  Created: \(.changes.new_issues | length)",
    "  Active: \(.changes.modified_issues | length)"
'

# Changes since specific date
bv --robot-diff --diff-since "2025-01-14" --format json

# Changes since last release tag
bv --robot-diff --diff-since "v1.0.0" --format json
```

### 4. Auto-Build Harness Integration

Track changes between autonomous build runs:

```bash
# At start of build loop
LAST_BUILD_COMMIT=$(cat .beads/last-build-commit 2>/dev/null || echo "HEAD~1")

echo "Checking changes since last build ($LAST_BUILD_COMMIT)..."
DIFF=$(bv --robot-diff --diff-since "$LAST_BUILD_COMMIT" --format json)

# Log summary
NEW_ISSUES=$(echo "$DIFF" | jq -r '.changes.new_issues | length')
CLOSED_ISSUES=$(echo "$DIFF" | jq -r '.changes.closed_issues | length')
NEW_CYCLES=$(echo "$DIFF" | jq -r '.graph_changes.new_cycles | length')

echo "Changes detected:"
echo "  New issues: $NEW_ISSUES"
echo "  Closed issues: $CLOSED_ISSUES"
echo "  New cycles: $NEW_CYCLES"

# Warn if cycles introduced
if [[ "$NEW_CYCLES" -gt 0 ]]; then
    echo "⚠️  WARNING: New dependency cycles detected!"
    echo "$DIFF" | jq -r '.graph_changes.new_cycles[] | "  " + (. | join(" → "))'

    # Optionally block build
    read -p "Continue despite cycles? [y/N]: " continue_build
    [[ ! "$continue_build" =~ ^[Yy]$ ]] && exit 1
fi

# Store current commit for next diff
git rev-parse HEAD > .beads/last-build-commit
```

## Basic Usage

### Diff Since Last Commit

```bash
# Show what changed since last commit
bv --robot-diff --diff-since HEAD~1 --format json
```

### Diff Since Specific Time

```bash
# Changes in last 24 hours
bv --robot-diff --diff-since "24h" --format json

# Changes in last week
bv --robot-diff --diff-since "7d" --format json

# Changes since yesterday
bv --robot-diff --diff-since "2025-01-14" --format json
```

### Diff Between Tags

```bash
# Changes between releases
bv --robot-diff --diff-since v1.0.0 --format json

# Changes between branches
git checkout feature-branch
bv --robot-diff --diff-since main --format json
```

## Output Format

```json
{
  "generated_at": "2025-01-15T10:30:00Z",
  "diff_since": "HEAD~1",
  "changes": {
    "new_issues": [
      {
        "id": "bd-x1",
        "title": "Add email validation",
        "created_at": "2025-01-15T09:00:00Z",
        "priority": 1,
        "tags": ["atom"],
        "assignee": "atom-writer"
      }
    ],
    "closed_issues": [
      {
        "id": "bd-y1",
        "title": "User authentication",
        "closed_at": "2025-01-15T10:00:00Z",
        "closed_by": "implementer",
        "reason": "Implemented and tested"
      }
    ],
    "reopened_issues": [],
    "removed_issues": [],
    "modified_issues": [
      {
        "id": "bd-z1",
        "title": "Password validation",
        "changes": {
          "status": {"old": "open", "new": "in_progress"},
          "priority": {"old": 3, "new": 1},
          "assignee": {"old": null, "new": "atom-writer"}
        }
      }
    ]
  },
  "graph_changes": {
    "new_cycles": [],
    "resolved_cycles": [],
    "density_change": -0.003
  },
  "metrics_changes": {
    "pagerank": {
      "bd-z1": {"old": 0.05, "new": 0.12}
    },
    "betweenness": {
      "bd-z1": {"old": 0.0, "new": 0.25}
    }
  }
}
```

## Integration Points

### Auto-Build Command

**File:** `/commands/auto-build/auto-build.md`

**Location:** At start of build loop

```bash
# Track changes since last build
if command -v bv &> /dev/null; then
    LAST_BUILD_COMMIT=$(cat .beads/last-build-commit 2>/dev/null || echo "HEAD~1")

    echo "Checking changes since last build ($LAST_BUILD_COMMIT)..."
    DIFF=$(bv --robot-diff --diff-since "$LAST_BUILD_COMMIT" --format json)

    # Log summary
    NEW_ISSUES=$(echo "$DIFF" | jq -r '.changes.new_issues | length')
    CLOSED_ISSUES=$(echo "$DIFF" | jq -r '.changes.closed_issues | length')
    NEW_CYCLES=$(echo "$DIFF" | jq -r '.graph_changes.new_cycles | length')

    echo "Changes detected:"
    echo "  New issues: $NEW_ISSUES"
    echo "  Closed issues: $CLOSED_ISSUES"
    echo "  New cycles: $NEW_CYCLES"

    # Warn if cycles introduced
    if [[ "$NEW_CYCLES" -gt 0 ]]; then
        echo "⚠️  WARNING: New dependency cycles detected!"
        echo "$DIFF" | jq -r '.graph_changes.new_cycles[] | "  " + (. | join(" → "))'

        # Optionally block build
        read -p "Continue despite cycles? [y/N]: " continue_build
        [[ ! "$continue_build" =~ ^[Yy]$ ]] && exit 1
    fi

    # Store current commit for next diff
    git rev-parse HEAD > .beads/last-build-commit
fi
```

### Session Summary at End

**File:** `/workflows/implementation/implement-with-beads.md`

**Location:** At end of implementation session

```bash
# Generate session summary using diff
if command -v bv &> /dev/null; then
    SESSION_START=$(cat .beads/session-start-commit 2>/dev/null || echo "HEAD~1")

    echo ""
    echo "=== Session Summary ==="
    bv --robot-diff --diff-since "$SESSION_START" --format json | jq -r '
        "Work completed:",
        "  • \(.changes.closed_issues | length) issues closed",
        "  • \(.changes.new_issues | length) new issues discovered",
        "  • \(.changes.modified_issues | length) issues updated",
        "",
        "Closed issues:",
        (.changes.closed_issues[] | "  ✓ \(.id): \(.title)"),
        "",
        (.changes.new_issues | if length > 0 then
            "Discovered issues:",
            (.[] | "  + \(.id): \(.title)")
        else "" end)
    '

    # Save current commit for next session
    git rev-parse HEAD > .beads/session-start-commit
fi
```

### Implementation Commands

**File:** `/commands/implement-tasks/single-agent/2-implement-tasks.md`

**Location:** After completing all work

```bash
# Display session summary
if command -v bv &> /dev/null; then
    echo ""
    echo "=== Implementation Summary ==="

    # Get diff since session start
    SESSION_START=$(cat .beads/session-start-commit 2>/dev/null || echo "HEAD~1")
    DIFF=$(bv --robot-diff --diff-since "$SESSION_START" --format json)

    # Display accomplishments
    echo "$DIFF" | jq -r '
        "Closed: \(.changes.closed_issues | length) issues",
        "Created: \(.changes.new_issues | length) issues",
        "Modified: \(.changes.modified_issues | length) issues"
    '

    # Warn if cycles introduced
    NEW_CYCLES=$(echo "$DIFF" | jq -r '.graph_changes.new_cycles | length')
    if [[ "$NEW_CYCLES" -gt 0 ]]; then
        echo ""
        echo "⚠️  WARNING: $NEW_CYCLES new cycles introduced during this session"
        echo "$DIFF" | jq -r '.graph_changes.new_cycles[] | "  " + (. | join(" → "))'
    fi
fi
```

## Harness Integration Notes

The autonomous coding harness (separate repo) can leverage time-travel diffs for:

- **Build-to-build tracking**: What changed between autonomous build runs?
- **Regression detection**: Did the harness introduce circular dependencies?
- **Progress metrics**: Track velocity (issues closed per build cycle)
- **Automatic rollback**: If regression detected, revert to last good state

Implementation details are in the harness repo, not agent-os. Agent-os provides the diff tracking foundation.

## Helper Functions

These functions are already available in `bv-helpers.md`:

```bash
# Get diff since reference
DIFF=$(get_session_diff "HEAD~1")
DIFF=$(get_session_diff "24h")
DIFF=$(get_session_diff "v1.0.0")

# Extract specific changes
CLOSED=$(echo "$DIFF" | jq -r '.changes.closed_issues')
NEW=$(echo "$DIFF" | jq -r '.changes.new_issues')
MODIFIED=$(echo "$DIFF" | jq -r '.changes.modified_issues')
CYCLES=$(echo "$DIFF" | jq -r '.graph_changes.new_cycles')
```

## Fallback Behavior

If BV unavailable, diff tracking returns empty:

```json
{
  "generated_at": null,
  "diff_since": null,
  "changes": {
    "new_issues": [],
    "closed_issues": [],
    "reopened_issues": [],
    "removed_issues": [],
    "modified_issues": []
  },
  "graph_changes": {
    "new_cycles": [],
    "resolved_cycles": [],
    "density_change": 0
  },
  "metrics_changes": {}
}
```

Session summaries gracefully degrade - no hard failures.

## Advanced Usage

### Compare Metrics Over Time

```bash
# Get diff with metrics changes
DIFF=$(bv --robot-diff --diff-since "7d" --format json)

# Find issues with biggest PageRank increase
echo "$DIFF" | jq -r '
    .metrics_changes.pagerank |
    to_entries |
    map({id: .key, change: (.value.new - .value.old)}) |
    sort_by(-.change) |
    .[:5] |
    .[] | "\(.id): PageRank +\(.change | . * 100 | round / 100)"
'
```

### Track Density Changes

```bash
# Monitor graph density over time
DIFF=$(bv --robot-diff --diff-since "HEAD~5" --format json)
DENSITY_CHANGE=$(echo "$DIFF" | jq -r '.graph_changes.density_change')

if (( $(echo "$DENSITY_CHANGE > 0.1" | bc -l) )); then
    echo "⚠️  Graph density increased by $DENSITY_CHANGE"
    echo "   Adding many dependencies - consider simplifying"
fi
```

### Detect Resolved Cycles

```bash
# Find cycles that were fixed
DIFF=$(bv --robot-diff --diff-since "HEAD~1" --format json)
RESOLVED=$(echo "$DIFF" | jq -r '.graph_changes.resolved_cycles | length')

if [[ "$RESOLVED" -gt 0 ]]; then
    echo "✓ Fixed $RESOLVED circular dependencies!"
    echo "$DIFF" | jq -r '.graph_changes.resolved_cycles[]'
fi
```

## Best Practices

1. **Record session start** - Always mark session start with `git rev-parse HEAD > .beads/session-start-commit`
2. **Commit frequently** - More commits = finer-grained diffs
3. **Check before risky operations** - Record state before bulk dependency changes
4. **Include in CI/CD** - Run diff checks in build pipelines
5. **Monitor cycle introduction** - Treat new cycles as build failures

## Troubleshooting

### "No changes detected" but work was done

**Causes:**
- Changes not committed to git
- Working in detached HEAD state
- `.beads/issues.jsonl` not committed

**Solution:** Commit beads changes to git before running diff

### Metrics changes show "null"

**Causes:**
- Graph too small for meaningful metrics
- No dependencies (flat structure)

**Solution:** Normal for small or flat graphs - metrics stabilize with ≥10 issues

### Diff performance is slow

**Causes:**
- Large git history
- Many beads issues (>1000)

**Solution:** Use recent reference points (`HEAD~5` not `v1.0.0`), or use date-based refs (`24h`)
