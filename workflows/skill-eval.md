---
description: "Run full 3-tier skill evaluation suite — lint, semantic triggers, and A/B effectiveness judge"
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
tools:
  web-fetch:
network:
  allowed:
    - defaults
    - python
secrets:
  MODELS_PAT:
    value: ${{ secrets.MODELS_PAT }}
    description: "GitHub Models API token for Tier 2/3 evaluation"
safe-outputs:
  create-issue:
    title-prefix: "[skill-eval] "
    labels: [skill-evaluation, automated]
    close-older-issues: true
  add-labels:
    allowed: [skill-improvement, needs-attention]
timeout-minutes: 30
---

# Skill Evaluation Suite

Run the complete 3-tier evaluation across all skills and create a detailed report.

## Steps

### Tier 1: Structural Quality (Free)

Run all three structural checks:

```bash
python scripts/lint-skills.py
python scripts/score-skills.py
python scripts/trigger-coverage.py
```

Record results: pass/fail per skill, scores, coverage percentages.

### Tier 2: Semantic Trigger Evaluation

Install dependencies and run semantic evaluation:

```bash
pip install requests
python scripts/eval-trigger-semantic.py --output eval-trigger-results.json
```

This uses the MODELS_PAT secret to call the GitHub Models API.
Record: precision and recall per skill.

### Tier 3: LLM A/B Effectiveness Judge

Run both quality and effectiveness evaluations:

```bash
python scripts/eval-skill-quality.py --output eval-quality-results.json
python scripts/eval-skill-effectiveness.py --max-comparisons 25 --output eval-effectiveness-results.json
```

Record: win/loss/tie per skill, fleet win rate.

## Report

Create an issue with this structure:

### Skill Evaluation Report — {date}

#### Fleet Summary
- **Lint**: {pass}/{total} pass
- **Average Score**: {avg}/100
- **Trigger Coverage**: {coverage}%
- **Tier 3 Win Rate**: {wins}/{total} ({pct}%)

#### Skills Below Threshold

| Skill | Score | Lint | Tier 3 | Action Needed |
|-------|-------|------|--------|--------------|
| {name} | {score} | {pass/fail} | {win/loss} | {diagnosis} |

#### Recommendations
For each skill below threshold (80% lint OR 85 score), explain what needs fixing.

If any skills need improvement, note:
> 💡 Create an issue labeled `skill-improvement` with the skill name to trigger automatic improvement.
