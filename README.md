# Plan Refiner

An agent skill that generates and iteratively refines implementation plans through parallel multi-reviewer passes with fresh agent context.

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
┌───────────────────────────┴────────────────────────────┐
│                     REVIEW LOOP (3+ passes)            │
│                                                        │
│ ┌────────────── Agent Team (if available) ───────────┐ │
│ │ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │ │
│ │ │   Standard   │ │ Adversarial  │ │   Custom     │ │ │
│ │ │   Review     │ │   Review     │ │   Review     │ │ │
│ │ │ (alignment,  │ │ (assumptions,│ │ (security,   │ │ │
│ │ │  versions    │ │  failure     │ │  perf, …)    │ │ │
│ │ │  via C7)     │ │  modes)      │ │  (optional)  │ │ │
│ │ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ │ │
│ └────────┼────────────────┼────────────────┼─────────┘ │
│          └────────────────┼────────────────┘           │
│                           ▼                            │
│                ┌─────────────────────┐                 │
│                │  Surface Issues &   │                 │
│                │  Assumptions        │                 │
│                └──────────┬──────────┘                 │
│                           ▼                            │
│                ┌─────────────────────┐                 │
│                │  User Feedback      │                 │
│                │  & Answers Q's      │                 │
│                └──────────┬──────────┘                 │
│                           ▼                            │
│                ┌─────────────────────┐                 │
│                │  Update Agent       │  ← changelog    │
│                │  (checks spec)      │                 │
│                └──────────┬──────────┘                 │
│                           └─────▶▶─────────────────────┤
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │   Finalized Plan    │
                 │  (audit preserved)  │
                 └─────────────────────┘
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

1. **Initial Plan Generation** — A fresh agent creates a comprehensive plan from your specification
2. **Review Loop (3+ passes)** — Each pass creates a fresh review team (when Agent Teams are available) or spawns parallel subagents (fallback). Up to three reviewers run in parallel:
   - **Standard Review** — checks spec alignment, completeness, feasibility, and verifies versions via Context7
   - **Adversarial Review** — challenges assumptions, questions implementation choices, identifies failure modes
   - **Custom Review** (optional) — adds a configurable specialized perspective (security, performance, etc.)
   - Issues and critical assumptions are surfaced to you
   - Your feedback is collected and applied; a changelog is generated for each pass
3. **Finalization** — After 3 passes, you decide when the plan is ready. Reviewers signal convergence when no significant issues remain.

### Key Features

- **Parallel Multi-Reviewer**: Standard, adversarial, and optional custom reviewers run in parallel for independent feedback
- **Custom Reviewer**: Add a specialized reviewer (security, performance, accessibility) configurable per run or globally
- **Version Verification**: Reviewers verify package versions and APIs against live docs via Context7
- **Fresh Perspective**: Each review pass uses new agents with no accumulated context bias
- **Drift Prevention**: All agents re-read the original spec to prevent scope creep
- **User in the Loop**: Issues are surfaced as questions; you provide feedback each pass
- **Audit Trail**: Every plan version is preserved, with a changelog documenting each pass
- **Error Resilience**: Graceful fallbacks when individual reviewers or Context7 are unavailable
- **Agent Teams** (experimental): When available, review agents are organized as a coordinated team per pass with structured task tracking via TaskCreate/TaskUpdate. Fresh team each pass preserves independent perspective. Falls back to parallel subagents automatically when teams are unavailable.

### Output Structure

Plans are saved to `~/.claude/plans/plan-refiner/{spec-slug}/`:

```
{spec-slug}/
├── initial_spec.md                  # Your original specification
├── plan.md                          # Current plan version
├── clarifications.md                # Accumulated Q&A and feedback
├── config.json                      # Custom reviewer settings
├── audit/                           # All plan versions
│   ├── plan_v0.md
│   ├── plan_v1.md
│   └── ...
├── pass_N_feedback.md               # Standard review feedback
├── pass_N_adversarial_feedback.md   # Adversarial review feedback
├── pass_N_custom_feedback.md        # Custom review feedback (if enabled)
└── pass_N_changelog.md              # Changes applied and feedback addressed
```

Per-project preferences are stored at `{project-root}/.claude/plan-refiner/preferences.json`.

## Agent Teams (Experimental)

When Claude Code Agent Teams are available (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`), the skill
organizes each pass's review agents as a coordinated team:

```
Pass N:
  ┌─────────────────── Agent Team ───────────────────┐
  │                                                   │
  │  [Task 1]          [Task 2]          [Task 3]    │
  │  Standard          Adversarial       Custom       │
  │  Reviewer          Reviewer          Reviewer     │
  │  (teammate)        (teammate)        (optional)   │
  │                                                   │
  │  TaskCreate → TaskUpdate(owner) → TaskList(done)  │
  └───────────────────────────────────────────────────┘
         ↓ Team destroyed after each pass (fresh perspective)
```

### How it works

1. At the start of each review pass, `TeamCreate` creates a fresh team
2. Review tasks are created via `TaskCreate` with full review instructions and assigned owners
3. Reviewer teammates are spawned in parallel (tasks are pre-assigned before spawn to avoid race conditions)
4. Teammates execute reviews, write feedback files, and mark tasks completed (5-minute timeout per task)
5. The team is shut down and deleted before proceeding to feedback processing

### Fallback

If Agent Teams are unavailable (env var not set, tool unavailable), the skill
automatically falls back to spawning parallel subagents — the same behavior as
before this feature was added. The fallback is detected on the first review pass
and applies to all subsequent passes in the session.

### Why per-pass teams?

Each review pass creates and destroys its own team rather than reusing a persistent
team. This preserves the skill's core **fresh-perspective principle**: every pass
gets new agents with no accumulated context bias from previous passes.

## License

MIT
