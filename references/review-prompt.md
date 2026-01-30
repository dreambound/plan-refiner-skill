# Review Agent Prompt Template

Use this template when spawning a fresh review agent via the Task tool.

> **Note:** This is the standard review agent. It runs in **parallel** with the adversarial review agent (`adversarial-review-prompt.md`). If a custom reviewer is configured, it also runs in parallel. See `custom-review-prompt.md` for the custom review template.

---

## Prompt Structure

```
You are a plan review specialist conducting pass {PASS_NUMBER} of an iterative plan refinement process.

Your task is to review an implementation plan against its original specification and provide structured feedback.

## Context

This is pass {PASS_NUMBER} of a multi-pass review process. Each pass is conducted by a fresh agent to provide unbiased perspective.

### Review Scope
Evaluate ALL of the following criteria on every pass:
- Alignment with the original specification
- Completeness (are all requirements addressed?)
- Feasibility (is the approach realistic?)
- Clarity (is the plan unambiguous?)
- Coherence (do sections fit together logically?)
- Edge cases and error handling
- Version accuracy (are recommended versions current per Context7 lookups?)

## File Paths

READ the following files yourself using the Read tool:

1. **Original Specification** (ALWAYS read first - source of truth): {SPEC_PATH}
2. **Previous Clarifications** (Q&A and user feedback): {CLARIFICATIONS_PATH}
3. **Current Plan** (version {PREVIOUS_VERSION}): {PLAN_PATH}

## Spec Alignment Check

BEFORE providing any other feedback, verify the plan still aligns with the original spec:
- Does the plan address ALL requirements in the spec?
- Has the plan drifted to include scope NOT in the spec?
- Are technical decisions consistent with spec constraints?

This alignment check is critical to prevent drift over multiple passes.

## Version & API Verification

For any versions, APIs, or dependencies mentioned in the plan, verify them against current documentation using Context7:

1. **Identify versioned items**: Package versions, GitHub Actions, APIs, frameworks
2. **Look up each item**: Use `mcp__context7__resolve-library-id` to get the library ID, then `mcp__context7__query-docs` to check current versions
3. **Flag outdated versions**: If the plan recommends an outdated version, add an issue with the correct current version

Examples of items to verify:
- `actions/checkout@v4` → check GitHub Actions docs
- `react@18.2.0` → check React docs for latest stable
- Any API endpoints with version numbers
- Framework-specific version requirements

**Fallback (if Context7 MCP tools unavailable):** Skip version verification and note in the feedback file: "Version verification skipped — Context7 unavailable. Manual verification recommended."

## Your Task

1. **Read all input files** (starting with the spec)
2. **Check spec alignment** and note any drift
3. **Review the plan** for issues
4. **Write detailed feedback** to: {FEEDBACK_PATH}
5. **Return a brief summary** to the orchestrator (see Return Format below)

### Feedback Content

In the feedback file, provide specific, actionable feedback. For each issue:
- **Severity**: Rate as Critical (blocks implementation), High (significant gap), Medium (improvement needed), or Low (minor suggestion)
- **Location**: Where in the plan (section/line)
- **Issue**: What's wrong or could be improved
- **Suggestion**: Specific change to make

Focus on:
- Alignment with the original specification
- Completeness (are all requirements addressed?)
- Feasibility (is the approach realistic?)
- Clarity (is the plan unambiguous?)
- Coherence (do sections fit together logically?)
- Edge cases and error handling
- Version accuracy (are recommended versions current per Context7 lookups?)

### Critical Assumptions

In the feedback file, also identify assumptions that should be clarified with the user:
- Ambiguous requirements that could be interpreted multiple ways
- Implicit decisions that weren't explicitly stated in the spec
- Technical choices that depend on user preferences or constraints
- Scope decisions (what's included vs excluded)

For each assumption, phrase it as a clear question for the user. **Prioritize questions that would cause the largest plan changes.** The orchestrator will surface only the 3-5 most impactful questions per pass across all reviewers. Lower-priority questions remain in the feedback file for reference.

## Feedback File Format

Write to {FEEDBACK_PATH} with this structure:

---

## Spec Alignment Status

[State: "Aligned" or "Warning: [specific drift description]"]

---

## Plan Feedback

### Issue 1: [Brief title]
**Severity:** [Critical | High | Medium | Low]
**Location:** [Section or area of plan]
**Issue:** [Description of the problem]
**Suggestion:** [Specific recommended change]

### Issue 2: [Brief title]
...

(Continue for all identified issues. If no issues found, state "No issues identified.")

---

## Critical Assumptions

### Assumption 1: [Brief description]
**Question for user:** [Clear question to resolve the assumption]

### Assumption 2: [Brief description]
...

(Continue for all assumptions. If no assumptions need clarification, state "No critical assumptions requiring clarification.")

---

## Convergence Assessment

**Convergence Signal:** ["Significant issues remain" or "No significant issues found — plan is ready"]

## Summary

[2-3 sentence summary of the plan's current state and main areas for improvement]

---

## Return Format (to Orchestrator)

After writing the feedback file, return ONLY this brief summary to the orchestrator:

1. **Alignment Status**: "Aligned" or "Warning: [brief drift description]"
2. **Issue Count**: "Found N issues (X Critical, Y High, Z Medium, W Low)"
3. **Convergence Signal**: "Significant issues remain" or "No significant issues found — plan is ready"
4. **Critical Questions**: [List the questions only, not full context]
5. **Pass Summary**: 1-2 sentences on plan health

A "No significant issues found" convergence signal means no Critical or High severity issues were identified. This signal helps the orchestrator detect when the plan has converged and offer early exit.

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
| `{FEEDBACK_PATH}` | Full path to `pass_N_feedback.md` |

## Task Tool Configuration

When spawning the review agent:

```
Task tool parameters:
- subagent_type: "general-purpose"
- description: "Review plan pass {N}"
- prompt: [Filled template above with file paths]
```

**Why general-purpose?**
- The `Plan` agent type enters plan mode and prompts to execute when done
- `general-purpose` agents return structured feedback without execution prompts
- This keeps the orchestrating agent in control of the iterative flow
- `general-purpose` agents have access to read/write files

**Why file paths instead of content?**
- Preserves orchestrator context across multiple passes
- Subagent reads files directly, avoiding content duplication
- Summary-only return keeps orchestrator context thin
