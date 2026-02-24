---
description: Run skill evaluation locally — Tier 1 (lint + score + triggers) by default, optionally Tier 2/3
argument-hint: "[tier1|tier2|tier3|all]"
---

# Evaluate Skills

Run the skill evaluation suite locally. Defaults to Tier 1 (free, no API calls).

Initial request: $ARGUMENTS

## Determine Tier

Parse `$ARGUMENTS`:
- Empty or `tier1` → Run Tier 1 only
- `tier2` → Run Tier 1 + Tier 2 (semantic triggers, requires MODELS_PAT)
- `tier3` → Run Tier 1 + Tier 3 (LLM judge, requires MODELS_PAT)
- `all` → Run all tiers

## Tier 1: Structural Quality (Free)

**Always runs.** No API calls needed.

```bash
python scripts/lint-skills.py
python scripts/score-skills.py
python scripts/trigger-coverage.py
```

**What it checks:**
- Frontmatter format and required fields
- Section structure matches SKILL_TEMPLATE.md
- Content depth (code examples, references, body length)
- Trigger keyword coverage against test queries
- Score ≥ 50 required to pass (100 = perfect)

**Report format:** Show pass/fail for each skill, highlight any below 80, show fleet average.

---

## Tier 2: Semantic Triggers (7 API calls)

**Runs only if `tier2` or `all` specified.**

Requires `MODELS_PAT` environment variable.

```bash
python scripts/eval-trigger-semantic.py --output results/eval-trigger-latest.json
```

**What it checks:**
- LLM-judged: "Given this user query, would this skill's description cause it to be selected?"
- Tests each skill against its mapped queries AND negative queries
- Measures precision (correct activations) and recall (missed activations)

---

## Tier 3: LLM A/B Judge (75+ API calls)

**Runs only if `tier3` or `all` specified.**

Requires `MODELS_PAT` environment variable.

### Quality Judge
```bash
python scripts/eval-skill-quality.py --output results/eval-quality-latest.json
```

### Effectiveness Comparison
```bash
python scripts/eval-skill-effectiveness.py --max-comparisons 25 --output results/eval-effectiveness-latest.json
```

**What it checks:**
- Side-by-side comparison: your skill vs a baseline LLM response (no skill)
- Judge rates: which response is more helpful, accurate, complete?
- Win/loss/tie per skill — target: ≥80% win rate across fleet

---

## Summary

After all requested tiers complete:
1. Show a summary table: skill name, lint, score, trigger coverage, and (if run) semantic/effectiveness results
2. Highlight skills that need attention (score < 80, lint fail, or judge loss)
3. Suggest running `/improve-skill <name>` for any failing skills

## CI/CD Alternative: Agentic Workflows

These evaluations also run automatically via GitHub Agentic Workflows:

- **On push/PR**: `skill-lint` workflow runs Tier 1 and comments results
- **Weekly + manual**: `skill-eval` workflow runs all 3 tiers and creates an issue report
- **Fleet orchestrator**: `skill-eval-orchestrator` evaluates all 25 skills and dispatches improvement workers

To trigger manually from the CLI:
```bash
gh aw run skill-lint              # Tier 1 only
gh aw run skill-eval-orchestrator # Full fleet eval + auto-dispatch workers
```

## Success Criteria

- [ ] All requested tiers ran without errors
- [ ] Results saved to `results/` directory
- [ ] Summary table displayed with actionable next steps
