---
name: plan-refiner
description: Generate and iteratively refine implementation plans from an initial spec/prompt. Takes a specification as input, generates an initial plan, then refines it through parallel multi-reviewer passes (minimum 3) with fresh agent context. User can continue beyond 3 passes until satisfied. Use when turning requirements into polished implementation plans.
license: MIT
metadata:
  author: dreambound
  version: "1.0.0"
---

# Plan Refiner Skill

This skill implements an iterative plan refinement process that transforms an initial specification into a polished implementation plan through multiple review passes with fresh agent perspective.

## Workflow Overview

```
Input: initial_spec.md (starting prompt/requirements)
         ↓
Step 1: Generate Initial Plan (plan.md v0)
         ↓
Step 2: Review Loop (minimum 3 passes)
  - Spawn up to 3 review agents in parallel:
    • Standard reviewer (spec alignment, completeness, versions via C7)
    • Adversarial reviewer (assumptions, failure modes)
    • Custom reviewer (optional — specialized focus)
  - Surface identified issues and assumptions to user
  - Prompt user for optional additional feedback
  - Apply all feedback to plan via update agent
  - After pass 3+, ask user to continue or finalize
         ↓
Output: Final plan.md + clarifications.md + audit trail
```

**Note:** Review agents use Context7 (`mcp__context7__resolve-library-id` → `mcp__context7__query-docs`) to verify that any package versions, GitHub Actions, APIs, or framework versions mentioned in the plan are current. This prevents recommending outdated versions based on training data.

**Fallback:** If Context7 MCP tools are unavailable, skip version verification and note in review summaries: "Version verification skipped — Context7 unavailable. Manual verification recommended."

## Input Requirements

The user must provide an initial specification or prompt. This can be:
- A file path to an existing spec document
- Text content to save as `initial_spec.md`
- A description of what they want to plan

## Working Directory Structure

Create a namespaced directory under Claude's plan directory:

```
~/.claude/plans/plan-refiner/{spec-slug}/
├── initial_spec.md           # The starting requirements (immutable)
├── plan.md                   # Current plan (updated each pass)
├── clarifications.md         # Accumulated Q&A and user feedback
├── config.json               # Run configuration (includes custom_reviewer settings)
├── audit/
│   ├── plan_v0.md            # Initial plan from spec
│   ├── plan_v1.md            # After pass 1
│   ├── plan_v2.md            # After pass 2
│   └── plan_v3.md            # After pass 3, etc.
├── pass_N_feedback.md              # Standard review feedback for each pass
├── pass_N_adversarial_feedback.md  # Adversarial review feedback for each pass
├── pass_N_custom_feedback.md       # Custom review feedback (if custom reviewer enabled)
└── pass_N_changelog.md             # Changes made in each pass and why
```

### Global Preferences

```
~/.claude/plans/plan-refiner/
└── preferences.json          # Global preferences across all runs
```

**Spec Slug Generation:**
- From file path: Use filename without extension (e.g., `my-feature.md` → `my-feature`)
- From content: Slugify first heading or first line (e.g., "Auth Feature Spec" → `auth-feature-spec`)
- Add timestamp suffix if directory exists: `my-feature-20260128`

## Context Preservation Strategy

To preserve orchestrator context across multiple passes, this skill uses **file-based delegation**:

1. **Subagents read from files** - Don't embed full content in prompts; tell subagents which files to read
2. **Subagents write to files** - Feedback and plan updates go directly to disk
3. **Summaries only returned** - Subagents return brief summaries, not full content
4. **Orchestrator stays thin** - Main agent manages paths and workflow, not content

### Drift Mitigation: Spec-Always-Included

To prevent drift from the original specification:
- **All subagents always re-read `initial_spec.md`** before performing their task
- **Review agents explicitly check spec alignment** and flag drift in their feedback
- **Update agents verify changes align with spec** before writing
- **Summaries include alignment status** (e.g., "Aligned" or "Warning: divergence detected")

### Error Handling

