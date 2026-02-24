---
description: "Shared tool and runtime configuration for skill evaluation workflows"
tools:
  bash: ["python", "pip", "cat", "ls", "head", "tail", "grep", "wc", "sort", "echo"]
  edit:
runtimes:
  python:
    version: "3.12"
---

# Evaluation Tools

Common tools and runtimes shared across all skill evaluation workflows.

## Python Scripts Available

- `scripts/lint-skills.py` — Structural lint (frontmatter, sections, references)
- `scripts/score-skills.py` — Content depth scorecard (0–100 across 8 dimensions)
- `scripts/trigger-coverage.py` — Trigger keyword coverage against test queries
- `scripts/eval-trigger-semantic.py` — Tier 2: LLM semantic trigger evaluation
- `scripts/eval-skill-quality.py` — Tier 3: LLM quality judge
- `scripts/eval-skill-effectiveness.py` — Tier 3: A/B effectiveness comparison

## Thresholds

- **Lint**: 100% pass rate required
- **Score**: ≥85/100 per skill, fleet average ≥95
- **Trigger coverage**: 100% pass rate required
- **Tier 3 win rate**: ≥80% across fleet
