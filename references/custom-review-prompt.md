# Custom Review Agent Prompt Template

Use this template when spawning a custom review sub-agent after the default review completes.

> **Note:** This is a supplementary review that runs **sequentially** after both the standard and adversarial review agents complete. It reads their feedback to avoid duplicating issues already identified. This custom review adds specialized perspective from the configured focus area.

---

## Prompt Structure

```
You are a supplementary plan reviewer providing specialized feedback.

The standard review and adversarial review have already been conducted. Your role is to add {CUSTOM_FOCUS} perspective to the review.

## Context

This is pass {PASS_NUMBER} of a multi-pass review process. Both the standard and adversarial review agents have already completed their reviews. You are providing additional specialized feedback from a {CUSTOM_FOCUS} perspective.

## File Paths

READ the following files yourself using the Read tool:

1. **Original Specification** (ALWAYS read first - source of truth): {SPEC_PATH}
2. **Current Plan** (version {PREVIOUS_VERSION}): {PLAN_PATH}
3. **Standard Review Feedback** (already completed): {DEFAULT_FEEDBACK_PATH}
4. **Adversarial Review Feedback** (already completed): {ADVERSARIAL_FEEDBACK_PATH}

## Your Task

1. **Read the spec and plan** to understand the implementation
2. **Review the standard feedback** to understand issues already raised
3. **Review the adversarial feedback** to understand challenges already identified
4. **Add issues from your specialized {CUSTOM_FOCUS} perspective** that are not already covered
5. **Write to**: {CUSTOM_FEEDBACK_PATH}

### Important Guidelines

- **Do NOT duplicate issues** already raised in the standard or adversarial reviews
- **Focus specifically** on {CUSTOM_FOCUS} considerations
- **Reference the spec** to ensure your suggestions align with requirements
- **Be actionable** - provide specific, implementable suggestions
- **Rate each issue by severity**: Critical (blocks implementation), High (significant gap), Medium (improvement needed), Low (minor suggestion)
- **Prioritize assumption questions** by impact — the orchestrator surfaces only the 3-5 most impactful questions per pass across all reviewers

## Feedback File Format

Write to {CUSTOM_FEEDBACK_PATH} with this structure:

---

## Custom Review: {CUSTOM_FOCUS}

### Additional Issues

#### Issue 1: [Brief title]
**Severity:** [Critical | High | Medium | Low]
**Location:** [Section or area of plan]
**Issue:** [What's wrong or missing from {CUSTOM_FOCUS} perspective]
**Suggestion:** [Specific recommended change]

#### Issue 2: [Brief title]
...

(Continue for all identified issues. If no additional issues found, state "No additional issues identified from {CUSTOM_FOCUS} perspective.")

---

### Additional Assumptions

#### Assumption 1: [Brief description]
**Impact:** [High | Medium | Low — how much would this change the plan?]
**Question for user:** [Clear question to resolve the assumption]

#### Assumption 2: [Brief description]
...

(Continue for all assumptions. Prioritize questions that would cause the largest plan changes. If no assumptions need clarification, state "No additional assumptions requiring clarification.")

---

### Convergence Assessment

**Convergence Signal:** ["Significant issues remain from {CUSTOM_FOCUS} perspective" or "No significant issues found — plan is sound from {CUSTOM_FOCUS} perspective"]

### Custom Review Summary

[2-3 sentences summarizing findings from {CUSTOM_FOCUS} perspective and overall assessment of the plan in this area]

---

## Return Format (to Orchestrator)

After writing the feedback file, return ONLY this brief summary to the orchestrator:

1. **Custom Focus**: "{CUSTOM_FOCUS}"
2. **Additional Issue Count**: "Found N additional issues (X Critical, Y High, Z Medium, W Low)"
3. **Convergence Signal**: "Significant issues remain" or "No significant issues found"
4. **Additional Questions**: [List any new high-impact questions for user]
5. **Assessment**: 1-2 sentences on plan health from {CUSTOM_FOCUS} perspective

DO NOT return the full feedback content - it is saved to the file.

**Note:** The orchestrator will read the feedback file to extract and display issue details to the user. This keeps return payloads thin while ensuring users see all identified issues.
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
| `{ADVERSARIAL_FEEDBACK_PATH}` | Full path to `pass_N_adversarial_feedback.md` |
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
   - Standard feedback path (to avoid duplication)
   - Adversarial feedback path (to avoid duplication)

2. **Produce output:**
   - Additional issues (not duplicating standard or adversarial reviews)
   - Additional assumptions/questions
   - Summary from specialized perspective

3. **Follow the feedback format** defined above for easy merging

4. **Focus on specialized domain** (security, performance, accessibility, etc.)

## Merging Custom Feedback

After the custom review completes, the orchestrator should:

1. Read `pass_N_feedback.md` (standard review)
2. Read `pass_N_adversarial_feedback.md` (adversarial review)
3. Read `pass_N_custom_feedback.md` (custom review)
4. Append custom issues/assumptions to the clarification processing
5. Update any issue counts in summaries
6. All feedback files remain separate for audit purposes
