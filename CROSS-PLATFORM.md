# Cross-Platform Implementation Guide

> **Appendix to the [Skill Eval Toolkit README](README.md)**
>
> This plugin was built for **GitHub Copilot CLI** + **Agentic Workflows (`gh aw`)**. But the same skill evaluation and improvement patterns can be implemented on other AI coding agent platforms. This guide shows how each workflow maps to equivalent primitives on **Claude Code**, **Gemini CLI**, **OpenAI Codex CLI**, and **OpenCode**.
>
> For a full feature-by-feature comparison of all five platforms, see the [AI Coding Agent Comparison Matrix](https://github.com/MrFixit96/skill-eval-toolkit/blob/main/docs/ai-coding-agent-comparison.md).

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
  - [OpenCode](#opencode-anomaly)
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

#### OpenCode

```yaml
# .github/workflows/opencode-skill-lint.yml
name: OpenCode Skill Lint
on:
  pull_request:
    paths: ['skills/**']
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anomalyco/opencode/github@latest
        with:
          prompt: |
            Run python scripts/lint-skills.py on all skills in skills/.
            Post a summary of lint results as a PR comment.
            If any skill fails lint, exit with error code 1.
```

**OpenCode advantage:** Can use ANY model provider for the lint check. Configure model in `opencode.json`.

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

#### OpenCode

**Option A — GitHub Action with `workflow_dispatch`:**

```yaml
name: OpenCode Skill Eval Fleet
on:
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anomalyco/opencode/github@latest
        with:
          prompt: |
            Run all 3 evaluation tiers using scripts in scripts/.
            For each skill below threshold (80% lint, 85 score):
              Create a GitHub issue labeled skill-improvement.
            Create a fleet summary issue with all results.
```

**Option B — `opencode serve` for local fleet eval:**

```bash
opencode serve &  # Start HTTP API server
curl http://localhost:3000/api/run -d '{"prompt": "Run lint-skills.py and score all skills"}'
```

**OpenCode advantage:** HTTP API server mode (`opencode serve`) enables integration with ANY CI/CD system, not just GitHub Actions.

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

#### OpenCode

OpenCode has native SKILL.md support, making it ideal for the improvement cycle:

```yaml
name: OpenCode Skill Improver
on:
  issues:
    types: [opened, labeled]
jobs:
  improve:
    if: contains(github.event.issue.labels.*.name, 'skill-improvement')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anomalyco/opencode/github@latest
        with:
          prompt: |
            Read the skill-improvement issue body for the failing skill name and scores.
            Use the skill tool to load skill-eval knowledge.
            Read the failing SKILL.md, diagnose issues, research official docs.
            Edit the SKILL.md to fix all issues.
            Run lint-skills.py to validate.
            Create a PR with the fixes.
```

**OpenCode advantage:** Native `skill` tool loads SKILL.md lazily — no need to cat files manually.

---

### 4. GEPA Optimizer (evolutionary search)

**What it does:** Runs a Genetic-Pareto optimization loop: seeds a population from the current SKILL.md, generates mutations guided by Actionable Side Information (Tier 3 judge reasoning), evaluates fitness, selects Pareto-dominant candidates, and iterates. Creates a PR with the best evolved SKILL.md and a detailed evolution report.

> **GEPA v2:** The optimizer now uses the native `gepa` Python package (`pip install gepa`) with per-dimension Pareto scoring across 8 objectives, multi-task batch mode, and generalization validation. See the [main README GEPA v2 section](README.md#gepa-v2-native-python-api) for scripts, workflows, and usage examples.

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

#### OpenCode

```bash
# Use opencode run in headless mode for each generation
for gen in $(seq 1 5); do
  opencode run "
    Read the current candidate SKILL.md.
    Generate 3 mutations based on prior eval feedback.
    For each mutation:
      1. Write to temp file
      2. Run lint-skills.py and score-skills.py
      3. Record score
    Select the best-scoring mutation as the next generation parent.
    Save the winner to skills/<name>/SKILL.md.
  " --format json > generation_${gen}.json
done
```

**OpenCode advantage:** `--format json` structured output makes it easy to parse generation results in scripts. Multi-provider means you can use GPT-4.1 for mutations and Claude for evaluation.

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

### OpenCode (Anomaly)

**What works out of the box:**
- ✅ Plugin system — `.opencode/plugins/` reads JS/TS modules
- ✅ SKILL.md — Native support, reads from `.opencode/skills/`, `.claude/skills/`, `.agents/skills/`
- ✅ Commands — `.opencode/commands/*.md` with `$ARGUMENTS`
- ✅ Agents — `.opencode/agents/*.md` (primary/subagent)
- ✅ Python eval scripts are platform-agnostic

**What needs adaptation:**
- CI/CD: Use `anomalyco/opencode/github@latest` Action (examples above)
- Config: Create `opencode.json` (project or global) with JSON Schema validation
- Multi-provider: Set different models per task (e.g., cheap model for lint, strong model for improvement)

**OpenCode-specific advantages:**
- Multi-provider model config — use the best model for each task (GPT-4.1 for mutations, Claude for evaluation, etc.)
- HTTP API server mode (`opencode serve`) enables integration with any CI/CD system
- `--format json` structured output simplifies scripting and GEPA orchestration
- Native `skill` tool loads SKILL.md lazily — no manual file reading needed
- LSP integration provides real-time diagnostics during skill editing
- 30+ plugin hook event types for fine-grained automation
- MCP support for both local and remote servers with OAuth
- Apache 2.0 open source — no platform fee

**Quick start:**
```bash
# Install OpenCode
npm i -g opencode-ai
# or: brew install anomalyco/tap/opencode
# or: curl -fsSL https://opencode.ai/install.sh | sh

# Add the GitHub Action to your repo automatically
opencode github install

# Use locally
opencode run "Run python scripts/lint-skills.py on all skills in skills/"
```

---

## Plugin Compatibility

| Component | Copilot CLI | Claude Code | Gemini CLI | Codex CLI | OpenCode |
|-----------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| `plugin.json` manifest | ✅ native | ✅ native | ⚠️ convert | ❌ manual | ✅ native |
| `commands/*.md` | ✅ native | ✅ native | ⚠️ convert to TOML | ❌ N/A | ✅ native |
| `agents/*.md` | ✅ native | ✅ native | ✅ copy to `.gemini/agents/` | ⚠️ use config.toml roles | ✅ native |
| `skills/*/SKILL.md` | ✅ native | ✅ native | ✅ copy to `.gemini/skills/` | ❌ paste into AGENTS.md | ✅ native |
| `workflows/*.md` | ✅ `gh aw compile` | ⚠️ rewrite as Actions | ⚠️ rewrite as Actions | ⚠️ rewrite as Actions | ⚠️ rewrite as Actions |
| Eval scripts (Python) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Hooks | ✅ `hooks.json` | ✅ `hooks.json` | ✅ extension hooks | ❌ | ✅ plugin events (30+ types) |
| MCP | ✅ | ✅ | ✅ | ✅ | ✅ local + remote with OAuth |

**Legend:** ✅ = works as-is | ⚠️ = needs conversion | ❌ = not supported

---

## Cost Comparison

Running the full skill evaluation suite (25 skills, weekly) across platforms:

| Scenario | Copilot CLI | Claude Code | Gemini CLI | Codex CLI | OpenCode |
|----------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| **Tier 1 lint (per run)** | Free | ~$0.02 | ~$0.01 | ~$0.03 | ~$0.01–0.03* |
| **Fleet orchestrator (weekly)** | Free | ~$0.50 | ~$0.25 | ~$0.40 | ~$0.20–0.50* |
| **Skill improvement (per skill)** | Free | ~$0.30 | ~$0.15 | ~$0.25 | ~$0.10–0.30* |
| **GEPA optimization (5 gen × 3 pop)** | Free | ~$5.00 | ~$2.50 | ~$4.00 | ~$2.00–5.00* |
| **Monthly total (25 skills)** | **$0** | **~$8–12** | **~$4–6** | **~$7–10** | **~$3–12*** |

> **Note:** Copilot CLI costs are $0 because Agentic Workflows use the Copilot API included in your GitHub Copilot subscription. Other platforms bill per API token. Estimates assume typical skill file sizes (150–200 lines) and standard model pricing as of February 2026.
>
> *OpenCode costs vary by provider chosen (BYO API key). The tool itself is free (Apache 2.0 open source). CI/CD uses standard GitHub Actions runner costs. Optional OpenCode Zen provides curated model access at additional cost. Range reflects cheapest provider (e.g., Gemini Flash) to most expensive (e.g., Claude Opus).

---

## Cross-Platform Relationship Insights

Understanding how these platforms relate to each other helps you plan migrations and multi-platform strategies.

### Compatibility Triangle — Zero-Conversion Portability

Three platforms share configuration formats, forming a **strong compatibility cluster**:

```
         Claude Code
        /            \
  plugin.json    .claude/skills/
  SKILL.md          CLAUDE.md
      /                  \
Copilot CLI ———————— OpenCode
  .agents/skills/    reads ALL three:
  AGENTS.md          .opencode/, .claude/, .agents/
```

**What this means in practice:**
- A SKILL.md written for this toolkit works on **Copilot CLI**, **Claude Code**, AND **OpenCode** — no changes needed
- Plugin commands/agents built for Copilot CLI install directly on Claude Code via shared `plugin.json`
- OpenCode is the most permissive reader — it consumes skills from all three directory conventions
- Gemini CLI requires conversion (TOML commands, `.gemini/` paths) — medium compatibility
- Codex CLI requires the most porting (no plugins, no skills, instructions via AGENTS.md) — lowest compatibility

### Why Every Platform Can Run These Workflows

All 4 workflows rely only on **universal capabilities** (supported by all 5 platforms):

| Workflow | Required Capabilities | Universal? |
|----------|----------------------|:----------:|
| Skill Lint | CLI Agent + Headless + CI/CD | ✅ 5/5 |
| Orchestrator | CLI Agent + Headless + Scheduled Eval + CI/CD | ✅ 5/5 |
| Skill Improver | CLI Agent + Headless + CI/CD + Skills + Custom Agents | ✅ 5/5 |
| GEPA Optimizer | CLI Agent + Headless + Structured Output* | ⚠️ 3/5 |

> *GEPA benefits from structured JSON output (Claude Code, Gemini CLI, OpenCode) but can work without it by parsing text on Copilot CLI and Codex CLI.

### Platform Leaders — Pick Based on Your Priority

| If you prioritize... | Choose | Why |
|---------------------|--------|-----|
| **Richest hook system** | OpenCode | 30+ plugin events vs 6-8 on others |
| **Multi-agent parallelism** | Claude Code | Agent Teams with git worktree isolation |
| **Security sandbox** | Copilot CLI | AWF safe-outputs + network allowlist |
| **Provider flexibility** | OpenCode | 75+ models from any vendor |
| **Zero vendor lock-in** | Gemini CLI or OpenCode | Both Apache 2.0 open source |
| **Lowest migration effort** | Claude Code | Shared plugin.json, direct install |

### MCP Fills the Gaps

Every platform supports MCP (5/5). Missing a native capability? Add it as an MCP server:

| Gap | MCP Solution | Platforms That Benefit |
|-----|-------------|----------------------|
| Web Search | `web-search` MCP server | Claude, Copilot, OpenCode |
| Browser Agent | Playwright MCP server | Copilot, Codex, OpenCode |
| Extra LLM providers | Provider-specific MCP servers | Copilot (locked to GitHub Models), Codex (locked to OpenAI) |

> For the full 26-capability breakdown with detailed profiles, see the [AI Coding Agent Comparison Matrix](docs/ai-coding-agent-comparison.md#key-relationship-insights).

---

## Getting Started

Choose your path:

1. **Already using Copilot CLI?** → Follow the [main README](README.md) — everything works out of the box.

2. **Using Claude Code?** → Install the plugin directly (`claude plugin add <url>`), then set up `claude-code-action@v1` for CI/CD using the examples above.

3. **Using Gemini CLI?** → Copy agents and skills to `.gemini/`, convert commands to TOML, set up `run-gemini-cli` for CI/CD.

4. **Using Codex CLI?** → Port instructions to AGENTS.md, set up `codex-action@v1` for CI/CD, consider the TypeScript SDK for GEPA.

5. **Using OpenCode?** → Install with `npm i -g opencode-ai`, run `opencode github install` to add the Action, configure models in `opencode.json`. Native plugin/skill/agent support means minimal conversion needed.

6. **Want the comparison matrix?** → See [AI Coding Agent Comparison Matrix](docs/ai-coding-agent-comparison.md) for a full capability breakdown of all five platforms.
