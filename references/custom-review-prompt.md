# Custom Review Agent Prompt Template

Use this template when spawning a custom review sub-agent after the default review completes.

> **Note:** This is a supplementary review. The default review has already been conducted and its feedback saved. This custom review adds specialized perspective without duplicating issues already identified.

---

## Prompt Structure

```
You are a supplementary plan reviewer providing specialized feedback.

The default review has already been conducted. Your role is to add {CUSTOM_FOCUS} perspective to the review.

## Context

This is pass {PASS_NUMBER} of a multi-pass review process. The default review agent has already completed its review. You are providing additional specialized feedback from a {CUSTOM_FOCUS} perspective.

## File Paths

READ the following files yourself using the Read tool:

1. **Original Specification** (ALWAYS read first - source of truth): {SPEC_PATH}
2. **Current Plan** (version {PREVIOUS_VERSION}): {PLAN_PATH}
3. **Default Review Feedback** (already completed): {DEFAULT_FEEDBACK_PATH}

## Your Task

1. **Read the spec and plan** to understand the implementation
2. **Review the default feedback** to understand issues already raised
3. **Add issues from your specialized {CUSTOM_FOCUS} perspective**
4. **Write to**: {CUSTOM_FEEDBACK_PATH}

### Important Guidelines

- **Do NOT duplicate issues** already raised in the default review
- **Focus specifically** on {CUSTOM_FOCUS} considerations
- **Reference the spec** to ensure your suggestions align with requirements
- **Be actionable** - provide specific, implementable suggestions

## Feedback File Format

Write to {CUSTOM_FEEDBACK_PATH} with this structure:

---

## Custom Review: {CUSTOM_FOCUS}

### Additional Issues

#### Issue 1: [Brief title]
**Location:** [Section or area of plan]
**Issue:** [What's wrong or missing from {CUSTOM_FOCUS} perspective]
**Suggestion:** [Specific recommended change]

#### Issue 2: [Brief title]
...

(Continue for all identified issues. If no additional issues found, state "No additional issues identified from {CUSTOM_FOCUS} perspective.")

---

### Additional Assumptions

#### Assumption 1: [Brief description]
**Question for user:** [Clear question to resolve the assumption]

#### Assumption 2: [Brief description]
...

(Continue for all assumptions. If no assumptions need clarification, state "No additional assumptions requiring clarification.")

---

### Custom Review Summary

[2-3 sentences summarizing findings from {CUSTOM_FOCUS} perspective and overall assessment of the plan in this area]

---

## Return Format (to Orchestrator)

After writing the feedback file, return ONLY this brief summary to the orchestrator:

1. **Custom Focus**: "{CUSTOM_FOCUS}"
2. **Additional Issue Count**: "Found N additional issues"
3. **Additional Questions**: [List any new questions for user]
4. **Assessment**: 1-2 sentences on plan health from {CUSTOM_FOCUS} perspective

DO NOT return the full feedback content - it is saved to the file.
```

---

## Variable Placeholders

When using this template, replace:

| Placeholder | Source |
|-------------|--------|
| `{PASS_NUMBER}` | Current pass (1, 2, 3, ...) |
| `{PREVIOUS_VERSION}` | Pass number - 1 (0 for first review) |
| `{SPEC_PATH}` | Full path to `initial_spec.md` |
| `{PLAN_PATH}` | Full path to `plan.md` |
| `{DEFAULT_FEEDBACK_PATH}` | Full path to `pass_N_feedback.md` |
| `{CUSTOM_FEEDBACK_PATH}` | Full path to `pass_N_custom_feedback.md` |
| `{CUSTOM_FOCUS}` | The specialized focus area (e.g., "security", "performance", "accessibility") |

## Task Tool Configuration

When spawning the custom review agent:

```
Task tool parameters:
- subagent_type: "general-purpose"
- description: "Custom {CUSTOM_FOCUS} review pass {N}"
- prompt: [Filled template above with file paths and custom focus]
```

**Why general-purpose?**
- Consistent with the default review agent approach
- Returns structured feedback without execution prompts
- Has access to read/write files

## Custom Review Skill Contract

If using a named skill as the custom reviewer, the skill should:

1. **Accept inputs:**
   - Spec path
   - Plan path
   - Default feedback path (to avoid duplication)

2. **Produce output:**
   - Additional issues (not duplicating default)
   - Additional assumptions/questions
   - Summary from specialized perspective

3. **Follow the feedback format** defined above for easy merging

4. **Focus on specialized domain** (security, performance, accessibility, etc.)

## Merging Custom Feedback

After the custom review completes, the orchestrator should:

1. Read `pass_N_feedback.md` (default review)
2. Read `pass_N_custom_feedback.md` (custom review)
3. Append custom issues/assumptions to the clarification processing
4. Update any issue counts in summaries
5. Both feedback files remain separate for audit purposes
