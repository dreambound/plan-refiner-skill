# Plan Update Agent Prompt Template

Use this template when spawning the plan update agent after a review pass.

---

## Prompt Structure

```
You are incorporating feedback into an implementation plan.

Your task is to update the plan based on review feedback and user clarifications while maintaining alignment with the original specification.

## File Paths

READ the following files using the Read tool:
1. **Original Specification** (ALWAYS read first - source of truth): {SPEC_PATH}
2. **Current Plan**: {PLAN_PATH}
3. **Review Feedback**: {FEEDBACK_PATH}
4. **User Clarifications**: {CLARIFICATIONS_PATH}

WRITE your outputs to:
- **Updated Plan**: {PLAN_PATH} (overwrite)
- **Audit Backup**: {AUDIT_PATH}

## Critical: Spec Alignment

BEFORE making any changes, verify alignment with the original spec:
- The spec is the SOURCE OF TRUTH
- Changes that would drift from spec should be REJECTED, not implemented
- If feedback suggests scope expansion beyond spec, note it in your summary instead of implementing

This prevents the plan from drifting over multiple passes.

## Your Task

1. **Read the original spec** first (source of truth)
2. **Read the current plan** to understand existing structure
3. **Read the review feedback** for issues to address
4. **Read clarifications** for user answers and additional feedback
5. **Verify each proposed change** aligns with the spec
6. **Update the plan** incorporating valid feedback
7. **Write** updated plan and backup
8. **Return** brief summary to orchestrator

## Update Guidelines

### What to Incorporate
- All issues raised in the feedback that align with spec
- User answers to assumption questions
- Additional user feedback from clarifications
- Improved clarity, structure, and completeness

### What to Reject
- Changes that would drift from the original spec
- Scope expansions not supported by the spec
- Features or requirements the user didn't ask for
- Speculative additions not in feedback

When rejecting a change, note it in your summary so the orchestrator can surface it to the user if needed.

### Quality Standards
- Address every issue in the feedback (or explain why rejected)
- Incorporate user clarifications naturally into the plan
- Maintain consistent plan structure and formatting
- Keep the plan actionable and specific
- Don't add commentary about changes made - just make them

## Writing the Updated Plan

1. Preserve the original plan structure
2. Make targeted edits to address feedback
3. Don't rewrite sections that weren't flagged
4. Ensure the plan remains coherent after changes

## Return Format (to Orchestrator)

After writing the updated plan, return ONLY this brief summary:

1. **Alignment Status**: "Aligned" or "Warning: [rejected N changes that would drift from spec]"
2. **Issues Addressed**: "Addressed N of M issues from feedback"
3. **Key Changes**: 2-3 bullet points on main changes made
4. **Rejected Changes**: List any feedback items rejected (with brief reason)
5. **Concerns**: Any remaining issues that couldn't be resolved

DO NOT return the full updated plan - it is saved to the files.
```

---

## Variable Placeholders

When using this template, replace:

| Placeholder | Source |
|-------------|--------|
| `{SPEC_PATH}` | Full path to `initial_spec.md` |
| `{PLAN_PATH}` | Full path to `plan.md` |
| `{FEEDBACK_PATH}` | Full path to `pass_N_feedback.md` |
| `{CLARIFICATIONS_PATH}` | Full path to `clarifications.md` |
| `{AUDIT_PATH}` | Full path to `audit/plan_v{pass}.md` |

## Task Tool Configuration

When spawning the update agent:

```
Task tool parameters:
- subagent_type: "general-purpose"
- description: "Apply pass {N} feedback"
- prompt: [Filled template above with file paths]
```

**Why general-purpose?**
- Needs file read/write access
- Returns structured summary without execution prompts
- Keeps orchestrator context thin by not returning full plan content

**Why always read the spec?**
- Prevents drift from original requirements over multiple passes
- Each update agent verifies alignment independently
- Rejected drift-causing changes can be surfaced to user
