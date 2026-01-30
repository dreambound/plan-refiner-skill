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
3. **User Clarifications** (user answers supersede reviewer suggestions): {CLARIFICATIONS_PATH}
4. **Standard Review Feedback**: {FEEDBACK_PATH}
5. **Adversarial Review Feedback**: {ADVERSARIAL_FEEDBACK_PATH}
6. **Custom Review Feedback** (if custom review is enabled): {CUSTOM_FEEDBACK_PATH}

WRITE your outputs to:
- **Updated Plan**: {PLAN_PATH} (overwrite)
- **Audit Backup**: {AUDIT_PATH}
- **Changelog**: {CHANGELOG_PATH}

## Critical: Spec Alignment

BEFORE making any changes, verify alignment with the original spec:
- The spec is the SOURCE OF TRUTH
- Changes that would drift from spec should be REJECTED, not implemented
- If feedback suggests scope expansion beyond spec, note it in your summary instead of implementing

This prevents the plan from drifting over multiple passes.

## Your Task

1. **Read the original spec** first (source of truth)
2. **Read the current plan** to understand existing structure
3. **Read clarifications** for user answers and additional feedback (user answers supersede reviewer suggestions)
4. **Read the standard review feedback** for issues to address
5. **Read the adversarial review feedback** for additional challenges and concerns
6. **Read custom review feedback** if it exists (file may not be present if custom reviewer is disabled)
7. **Verify each proposed change** aligns with the spec
8. **Update the plan** incorporating valid feedback from all sources
9. **Write** updated plan and changelog
10. **Return** brief summary to orchestrator

## Update Guidelines

### What to Incorporate
- All issues raised in the standard review feedback that align with spec
- Valid concerns from the adversarial review (assumptions challenged, failure modes, simplification opportunities)
- User answers to assumption questions
- Additional user feedback from clarifications
- Improved clarity, structure, and completeness

### What to Reject
- Changes that would drift from the original spec
- Scope expansions not supported by the spec
- Features or requirements the user didn't ask for
- Speculative additions not in feedback

When rejecting a change, note it in your summary so the orchestrator can surface it to the user if needed.

### Conflict Resolution

When standard, adversarial, and custom reviewers raise contradictory feedback:

1. **Spec alignment takes precedence over all feedback** — if one reviewer's suggestion aligns with the spec and another's contradicts it, follow the spec-aligned suggestion
2. **When reviewers directly contradict** (e.g., standard says "add error handling for X" and adversarial says "error handling is over-engineered"), note the disagreement in your summary and present both options for the user to decide
3. **Prefer simplification over addition** when both options are spec-compatible
4. **Later user clarifications supersede earlier ones** on the same topic — if the user answered "Yes, support pagination" in pass 1 but "No, pagination is not needed" in pass 3, follow the pass 3 answer

### Deduplication

Standard and adversarial reviews may raise the same underlying issue in different terms (e.g., "missing edge case for empty input" vs "failure mode: no handling for empty input"). When this happens:
- Address the issue **once** in the plan update
- Note the convergence in your summary (e.g., "Both reviewers flagged empty input handling — addressed once")

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

## Writing the Changelog

After updating the plan, write a changelog file to `{CHANGELOG_PATH}` with this structure:

```markdown
# Pass {PASS_NUMBER} Changelog

## Changes Made
- [Brief description of change 1 and why]
- [Brief description of change 2 and why]
- ...

## Feedback Addressed
- Standard review: [N of M issues addressed]
- Adversarial review: [N of M issues addressed]
- Custom review: [N of M issues addressed, if applicable]

## Rejected Changes
- [Description of rejected change and reason, if any]

## Reviewer Conflicts Noted
- [Description of conflict and resolution, if any]
```

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
| `{ADVERSARIAL_FEEDBACK_PATH}` | Full path to `pass_N_adversarial_feedback.md` |
| `{CUSTOM_FEEDBACK_PATH}` | Full path to `pass_N_custom_feedback.md` (conditional: only if custom review is enabled) |
| `{CLARIFICATIONS_PATH}` | Full path to `clarifications.md` |
| `{AUDIT_PATH}` | Full path to `audit/plan_v{pass}.md` |
| `{CHANGELOG_PATH}` | Full path to `pass_N_changelog.md` |
| `{PASS_NUMBER}` | Current pass number (1, 2, 3, ...) |

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