If a subagent fails (returns an error, writes an empty file, or doesn't write its output file):

1. **Generation agent failure** (Step 1): Abort and report error to user. Cannot proceed without an initial plan.
2. **One review agent fails** (Step 2a): Proceed with the remaining agents' feedback. Note in the pass summary: "Warning: [standard/adversarial/custom] review unavailable this pass — proceeding with partial feedback." Custom reviewer failure is always non-critical (it's optional).
3. **Two review agents fail** (Step 2a): Proceed if the standard reviewer succeeded. If the standard reviewer is among the failures, abort the pass and offer to retry or finalize with the current plan.
4. **All review agents fail** (Step 2a): Abort the current pass and report error to user. Offer to retry or finalize with the current plan.
5. **Update agent fails** (Step 2e): Restore plan.md from the latest audit copy (`audit/plan_v{pass-1}.md`). Report error and offer to retry or skip this pass.

**Verification after each agent:** After each subagent completes, verify its output file exists and is non-empty before proceeding. If verification fails, follow the failure handling above.

### Context Budget Considerations

The orchestrator accumulates ~4,000-8,000 tokens per pass from agent spawn prompts (up to 3 agents), return summaries, issue display, and user interactions. With initial setup overhead (~5,000-8,000 tokens), the orchestrator context remains healthy through **7-8 passes**. Beyond that, quality may degrade.

- **Passes 1-5:** Safe. No action needed.
- **Passes 6-7:** Monitor. Consider compacting orchestrator context by summarizing prior pass interactions.
- **Passes 8+:** Extended sessions may exhibit degraded orchestrator performance. Consider finalizing or restarting with the current plan as the new baseline.

For subagents, each starts fresh per pass (no accumulation). However, `clarifications.md` grows by ~500 tokens per pass. For sessions exceeding 8 passes, earlier clarifications that have already been incorporated into the plan can be summarized to reduce subagent input size.

## Startup Configuration

Before beginning refinement, ask about optional custom review.

### Startup Questions

**Question 1: Custom Reviewer**

Check for `~/.claude/plans/plan-refiner/preferences.json`:

If `preferences.json` exists and has a previous `custom_reviewer`:
- Ask: "Last time you used '{skill-name}' as additional reviewer. Use it again?"
- Options: Yes / No / Different skill

Otherwise:
- Ask: "Add a custom review agent for specialized feedback? (e.g., security, performance)"
- Options: No (default only) / Yes, specify skill

**Question 2: Skill Specification** (if yes to custom reviewer)

Ask: "Specify the custom review skill:"
- Skill name (e.g., 'security-reviewer')
- Skill path (absolute path)
- Skip (use default only)

Also ask for the focus description (e.g., "security considerations", "performance optimization", "accessibility compliance").

### Preferences Persistence

Store in `~/.claude/plans/plan-refiner/preferences.json`:

```json
{
  "custom_reviewer": {
    "enabled": true,
    "type": "skill_name",
    "value": "security-reviewer",
    "focus": "security considerations"
  },
  "custom_reviewer_history": ["security-reviewer", "performance-reviewer"],
  "updated_at": "2026-01-28T..."
}
```

- `custom_reviewer`: Current configuration (or null if disabled)
- `custom_reviewer_history`: Previously used skills (for suggestions)
- `updated_at`: Last modification timestamp

---

## Progress Tracking

Create all progress tasks upfront after setup using `TaskCreate`, then update status as each phase progresses.

### Initial Tasks (created after setup, before generation)

| # | subject | activeForm |
|---|---------|------------|
| 1 | Generate initial plan from spec (v0) | Generating initial plan from spec |
| 2 | Review pass 1 | Running review pass 1 |
| 3 | Review pass 2 | Running review pass 2 |
| 4 | Review pass 3 | Running review pass 3 |

### Status Transitions

Tasks follow: `pending` → `in_progress` → `completed`

- Set a task to `in_progress` immediately before starting its work
- Set a task to `completed` immediately after its work finishes
- For passes beyond 3, dynamically create a new task via `TaskCreate` before starting:
  - `subject: "Review pass {N}"`
  - `activeForm: "Running review pass {N}"`

---

## Execution Steps

### Step 1: Setup and Initial Plan Generation

