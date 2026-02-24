---
description: "Run structural lint, content depth scoring, and trigger coverage on skill changes"
on:
  push:
    branches: [main]
    paths:
      - 'skills/**'
      - 'scripts/**'
      - 'tests/**'
  pull_request:
    branches: [main]
    paths:
      - 'skills/**'
      - 'scripts/**'
      - 'tests/**'
  workflow_dispatch:
  skip-bots: [github-actions, copilot]
permissions:
  contents: read
  pull-requests: read
imports:
  - shared/eval-tools.md
safe-outputs:
  add-comment:
---

# Skill Quality Lint

Run the Tier 1 skill quality suite on changed skills and report results.

## Steps

1. Run structural lint to check all skills:
   ```bash
   python scripts/lint-skills.py
   ```

2. Run content depth scoring:
   ```bash
   python scripts/score-skills.py
   ```

3. Run trigger coverage tests:
   ```bash
   python scripts/trigger-coverage.py
   ```

4. Analyze the results:
   - If all checks pass: post a summary comment with the fleet average score
   - If any check fails: post a detailed comment listing each failure with the specific issue

## Output Format

Post a comment with this structure:

### ✅ Skill Quality Report (or ❌ if failures)

| Check | Result | Details |
|-------|--------|---------|
| Structural Lint | ✅ 25/25 | All skills pass |
| Content Score | ✅ 98.6 avg | 19 skills at 100 |
| Trigger Coverage | ✅ 40/40 | 100% pass rate |

If any skill scores below 85, flag it:
> ⚠️ **{skill}** scored {score}/100 — consider running the skill improver workflow.
