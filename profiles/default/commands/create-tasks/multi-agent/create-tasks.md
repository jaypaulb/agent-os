# Task List Creation Process

You are creating a tasks breakdown from a given spec and requirements for a new feature.

## PHASE 1: Get and read the spec.md and/or requirements document(s)

You will need ONE OR BOTH of these files to inform your tasks breakdown:
- `agent-os/specs/[this-spec]/spec.md`
- `agent-os/specs/[this-spec]/planning/requirements.md`

IF you don't have ONE OR BOTH of those files in your current conversation context, then ask user to provide direction on where to you can find them by outputting the following request then wait for user's response:

```
I'll need a spec.md or requirements.md (or both) in order to build a tasks list.

Please direct me to where I can find those.  If you haven't created them yet, you can run /shape-spec or /write-spec.
```

## PHASE 2: Create task breakdown

{{IF tracking_mode_beads}}
Once you have `spec.md` AND/OR `requirements.md`, auto-generate beads issues with atomic design hierarchy.

Follow the workflow at: `@agent-os/workflows/implementation/create-beads-issues.md`

This will create issues in `.beads/issues.jsonl` with:
- Parent epic for the feature
- Organism issues (database, API, UI layers)
- Molecule issues as children of organisms
- Atom issues as children of molecules
- Blocking dependencies (atoms block molecules, molecules block organisms)

{{ELSE}}
Once you have `spec.md` AND/OR `requirements.md`, use the **tasks-list-creator** subagent to break down the spec and requirements into an actionable tasks list with strategic grouping and ordering.

Provide the tasks-list-creator:
- `agent-os/specs/[this-spec]/spec.md` (if present)
- `agent-os/specs/[this-spec]/planning/requirements.md` (if present)
- `agent-os/specs/[this-spec]/planning/visuals/` and its' contents (if present)

The tasks-list-creator will create `tasks.md` inside the spec folder.
{{ENDIF tracking_mode_beads}}

## PHASE 3: Inform user

{{IF tracking_mode_beads}}
Output the following to inform the user:

```
Your beads issues are ready!

✅ Issues created in: `.beads/issues.jsonl`

View ready work: `bd ready`
View dependency tree: `bd dep tree [epic-id]`

NEXT STEP 👉 Run `/implement-tasks` or `/orchestrate-tasks` to start building!
- Agents will use `bd ready` to find unblocked work
- Start with atoms (no dependencies), then molecules, then organisms
```
{{ELSE}}
Output the following to inform the user:

```
Your tasks list is ready!

✅ Tasks list created: `agent-os/specs/[this-spec]/tasks.md`

NEXT STEP 👉 Run `/implement-tasks` (simple, effective) or `/orchestrate-tasks` (advanced, powerful) to start building!
```
{{ENDIF tracking_mode_beads}}
