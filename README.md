# Plan Refiner

An agent skill that generates and iteratively refines implementation plans through multiple review passes with fresh agent context.

## Flow

```
                      ┌──────────────────────┐
                      │      User Spec       │
                      └──────────┬───────────┘
                                 │
                                 ▼
                      ┌──────────────────────┐
                      │   Generate Initial   │  ← Fresh Agent
                      │        Plan          │
                      └──────────┬───────────┘
                                 │
┌────────────────────────────────┴────────────────────────────────┐
│                     REVIEW LOOP (3+ passes)                     │
│                     (repeat until satisfied)                    │
│                                                                 │
│      ┌─────────────┐           ┌──────────────────┐             │
│      │ Fresh Review│  ───────▶ │  Surface Issues  │             │
│      │    Agent    │           │  & Assumptions   │             │
│      └─────────────┘           └────────┬─────────┘             │
│                                         │                       │
│                                         ▼                       │
│                                ┌──────────────────┐             │
│                                │  User Feedback   │             │
│                                │  & Answers Q's   │             │
│                                └────────┬─────────┘             │
│                                         │                       │
│                                         ▼                       │
│      ┌─────────────┐           ┌──────────────────┐             │
│      │Update Agent │  ◀─────── │    Apply All     │             │
│      │(checks spec)│           │    Feedback      │             │
│      └──────┬──────┘           └──────────────────┘             │
│             │                                                   │
│             └───────────────────────────────────────────────────┤
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
                      ┌──────────────────────┐
                      │    Finalized Plan    │
                      │   (audit preserved)  │
                      └──────────────────────┘
```

## Installation

```bash
npx skills add dreambound/plan-refiner-skill
```

## Usage

Invoke the skill with a specification:

```
/plan-refiner on my feature spec
```

Or provide content directly:

```
/plan-refiner "Build a user authentication system with OAuth support"
```

## How It Works

The skill implements an iterative refinement process:

1. **Initial Plan Generation** - Creates a comprehensive plan from your specification
2. **Review Loop (3+ passes)** - Each pass uses a fresh agent to:
   - Review the plan against the original spec
   - Identify issues and suggest improvements
   - Surface critical assumptions as questions
   - Collect your feedback
   - Apply all changes
3. **Finalization** - After 3 passes, you decide when the plan is ready

### Key Features

- **Fresh Perspective**: Each review pass uses a new agent with no accumulated context bias
- **Drift Prevention**: All agents re-read the original spec to prevent scope creep
- **User in the Loop**: Critical assumptions require your clarification
- **Audit Trail**: Every plan version is preserved in the audit directory

### Output Structure

Plans are saved to `~/.claude/plans/plan-refiner/{spec-slug}/`:

```
{spec-slug}/
├── initial_spec.md      # Your original specification
├── plan.md              # Current plan version
├── clarifications.md    # Accumulated Q&A and feedback
├── audit/               # All plan versions
│   ├── plan_v0.md
│   ├── plan_v1.md
│   └── ...
└── pass_N_feedback.md   # Feedback from each review pass
```

## License

MIT
