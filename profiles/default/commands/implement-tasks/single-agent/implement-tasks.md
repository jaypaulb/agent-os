Now that we have a spec and tasks list ready for implementation, we will proceed with implementation of this spec by following this multi-phase process:

PHASE 1: Determine which task group(s) from tasks.md should be implemented
PHASE 2: Implement the given task(s)
PHASE 3: After ALL task groups have been implemented, produce the final verification report.

**CRITICAL EFFICIENCY REQUIREMENTS:**
- Token budget: 60K tokens (30% of context window)
- Use Grep/Glob to explore, Read only specific files needed
- Read each file ONCE, take notes in your response
- Commit after each atomic design phase completes
- Follow atomic workflow precisely, don't skip levels

Carefully read and execute the instructions in the following files IN SEQUENCE, following their numbered file names.  Only proceed to the next numbered instruction file once the previous numbered instruction has been executed.

Instructions to follow in sequence:

{{PHASE 1: @.agent-os/commands/implement-tasks/1-determine-tasks.md}}

{{PHASE 2: @.agent-os/commands/implement-tasks/2-implement-tasks.md}}

{{PHASE 3: @.agent-os/commands/implement-tasks/3-verify-implementation.md}}
