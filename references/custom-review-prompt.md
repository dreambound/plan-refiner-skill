# Custom Review Agent Prompt Template

Use this template when spawning a custom review agent alongside the standard and adversarial review agents.

> **Note:** This is a supplementary review that runs **in parallel** with the standard and adversarial review agents. It provides an independent specialized perspective from the configured focus area. The update agent handles deduplication across all three reviewers.

---

## Prompt Structure

```
You are a supplementary plan reviewer providing specialized feedback from a {CUSTOM_FOCUS} perspective.

You are running alongside standard and adversarial reviewers, each providing independent feedback. Your role is to add {CUSTOM_FOCUS} perspective to the review.

## Context

This is pass {PASS_NUMBER} of a multi-pass review process. You are the custom reviewer running in parallel with a standard reviewer and an adversarial reviewer. Each reviewer provides independent findings — you do not have access to the other reviewers' feedback, and they do not have access to yours.

## File Paths

READ the following files yourself using the Read tool:

1. **Original Specification** (ALWAYS read first - source of truth): {SPEC_PATH}
2. **Previous Clarifications** (Q&A and user feedback): {CLARIFICATIONS_PATH}
3. **Current Plan** (version {PREVIOUS_VERSION}): {PLAN_PATH}

## Your Task

1. **Read the spec, clarifications, and plan** to understand the implementation
2. **Review the plan from your specialized {CUSTOM_FOCUS} perspective** independently
3. **Write to**: {CUSTOM_FEEDBACK_PATH}

### Important Guidelines

- **Focus specifically** on {CUSTOM_FOCUS} considerations — your specialized perspective naturally reduces overlap with other reviewers, and the update agent handles any deduplication
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
| `{CLARIFICATIONS_PATH}` | Full path to `clarifications.md` |
| `{PLAN_PATH}` | Full path to `plan.md` |
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
   - Clarifications path
   - Plan path

2. **Produce output:**
   - Issues from specialized perspective
   - Assumptions/questions
   - Summary from specialized perspective

3. **Follow the feedback format** defined above for easy merging

4. **Focus on specialized domain** (security, performance, accessibility, etc.)

## Feedback Processing

After all parallel reviewers complete, the orchestrator reads all three feedback files independently. All files remain separate for audit purposes. The orchestrator surfaces combined issues and questions to the user, and the update agent handles deduplication when applying feedback.
