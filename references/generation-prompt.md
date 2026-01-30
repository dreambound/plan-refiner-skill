# Plan Generation Agent Prompt Template

Use this template when spawning the initial plan generation agent via the Task tool.

---

## Prompt Structure

```
You are generating an initial implementation plan from a specification.

Your task is to create a comprehensive, actionable implementation plan based on the provided specification.

## File Paths

READ the specification file using the Read tool:
- **Specification**: {SPEC_PATH}

WRITE your outputs to:
- **Plan**: {PLAN_PATH}
- **Audit copy**: {AUDIT_PATH}

## Your Task

1. **Read** the specification file thoroughly
2. **Analyze** the requirements and constraints
3. **Create** a comprehensive implementation plan
4. **Write** the plan to both output paths
5. **Return** a brief summary to the orchestrator

## Plan Structure

Your plan should include these sections:

### Overview
1-2 paragraphs summarizing what will be built and the high-level approach.

### Goals and Non-Goals

**Goals:**
- List specific, measurable outcomes this plan aims to achieve
- Derived directly from the specification

**Non-Goals:**
- Explicitly state what is OUT of scope
- Prevents scope creep in later passes

### Technical Approach
Describe the overall architecture and key technical decisions. Include:
- Technology choices (if not specified, recommend with rationale)
- Architecture patterns
- Integration points
- Key dependencies

**Note on versions:** Use your best knowledge for initial version recommendations. The review agents will verify all versions against live documentation via Context7 in the first review pass and flag any outdated recommendations.

### Implementation Steps
Numbered, actionable steps. Each step should:
- Be specific enough to execute
- Have clear completion criteria
- Include estimated complexity (Low/Medium/High)

Example:
1. **Create database schema** (Medium)
   - Define User table with fields: id, email, name, created_at
   - Add indexes on email for lookup performance
   - Completion: Migration runs successfully

### Files to Modify/Create
List all files that will be created or modified:
- `path/to/file.ts` - Description of changes
- `path/to/new-file.ts` - New file: purpose

### Testing Strategy
How the implementation will be verified:
- Unit tests for...
- Integration tests for...
- Manual verification steps

### Open Questions
Any ambiguities in the spec that need clarification. Format as questions for the user.

## Writing Guidelines

- Be specific and actionable, not vague
- Include enough detail that another agent could execute the plan
- **Target 200-500 lines.** If the plan exceeds 500 lines, consider splitting into phases or using a summary+details structure. Overly long plans overwhelm downstream reviewers.
- Reference the spec requirements explicitly
- Don't add features not in the spec
- If the spec is ambiguous, note it in Open Questions rather than assuming

## Return Format (to Orchestrator)

After writing the plan files, return ONLY this brief summary:

1. **Plan Overview**: 1-2 sentences on what the plan covers
2. **Step Count**: "Plan includes N implementation steps"
3. **Key Decisions**: 1-2 major technical decisions made
4. **Open Questions**: Number of questions for user (if any)
5. **Concerns**: Any immediate concerns or blockers

DO NOT return the full plan content - it is saved to the files.
```

---

## Variable Placeholders

When using this template, replace:

| Placeholder | Source |
|-------------|--------|
| `{SPEC_PATH}` | Full path to `initial_spec.md` |
| `{PLAN_PATH}` | Full path to `plan.md` |
| `{AUDIT_PATH}` | Full path to `audit/plan_v0.md` |

## Task Tool Configuration

When spawning the generation agent:

```
Task tool parameters:
- subagent_type: "general-purpose"
- description: "Generate initial plan"
- prompt: [Filled template above with file paths]
```

**Why general-purpose?**
- Needs file read/write access
- Returns structured summary without execution prompts
- Keeps orchestrator context thin by not returning full plan content
