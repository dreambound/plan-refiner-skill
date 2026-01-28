---
name: plan-refiner
description: Generate and iteratively refine implementation plans from an initial spec/prompt. Takes a specification as input, generates an initial plan, then refines it through multiple review passes (minimum 3) with fresh agent context. User can continue beyond 3 passes until satisfied. Use when turning requirements into polished implementation plans.
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
  - Spawn fresh review agent with spec + clarifications + plan
  - Agent provides feedback + identifies critical assumptions
  - Surface assumptions as questions to user
  - Prompt user for optional additional feedback
  - Apply all feedback to plan
  - After pass 3+, ask user to continue or finalize
         ↓
Output: Final plan.md + clarifications.md + audit trail
```

## Input Requirements

The user must provide an initial specification or prompt. This can be:
- A file path to an existing spec document
- Text content to save as `initial_spec.md`
- A description of what they want to plan

## Working Directory Structure

Create a namespaced directory under Claude's plan directory:

```
~/.claude/plans/plan-refiner/{spec-slug}/
├── initial_spec.md      # The starting requirements (immutable)
├── plan.md              # Current plan (updated each pass)
├── clarifications.md    # Accumulated Q&A and user feedback
├── audit/
│   ├── plan_v0.md       # Initial plan from spec
│   ├── plan_v1.md       # After pass 1
│   ├── plan_v2.md       # After pass 2
│   └── plan_v3.md       # After pass 3, etc.
└── pass_N_feedback.md   # Feedback from each review pass
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

## Execution Steps

### Step 1: Setup and Initial Plan Generation

1. **Generate spec slug** from input (filename or content title)
2. **Create namespaced directory** at `~/.claude/plans/plan-refiner/{spec-slug}/`
   - If directory exists, append timestamp suffix (e.g., `-20260128`)
3. **Save or verify initial_spec.md** from user input
4. **Create clarifications.md** (empty initially)
5. **Create audit/ directory**
6. **Spawn Plan Generation Agent** to create initial plan:
   - Use Task tool with `subagent_type: general-purpose`
   - Provide file paths (not contents): `initial_spec.md`, `plan.md`, `audit/plan_v0.md`
   - Agent reads spec, writes plan to both locations
   - Agent returns: brief summary of plan structure (not full content)
   - See `references/generation-prompt.md` for the prompt template

### Step 2: Review Loop (Minimum 3 Passes)

For each pass (1, 2, 3, ...):

#### 2a. Spawn Fresh Review Agent

Use the Task tool with `subagent_type: general-purpose` to create a fresh agent context.

**Important:** Do NOT use `subagent_type: Plan` as it triggers plan mode behavior and will prompt to execute instead of returning feedback.

**Prompt the agent with file paths, not contents:**
- Path to `initial_spec.md` (agent reads it - source of truth)
- Path to `clarifications.md` (agent reads it)
- Path to `plan.md` (agent reads it)
- Current pass number
- Path to write feedback: `pass_N_feedback.md`
- Review instructions from `references/review-prompt.md`

**Agent responsibilities:**
1. Read all input files (always starting with spec)
2. Verify plan alignment with original spec
3. Perform review
4. Write detailed feedback to `pass_N_feedback.md`
5. Return ONLY: alignment status + summary of issues + critical assumption questions

#### 2b. Process Agent Summary

The review agent returns a brief summary (not full feedback):
1. **Alignment Status**: "Aligned" or "Warning: [drift description]"
2. **Issue Count**: Number of issues found
3. **Critical Assumptions**: Questions requiring user input

Full feedback is already saved to `pass_N_feedback.md` by the agent.

**If alignment warning received:** Surface to user via AskUserQuestion before proceeding.

#### 2c. Surface Assumptions to User

If the agent identified critical assumptions:
- Use AskUserQuestion to present each question
- Append user answers to `clarifications.md` with format:

```markdown
## Pass N Clarifications

**Q: [Question from agent]**
A: [User's answer]
```

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
- Path to `pass_N_feedback.md` (agent feedback)
- Path to `clarifications.md` (includes user answers)
- Path to write: `audit/plan_v{pass}.md` (backup) and updated `plan.md`
- Update instructions from `references/update-prompt.md`

**Agent responsibilities:**
1. Read spec, current plan, feedback, and clarifications
2. Verify changes align with original spec
3. Incorporate all feedback and user clarifications
4. Write updated plan to `plan.md`
5. Copy to `audit/plan_v{pass}.md`
6. Return ONLY: alignment status + 2-3 sentence summary of changes

**If alignment warning received:** Surface to user via AskUserQuestion:
"The update agent detected potential drift: [warning]. Continue with changes or revert?"

#### 2f. Continuation Check (After Pass 3+)

After pass 3 and each subsequent pass, ask the user:
"Would you like to continue refining the plan, or is it ready?"

Options:
- **Continue**: Proceed to next pass
- **Finalize**: Exit loop with current plan

### Step 3: Finalization

Present the user with:
- Final `plan.md` location
- Summary of changes across all passes
- Location of audit trail

## Subagent Prompt Templates

See the following templates in `references/`:
- `generation-prompt.md` - Initial plan generation agent
- `review-prompt.md` - Review agents for each pass
- `update-prompt.md` - Plan update agent after feedback

## Pass Focus Areas

Each pass has a primary focus while still reviewing the full plan:

| Pass | Primary Focus |
|------|---------------|
| 1 | Alignment with spec, surface major assumptions |
| 2 | Completeness and feasibility, clarify remaining gaps |
| 3+ | Final polish, coherence, edge cases |

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
