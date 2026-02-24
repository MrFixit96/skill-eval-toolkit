# Agentic Workflow Templates

These are GitHub Agentic Workflow templates for the skill-eval-toolkit.

## Installation

```bash
# Prerequisites
gh extension install github/gh-aw
gh aw init

# Add workflows
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-lint
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-eval
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-improver
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-eval-orchestrator
gh aw add MrFixit96/skill-eval-toolkit/workflows/gepa-evaluator
gh aw add MrFixit96/skill-eval-toolkit/workflows/gepa-optimizer

# Configure secrets
gh aw secrets bootstrap
gh secret set MODELS_PAT --body "your-github-models-pat"

# Compile
gh aw compile --strict
```

## Workflows

| Workflow | Description | Phase |
|----------|-------------|-------|
| skill-lint | Structural lint + scoring on push/PR | 1 |
| skill-eval | Full 3-tier evaluation suite | 1 |
| skill-improver | Autonomous improvement agent | 1 |
| skill-eval-orchestrator | Fleet evaluation + worker dispatch | 2 |
| gepa-evaluator | GEPA fitness function | 3 |
| gepa-optimizer | GEPA evolutionary search | 3 |

## Shared Components

| Component | Description |
|-----------|-------------|
| shared/eval-tools.md | Common tool + runtime config |
| shared/skill-knowledge.md | Evaluation framework knowledge |
| shared/gepa-config.md | GEPA default parameters |
