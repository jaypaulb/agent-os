First, check if the user has already provided instructions about which task group(s) to implement.

**If the user HAS provided instructions:** Proceed to PHASE 2 to delegate implementation to the appropriate agent(s).

**If the user has NOT provided instructions:**

{{IF tracking_mode_beads}}
Query beads to find ready work using execution planning (from project root):

```bash
# Source BV helpers
source agent-os/workflows/implementation/bv-helpers.md

# Check if bv is available
if bv_available; then
    echo "Using BV execution planning..."
    echo ""

    # Get full execution plan
    PLAN=$(get_execution_plan)

    # Display execution plan with track information
    echo "$PLAN" | jq -r '
        "=== Execution Plan ===",
        "",
        "Parallel tracks available: \(.tracks | length)",
        "",
        (.tracks[] |
         "Track \(.track_id): \(.reason)",
         "  Priority: \(.items[0].priority // "N/A")",
         "  Items: \(.items | length)",
         "  Top item: \(.items[0].id) - \(.items[0].title)",
         "  Impact: Unblocks \(.items[0].unblocks_count) downstream tasks",
         "")
    '

    # Show recommended next work with reasoning
    echo ""
    echo "=== Recommended Next Task ==="
    echo "$PLAN" | jq -r '
        .tracks[0].items[0] |
        "Issue: \(.id)",
        "Title: \(.title)",
        "Priority: P\(.priority)",
        "Impact: Unblocks \(.unblocks_count) downstream tasks",
        "Reasoning: Highest impact unblocked work"
    '

    # Show parallelization opportunities
    echo ""
    TRACK_COUNT=$(echo "$PLAN" | jq -r '.tracks | length')
    if [[ "$TRACK_COUNT" -gt 1 ]]; then
        echo "⚡ Parallelization opportunity: $TRACK_COUNT independent work streams available"
        echo "   Multiple agents could work simultaneously on different tracks"
    fi

    # Quick recipe checks
    echo ""
    echo "=== Quick Views ==="
    ACTIONABLE=$(get_recipe actionable)
    HIGH_IMPACT=$(get_recipe high-impact)
    BLOCKED=$(get_recipe blocked)

    echo "Actionable items: $(echo "$ACTIONABLE" | jq -r '. | length')"
    echo "High-impact items: $(echo "$HIGH_IMPACT" | jq -r '. | length')"
    echo "Blocked items: $(echo "$BLOCKED" | jq -r '. | length')"
else
    echo "Using basic task selection (BV unavailable)..."
    echo ""
    bd ready
fi
```

Show the user the ready work and ask:

```
Ready work from execution plan:

[Display output from BV execution plan or bd ready]

Should we proceed with the recommended task, all ready work, or specify which issues to implement?
```

{{ELSE}}
Read `agent-os/specs/[this-spec]/tasks.md` to review the available task groups, then output the following message to the user and WAIT for their response:

```
Should we proceed with implementation of all task groups in tasks.md?

If not, then please specify which task(s) to implement.
```
{{ENDIF tracking_mode_beads}}
