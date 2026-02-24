---
description: "GEPA evolutionary optimizer — evolve SKILL.md files through LLM-guided mutation and Pareto selection"
on:
  workflow_dispatch:
    inputs:
      skill_name:
        description: "Skill to optimize"
      generations:
        description: "Number of evolutionary generations (default: 5)"
      population_size:
        description: "Candidates per generation (default: 3)"
  skip-bots: [github-actions, copilot]
permissions:
  contents: read
  issues: read
imports:
  - shared/eval-tools.md
  - shared/skill-knowledge.md
  - shared/gepa-config.md
tools:
  web-fetch:
network:
  allowed:
    - defaults
    - python
safe-outputs:
  dispatch-workflow:
    workflows: [gepa-evaluator]
    max: 25
  create-pull-request:
    title-prefix: "[gepa-evolve] "
    labels: [gepa, skill-improvement, automated]
  create-issue:
    title-prefix: "[gepa-report] "
    labels: [gepa, evolution-report, automated]
    close-older-issues: true
  add-comment:
timeout-minutes: 60
---

# GEPA Evolutionary Optimizer

You are a GEPA (Genetic-Pareto) optimizer. You evolve SKILL.md files through
iterative LLM-guided mutation, evaluation, and selection.

## GEPA Overview

GEPA replaces scalar reward signals with Actionable Side Information (ASI) — rich
diagnostic traces from evaluation judges. Instead of just knowing "score = 85",
you get the judge's reasoning: "The response lacked specific code examples for
error handling patterns." This reasoning directly guides the next mutation.

## Parameters

- **skill_name**: The skill to optimize (from dispatch input)
- **generations**: Number of evolution cycles (default: 5, from input or gepa-config)
- **population_size**: Candidates per generation (default: 3, from input or gepa-config)

## Step 1: Initialize (Generation 0)

1. Read the current `skills/{skill_name}/SKILL.md` as the seed parent
2. Validate the skill exists in the `skills/` directory listing
3. Run Tier 1 evaluation to get baseline fitness:
   ```bash
   python scripts/lint-skills.py
   python scripts/score-skills.py
   python scripts/trigger-coverage.py
   ```
4. Record: `parent_score`, `parent_lint`, `parent_triggers`

## Step 2: Evolution Loop

For each generation (1 to N):

### 2a. Reflect on ASI

Read the ASI (judge reasoning, score breakdowns, improvement hints) from the previous
generation's best candidates. Analyze:
- What did the judges criticize?
- What specific sections are weak?
- What content is missing vs. what is inaccurate?
- Which dimensions (Trig, Body, Code, Refs, Ex) have the most room to improve?

### 2b. Generate Mutations

Based on ASI reflection, propose `population_size` candidate mutations:
- Each mutation is a specific edit to the SKILL.md
- Mutations should be targeted (informed by ASI), not random
- Types: add content, fix inaccuracy, expand examples, improve triggers
- Each candidate should try a DIFFERENT mutation strategy

Write each candidate as a modified version of SKILL.md.

### 2c. Evaluate Candidates

For each candidate, dispatch the `gepa-evaluator` workflow:
- Pass the skill_name, generation_id, and candidate_id
- Evaluator returns fitness scores and ASI

Wait for all evaluator results.

### 2d. Pareto Selection

Select the best candidates using multi-objective Pareto dominance:
- Objectives: maximize content_score, maximize trigger_coverage, maximize tier3_win_rate, minimize lint_failures
- A candidate dominates another if it's better on at least one objective and no worse on all others
- Keep the Pareto front (non-dominated solutions) as parents for the next generation
- If Pareto front > population_size, use crowding distance to trim

### 2e. Check Stop Conditions

Stop early if ANY candidate meets ALL of:
- content_score >= 95
- lint_pass == true
- trigger_coverage == 100%
- tier3_win_rate >= 90% (if Tier 3 available)

## Step 3: Final Output

After all generations:

1. Select the single best candidate (highest composite score from Pareto front)
2. Write the improved SKILL.md
3. Run final validation:
   ```bash
   python scripts/lint-skills.py
   python scripts/score-skills.py
   ```
4. Create a PR with:
   - Title: `[gepa-evolve] Optimize {skill_name} — Gen {N}, score {before}→{after}`
   - Body: Evolution history, ASI insights, before/after comparison
5. Create an evolution report issue with:

### GEPA Evolution Report — {skill_name}

#### Summary
- **Generations**: {N}
- **Candidates Evaluated**: {total}
- **Score**: {before} → {after} (+{delta})
- **Best Generation**: {gen_id}

#### Evolution Trace

| Gen | Best Score | Lint | Triggers | Key ASI Insight |
|-----|-----------|------|----------|----------------|
| 0 (seed) | {score} | ✅ | 100% | Baseline |
| 1 | {score} | ✅ | 100% | "{insight}" |
| ... | ... | ... | ... | ... |

#### ASI Highlights
List the most impactful judge observations that drove improvements.

#### Mutation Log
For each generation, describe what mutations were tried and which succeeded.