1. **Generate spec slug** from input (filename or content title)
2. **Create namespaced directory** at `~/.claude/plans/plan-refiner/{spec-slug}/`
   - If directory exists, append timestamp suffix (e.g., `-20260128`)
3. **Save or verify initial_spec.md** from user input
4. **Create clarifications.md** (empty initially)
5. **Create audit/ directory**
6. **Create config.json** with run configuration:
   ```json
   {
     "created_at": "2026-01-28T...",
     "custom_reviewer": {
       "enabled": true,
       "type": "skill_name",
       "value": "security-reviewer",
       "focus": "security considerations"
     },
     "current_pass": 0,
     "status": "in_progress"
   }
   ```
   - Set `custom_reviewer` to null if not configured
7. **Create progress tasks** via `TaskCreate` — create all 4 tasks from the Progress Tracking table above; store task IDs for status updates
8. **Set Task 1 to `in_progress`** (`TaskUpdate` with status `in_progress`)
9. **Spawn Plan Generation Agent** to create initial plan:
   - Use Task tool with `subagent_type: general-purpose`
   - Provide file paths (not contents): `initial_spec.md`, `plan.md`, `audit/plan_v0.md`
   - Agent reads spec, writes plan to both locations
   - Agent returns: brief summary of plan structure (not full content)
   - See `references/generation-prompt.md` for the prompt template
10. **Set Task 1 to `completed`** (`TaskUpdate` with status `completed`)

### Step 2: Review Loop (Minimum 3 Passes)

For each pass (1, 2, 3, ...):

#### 2-pre. Set Pass Task to In Progress

Set the current pass's task to `in_progress` via `TaskUpdate`. For passes 1-3 this is one of the pre-created tasks. For passes 4+, this is the dynamically created task (see 2f).

#### 2a. Spawn Review Agents (Parallel)

Spawn **up to three review agents in parallel** using a single message with multiple Task tool calls. All use `subagent_type: general-purpose`.

**Important:** Do NOT use `subagent_type: Plan` as it triggers plan mode behavior and will prompt to execute instead of returning feedback.

**Agent 1 — Standard Review:**
- Prompt with file paths (not contents):
  - Path to `initial_spec.md` (source of truth)
  - Path to `clarifications.md`
  - Path to `plan.md`
  - Current pass number
  - Path to write feedback: `pass_N_feedback.md`
- Review instructions from `references/review-prompt.md`
- Evaluates ALL criteria every pass: spec alignment, completeness, feasibility, clarity, coherence, edge cases, version accuracy
- Returns: alignment status + summary of issues + critical assumption questions

**Agent 2 — Adversarial Review:**
- Prompt with file paths (not contents):
  - Path to `initial_spec.md` (source of truth)
  - Path to `clarifications.md`
  - Path to `plan.md`
  - Current pass number
  - Path to write feedback: `pass_N_adversarial_feedback.md`
- Review instructions from `references/adversarial-review-prompt.md`
- Does NOT read standard feedback (runs in parallel — independent perspective)
- Challenges: assumptions, implementation choices, unstated dependencies, failure modes, over-engineering, under-specification
- Returns: issue count + top concerns + assumptions questioned + resilience assessment

**Agent 3 — Custom Review** (conditional: only if `config.json` has `custom_reviewer.enabled: true`):
- Prompt with file paths (not contents):
  - Path to `initial_spec.md` (source of truth)
  - Path to `clarifications.md`
  - Path to `plan.md`
  - Current pass number
  - Custom focus area from `config.json` (e.g., "security considerations")
  - Path to write feedback: `pass_N_custom_feedback.md`
- Review instructions from `references/custom-review-prompt.md`
- Does NOT read standard or adversarial feedback (runs in parallel — independent specialized perspective)
- Focuses on the configured custom area (security, performance, accessibility, etc.)
- Returns: custom focus + additional issue count + convergence signal + additional questions

**All enabled agents must be launched in a single message** to run in parallel.

**Error handling:** After all agents return, verify that each expected feedback file exists and is non-empty. See Error Handling section above for failure scenarios.

#### 2b. Process Agent Summaries

All review agents return brief summaries (not full feedback):

