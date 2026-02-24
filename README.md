# Skill Eval Toolkit

A [Copilot CLI](https://githubnext.com/projects/copilot-cli/) plugin for evaluating, diagnosing, and improving AI agent skill files. Codifies the **diagnose → research → fix → validate** improvement cycle as reusable commands, agents, and GitHub Agentic Workflow templates.

**v2.0.0** — Now includes [GitHub Agentic Workflows](https://github.github.io/gh-aw/introduction/overview/) for fully autonomous CI/CD skill quality management.

## Installation

### As a Copilot CLI Plugin

```bash
# Clone into your plugins directory
git clone https://github.com/MrFixit96/skill-eval-toolkit.git \
  ~/.copilot/installed-plugins/skill-eval-toolkit

# Or add as a git submodule in an existing plugins repo
git submodule add https://github.com/MrFixit96/skill-eval-toolkit.git \
  plugins/skill-eval-toolkit
```

Copilot CLI auto-discovers plugins from `~/.copilot/installed-plugins/` and `plugins/` directories — no further configuration needed.

### Agentic Workflows (CI/CD)

To also install the agentic workflow templates into your repository:

```bash
# Prerequisites
gh extension install github/gh-aw
cd your-skills-repo
gh aw init

# Add workflow templates (pick what you need)
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-lint           # Phase 1: lint on push/PR
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-eval           # Phase 1: full 3-tier eval
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-improver       # Phase 1: autonomous fix cycle
gh aw add MrFixit96/skill-eval-toolkit/workflows/skill-eval-orchestrator  # Phase 2: fleet orchestrator
gh aw add MrFixit96/skill-eval-toolkit/workflows/gepa-evaluator       # Phase 3: GEPA fitness function
gh aw add MrFixit96/skill-eval-toolkit/workflows/gepa-optimizer       # Phase 3: evolutionary search

# Configure required secrets
gh aw secrets bootstrap                                    # COPILOT_GITHUB_TOKEN (required)
gh secret set MODELS_PAT --body "your-github-models-pat"   # Tier 2/3 eval (optional)

# Compile and push
gh aw compile
git add .github && git commit -m "Add skill-eval agentic workflows" && git push
```

## What's Included

### Commands

| Command | Description |
|---------|-------------|
| `/eval-skills` | Run Tier 1 validation locally (lint + score + trigger coverage) |
| `/eval-skills tier2` | Also run semantic trigger evaluation (requires `MODELS_PAT`) |
| `/eval-skills tier3` | Run full A/B effectiveness judge |
| `/improve-skill <name>` | Full improvement cycle on a named skill |

### Agents

| Agent | Description |
|-------|-------------|
| `skill-improver` | Reads eval failures, researches official docs, makes surgical fixes, validates |
| `skill-diagnostician` | Analyzes eval results and produces actionable diagnoses with priorities |

### Skills

| Skill | Description |
|-------|-------------|
| `skill-eval` | Reference knowledge about the 3-tier eval framework, scoring rubric, and improvement methodology |

### Agentic Workflows

| Workflow | Trigger | Description | Phase |
|----------|---------|-------------|-------|
| `skill-lint` | push, PR, manual | Structural lint + content scoring + trigger coverage | 1 |
| `skill-eval` | weekly, manual | Full 3-tier evaluation suite with issue report | 1 |
| `skill-improver` | issue label, manual | Autonomous diagnose → research → fix → validate cycle | 1 |
| `skill-eval-orchestrator` | weekly, manual | Fleet-wide eval, dispatches improvement workers per skill | 2 |
| `gepa-evaluator` | manual | GEPA fitness function — evaluates a single skill candidate with ASI | 3 |
| `gepa-optimizer` | manual | GEPA evolutionary search — evolves skills through Pareto selection | 3 |

### Shared Components

| Component | Description |
|-----------|-------------|
| `shared/eval-tools.md` | Common tool and runtime configuration (Python 3.12, bash commands) |
| `shared/skill-knowledge.md` | Evaluation framework knowledge (tiers, scoring, methodology) |
| `shared/gepa-config.md` | GEPA default parameters, Pareto criteria, stop conditions |

## Usage

### Local Commands (Interactive)

```bash
# Run Tier 1 evaluation across all skills
/eval-skills

# Improve a specific skill that's below threshold
/improve-skill golang

# Ask the diagnostician to analyze recent eval results
# (invoke skill-diagnostician agent via task tool or by name)
```

### Agentic Workflows (CI/CD)

```bash
# Trigger lint check manually
gh aw run skill-lint

# Trigger full fleet evaluation
gh aw run skill-eval-orchestrator

# Improve a specific skill via workflow dispatch
gh aw run skill-improver -- --skill_name golang --threshold_score 85

# Run GEPA evolutionary optimization on a skill
gh aw run gepa-optimizer -- --skill_name rust --generations 5 --population_size 3

# Check workflow status
gh aw status
```

### Automated Triggers

Once installed and compiled, workflows run automatically:

- **On push/PR** (paths: `skills/**`, `scripts/**`, `tests/**`): `skill-lint` runs Tier 1 checks and comments results
- **Weekly**: `skill-eval-orchestrator` evaluates all skills and dispatches improvement workers for any below threshold
- **On issue label** (`skill-improvement`): `skill-improver` autonomously fixes the named skill and creates a PR

## The Improvement Cycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Diagnose │───→│ Research  │───→│   Fix    │───→│ Validate │
│          │    │          │    │          │    │          │
│ Parse    │    │ Fetch    │    │ Surgical │    │ Lint +   │
│ judge    │    │ official │    │ edits to │    │ score +  │
│ reasoning│    │ docs for │    │ SKILL.md │    │ trigger  │
│ from eval│    │ accuracy │    │ only     │    │ coverage │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
      ↑                                               │
      └───────────────────────────────────────────────┘
                    Re-evaluate if needed
```

## 3-Tier Evaluation System

| Tier | Cost | What It Measures | Scripts |
|------|------|-----------------|---------|
| **1** | Free (no API) | Structure, content depth (0–100), trigger keyword coverage | `lint-skills.py`, `score-skills.py`, `trigger-coverage.py` |
| **2** | 7 API calls | Semantic trigger accuracy — LLM judges description activation | `eval-trigger-semantic.py` |
| **3** | 75+ API calls | A/B effectiveness — LLM compares skill-augmented vs baseline responses | `eval-skill-effectiveness.py` |

**Thresholds**: Skills must pass lint (100%) and score ≥85/100. Fleet target: ≥95 average, ≥80% Tier 3 win rate.

## GEPA Evolutionary Optimization (Phase 3)

GEPA (Genetic-Pareto optimizer) evolves skill files through LLM-guided mutation and multi-objective selection. Instead of scalar rewards, it uses **Actionable Side Information (ASI)** — the judge's full reasoning from Tier 3 evaluation — to guide mutations.

```
Seed SKILL.md → [Mutate based on ASI] → Evaluate → Pareto Select → Repeat
```

- **Population**: 3 candidates per generation (configurable)
- **Generations**: 5 cycles (configurable)
- **Selection**: Multi-objective Pareto dominance (content score ↑, trigger coverage ↑, win rate ↑, lint failures ↓)
- **Stop early**: When any candidate hits score ≥95, lint pass, 100% triggers, ≥90% win rate

> ⚠️ GEPA workflows are expensive. They're gated behind manual `workflow_dispatch` only.

## Prerequisites

- [GitHub Copilot CLI](https://githubnext.com/projects/copilot-cli/) v0.0.415+
- [gh CLI](https://cli.github.com/) v2.50+ (for agentic workflows)
- [gh-aw extension](https://github.com/github/gh-aw) v0.50+ (for agentic workflows)
- Python 3.12+ (for eval scripts)
- Repository secrets:
  - `COPILOT_GITHUB_TOKEN` — Fine-grained PAT with `Copilot Requests: Read-only` (required for agentic workflows)
  - `MODELS_PAT` — GitHub Models API token (optional, for Tier 2/3 evaluation)

## Repository Structure

```
skill-eval-toolkit/
├── .claude-plugin/
│   └── plugin.json           # Plugin manifest (v2.0.0)
├── agents/
│   ├── skill-improver.md     # Autonomous improvement agent
│   └── skill-diagnostician.md # Eval result analysis agent
├── commands/
│   ├── eval-skills.md        # /eval-skills command
│   └── improve-skill.md      # /improve-skill command
├── skills/
│   └── skill-eval/
│       └── SKILL.md          # Eval framework knowledge
├── workflows/
│   ├── shared/
│   │   ├── eval-tools.md     # Common tool config (import)
│   │   ├── skill-knowledge.md # Eval knowledge (import)
│   │   └── gepa-config.md    # GEPA parameters (import)
│   ├── skill-lint.md         # Phase 1: lint on push/PR
│   ├── skill-eval.md         # Phase 1: 3-tier eval suite
│   ├── skill-improver.md     # Phase 1: autonomous fix cycle
│   ├── skill-eval-orchestrator.md # Phase 2: fleet orchestrator
│   ├── gepa-evaluator.md     # Phase 3: fitness function
│   ├── gepa-optimizer.md     # Phase 3: evolutionary search
│   └── README.md             # Workflow-specific docs
├── .gitignore
├── LICENSE                   # MIT
└── README.md                 # This file
```

## Security

- Agentic workflows run in **read-only sandboxed containers** with firewall (AWF)
- All writes go through **safe-outputs** (create-pull-request, create-issue, add-comment) — never direct pushes
- Skill names are **validated against the `skills/` directory listing** before any file operations to prevent path traversal
- Network access is restricted to an **explicit domain allowlist** per workflow
- Bot-triggered runs are skipped via `skip-bots` to prevent infinite loops

## License

MIT — see [LICENSE](LICENSE)
