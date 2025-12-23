Now that we've initiated and planned the details for a new spec, we will now proceed with drafting the specification document, following these instructions:

**EFFICIENCY REQUIREMENTS:**
- Token budget: 50-70K (write-spec is detailed technical work)
- Use Grep/Glob to explore existing code patterns
- Read files ONCE, extract key information
- Commit/save progress after each phase

{{workflows/specification/write-spec}}

## Display confirmation and next step

Display the following message to the user:

```
The spec has been created at `.agent-os/specs/[this-spec]/spec.md`.

Review it closely to ensure everything aligns with your vision and requirements.

{{IF tracking_mode_beads}}
Next step: Create beads issues from the spec
- Run `/create-tasks` to auto-generate beads issues with atomic design hierarchy
- Issues will be created in `.beads/issues.jsonl`
- Use `bd ready` to see unblocked work
{{ELSE}}
Next step: Create task breakdown
- Run `/create-tasks` to generate hierarchical task groups in tasks.md
{{ENDIF tracking_mode_beads}}
```

{{UNLESS standards_as_claude_code_skills}}
## User Standards & Preferences Compliance

IMPORTANT: Ensure that the specification document's content is ALIGNED and DOES NOT CONFLICT with the user's preferences and standards as detailed in the following files:

{{standards/*}}
{{ENDUNLESS standards_as_claude_code_skills}}