**Standard Review Summary:**
1. **Alignment Status**: "Aligned" or "Warning: [drift description]"
2. **Issue Count**: Number of issues found (with severity breakdown)
3. **Convergence Signal**: "Significant issues remain" or "No significant issues found — plan is ready"
4. **Critical Assumptions**: Questions requiring user input

**Adversarial Review Summary:**
1. **Issue Count**: Number of adversarial issues found (with severity breakdown)
2. **Convergence Signal**: "Significant issues remain" or "No significant issues found — plan is ready"
3. **Top Concerns**: Most significant challenges
4. **Assumptions Questioned**: Questions to validate assumptions

**Custom Review Summary** (if custom reviewer enabled):
1. **Custom Focus**: The specialized area reviewed
2. **Additional Issue Count**: Number of issues found (with severity breakdown)
3. **Convergence Signal**: "Significant issues remain" or "No significant issues found — plan is sound from [focus] perspective"
4. **Additional Questions**: New questions from the specialized perspective

Full feedback is already saved to the respective `pass_N_*_feedback.md` files by the agents.

**If alignment warning received:** Surface to user via AskUserQuestion before proceeding.

#### 2b-bis. Surface Issues to User

After all review agents complete and summaries are processed, read all feedback files and display identified issues:

1. **Read feedback files**:
   - Read `pass_N_feedback.md` for standard review issues
   - Read `pass_N_adversarial_feedback.md` for adversarial review issues
   - If custom reviewer enabled, also read `pass_N_custom_feedback.md`

2. **Extract issues** from each file:
   - Standard: parse each `### Issue N:` block
   - Adversarial: parse each `#### Issue N:` block
   - Custom: parse each `#### Issue N:` block
   - Extract: title, severity, location/category, problem/challenge, suggestion

3. **Display issues to the user** before asking questions:

```
## Pass {N} Review Findings

### Standard Review Issues ({count})

**Issue 1: {title}** [{severity}]
- Location: {location}
- Problem: {description}
- Suggestion: {recommendation}

**Issue 2: {title}**
...

### Adversarial Review Issues ({count})

**Issue 1: {title}** [{severity}]
- Category: {category}
- Challenge: {description}
- Suggestion: {recommendation}

**Issue 2: {title}**
...

[If custom reviewer enabled:]
### Additional Issues from {custom_focus} Review ({count})

**Issue 1: {title}**
...
```

If no issues found across all reviewers, display: "No issues identified in this pass."

#### 2c. Surface Questions to User

If the agents identified critical assumptions:
- Collect questions from all reviewers (standard, adversarial, and custom if enabled)
- Limit to the **3-5 most impactful questions** per pass — prioritize questions that would cause the largest plan changes
- Lower-priority questions remain in the feedback files for reference but are not surfaced as interactive prompts
- Use AskUserQuestion to present the selected questions
- Append user answers to `clarifications.md` with format:

```markdown
## Pass N Clarifications

**Q: [Question from agent]**
A: [User's answer]
```

**Note:** Later entries in `clarifications.md` supersede earlier entries on the same topic. If the user answered "Yes, support pagination" in pass 1 but "No, pagination is not needed" in pass 3, the pass 3 answer takes precedence. The update agent is instructed to resolve conflicts by preferring the most recent user input.

#### 2d. Prompt for Additional User Feedback

After answering questions, ask the user:
"Do you have any additional feedback on the current plan? (You can skip this)"

If user provides feedback:
- Append to `clarifications.md`:

```markdown
### User Feedback (Pass N)
[User's feedback]
```

#### 2e. Apply Feedback via Subagent

Spawn a **Plan Update Agent** (`subagent_type: general-purpose`):

**Provide file paths:**
- Path to `initial_spec.md` (source of truth)
- Path to `plan.md` (current version)
- Path to `pass_N_feedback.md` (standard review feedback)
- Path to `pass_N_adversarial_feedback.md` (adversarial review feedback)
- Path to `pass_N_custom_feedback.md` (custom review feedback — only if custom reviewer is enabled)
- Path to `clarifications.md` (includes user answers)
- Path to write: `audit/plan_v{pass}.md` (backup), updated `plan.md`, and `pass_N_changelog.md`
- Update instructions from `references/update-prompt.md`

