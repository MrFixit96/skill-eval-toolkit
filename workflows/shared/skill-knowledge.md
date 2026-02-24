---
description: "Knowledge about the skill evaluation framework, format, and improvement methodology"
---

# Skill Evaluation Knowledge

## Skill File Format

Skills are markdown files at `skills/{name}/SKILL.md` with:
- YAML frontmatter between `---` fences containing `name` and `description` fields
- `description` must contain quoted trigger keywords (phrases users would say)
- Body sections: Architecture, Key Features, Quick Examples, Common Pitfalls, Success Criteria, References
- Progressive disclosure: SKILL.md is lean (1,000–3,000 words), details in `references/` subdirectory

## 3-Tier Evaluation System

- **Tier 1** (free, no API): lint-skills.py + score-skills.py + trigger-coverage.py — runs on every push
- **Tier 2** (7 API calls): eval-trigger-semantic.py — LLM judges whether descriptions activate correctly
- **Tier 3** (75+ API calls): eval-skill-effectiveness.py — A/B judge compares skill-augmented vs baseline responses

## Content Depth Scoring (0–100)

| Dimension | Max | What It Measures |
|-----------|-----|-----------------|
| Frontmatter | 10 | Valid YAML, name, description present |
| Triggers | 15 | Keyword density and specificity |
| Body | 15 | Word count, section coverage, depth |
| Code | 15 | Number and quality of code examples |
| References | 15 | Links to reference modules, external sources |
| Examples | 10 | Worked examples with context |
| Sources | 10 | Citations to official documentation |
| Clean | 10 | No TODOs, no placeholder text |

## Improvement Methodology

When a skill scores below threshold (80% lint, 85 score), follow:
1. **Diagnose**: Read eval results, classify failure (inaccuracy HIGH, missing HIGH, insufficient MEDIUM, poor examples MEDIUM)
2. **Research**: Fetch official docs, verify facts before editing
3. **Fix**: Surgical minimal edits — only change what diagnosis identified
4. **Validate**: Re-run Tier 1, confirm lint passes and score maintained/improved

## Critical Rules
- Never add content without researching official docs first
- Never reorganize a skill — only fix what's broken
- Code examples must be complete and runnable
- Preserve all working content
