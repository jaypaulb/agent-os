# BV Helper Functions

Reusable bash functions for BV detection and graceful fallback to basic beads commands.

## Purpose

These functions provide a consistent way to:
- Check if BV is available in the current spec
- Fall back gracefully to basic `bd` commands if BV is unavailable
- Abstract BV command execution with automatic fallback handling

## Usage

Source this file in workflows that use BV:

```bash
# Source BV helpers (adjust path as needed)
source "$(dirname "$0")/bv-helpers.md"

# Use helper functions
if bv_available; then
    PLAN=$(get_execution_plan)
    # ... use plan ...
fi
```

## Helper Functions

### bv_available()

Check if BV is available in the current spec.

**Parameters:**
- `$1` - Spec path (optional, defaults to current directory)

**Returns:**
- `0` if BV is available
- `1` if BV is not available

**Usage:**
```bash
if bv_available; then
    echo "BV is available - using graph intelligence"
else
    echo "BV unavailable - using basic beads commands"
fi

# Check specific spec
if bv_available "/path/to/spec"; then
    # ...
fi
```

**Implementation:**
```bash
bv_available() {
    local spec_path=${1:-.}

    # Check if bv command exists
    if ! command -v bv &> /dev/null; then
        return 1
    fi

    # Check spec config (if exists)
    if [[ -f "$spec_path/spec-config.yml" ]]; then
        local enabled=$(grep "bv_enabled:" "$spec_path/spec-config.yml" | cut -d: -f2 | xargs)
        [[ "$enabled" == "true" ]] && return 0 || return 1
    fi

    return 0  # Default to available if command exists
}
```

### get_execution_plan()

Get execution plan using BV or fall back to bd ready.

**Returns:**
- JSON execution plan (from `bv --robot-plan` or `bd ready`)

**Fallback behavior:**
- If BV available: Returns `bv --robot-plan --format json`
- If BV fails: Logs warning, returns `bd ready --format json`
- If BV unavailable: Returns `bd ready --format json`

**Usage:**
```bash
PLAN=$(get_execution_plan)
echo "$PLAN" | jq -r '.tracks[] | .track_id'
```

**Implementation:**
```bash
get_execution_plan() {
    if bv_available; then
        bv --robot-plan --format json 2>/dev/null || {
            echo "⚠️  BV plan failed, falling back to bd ready" >&2
            bd ready --format json
        }
    else
        bd ready --format json
    fi
}
```

### get_priority_recommendations()

Get priority recommendations from BV.

**Returns:**
- JSON priority recommendations (from `bv --robot-priority`)
- Empty recommendations object if BV unavailable

**Usage:**
```bash
RECS=$(get_priority_recommendations)
echo "$RECS" | jq -r '.recommendations[] | "\(.issue_id): P\(.suggested_priority)"'
```

**Implementation:**
```bash
get_priority_recommendations() {
    if bv_available; then
        bv --robot-priority --format json 2>/dev/null || {
            echo "⚠️  BV priority analysis unavailable" >&2
            echo '{"recommendations": [], "summary": {"total_issues": 0, "recommendations": 0}}'
        }
    else
        echo '{"recommendations": [], "summary": {"total_issues": 0, "recommendations": 0}}'
    fi
}
```

### get_session_diff()

Get diff since last session or specified reference.

**Parameters:**
- `$1` - Git reference to diff since (optional, defaults to HEAD~1)

**Returns:**
- JSON diff object (from `bv --robot-diff`)
- Empty diff object if BV unavailable

**Usage:**
```bash
# Diff since last commit
DIFF=$(get_session_diff)

# Diff since specific time
DIFF=$(get_session_diff "24h")

# Diff since specific commit
DIFF=$(get_session_diff "HEAD~3")

echo "$DIFF" | jq -r '.changes.closed_issues[] | .id'
```

**Implementation:**
```bash
get_session_diff() {
    local since="${1:-HEAD~1}"

    if bv_available; then
        bv --robot-diff --diff-since "$since" --format json 2>/dev/null || {
            echo "⚠️  BV diff unavailable" >&2
            echo '{"changes": {}, "graph_changes": {}, "metrics_changes": {}}'
        }
    else
        echo '{"changes": {}, "graph_changes": {}, "metrics_changes": {}}'
    fi
}
```

### get_graph_insights()

Get graph insights (bottlenecks, keystones, influencers).

**Returns:**
- JSON insights object (from `bv --robot-insights`)
- Empty insights object if BV unavailable

**Usage:**
```bash
INSIGHTS=$(get_graph_insights)
echo "$INSIGHTS" | jq -r '.bottlenecks[:5] | .[] | "\(.id) (betweenness: \(.value))"'
```

**Implementation:**
```bash
get_graph_insights() {
    if bv_available; then
        bv --robot-insights --format json 2>/dev/null || {
            echo "⚠️  BV insights unavailable" >&2
            echo '{
                "bottlenecks": [],
                "keystones": [],
                "influencers": [],
                "hubs": [],
                "authorities": [],
                "cycles": [],
                "clusterDensity": 0
            }'
        }
    else
        echo '{
            "bottlenecks": [],
            "keystones": [],
            "influencers": [],
            "hubs": [],
            "authorities": [],
            "cycles": [],
            "clusterDensity": 0
        }'
    fi
}
```

