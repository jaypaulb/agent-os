# Autonomous Planning Loop

You are orchestrating a fully autonomous planning loop that will take a product vision through iterative spec development across multiple phases, culminating in a complete implementation plan.

This command automates the following workflow:
1. **Plan Product**: Create mission, roadmap, and tech stack
2. **Iterative Phase Loop**: For each phase in the roadmap, run shape-spec → write-spec → update plan-product
3. **Create Tasks**: Generate Beads issues with atomic design hierarchy

The loop is designed to be fully autonomous once started, with plan-product updated after each phase to incorporate learnings.

---

## PHASE 1: Initial Product Planning

Run the plan-product workflow to establish the foundation:

{{@agent-os/commands/plan-product/plan-product.md}}

Once complete, proceed immediately to PHASE 2.

---

## PHASE 2: Extract Phases from Roadmap

Read the roadmap file to identify all phases for iterative spec development:

```bash
cat agent-os/planning/roadmap.md
```

Parse the roadmap to extract phase titles. Typically these are organized as:
- Phase 1: [Phase Title]
- Phase 2: [Phase Title]
- Phase 3: [Phase Title]
- etc.

Create a list of phases to iterate through. Store this mentally for the loop.

---

## PHASE 3: Iterative Spec Development Loop

For EACH phase identified in the roadmap, execute the following sub-phases IN SEQUENCE:

### Sub-Phase A: Shape Spec for Current Phase

Run the shape-spec workflow for this specific phase:

{{@agent-os/commands/shape-spec/shape-spec.md}}

When initializing the spec, use the phase title as the spec name (e.g., "user-authentication" for "Phase 1: User Authentication").

### Sub-Phase B: Write Spec for Current Phase

Run the write-spec workflow for this specific phase:

{{@agent-os/commands/write-spec/write-spec.md}}

This creates `agent-os/specs/[phase-spec]/spec.md` with complete implementation details.

### Sub-Phase C: Update Plan-Product with Learnings

After completing the spec for this phase, update the plan-product documents to incorporate learnings:

1. **Read the newly created spec**:
   ```bash
   cat agent-os/specs/[phase-spec]/spec.md
   ```

2. **Identify key insights** that should inform subsequent phases:
   - Technical dependencies discovered
   - Architectural decisions made
   - Constraints or requirements clarified
   - Integration points with other phases

3. **Update roadmap.md** with refined details:
   - Add technical notes to this phase's entry
   - Update subsequent phase descriptions if dependencies changed
   - Clarify phase boundaries based on what was learned

4. **Update tech-stack.md** if new technical decisions were made:
   - Add libraries or frameworks chosen for this phase
   - Document integration patterns discovered
   - Note any tech stack adjustments for future phases

Example update to roadmap.md:
```markdown
### Phase 1: User Authentication ✓
**Status**: Spec Complete
**Technical Notes**:
- JWT chosen for session management
- Password hashing via bcrypt
- Requires database schema migration before Phase 2
- API endpoints: POST /auth/login, POST /auth/register, POST /auth/logout

**Integration Points**:
- Phase 2 (User Dashboard) depends on auth middleware from this phase
- Phase 3 (Profile Management) uses user context established here
```

### Sub-Phase D: Continue to Next Phase

Repeat Sub-Phases A, B, C for the next phase in the roadmap.

**Autonomous Loop Execution**:
- DO NOT ask for permission between phases
- Automatically proceed to the next phase after updating plan-product
- Only stop if all phases are complete or if an error occurs

---

## PHASE 4: Create Implementation Tasks

After ALL phases have been shaped, written, and plan-product has been updated, create the implementation task breakdown:

{{@agent-os/commands/create-tasks/create-tasks.md}}

{{IF tracking_mode_beads}}
This will automatically create Beads issues using the workflow defined in:
{{workflows/implementation/create-beads-issues.md}}

The result will be:
- `.beads/` directory initialized in each spec folder
- Epic, organism, molecule, and atom issues created with proper hierarchy
- Blocking dependencies set (atoms block molecules, molecules block organisms)
- Ready work queue populated (`bd ready` shows unblocked atoms)

**Note**: Since you've iterated through multiple phases, you may need to run create-tasks for EACH phase's spec folder, or create a combined implementation plan that sequences phases properly.

For a multi-phase implementation:
1. Create Beads issues for Phase 1
2. Set Phase 2's epic to be blocked by Phase 1's integration issue
3. Repeat for each subsequent phase to ensure sequential execution
{{ENDIF tracking_mode_beads}}

---

## PHASE 5: Output Summary

Display the following message to the user:

```
Autonomous planning loop complete!

**Product Planning**:
- Mission: agent-os/planning/mission.md
- Roadmap: agent-os/planning/roadmap.md
- Tech Stack: agent-os/planning/tech-stack.md

**Phases Completed**:
[List each phase with its spec location]
- Phase 1: [Phase Title] → agent-os/specs/[phase-spec]/spec.md
- Phase 2: [Phase Title] → agent-os/specs/[phase-spec]/spec.md
- Phase 3: [Phase Title] → agent-os/specs/[phase-spec]/spec.md
[etc.]

**Plan-Product Updates**:
The roadmap and tech stack were updated after each phase to incorporate learnings and refine subsequent phase requirements.

{{IF tracking_mode_beads}}
**Beads Issues Created**:
Total issues created across all phases:
[Count from all .beads/ directories]

Ready to start implementation:
```bash
cd agent-os/specs/[first-phase-spec]/
bd ready
```

**Next Step**: Run `/auto-build` to launch the autonomous coding harness
{{ELSE}}
**Tasks Created**:
tasks.md files created for each phase spec.

**Next Step**: Run `/orchestrate-tasks` to begin implementation
{{ENDIF tracking_mode_beads}}
```

---

## Autonomous Execution Notes

**This command is designed to run fully autonomously**:
- No user interaction required after initial product concept gathering
- Loops through all phases automatically
- Updates plan-product iteratively as context grows
- Handles multi-phase dependencies in Beads issue creation

**Expected Duration**:
- Depends on number of phases and complexity of each
- Typical range: 5-20 phases
- Each phase involves: shape (10-30 mins) + write (15-45 mins) + update (5-10 mins)

**Failure Handling**:
- If a phase fails to shape or write, STOP and report the error
- User can then provide clarification and resume from that phase
- Plan-product updates are cumulative, so no context is lost
