Now that you have the task group(s) to be implemented, proceed with implementation by following these instructions:

{{workflows/implementation/implement-tasks}}

## Display confirmation and next step

Display a summary of what was implemented.

{{IF tracking_mode_beads}}
Generate implementation summary using BV diff tracking:

```bash
cd agent-os/specs/[this-spec]/

# Source BV helpers
source ../../../workflows/implementation/bv-helpers.md

if bv_available; then
    echo ""
    echo "=== Implementation Summary ==="

    # Get diff since session start
    SESSION_START=$(cat .beads/session-start-commit 2>/dev/null || echo "HEAD~1")
    DIFF=$(get_session_diff "$SESSION_START")

    # Display accomplishments
    echo "$DIFF" | jq -r '
        "Closed: \(.changes.closed_issues | length) issues",
        "Created: \(.changes.new_issues | length) issues",
        "Modified: \(.changes.modified_issues | length) issues",
        "",
        "Closed issues:",
        (.changes.closed_issues[] | "  ✓ \(.id): \(.title)")
    '

    # Warn if cycles introduced
    NEW_CYCLES=$(echo "$DIFF" | jq -r '.graph_changes.new_cycles | length')
    if [[ "$NEW_CYCLES" -gt 0 ]]; then
        echo ""
        echo "⚠️  WARNING: $NEW_CYCLES new cycles introduced during this session"
        echo "$DIFF" | jq -r '.graph_changes.new_cycles[] | "  " + (. | join(" → "))'
    fi

    echo ""
fi
```

Check beads status to see if all work is complete:

```bash
cd agent-os/specs/[this-spec]/
bd list --json | jq -r '.[] | select(.status!="closed")'
```

IF all issues are closed, display:

```
All beads issues have been implemented and closed.

NEXT STEP 👉 Run verification to ensure everything works end-to-end.
```

IF there are still open issues, display:

```
Remaining open issues: [count]

Run `bd ready` to see unblocked work.

Would you like to continue with the remaining work?
```

{{ELSE}}
IF all tasks are now marked as done (with `- [x]`) in tasks.md, display this message to user:

```
All tasks have been implemented: `agent-os/specs/[this-spec]/tasks.md`.

NEXT STEP 👉 Run `3-verify-implementation.md` to verify the implementation.
```

IF there are still tasks in tasks.md that have yet to be implemented (marked unfinished with `- [ ]`) then display this message to user:

```
Would you like to proceed with implementation of the remaining tasks in tasks.md?

If not, please specify which task group(s) to implement next.
```
{{ENDIF tracking_mode_beads}}

{{UNLESS standards_as_claude_code_skills}}
## User Standards & Preferences Compliance

IMPORTANT: Ensure that the tasks list is ALIGNED and DOES NOT CONFLICT with the user's preferences and standards as detailed in the following files:

{{standards/*}}
{{ENDUNLESS standards_as_claude_code_skills}}