**Agent responsibilities:**
1. Read spec, current plan, all feedback files, and clarifications
2. Verify changes align with original spec
3. Incorporate feedback from standard, adversarial, and (if present) custom reviews
4. Write updated plan to `plan.md`
5. Copy to `audit/plan_v{pass}.md`
6. Return ONLY: alignment status + 2-3 sentence summary of changes
7. Write a changelog to `pass_N_changelog.md` documenting what changed and why

**Error handling:** After the update agent completes, verify `plan.md` was updated, `audit/plan_v{pass}.md` was written, and `pass_N_changelog.md` exists. If the plan update failed, restore `plan.md` from `audit/plan_v{pass-1}.md` (or `audit/plan_v0.md` for pass 1) and report the failure to the user. A missing changelog is non-critical — note the gap for the finalization summary but do not abort.

**If alignment warning received:** Surface to user via AskUserQuestion:
"The update agent detected potential drift: [warning]. Continue with changes or revert?"

#### 2e-post. Set Pass Task to Completed

Set the current pass's task to `completed` via `TaskUpdate`.

#### 2f. Continuation Check (After Pass 3+)

After pass 3 and each subsequent pass:

**Convergence check:** Re-read the Convergence Assessment sections from `pass_N_feedback.md` and `pass_N_adversarial_feedback.md` (and `pass_N_custom_feedback.md` if custom review is enabled) to determine whether significant issues remain. Do not rely on memory of earlier summaries from step 2b.

If all reviewers reported "No significant issues found" (no High or Critical severity issues), recommend finalization:
"All reviewers found no significant issues. The plan appears ready. Finalize or continue?"

**Otherwise**, ask:
"Would you like to continue refining the plan, or is it ready?"

Options:
- **Continue**: Create a new progress task via `TaskCreate` (`subject: "Review pass {N}"`, `activeForm: "Running review pass {N}"`), then proceed to next pass
- **Finalize**: Exit loop with current plan

### Step 3: Finalization

Present the user with:
- Final `plan.md` location
- Summary of changes across all passes (compiled from `pass_N_changelog.md` files)
- Location of audit trail
- Total passes completed and convergence status

## Subagent Prompt Templates

See the following templates in `references/`:
- `generation-prompt.md` - Initial plan generation agent
- `review-prompt.md` - Standard review agent (all criteria, every pass)
- `adversarial-review-prompt.md` - Adversarial review agent (challenges and stress-tests)
- `custom-review-prompt.md` - Custom review agent (parallel, specialized feedback)
- `update-prompt.md` - Plan update agent after feedback

## Review Architecture

Every pass spawns up to three review agents in parallel. Reviewers evaluate all criteria each pass but may report "no significant issues" for clean areas, enabling convergence detection:

| Agent | Scope | Output |
|-------|-------|--------|
| Standard Reviewer | All criteria: spec alignment, completeness, feasibility, clarity, coherence, edge cases, version accuracy | `pass_N_feedback.md` |
| Adversarial Reviewer | Challenges: assumptions, simpler alternatives, unstated dependencies, failure modes, over/under-engineering | `pass_N_adversarial_feedback.md` |
| Custom Reviewer (optional) | Specialized focus (e.g., security, performance) — runs in parallel, independent specialized perspective | `pass_N_custom_feedback.md` |

## Example Invocation

User: "Run /plan-refiner on my feature spec"

1. Agent asks for spec location or content
2. Agent generates spec slug (e.g., `my-feature` from `my-feature.md`)
3. Agent creates directory at `~/.claude/plans/plan-refiner/my-feature/` and saves spec
4. Agent generates initial plan (v0)
5. Agent runs 3 review passes with fresh context
6. After pass 3, agent asks to continue or finalize
7. Agent presents final plan location and audit trail

## Key Principles

1. **Fresh Perspective**: Each review pass uses a new agent with no accumulated context bias
2. **User in the Loop**: Critical assumptions require user clarification before proceeding
3. **Audit Trail**: Every plan version is preserved for reference
4. **Accumulated Context**: Clarifications (Q&A + feedback) persist across all passes
5. **Flexible Iteration**: Minimum 3 passes, but user controls when to stop
