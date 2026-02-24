---
description: "GEPA evolutionary optimizer default parameters and configuration"
---

# GEPA Configuration

## Default Parameters

- **population_size**: 3 (candidates per generation)
- **generations**: 5 (evolution cycles)
- **mutation_strategies**: ["add_content", "fix_inaccuracy", "expand_examples", "improve_triggers", "restructure_sections"]

## Pareto Criteria

Multi-objective optimization across:
1. **content_score** ↑ (maximize, weight 0.4)
2. **trigger_coverage** ↑ (maximize, weight 0.2)
3. **tier3_win_rate** ↑ (maximize, weight 0.4)
4. **lint_failures** ↓ (minimize, hard constraint)

## Composite Score Formula

When Tier 3 available:
```
composite = (content_score * 0.4) + (trigger_coverage * 0.2) + (tier3_win_rate * 40)
```

When only Tier 1 available:
```
composite = (content_score * 0.6) + (trigger_coverage * 0.4)
```

## Stop Conditions

Stop evolution early if any candidate meets ALL:
- content_score ≥ 95
- lint_pass = true
- trigger_coverage = 100%
- tier3_win_rate ≥ 90% (if Tier 3 available)

## Resource Limits

- Maximum generations: 10
- Maximum population size: 5
- Maximum Tier 3 evaluations per run: 50
- This workflow is expensive — gate behind manual trigger only

## ASI Reflection Prompt

When reflecting on ASI to generate mutations, the LLM should:
1. Read the parent SKILL.md
2. Read ALL judge reasoning from the previous generation
3. Identify the most actionable criticism
4. Propose mutations that DIRECTLY address the criticism
5. Predict the expected score impact of each mutation
6. Prioritize mutations with highest expected improvement
