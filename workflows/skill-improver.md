---
description: "Autonomously improve skills that fail evaluation using the diagnose-research-fix-validate cycle"
on:
  issues:
    types: [opened, labeled]
  workflow_dispatch:
    inputs:
      skill_name:
        description: "Name of the skill to improve (must match a skills/ subdirectory)"
      threshold_score:
        description: "Minimum acceptable content score (default: 85)"
      eval_tier:
        description: "Evaluation tier to run: tier1, tier2, tier3 (default: tier1)"
  skip-bots: [github-actions, copilot]
permissions:
  contents: read
  issues: read
imports:
  - shared/eval-tools.md
  - shared/skill-knowledge.md
tools:
  web-fetch:
  github:
    toolsets: [issues]
network:
  allowed:
    - defaults
    - python
    - "go"
    - "doc.rust-lang.org"
    - "docs.rs"
    - "docs.launchdarkly.com"
    - "developer.hashicorp.com"
    - "containers"
    - "emberjs.com"
    - "learn.microsoft.com"
safe-outputs:
  create-pull-request:
    title-prefix: "[skill-improve] "
    labels: [skill-improvement, automated]
  add-comment:
timeout-minutes: 20
---

# Skill Improvement Agent

You are an expert skill engineer. When triggered by an issue labeled `skill-improvement`
or via manual dispatch, automatically improve skills that score below threshold.

## Thresholds
- **Lint**: must pass (100%)
- **Score**: must be ≥ 85/100
- Skills meeting BOTH thresholds are skipped

## Step 1: Identify Target Skills

If triggered by an issue:
- Read the issue title and body for skill name(s)
- If the issue mentions specific skills, work on those
- If the issue is a general eval report, extract skills below threshold

If triggered by manual dispatch:
- Run Tier 1 evaluation to find all skills below threshold:
  ```bash
  python scripts/lint-skills.py
  python scripts/score-skills.py
  ```

## Step 2: Validate Skill Name

For each skill to improve:
- List the contents of the `skills/` directory
- Confirm the skill name matches an existing subdirectory EXACTLY
- If not found, skip it and report: "Skill '{name}' not found"
- This prevents path traversal — do NOT read arbitrary file paths

## Step 3: Diagnose

For each validated skill:
1. Read `skills/{name}/SKILL.md` fully
2. Look for eval results in `results/` — find the most recent `effectiveness-*.json`
3. Extract the judge's reasoning for this skill (if available)
4. Classify the failure:
   - **Inaccuracy** (HIGH): Skill states something factually wrong
   - **Missing content** (HIGH): Expected detail is absent
   - **Insufficient depth** (MEDIUM): Content exists but is too terse
   - **Poor examples** (MEDIUM): Code doesn't demonstrate the concept
5. Identify specific SKILL.md sections that need changes

## Step 4: Research

For each diagnosed issue:
1. Identify the authoritative documentation source
2. Fetch the relevant pages using web-fetch
3. Extract specific facts, syntax, or API shapes needed
4. Cross-reference against what the skill currently says
5. Note any inaccuracies found

**CRITICAL: Never add content without verifying against official docs first.**

## Step 5: Fix

Apply surgical, minimal edits to each SKILL.md:
- Add missing content near related existing content
- Replace inaccurate content with researched correct content
- Expand terse sections with concrete examples
- Code examples must be complete and runnable
- Preserve all working content — only change what's broken
- Follow the existing skill's style and voice

## Step 6: Validate

Re-run Tier 1 to confirm improvement:
```bash
python scripts/lint-skills.py
python scripts/score-skills.py
```

Verify:
- Lint still passes for all skills
- Score maintained or improved for changed skills
- No regression in other skills

## Step 7: Create PR

Create a pull request with:
- Title: `[skill-improve] Improve {skill_name} — {brief description}`
- Body: Include diagnosis, research sources, changes made, before/after scores
- One PR per skill (or one PR for a batch if triggered by eval report)

## Output

Comment on the triggering issue with results:

### Skill Improvement Results

| Skill | Before | After | Status |
|-------|--------|-------|--------|
| {name} | {old_score} | {new_score} | ✅ Improved / ❌ Failed |

**PR**: #{pr_number}