### check_cycles()

Check for circular dependencies in the graph.

**Returns:**
- `0` if no cycles detected
- `1` if cycles detected

**Side effects:**
- Prints cycle paths to stderr if cycles detected

**Usage:**
```bash
if ! check_cycles; then
    echo "❌ Circular dependencies detected - fix before proceeding"
    exit 1
fi
```

**Implementation:**
```bash
check_cycles() {
    if bv_available; then
        local cycles=$(bv --robot-insights --format json 2>/dev/null | jq -r '.cycles')
        local cycle_count=$(echo "$cycles" | jq '. | length')

        if [[ "$cycle_count" -gt 0 ]]; then
            echo "⚠️  WARNING: Circular dependencies detected!" >&2
            echo "$cycles" | jq -r '.[] | "  Cycle: " + (. | join(" → "))' >&2
            return 1
        fi
    fi

    return 0
}
```

### get_recipe()

Get filtered view using BV recipe system.

**Parameters:**
- `$1` - Recipe name (actionable, high-impact, blocked, stale, recent)

**Returns:**
- JSON array of issues matching recipe
- Empty array if BV unavailable

**Usage:**
```bash
# Get actionable work
ACTIONABLE=$(get_recipe actionable)
echo "$ACTIONABLE" | jq -r '.[] | .id'

# Get high-impact work
HIGH_IMPACT=$(get_recipe high-impact)
```

**Implementation:**
```bash
get_recipe() {
    local recipe_name="$1"

    if bv_available; then
        bv --recipe "$recipe_name" --format json 2>/dev/null || {
            echo "⚠️  BV recipe '$recipe_name' unavailable" >&2
            echo '[]'
        }
    else
        echo '[]'
    fi
}
```

### list_recipes()

List available BV recipes.

**Returns:**
- JSON array of recipe definitions
- Empty array if BV unavailable

**Usage:**
```bash
RECIPES=$(list_recipes)
echo "$RECIPES" | jq -r '.recipes[] | "• \(.name): \(.description)"'
```

**Implementation:**
```bash
list_recipes() {
    if bv_available; then
        bv --robot-recipes --format json 2>/dev/null || {
            echo "⚠️  BV recipe list unavailable" >&2
            echo '{"recipes": []}'
        }
    else
        echo '{"recipes": []}'
    fi
}
```

## Fallback Behavior Summary

All helper functions implement graceful degradation:

| Function | BV Available | BV Unavailable | BV Command Fails |
|----------|--------------|----------------|------------------|
| `bv_available()` | Returns 0 | Returns 1 | Returns 1 |
| `get_execution_plan()` | BV plan JSON | BD ready JSON | BD ready JSON (logged warning) |
| `get_priority_recommendations()` | BV priority JSON | Empty recommendations | Empty recommendations (logged warning) |
| `get_session_diff()` | BV diff JSON | Empty diff | Empty diff (logged warning) |
| `get_graph_insights()` | BV insights JSON | Empty insights | Empty insights (logged warning) |
| `check_cycles()` | Checks cycles, returns 0/1 | Returns 0 (permissive) | Returns 0 (logged warning) |
| `get_recipe()` | BV recipe JSON | Empty array | Empty array (logged warning) |
| `list_recipes()` | BV recipes JSON | Empty array | Empty array (logged warning) |

## Error Handling

- All functions that call BV use `2>/dev/null` to suppress stderr noise from BV
- Warnings are logged to stderr using `>&2` when fallback occurs
- Functions never fail hard - they always return valid (possibly empty) JSON
- This ensures workflows continue even if BV is unavailable or malfunctions

## Integration Pattern

Typical workflow integration:

```bash
#!/bin/bash

# Source helpers
source "$(dirname "$0")/bv-helpers.md"

# Check availability
if bv_available; then
    echo "Using BV graph intelligence"

    # Get execution plan
    PLAN=$(get_execution_plan)
    NEXT_TASK=$(echo "$PLAN" | jq -r '.tracks[0].items[0].id')

    # Check for cycles
    if ! check_cycles; then
        echo "Fix cycles before proceeding"
        exit 1
    fi

    # Get priority recommendations
    RECS=$(get_priority_recommendations)
    HIGH_PRIORITY=$(echo "$RECS" | jq -r '.recommendations[] | select(.confidence > 0.8) | .issue_id')

    echo "Recommended task: $NEXT_TASK"
else
    echo "BV unavailable - using basic task selection"

    # Fallback to basic bd commands
    NEXT_TASK=$(bd ready --limit 1 | head -1)
    echo "Next task: $NEXT_TASK"
fi

# Proceed with selected task
bd update "$NEXT_TASK" --status in_progress
```

## Notes

- These functions are designed to be sourced, not executed directly
- All functions use `local` variables to avoid polluting the global namespace
- JSON output is always valid, even in error cases (enables safe `jq` piping)
- Warnings go to stderr (`>&2`) so they don't pollute JSON output on stdout
