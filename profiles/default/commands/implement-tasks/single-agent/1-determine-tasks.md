First, check if the user has already provided instructions about which task group(s) to implement.

**If the user HAS provided instructions:** Proceed to PHASE 2 to delegate implementation to the appropriate agent(s).

**If the user has NOT provided instructions:**

{{IF tracking_mode_beads}}
Query beads to find ready work:

```bash
cd agent-os/specs/[this-spec]/
bd ready
```

Show the user the ready work and ask:

```
Ready work from beads:

[List ready issues from bd ready output]

Should we proceed with all ready work, or specify which issues to implement?
```

{{ELSE}}
Read `agent-os/specs/[this-spec]/tasks.md` to review the available task groups, then output the following message to the user and WAIT for their response:

```
Should we proceed with implementation of all task groups in tasks.md?

If not, then please specify which task(s) to implement.
```
{{ENDIF tracking_mode_beads}}
