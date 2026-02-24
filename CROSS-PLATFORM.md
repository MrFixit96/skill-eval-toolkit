# Cross-Platform Implementation Guide

> **Appendix to the [Skill Eval Toolkit README](README.md)**
>
> This plugin was built for **GitHub Copilot CLI** + **Agentic Workflows (`gh aw`)**. But the same skill evaluation and improvement patterns can be implemented on other AI coding agent platforms. This guide shows how each workflow maps to equivalent primitives on **Claude Code**, **Gemini CLI**, and **OpenAI Codex CLI**.
>
> For a full feature-by-feature comparison of all four platforms, see the [AI Coding Agent Comparison Matrix](https://github.com/MrFixit96/skill-eval-toolkit/blob/main/docs/ai-coding-agent-comparison.md).

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Workflow Mapping](#workflow-mapping)
  - [1. Skill Lint (CI gate)](#1-skill-lint-ci-gate)
  - [2. Skill Eval Orchestrator (fleet management)](#2-skill-eval-orchestrator-fleet-management)
  - [3. Skill Improver (autonomous fix cycle)](#3-skill-improver-autonomous-fix-cycle)
  - [4. GEPA Optimizer (evolutionary search)](#4-gepa-optimizer-evolutionary-search)
- [Platform-Specific Guides](#platform-specific-guides)
  - [Claude Code](#claude-code-anthropic)
  - [Gemini CLI](#gemini-cli-google)
  - [Codex CLI](#codex-cli-openai)
- [Plugin Compatibility](#plugin-compatibility)
- [Cost Comparison](#cost-comparison)

---

## Architecture Overview

The toolkit implements a 3-phase architecture. Each phase can be ported independently:

```
Phase 1: Individual Workflows         Phase 2: Orchestration        Phase 3: Evolution
┌──────────┐ ┌──────────┐ ┌─────────┐  ┌───────────────┐         ┌──────────────┐
│skill-lint│ │skill-eval│ │skill-   │  │ orchestrator  │         │gepa-optimizer│
│(CI gate) │ │(3-tier)  │ │improver │  │ (fleet-wide)  │         │(Pareto search│
└──────────┘ └──────────┘ └─────────┘  └───────────────┘         └──────────────┘
```

**What stays the same across all platforms:**
- The plugin core (commands, agents, skills) works in Claude Code and Copilot CLI as-is
- The Python eval scripts (`lint-skills.py`, `score-skills.py`, `trigger-coverage.py`) are platform-agnostic
- The SKILL.md format and 3-tier evaluation methodology are universal
- The diagnose → research → fix → validate improvement cycle is the same

**What changes per platform:**
- CI/CD trigger mechanism and workflow syntax
- Agent execution environment and sandboxing
- Output mechanisms (PRs, issues, comments)
- Multi-agent dispatch patterns
- Cost model

---

## Workflow Mapping

### 1. Skill Lint (CI gate)

**What it does:** On push/PR, runs Tier 1 checks (structural lint, content scoring 0–100, trigger keyword coverage) and posts results as a comment.

#### Copilot CLI (current)

```yaml
# .github/workflows/skill-lint.md frontmatter
on:
  push:
    branches: [main]
    paths: [skills/**, scripts/**, tests/**]
  pull_request:
    paths: [skills/**, scripts/**, tests/**]
  workflow_dispatch:
tools: [bash, view, grep, glob]
safe-outputs: [add-comment]
```

#### Claude Code

```yaml
# .github/workflows/skill-lint.yml
name: Skill Lint
on:
  push:
    branches: [main]
    paths: [skills/**, scripts/**, tests/**]
  pull_request:
    paths: [skills/**, scripts/**, tests/**]
  workflow_dispatch:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          prompt: |
            Run Tier 1 skill evaluation:
            1. python scripts/lint-skills.py skills/
            2. python scripts/score-skills.py skills/
            3. python scripts/trigger-coverage.py skills/
            Report pass/fail. Flag any skill scoring below 85.
          allowed_tools: bash,view,glob,grep
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

#### Gemini CLI

```yaml
# .github/workflows/skill-lint.yml
name: Skill Lint
on:
  push:
    branches: [main]
    paths: [skills/**, scripts/**, tests/**]
  pull_request:
    paths: [skills/**, scripts/**, tests/**]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/run-gemini-cli@v1
        with:
          prompt: |
            Run all Tier 1 skill checks and comment the results:
            python scripts/lint-skills.py skills/
            python scripts/score-skills.py skills/
            python scripts/trigger-coverage.py skills/
          gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
```

#### Codex CLI

```yaml
# .github/workflows/skill-lint.yml
name: Skill Lint
on:
  push:
    branches: [main]
    paths: [skills/**, scripts/**, tests/**]
  pull_request:
    paths: [skills/**, scripts/**, tests/**]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: openai/codex-action@v1
        with:
          prompt: |
            Run Tier 1 skill evaluation scripts and report results.
            Flag any skill below 85/100.
          prompt-file: .github/codex/prompts/skill-lint.md
          sandbox: read-only
          safety-strategy: drop-sudo
          openai_api_key: ${{ secrets.OPENAI_API_KEY }}
```

---

### 2. Skill Eval Orchestrator (fleet management)

**What it does:** Weekly, evaluates all skills fleet-wide, identifies failures, and dispatches one improvement worker per failing skill. Creates a summary issue.

#### Copilot CLI (current)

Uses `gh aw` orchestrator → `dispatch-workflow` safe-output to fan out up to 25 `skill-improver` workers. Single markdown workflow with `safe-outputs: [dispatch-workflow, create-issue, close-issue]`.

#### Claude Code

```yaml
# .github/workflows/skill-eval-orchestrator.yml
name: Fleet Orchestrator
on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6 AM
  workflow_dispatch:

jobs:
  orchestrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run fleet evaluation
        uses: anthropics/claude-code-action@v1
        with:
          prompt: |
            1. Run python scripts/lint-skills.py and scripts/score-skills.py
            2. For each skill failing lint or scoring <85:
               - Create a GitHub issue titled "[skill-improve] <name>"
               - Label it "skill-improvement"
            3. Create a summary issue "[skill-fleet] Evaluation Report"
          allowed_tools: bash,view,glob,grep
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}

  # Improvement workers triggered by the "skill-improvement" label
  # via a separate workflow (see Skill Improver below)
```

**Multi-agent alternative using Agent Teams:**

```bash
# Local execution with parallel improvement agents
claude --agent-teams \
  -p "Evaluate all skills in skills/. For each failing skill, spawn a
      sub-agent to run the diagnose-research-fix-validate cycle.
      Create a PR for each improved skill."
```

#### Gemini CLI

```yaml
# Gemini equivalent — uses dispatch workflow pattern
name: Fleet Orchestrator
on:
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:

jobs:
  orchestrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: google-github-actions/run-gemini-cli@v1
        with:
          prompt: |
            Run fleet-wide Tier 1 evaluation. For each failing skill,
            use gh workflow dispatch to trigger skill-improver.yml
            with skill_name input. Create a summary issue.
          gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
```

#### Codex CLI

```typescript
// Using Codex TypeScript SDK for programmatic orchestration
import Codex from '@openai/codex-sdk';

const codex = new Codex();

// Run evaluation
const evalThread = await codex.threads.create();
const evalResult = await codex.threads.run(evalThread.id, {
  prompt: "Run python scripts/lint-skills.py and scripts/score-skills.py. Return JSON with failing skills.",
  sandbox_mode: "read-only"
});

// Parse failing skills and dispatch workers in parallel
const failingSkills = JSON.parse(evalResult.output);
await Promise.all(failingSkills.map(skill =>
  codex.threads.run(codex.threads.create().id, {
    prompt: `Improve ${skill.name}: ${skill.diagnosis}. Create a PR.`,
    sandbox_mode: "workspace-write"
  })
));
```

---

### 3. Skill Improver (autonomous fix cycle)

**What it does:** Given a skill name, runs the diagnose → research → fix → validate cycle: classifies failures, fetches official documentation, applies surgical SKILL.md edits, re-runs Tier 1, and creates a PR with before/after scores.

#### Copilot CLI (current)

Uses `web-fetch` for doc research, explicit network allowlist (python.org, go.dev, rust-lang.org, etc.), and `safe-outputs: [create-pull-request, add-comment]`. The agent reads from shared imports `eval-tools.md` and `skill-knowledge.md`.

#### Claude Code

```yaml
# .github/workflows/skill-improver.yml
name: Skill Improver
on:
  issues:
    types: [labeled]
  workflow_dispatch:
    inputs:
      skill_name: { required: true }
      threshold_score: { default: '85' }

jobs:
  improve:
    if: github.event.label.name == 'skill-improvement' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          prompt: |
            Improve the skill "${{ inputs.skill_name || 'from issue title' }}":
            1. DIAGNOSE: Run eval scripts, parse failures
            2. RESEARCH: Use web-fetch to consult official docs
            3. FIX: Make surgical edits to the SKILL.md only
            4. VALIDATE: Re-run eval, confirm score ≥${{ inputs.threshold_score || 85 }}
            Create a PR with before/after scores.
          allowed_tools: bash,view,edit,web_fetch,glob,grep
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Local equivalent:**

```bash
# Using the plugin directly (same commands work in both Claude Code and Copilot CLI)
/improve-skill golang
```

#### Gemini CLI

```yaml
# Gemini can leverage Google Search grounding for the research phase
- uses: google-github-actions/run-gemini-cli@v1
  with:
    prompt: |
      Improve skill "${{ inputs.skill_name }}".
      Use Google Search to find the latest official documentation for
      the technology this skill covers. Apply fixes and create a PR.
```

**Gemini advantage:** Google Search grounding means the research phase doesn't need explicit `web-fetch` calls — the model can search the web natively during its reasoning.

#### Codex CLI

```yaml
- uses: openai/codex-action@v1
  with:
    prompt-file: .github/codex/prompts/improve-skill.md
    sandbox: workspace-write
    safety-strategy: drop-sudo
    env: |
      SKILL_NAME=${{ inputs.skill_name }}
      THRESHOLD=${{ inputs.threshold_score }}
```

---

### 4. GEPA Optimizer (evolutionary search)

**What it does:** Runs a Genetic-Pareto optimization loop: seeds a population from the current SKILL.md, generates mutations guided by Actionable Side Information (Tier 3 judge reasoning), evaluates fitness, selects Pareto-dominant candidates, and iterates. Creates a PR with the best evolved SKILL.md and a detailed evolution report.

This is the most complex workflow. Multi-agent support matters here.

#### Copilot CLI (current)

Uses `dispatch-workflow` to fan out `gepa-evaluator` workers (one per candidate), collects results via issue comments, runs Pareto selection in the orchestrator agent.

#### Claude Code (Agent Teams)

```bash
# Agent Teams is ideal for GEPA — parallel candidate evaluation
claude --agent-teams \
  -p "Run GEPA optimization on skills/golang/SKILL.md:
      1. Create 3 mutant candidates from the current skill
      2. Each team member evaluates one candidate (run all 3 tiers)
      3. Collect scores, run Pareto selection
      4. Repeat for 5 generations
      5. Create a PR with the winner and an evolution report issue"
```

Each team member gets an isolated git worktree, so candidates don't interfere with each other. This is Claude Code's strongest use case for GEPA.

#### Gemini CLI

Gemini's sequential subagent execution (issue #14963) makes parallel candidate evaluation slow. Best approach: use the GitHub Action to dispatch parallel jobs.

```yaml
# Use matrix strategy for parallel candidate evaluation
jobs:
  generate:
    runs-on: ubuntu-latest
    outputs:
      candidates: ${{ steps.gen.outputs.candidates }}
    steps:
      - uses: google-github-actions/run-gemini-cli@v1
        id: gen
        with:
          prompt: |
            Generate 3 GEPA mutations of ${{ inputs.skill_name }}.
            Output a JSON array of file paths.

  evaluate:
    needs: generate
    strategy:
      matrix:
        candidate: ${{ fromJson(needs.generate.outputs.candidates) }}
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/run-gemini-cli@v1
        with:
          prompt: Evaluate candidate ${{ matrix.candidate }}

  select:
    needs: evaluate
    runs-on: ubuntu-latest
    steps:
      - uses: google-github-actions/run-gemini-cli@v1
        with:
          prompt: Run Pareto selection on evaluation results. Create PR.
```

#### Codex CLI (Multi-Agent + SDK)

```typescript
// Codex multi-agent with config.toml roles
// .codex/config.toml:
// [roles.evaluator]
// model = "o3"
// sandbox_mode = "read-only"
// config_file = ".codex/evaluator-instructions.md"
//
// [roles.mutator]
// model = "gpt-5.3-codex"
// sandbox_mode = "workspace-write"

import Codex from '@openai/codex-sdk';
const codex = new Codex();

for (let gen = 0; gen < 5; gen++) {
  // Mutator generates candidates
  const mutations = await codex.threads.run(thread.id, {
    prompt: `Generate 3 mutations of ${skillName} using ASI: ${asi}`,
    role: "mutator"
  });

  // Evaluators score in parallel
  const scores = await Promise.all(candidates.map(c =>
    codex.threads.run(codex.threads.create().id, {
      prompt: `Evaluate candidate: ${c}`,
      role: "evaluator"
    })
  ));

  // Pareto selection
  const front = paretoSelect(scores);
  if (meetsStopCriteria(front)) break;
}
```

---

## Platform-Specific Guides

### Claude Code (Anthropic)

**What works out of the box:**
- ✅ The plugin (`plugin.json`, commands, agents, skills) installs directly — same format
- ✅ `/eval-skills` and `/improve-skill` commands work identically
- ✅ Both agents (`skill-improver`, `skill-diagnostician`) work as-is
- ✅ The `skill-eval` SKILL.md is auto-invoked

**What needs adaptation:**
- CI/CD: Replace `gh aw` workflows with `claude-code-action@v1` GitHub Actions (examples above)
- Multi-agent: Use Agent Teams instead of orchestrator→dispatch pattern
- Cost: Anthropic API billing per token (no free tier for CI/CD)

**Quick start:**
```bash
# Install the plugin
claude plugin add https://github.com/MrFixit96/skill-eval-toolkit.git

# Use locally (identical to Copilot CLI)
/eval-skills
/improve-skill golang
```

### Gemini CLI (Google)

**What works out of the box:**
- ✅ SKILL.md files work (place in `.gemini/skills/`)
- ✅ Custom agents work (place in `.gemini/agents/`)
- ✅ Python eval scripts are platform-agnostic

**What needs adaptation:**
- Plugin format: Convert `plugin.json` → `gemini-extension.json`
- Commands: Convert markdown commands → TOML format in `commands/`
- CI/CD: Use `google-github-actions/run-gemini-cli` (examples above)
- Multi-agent: No native orchestration — use GitHub Actions matrix strategy

**Gemini-specific advantages:**
- Google Search grounding eliminates the explicit doc-fetch step in skill improvement
- A2A protocol could enable distributed GEPA evaluation across machines
- Browser agent could validate skills that reference web technologies

### Codex CLI (OpenAI)

**What works out of the box:**
- ✅ AGENTS.md can contain skill evaluation instructions
- ✅ Python eval scripts are platform-agnostic
- ✅ Agent roles approximate the improver/diagnostician split

**What needs adaptation:**
- No plugin system — embed instructions in AGENTS.md and `.codex/config.toml`
- No SKILL.md auto-invocation — use the [blog post method](https://blog.fsck.com/2025/10/27/skills-for-openai-codex/) to port skills
- No hooks — can't enforce pre-tool-use policies
- CI/CD: Use `openai/codex-action@v1` (examples above)

**Codex-specific advantages:**
- TypeScript SDK enables fully programmatic GEPA orchestration
- Sandbox modes provide fine-grained security per agent role
- Multi-agent with `max_threads` enables parallel evaluation without CI/CD fan-out

---

## Plugin Compatibility

| Component | Copilot CLI | Claude Code | Gemini CLI | Codex CLI |
|-----------|:-----------:|:-----------:|:----------:|:---------:|
| `plugin.json` manifest | ✅ native | ✅ native | ⚠️ convert | ❌ manual |
| `commands/*.md` | ✅ native | ✅ native | ⚠️ convert to TOML | ❌ N/A |
| `agents/*.md` | ✅ native | ✅ native | ✅ copy to `.gemini/agents/` | ⚠️ use config.toml roles |
| `skills/*/SKILL.md` | ✅ native | ✅ native | ✅ copy to `.gemini/skills/` | ❌ paste into AGENTS.md |
| `workflows/*.md` | ✅ `gh aw compile` | ⚠️ rewrite as Actions | ⚠️ rewrite as Actions | ⚠️ rewrite as Actions |
| Eval scripts (Python) | ✅ | ✅ | ✅ | ✅ |
| Hooks | ✅ `hooks.json` | ✅ `hooks.json` | ✅ extension hooks | ❌ |

**Legend:** ✅ = works as-is | ⚠️ = needs conversion | ❌ = not supported

---

## Cost Comparison

Running the full skill evaluation suite (25 skills, weekly) across platforms:

| Scenario | Copilot CLI | Claude Code | Gemini CLI | Codex CLI |
|----------|:-----------:|:-----------:|:----------:|:---------:|
| **Tier 1 lint (per run)** | Free | ~$0.02 | ~$0.01 | ~$0.03 |
| **Fleet orchestrator (weekly)** | Free | ~$0.50 | ~$0.25 | ~$0.40 |
| **Skill improvement (per skill)** | Free | ~$0.30 | ~$0.15 | ~$0.25 |
| **GEPA optimization (5 gen × 3 pop)** | Free | ~$5.00 | ~$2.50 | ~$4.00 |
| **Monthly total (25 skills)** | **$0** | **~$8–12** | **~$4–6** | **~$7–10** |

> **Note:** Copilot CLI costs are $0 because Agentic Workflows use the Copilot API included in your GitHub Copilot subscription. Other platforms bill per API token. Estimates assume typical skill file sizes (150–200 lines) and standard model pricing as of February 2026.

---

## Getting Started

Choose your path:

1. **Already using Copilot CLI?** → Follow the [main README](README.md) — everything works out of the box.

2. **Using Claude Code?** → Install the plugin directly (`claude plugin add <url>`), then set up `claude-code-action@v1` for CI/CD using the examples above.

3. **Using Gemini CLI?** → Copy agents and skills to `.gemini/`, convert commands to TOML, set up `run-gemini-cli` for CI/CD.

4. **Using Codex CLI?** → Port instructions to AGENTS.md, set up `codex-action@v1` for CI/CD, consider the TypeScript SDK for GEPA.

5. **Want the comparison matrix?** → See [AI Coding Agent Comparison Matrix](docs/ai-coding-agent-comparison.md) for a full 26-capability breakdown of all four platforms.
