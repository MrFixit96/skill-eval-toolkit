---
description: "Orchestrate fleet-wide skill evaluation and dispatch improvement workers for failing skills"
on:
  schedule: weekly
  workflow_dispatch:
  skip-bots: [github-actions, copilot]
permissions:
  contents: read
  issues: read
imports:
  - shared/eval-tools.md
  - shared/skill-knowledge.md
network:
  allowed:
    - defaults
    - python
safe-outputs:
  dispatch-workflow:
    workflows: [skill-improver]
    max: 25
  create-issue:
    title-prefix: "[skill-fleet] "
    labels: [skill-evaluation, fleet-report, automated]
    close-older-issues: true
  add-comment:
timeout-minutes: 15
---

# Skill Fleet Evaluation Orchestrator

You are a fleet evaluation manager. Run Tier 1 evaluation across all 25 skills,
identify any below threshold, and dispatch individual improvement workers.

## Step 1: Run Fleet Evaluation

Run the full Tier 1 suite:

```bash
python scripts/lint-skills.py
python scripts/score-skills.py
python scripts/trigger-coverage.py
```

Parse output to build a fleet scorecard — record each skill's lint result, content score, and trigger coverage.

## Step 2: Identify Failing Skills

A skill fails if:
- Lint fails (any structural issue), OR
- Content score < 85/100

List all failing skills with their specific failure reason.

## Step 3: Dispatch Improvement Workers

For each failing skill, dispatch the `skill-improver` workflow with the skill name:
- Use `dispatch-workflow` safe-output
- Pass skill name and current scores as context
- Each worker creates its own PR

Skip dispatching if all skills pass (fleet is healthy).

## Step 4: Create Fleet Report Issue

Create a summary issue with:

### Skill Fleet Evaluation Report — {date}

#### Fleet Health
- **Total Skills**: 25
- **Passing**: {count} ({pct}%)
- **Failing**: {count}
- **Average Score**: {avg}/100
- **Skills at 100**: {count}

#### Detailed Scorecard

| Skill | Lint | Score | Triggers | Status |
|-------|------|-------|----------|--------|
| boundary | ✅ | 100 | ✅ | 🟢 Healthy |
| ... | ... | ... | ... | ... |

#### Workers Dispatched

| Skill | Reason | Worker Status |
|-------|--------|--------------|
| {name} | Score {score} < 85 | 🔄 Dispatched |

If no workers dispatched:
> ✅ All skills meet quality thresholds. No improvements needed.
