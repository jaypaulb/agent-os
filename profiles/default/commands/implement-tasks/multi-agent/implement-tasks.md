# Spec Implementation Process

Now that we have a spec and tasks list ready for implementation, we will proceed with implementation of this spec by following this multi-phase process:

PHASE 1: Determine which task group(s) from tasks.md should be implemented
PHASE 2: Delegate implementation to the implementer subagent
PHASE 3: After ALL task groups have been implemented, delegate to implementation-verifier to produce the final verification report.

Follow each of these phases and their individual workflows IN SEQUENCE:

## Multi-Phase Process

### PHASE 1: Determine which task group(s) to implement

First, check if the user has already provided instructions about which task group(s) to implement.

**If the user HAS provided instructions:** Proceed to PHASE 2 to delegate implementation.

**If the user has NOT provided instructions:**

{{IF tracking_mode_beads}}
Query beads to find ready work for this spec:

```bash
# From project root (where .beads/ is located)

# Source BV helpers
source agent-os/workflows/implementation/bv-helpers.md

# Get spec name
SPEC_NAME="[this-spec-slug]"

# Get ready work for this spec using BV (or fall back to bd)
if bv_available; then
    echo "Finding ready work for $SPEC_NAME using BV..."
    READY=$(bv --unblocked --filter "label:$SPEC_NAME" --format json 2>/dev/null || \
            bd ready --label "$SPEC_NAME" --json)
else
    echo "Finding ready work for $SPEC_NAME using bd..."
    READY=$(bd ready --label "$SPEC_NAME" --json)
fi

# Display ready work
echo "$READY" | jq -r '.[] | "  • \(.id): \(.title) (P\(.priority // "?"))"'
```

Show the user the ready work and ask:

```
Ready work from beads for [this-spec]:

[List ready issues from above output]

Should we proceed with all ready work, or specify which issues to implement?
```

{{ELSE}}
Read `agent-os/specs/[this-spec]/tasks.md` to review the available task groups, then output the following message to the user and WAIT for their response:

```
Should we proceed with implementation of all task groups in tasks.md?

If not, then please specify which task(s) to implement.
```
{{ENDIF tracking_mode_beads}}

### PHASE 2: Delegate implementation to the implementer subagent

Delegate to the **implementer** subagent to implement the specified task group(s):

Provide to the subagent:
- The specific task group(s) from `agent-os/specs/[this-spec]/tasks.md` including the parent task, all sub-tasks, and any sub-bullet points
- The path to this spec's documentation: `agent-os/specs/[this-spec]/spec.md`
- The path to this spec's requirements: `agent-os/specs/[this-spec]/planning/requirements.md`
- The path to this spec's visuals (if any): `agent-os/specs/[this-spec]/planning/visuals`

Instruct the subagent to:
1. Analyze the provided spec.md, requirements.md, and visuals (if any)
2. Analyze patterns in the codebase according to its built-in workflow
3. Implement the assigned task group according to requirements and standards
4. Update `agent-os/specs/[this-spec]/tasks.md` to mark completed tasks with `- [x]`

### PHASE 3: Produce the final verification report

{{IF tracking_mode_beads}}
Check if ALL beads issues for this spec are closed:

```bash
# From project root

# Get spec name
SPEC_NAME="[this-spec-slug]"

# Check for open issues in this spec
OPEN_ISSUES=$(bd list --label "$SPEC_NAME" --json | jq -r '.[] | select(.status!="closed") | .id')

if [ -z "$OPEN_ISSUES" ]; then
    echo "✓ All issues for $SPEC_NAME are closed"
else
    echo "⚠️  Open issues remaining for $SPEC_NAME:"
    echo "$OPEN_ISSUES"
    echo ""
    echo "Return to PHASE 1 to continue implementation"
    exit 0
fi
```

IF ALL issues for this spec are closed, then proceed with this step. Otherwise, return to PHASE 1.

Assuming all issues are closed, then delegate to the **implementation-verifier** subagent to do its implementation verification and produce its final verification report.

{{ELSE}}
IF ALL task groups in tasks.md are marked complete with `- [x]`, then proceed with this step. Otherwise, return to PHASE 1.

Assuming all tasks are marked complete, then delegate to the **implementation-verifier** subagent to do its implementation verification and produce its final verification report.
{{ENDIF tracking_mode_beads}}

Provide to the subagent the following:
- The path to this spec: `agent-os/specs/[this-spec]`
Instruct the subagent to do the following:
  1. Run all of its final verifications according to its built-in workflow
  2. Produce the final verification report in `agent-os/specs/[this-spec]/verifications/final-verification.md`.
