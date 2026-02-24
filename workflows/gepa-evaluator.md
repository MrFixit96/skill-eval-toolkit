---
description: "GEPA fitness function — evaluate a single skill candidate and return structured results with ASI"
on:
  workflow_dispatch:
    inputs:
      skill_name:
        description: "Name of the skill to evaluate"
      generation_id:
        description: "GEPA generation number for tracking"
      candidate_id:
        description: "Candidate identifier within the generation"
  skip-bots: [github-actions, copilot]
permissions:
  contents: read
imports:
  - shared/eval-tools.md
  - shared/skill-knowledge.md
  - shared/gepa-config.md
network:
  allowed:
    - defaults
    - python
secrets:
  MODELS_PAT:
    value: ${{ secrets.MODELS_PAT }}
    description: "GitHub Models API token for Tier 3 evaluation"
safe-outputs:
  add-comment:
  upload-asset:
timeout-minutes: 15
---

# GEPA Fitness Evaluator

You are a GEPA (Genetic-Pareto) fitness evaluator. Your job is to evaluate a single
skill candidate and produce structured results that include Actionable Side Information (ASI).

## Context

GEPA optimizes text artifacts (SKILL.md files) using evolutionary search with LLM reflection.
Instead of scalar rewards, GEPA uses full execution traces and judge reasoning as ASI to guide
mutation. You are the fitness function in this pipeline.

## Step 1: Locate the Candidate

Read the skill file at `skills/{skill_name}/SKILL.md`.
If the skill doesn't exist in the `skills/` directory listing, abort with error.

## Step 2: Run Tier 1 Evaluation

```bash
python scripts/lint-skills.py
python scripts/score-skills.py
python scripts/trigger-coverage.py
```

Extract this skill's specific results:
- `lint_pass`: boolean
- `content_score`: integer 0-100
- `trigger_coverage`: percentage

## Step 3: Run Tier 3 Evaluation (if MODELS_PAT available)

```bash
pip install requests
python scripts/eval-skill-effectiveness.py --skill {skill_name} --max-comparisons 5 --output gepa-eval-{generation_id}-{candidate_id}.json
```

Extract:
- `wins`: number of A/B comparisons won
- `losses`: number lost
- `ties`: number tied
- `judge_reasoning`: array of verbatim judge explanations (THIS IS THE ASI)

## Step 4: Produce Structured Output

Create a JSON result:

```json
{
  "skill_name": "{skill_name}",
  "generation_id": "{generation_id}",
  "candidate_id": "{candidate_id}",
  "fitness": {
    "lint_pass": true,
    "content_score": 98,
    "trigger_coverage": 100,
    "tier3_wins": 4,
    "tier3_losses": 1,
    "tier3_ties": 0,
    "composite_score": 94.5
  },
  "asi": {
    "judge_reasoning": ["The skill-augmented response provided more specific..."],
    "lint_issues": [],
    "score_breakdown": {"FM": 10, "Trig": 15, "Body": 15, "...": "..."},
    "improvement_hints": ["Consider adding more concrete code examples for..."]
  }
}
```

The `composite_score` is: `(content_score * 0.4) + (trigger_coverage * 0.2) + (tier3_win_rate * 40)`
If Tier 3 unavailable: `(content_score * 0.6) + (trigger_coverage * 0.4)`

## Step 5: Upload Results

Upload the JSON result as an asset for the optimizer to consume.
Comment with a brief summary: "Gen {gen} Candidate {id}: score={score}, lint={pass/fail}, wins={w}/{total}"
