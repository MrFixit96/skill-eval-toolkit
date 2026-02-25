# Cross-Platform Implementation Guide

> **Appendix to the [Skill Eval Toolkit README](README.md)**
>
> This plugin was built for **GitHub Copilot CLI** + **Agentic Workflows (`gh aw`)**. But the same skill evaluation and improvement patterns can be implemented on other AI coding agent platforms. This guide shows how each workflow maps to equivalent primitives on **Claude Code**, **Gemini CLI**, **OpenAI Codex CLI**, **OpenCode**, and cloud-native platforms (**AWS Bedrock Agents**, **Google Vertex AI Agent Builder**, **Azure AI Agent Service**).
>
> For a full feature-by-feature comparison of all five platforms, see the [AI Coding Agent Comparison Matrix](https://github.com/MrFixit96/skill-eval-toolkit/blob/main/docs/ai-coding-agent-comparison.md).
>
> For a full cloud-native platform comparison, see the [Cloud-Native Agent Comparison Matrix](docs/cloud-agent-comparison.md).

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
- [Cloud-Native Agent Platforms](#cloud-native-agent-platforms)
  - [Architecture Comparison](#architecture-comparison)
  - [Cloud Workflow Mapping](#cloud-workflow-mapping)
  - [Cloud Platform-Specific Guides](#cloud-platform-specific-guides)

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

## Cloud-Native Agent Platforms

The platforms covered so far — Copilot CLI, Claude Code, Gemini CLI, Codex CLI, and OpenCode — are **CLI-first** tools: local development agents that run on a developer's machine, read and write files on disk, and integrate into CI/CD via GitHub Actions. They are optimized for interactive coding, code review, and repository-scoped automation.

**Cloud-native agent platforms** solve a different problem. They are fully managed services for building, deploying, and operating **production agent workloads** — customer-facing assistants, autonomous data pipelines, multi-step reasoning systems — that run 24/7 behind an API. The three major platforms are:

| Platform | Provider | SDK | GA API Version |
|----------|----------|-----|----------------|
| **AWS Bedrock Agents** | Amazon | `boto3` (`bedrock-agent`, `bedrock-agent-runtime`) | Current |
| **Vertex AI Agent Builder** | Google | `google-cloud-aiplatform[agent_engines,adk]` | Current |
| **Azure AI Agent Service** (now Foundry Agent Service) | Microsoft | `azure-ai-projects` / `azure-ai-agents` | `2025-05-01` |

---

### Architecture Comparison

CLI-first and cloud-native platforms follow fundamentally different execution models:

```
CLI-First (local dev tools):
  Developer → CLI Agent → Local Filesystem → CI/CD Workflow → GitHub PR/Issue

Cloud-Native (managed services):
  API Call → Managed Runtime → Orchestration Loop → Tool Execution → Response/Callback
```

In the CLI-first model, the developer is in the loop — the agent reads code, makes edits, and produces artifacts (PRs, comments) via the developer's terminal or a CI runner. In the cloud-native model, the agent is a **deployed service**: it receives requests via API, executes tools server-side (Lambda functions, code interpreters, knowledge base queries), and returns structured responses to calling applications.

```
                    CLI-First                          Cloud-Native
                ┌──────────────┐                  ┌───────────────────┐
  Trigger:      │ Developer /  │                  │ API / Event /     │
                │ CI workflow  │                  │ Schedule trigger  │
                └──────┬───────┘                  └────────┬──────────┘
                       │                                   │
  Runtime:      ┌──────▼───────┐                  ┌────────▼──────────┐
                │ Local process│                  │ Managed runtime   │
                │ (CLI binary) │                  │ (serverless)      │
                └──────┬───────┘                  └────────┬──────────┘
                       │                                   │
  Tools:        ┌──────▼───────┐                  ┌────────▼──────────┐
                │ bash, edit,  │                  │ Lambda/Functions, │
                │ grep, web    │                  │ KB/RAG, Code Exec │
                └──────┬───────┘                  └────────┬──────────┘
                       │                                   │
  Output:       ┌──────▼───────┐                  ┌────────▼──────────┐
                │ Files, PRs,  │                  │ API response,     │
                │ comments     │                  │ callback, stream  │
                └──────────────┘                  └───────────────────┘
```

---

### When to Use CLI-First vs Cloud-Native

| Use Case | CLI-First | Cloud-Native | Notes |
|----------|:---------:|:------------:|-------|
| **Interactive development** | ✅ Primary | ❌ Not designed for | CLI agents are built for real-time developer interaction |
| **Code review & PR automation** | ✅ Native | ❌ Not applicable | Cloud platforms don't operate on git repositories |
| **CI/CD pipeline automation** | ✅ Via GitHub Actions | ✅ Via API triggers | Both work; CLI-first is simpler for repo-scoped tasks |
| **Production customer-facing agents** | ⚠️ Limited | ✅ Primary | Cloud platforms provide managed scaling, SLAs, and monitoring |
| **Knowledge base / RAG** | ⚠️ Manual (web-fetch) | ✅ Built-in | All three cloud platforms offer managed vector stores and retrieval |
| **Multi-agent orchestration** | ⚠️ CI fan-out / Agent Teams | ✅ Native (supervisor/worker) | Bedrock, Vertex, and Azure all support hierarchical multi-agent patterns |
| **Guardrails & content safety** | ⚠️ Prompt-level only | ✅ Managed filters | Bedrock Guardrails, Vertex Model Armor, Azure Content Safety — all configurable |
| **Enterprise auth & network isolation** | ⚠️ GitHub token scopes | ✅ IAM / Entra ID / VPC | Cloud platforms integrate with enterprise identity and network controls |
| **Session memory across conversations** | ❌ Ephemeral | ✅ Managed persistence | Bedrock session summaries, Vertex Memory Bank, Azure thread persistence |
| **Observability & tracing** | ⚠️ CI logs only | ✅ Native (CloudWatch, Cloud Trace, App Insights) | Cloud platforms emit structured traces with token usage and step-level detail |
| **Cost tracking & billing** | ✅ Subscription-based | ⚠️ Per-token + infra | CLI tools have predictable costs; cloud platforms require usage monitoring |
| **Rapid prototyping** | ✅ Immediate | ⚠️ Setup overhead | CLI agents start instantly; cloud platforms require project/resource provisioning |

> **Rule of thumb:** Use CLI-first platforms when the agent's job is to **help a developer write and ship code**. Use cloud-native platforms when the agent's job is to **serve end users or run autonomous workflows in production**.

---

### Cloud Platform Feature Summary

Each cloud platform brings distinct strengths to production agent workloads:

| Capability | AWS Bedrock Agents | Vertex AI Agent Builder | Azure Foundry Agent Service |
|------------|:------------------:|:-----------------------:|:---------------------------:|
| **Tool registration** | OpenAPI + Lambda, Function Schema, Return-of-Control | Python functions (ADK), OpenAPI, MCP, 100+ connectors | Code Interpreter, File Search, Bing, Azure Functions, OpenAPI, MCP |
| **Knowledge base / RAG** | ✅ Managed KB (OpenSearch, Aurora, Pinecone, Kendra) | ✅ Google Search grounding, Vertex AI Search, RAG Engine | ✅ File Search vector stores, Azure AI Search |
| **Multi-agent** | ✅ Supervisor/Worker (hierarchical) | ✅ Sub-agents (ADK), A2A protocol | ✅ Connected Agents, Workflows (preview) |
| **Memory** | Session summaries (1–365 day retention) | Memory Bank (long-term, similarity search) | Threads (persistent until deleted, Cosmos DB) |
| **Orchestration** | ReAct loop, custom Lambda orchestration | Sequential, Parallel, Loop agents (ADK) | Server-side run model, polling/streaming |
| **Guardrails** | ✅ Bedrock Guardrails (content, PII, grounding, reasoning) | ✅ Model Armor, VPC-SC, agent threat detection | ✅ Content filters, XPIA protection |
| **Observability** | Traces + CloudWatch + model invocation logging | Cloud Trace + Cloud Logging + OpenTelemetry | OpenTelemetry + Application Insights |
| **Auth model** | IAM service roles + resource policies | IAM + Agent Identity (SPIFFE) + deny policies | Entra ID (RBAC only, no API keys) |
| **Deployment** | Serverless, versioned aliases | Agent Engine (Cloud Run), auto-scaling | Foundry projects, Basic/Standard setup |
| **SDK languages** | Python (boto3) | Python, TypeScript, Go, Java | Python, TypeScript/JS, .NET, Java |

---

For a full feature comparison across all three cloud-native platforms — including API surface details, pricing models, and migration considerations — see the [Cloud-Native Agent Comparison Matrix](docs/cloud-agent-comparison.md).


## Cloud Workflow Mapping

### Cloud: 1. Skill Lint (CI gate)

**What it does:** On push/PR, runs Tier 1 checks (structural lint, content scoring 0–100, trigger keyword coverage) and returns pass/fail. Cloud agents receive the skill file path, invoke a code-interpreter or function tool to run lint scripts, and parse the structured result.

#### AWS Bedrock Agents

✅ Code Interpreter runs Python natively in a sandbox
✅ Return-of-control lets the caller parse structured results
⚠️ Agent must be pre-created and prepared (`prepare_agent`) before first invocation

```python
import boto3, json, uuid

bedrock = boto3.client("bedrock-agent")
runtime = boto3.client("bedrock-agent-runtime")

# --- Build-time: create a lint agent with Code Interpreter ---
agent = bedrock.create_agent(
    agentName="skill-lint-agent",
    foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
    agentResourceRoleArn="arn:aws:iam::123456789012:role/BedrockAgentRole",
    instruction=(
        "You are a CI lint agent. When given a skill file path, "
        "use Code Interpreter to run: "
        "  python scripts/lint-skills.py <path> && "
        "  python scripts/score-skills.py <path>\n"
        "Return a JSON object: {\"pass\": bool, \"score\": int, \"errors\": [str]}"
    ),
)
agent_id = agent["agent"]["agentId"]

# Attach Code Interpreter built-in action group
bedrock.create_agent_action_group(
    agentId=agent_id,
    agentVersion="DRAFT",
    actionGroupName="CodeInterpreter",
    parentActionGroupSignature="AMAZON.CodeInterpreter",
    actionGroupState="ENABLED",
)
bedrock.prepare_agent(agentId=agent_id)

# --- Runtime: invoke lint on a skill path ---
session_id = str(uuid.uuid4())
response = runtime.invoke_agent(
    agentId=agent_id,
    agentAliasId="TSTALIASID",
    sessionId=session_id,
    enableTrace=True,
    inputText="Lint the skill at skills/golang/SKILL.md",
)

# Parse the streamed response for the final answer
result_text = ""
for event in response["completion"]:
    if "chunk" in event:
        result_text += event["chunk"]["bytes"].decode("utf-8")

lint_result = json.loads(result_text)
assert lint_result["pass"], f"Lint failed: {lint_result['errors']}"
print(f"Score: {lint_result['score']}/100")
```

#### Google Vertex AI Agent Builder

✅ ADK `Agent` with inline Python tool functions — simplest integration
✅ SequentialAgent can chain lint + score steps
⚠️ Requires `pip install google-cloud-aiplatform[agent_engines,adk]`

```python
import vertexai, json
from google.adk.agents import Agent, SequentialAgent
from vertexai import agent_engines

client = vertexai.Client(project="my-project", location="us-central1")

# --- Define lint tool as a plain Python function ---
def run_skill_lint(skill_path: str) -> dict:
    """Run structural lint and content scoring on a SKILL.md file.
    Returns {"pass": bool, "score": int, "errors": [str]}.
    """
    import subprocess
    lint = subprocess.run(
        ["python", "scripts/lint-skills.py", skill_path],
        capture_output=True, text=True,
    )
    score = subprocess.run(
        ["python", "scripts/score-skills.py", skill_path],
        capture_output=True, text=True,
    )
    errors = [l for l in lint.stdout.splitlines() if "FAIL" in l]
    score_val = int(score.stdout.strip().split(":")[-1])
    return {"pass": len(errors) == 0 and score_val >= 85, "score": score_val, "errors": errors}

# --- Create lint agent ---
lint_agent = Agent(
    model="gemini-2.5-flash",
    name="skill_lint_agent",
    instruction=(
        "You are a CI lint agent. Call run_skill_lint with the skill path "
        "the user provides. Return the JSON result unchanged."
    ),
    tools=[run_skill_lint],
)

# --- Deploy to Agent Engine for CI usage ---
app = agent_engines.AdkApp(agent=lint_agent)
remote = client.agent_engines.create(
    agent=app,
    config={
        "requirements": ["google-cloud-aiplatform[agent_engines,adk]"],
        "display_name": "Skill Lint Agent",
    },
)

# --- Invoke remotely ---
response = remote.query(
    user_id="ci-pipeline",
    message="Lint the skill at skills/golang/SKILL.md",
)
lint_result = json.loads(response["output"])
assert lint_result["pass"], f"Lint failed: {lint_result['errors']}"
print(f"Score: {lint_result['score']}/100")
```

#### Azure AI Agent Service

✅ Code Interpreter executes Python in a sandbox — no Lambda/Cloud Function needed
✅ Thread model persists results for audit trail
⚠️ File search / vector stores not needed here; Code Interpreter is the right tool

```python
import os, json
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.agents.models import CodeInterpreterTool

project = AIProjectClient(
    endpoint=os.environ["PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

# --- Create lint agent with Code Interpreter ---
code_tool = CodeInterpreterTool()
agent = project.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="skill-lint-agent",
    instructions=(
        "You are a CI lint agent. When given a skill file path, use Code Interpreter "
        "to run lint-skills.py and score-skills.py from the scripts/ directory. "
        "Return JSON: {\"pass\": bool, \"score\": int, \"errors\": [str]}"
    ),
    tools=code_tool.definitions,
    tool_resources=code_tool.resources,
)

# --- Create thread and run ---
thread = project.agents.threads.create()
project.agents.messages.create(
    thread_id=thread.id,
    role="user",
    content="Lint the skill at skills/golang/SKILL.md",
)
run = project.agents.runs.create_and_process(
    thread_id=thread.id,
    agent_id=agent.id,
)

# --- Parse result from assistant message ---
messages = project.agents.messages.list(thread_id=thread.id)
for msg in messages:
    if msg.role == "assistant":
        lint_result = json.loads(msg.content[0].text.value)
        break

assert lint_result["pass"], f"Lint failed: {lint_result['errors']}"
print(f"Score: {lint_result['score']}/100")

# Cleanup
project.agents.threads.delete(thread.id)
project.agents.delete_agent(agent.id)
```

---

### Cloud: 2. Orchestrator (fleet management)

**What it does:** Weekly, evaluates all 25 skills fleet-wide, identifies failures, and dispatches one improvement worker per failing skill. Collects and aggregates results into a summary report.

#### AWS Bedrock Agents

✅ Multi-agent collaboration (Supervisor/Worker) is a native primitive
✅ Supervisor routes to collaborator agents per-skill
⚠️ Collaborators are invoked synchronously — no built-in parallel fan-out across workers
⚠️ Each collaborator must be a fully created agent with its own alias

```python
import boto3, json, uuid

bedrock = boto3.client("bedrock-agent")
runtime = boto3.client("bedrock-agent-runtime")

SKILLS = [
    "golang", "terraform", "vault", "consul", "docker",
    "kubernetes", "aws", "azure", "gcp", "nomad",
    "boundary", "waypoint", "ember", "hds", "github",
    "python", "rust", "sentinel", "packer", "vagrant",
    "react", "typescript", "java", "csharp", "ruby",
]

# --- Build-time: create an evaluator worker agent ---
evaluator = bedrock.create_agent(
    agentName="skill-evaluator-worker",
    foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
    agentResourceRoleArn="arn:aws:iam::123456789012:role/BedrockAgentRole",
    instruction=(
        "You evaluate a single skill. Run lint-skills.py and score-skills.py "
        "on the skill path provided. Return JSON: "
        "{\"skill\": str, \"score\": int, \"pass\": bool, \"errors\": [str]}"
    ),
)
evaluator_id = evaluator["agent"]["agentId"]
bedrock.create_agent_action_group(
    agentId=evaluator_id, agentVersion="DRAFT",
    actionGroupName="CodeInterpreter",
    parentActionGroupSignature="AMAZON.CodeInterpreter",
    actionGroupState="ENABLED",
)
bedrock.prepare_agent(agentId=evaluator_id)
eval_alias = bedrock.create_agent_alias(
    agentId=evaluator_id, agentAliasName="prod",
)

# --- Build-time: create supervisor orchestrator ---
supervisor = bedrock.create_agent(
    agentName="skill-fleet-orchestrator",
    foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
    agentResourceRoleArn="arn:aws:iam::123456789012:role/BedrockAgentRole",
    agentCollaboration="SUPERVISOR",
    instruction=(
        "You orchestrate fleet-wide skill evaluation. For each skill in the "
        "fleet, delegate to the SkillEvaluator collaborator. Collect all "
        "results and produce a summary report with: total skills, passing, "
        "failing, and a list of failing skill names with scores."
    ),
)
supervisor_id = supervisor["agent"]["agentId"]

# Associate evaluator as collaborator
bedrock.associate_agent_collaborator(
    agentId=supervisor_id,
    agentVersion="DRAFT",
    agentDescriptor={"aliasArn": eval_alias["agentAlias"]["agentAliasArn"]},
    collaborationInstruction="Evaluate a single skill by name and return its score.",
    collaboratorName="SkillEvaluator",
)
bedrock.prepare_agent(agentId=supervisor_id)

# --- Runtime: invoke orchestrator ---
session_id = str(uuid.uuid4())
response = runtime.invoke_agent(
    agentId=supervisor_id,
    agentAliasId="TSTALIASID",
    sessionId=session_id,
    enableTrace=True,
    inputText=f"Evaluate all skills: {', '.join(SKILLS)}",
)

report = ""
for event in response["completion"]:
    if "chunk" in event:
        report += event["chunk"]["bytes"].decode("utf-8")

print(report)
```

#### Google Vertex AI Agent Builder

✅ `ParallelAgent` provides native fan-out — ideal for fleet evaluation
✅ `SequentialAgent` chains fan-out → aggregation
✅ Shared session state (`output_key`) collects results without external storage

```python
import vertexai, json
from google.adk.agents import Agent, LlmAgent, ParallelAgent, SequentialAgent
from vertexai import agent_engines

client = vertexai.Client(project="my-project", location="us-central1")

SKILLS = [
    "golang", "terraform", "vault", "consul", "docker",
    "kubernetes", "aws", "azure", "gcp", "nomad",
    "boundary", "waypoint", "ember", "hds", "github",
    "python", "rust", "sentinel", "packer", "vagrant",
    "react", "typescript", "java", "csharp", "ruby",
]

def evaluate_skill(skill_name: str) -> dict:
    """Run Tier 1 evaluation on a single skill. Returns score dict."""
    import subprocess
    path = f"skills/{skill_name}/SKILL.md"
    score_out = subprocess.run(
        ["python", "scripts/score-skills.py", path],
        capture_output=True, text=True,
    )
    lint_out = subprocess.run(
        ["python", "scripts/lint-skills.py", path],
        capture_output=True, text=True,
    )
    score = int(score_out.stdout.strip().split(":")[-1])
    errors = [l for l in lint_out.stdout.splitlines() if "FAIL" in l]
    return {"skill": skill_name, "score": score, "pass": score >= 85 and not errors, "errors": errors}

# --- Create per-skill evaluator agents ---
evaluators = []
for skill in SKILLS:
    evaluators.append(LlmAgent(
        name=f"eval_{skill}",
        model="gemini-2.5-flash",
        instruction=f"Call evaluate_skill('{skill}') and return the result.",
        tools=[evaluate_skill],
        output_key=f"result_{skill}",
    ))

# --- Fan-out: run all evaluators in parallel ---
fan_out = ParallelAgent(name="fleet_fan_out", sub_agents=evaluators)

# --- Aggregator: collect results from shared state ---
aggregator = LlmAgent(
    name="fleet_aggregator",
    model="gemini-2.5-flash",
    instruction=(
        "Read all result_* keys from session state. Produce a fleet report: "
        "total skills, passing count, failing count, and list each failing "
        "skill with its score and errors."
    ),
    output_key="fleet_report",
)

# --- Pipeline: fan-out → aggregate ---
orchestrator = SequentialAgent(
    name="fleet_orchestrator",
    sub_agents=[fan_out, aggregator],
)

app = agent_engines.AdkApp(agent=orchestrator)
remote = client.agent_engines.create(
    agent=app,
    config={
        "requirements": ["google-cloud-aiplatform[agent_engines,adk]"],
        "display_name": "Fleet Orchestrator",
        "max_instances": 10,
    },
)

# --- Invoke ---
response = remote.query(user_id="ci-weekly", message="Run fleet evaluation")
print(response["output"])
```

#### Azure AI Agent Service

✅ Connected Agents provide native multi-agent delegation
⚠️ Max depth of 2 (parent → child only) — sufficient for orchestrator → evaluator
⚠️ No built-in parallel fan-out; children are invoked sequentially by the orchestrator LLM
❌ Connected agents cannot use function calling — must use Code Interpreter or OpenAPI tools

```python
import os, json
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.agents.models import CodeInterpreterTool, ConnectedAgentTool

project = AIProjectClient(
    endpoint=os.environ["PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

SKILLS = [
    "golang", "terraform", "vault", "consul", "docker",
    "kubernetes", "aws", "azure", "gcp", "nomad",
    "boundary", "waypoint", "ember", "hds", "github",
    "python", "rust", "sentinel", "packer", "vagrant",
    "react", "typescript", "java", "csharp", "ruby",
]

# --- Create evaluator child agent ---
code_tool = CodeInterpreterTool()
evaluator = project.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="skill-evaluator",
    instructions=(
        "You evaluate a single skill. Use Code Interpreter to run "
        "scripts/lint-skills.py and scripts/score-skills.py on the given path. "
        "Return JSON: {\"skill\": str, \"score\": int, \"pass\": bool}"
    ),
    tools=code_tool.definitions,
    tool_resources=code_tool.resources,
)

# --- Wrap as connected agent tool ---
connected = ConnectedAgentTool(
    id=evaluator.id,
    name="skill-evaluator",
    description="Evaluates a single skill by path and returns its score.",
)

# --- Create orchestrator parent agent ---
orchestrator = project.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="fleet-orchestrator",
    instructions=(
        f"You manage fleet-wide skill evaluation for {len(SKILLS)} skills. "
        "For each skill, delegate to the skill-evaluator connected agent "
        "with the path skills/<name>/SKILL.md. Collect all results into a "
        "summary: total, passing, failing, and the list of failures."
    ),
    tools=connected.definitions,
)

# --- Execute fleet evaluation ---
thread = project.agents.threads.create()
project.agents.messages.create(
    thread_id=thread.id,
    role="user",
    content=f"Evaluate all skills: {', '.join(SKILLS)}",
)
run = project.agents.runs.create_and_process(
    thread_id=thread.id,
    agent_id=orchestrator.id,
)

# --- Retrieve fleet report ---
messages = project.agents.messages.list(thread_id=thread.id)
for msg in messages:
    if msg.role == "assistant":
        print(msg.content[0].text.value)
        break

# Cleanup
project.agents.threads.delete(thread.id)
project.agents.delete_agent(evaluator.id)
project.agents.delete_agent(orchestrator.id)
```

---

### Cloud: 3. Skill Improver (autonomous fix cycle)

**What it does:** Given a failing skill, runs the diagnose → research → fix → validate cycle. Configures tools for code execution, web search (documentation lookup), and knowledge retrieval. Produces an improved SKILL.md and creates a PR with before/after scores.

#### AWS Bedrock Agents

✅ Code Interpreter for running eval scripts in sandbox
✅ Knowledge Base (RAG) for skill-writing guidelines
✅ Return-of-control enables client-side PR creation via `gh`
⚠️ No built-in web search — use a Lambda action group wrapping an HTTP fetch, or use a KB pre-loaded with docs

```python
import boto3, json, uuid, subprocess

bedrock = boto3.client("bedrock-agent")
runtime = boto3.client("bedrock-agent-runtime")

# --- Build-time: create improver agent ---
agent = bedrock.create_agent(
    agentName="skill-improver",
    foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
    agentResourceRoleArn="arn:aws:iam::123456789012:role/BedrockAgentRole",
    instruction=(
        "You are a skill improvement agent. Follow this cycle:\n"
        "1. DIAGNOSE: Use Code Interpreter to run lint-skills.py and score-skills.py. "
        "Parse errors and identify failure categories.\n"
        "2. RESEARCH: Query the skill-guidelines knowledge base for best practices "
        "matching the failure categories.\n"
        "3. FIX: Use Code Interpreter to read the SKILL.md, apply surgical edits "
        "based on research findings, and write the improved version.\n"
        "4. VALIDATE: Re-run lint and score scripts. If score < threshold, "
        "repeat from step 1 (max 3 iterations).\n"
        "Return the final SKILL.md content and a JSON summary: "
        "{\"before\": int, \"after\": int, \"edits\": [str]}"
    ),
)
agent_id = agent["agent"]["agentId"]

# Attach Code Interpreter
bedrock.create_agent_action_group(
    agentId=agent_id, agentVersion="DRAFT",
    actionGroupName="CodeInterpreter",
    parentActionGroupSignature="AMAZON.CodeInterpreter",
    actionGroupState="ENABLED",
)

# Attach Knowledge Base with skill-writing guidelines
bedrock.associate_agent_knowledge_base(
    agentId=agent_id, agentVersion="DRAFT",
    knowledgeBaseId="KB_SKILL_GUIDELINES_ID",
    description="Skill-writing best practices and SKILL.md format reference.",
)

# Attach a return-of-control action group for PR creation
bedrock.create_agent_action_group(
    agentId=agent_id, agentVersion="DRAFT",
    actionGroupName="CreatePullRequest",
    actionGroupExecutor={"customControl": "RETURN_CONTROL"},
    functionSchema={"functions": [{
        "name": "create_pr",
        "description": "Create a GitHub pull request with the improved skill file.",
        "parameters": {
            "branch_name": {"type": "string", "required": True, "description": "Git branch name"},
            "title": {"type": "string", "required": True, "description": "PR title"},
            "body": {"type": "string", "required": True, "description": "PR body with before/after scores"},
            "file_content": {"type": "string", "required": True, "description": "New SKILL.md content"},
        },
    }]},
    actionGroupState="ENABLED",
)
bedrock.prepare_agent(agentId=agent_id)

# --- Runtime: invoke improver ---
session_id = str(uuid.uuid4())
response = runtime.invoke_agent(
    agentId=agent_id,
    agentAliasId="TSTALIASID",
    sessionId=session_id,
    enableTrace=True,
    inputText="Improve the skill at skills/golang/SKILL.md. Target score: 90.",
)

# Handle return-of-control for PR creation
for event in response["completion"]:
    if "returnControl" in event:
        payload = event["returnControl"]["invocationInputs"][0]["functionInvocationInput"]
        params = {p["name"]: p["value"] for p in payload["parameters"]}

        # Client-side PR creation using gh CLI
        subprocess.run(["git", "checkout", "-b", params["branch_name"]])
        with open(f"skills/golang/SKILL.md", "w") as f:
            f.write(params["file_content"])
        subprocess.run(["git", "add", "."])
        subprocess.run(["git", "commit", "-m", params["title"]])
        subprocess.run(["git", "push", "origin", params["branch_name"]])
        subprocess.run(["gh", "pr", "create",
            "--title", params["title"],
            "--body", params["body"],
        ])
        print(f"PR created: {params['title']}")
    elif "chunk" in event:
        print(event["chunk"]["bytes"].decode("utf-8"), end="")
```

#### Google Vertex AI Agent Builder

✅ Google Search grounding replaces explicit web-fetch — agent searches docs natively
✅ Code execution sandbox runs eval scripts
✅ `LoopAgent` implements the diagnose→fix→validate retry cycle natively
⚠️ LoopAgent terminates on `escalate=True` or max iterations — must signal success in custom agent

```python
import vertexai, json, subprocess
from google.adk.agents import Agent, LlmAgent, LoopAgent, SequentialAgent, BaseAgent
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event, EventActions
from google.genai import types
from vertexai import agent_engines

client = vertexai.Client(project="my-project", location="us-central1")

# --- Tool functions ---
def diagnose_skill(skill_path: str) -> dict:
    """Run lint and score, return diagnosis with error categories."""
    import subprocess as sp
    lint = sp.run(["python", "scripts/lint-skills.py", skill_path], capture_output=True, text=True)
    score = sp.run(["python", "scripts/score-skills.py", skill_path], capture_output=True, text=True)
    score_val = int(score.stdout.strip().split(":")[-1])
    errors = [l for l in lint.stdout.splitlines() if "FAIL" in l]
    return {"score": score_val, "errors": errors, "pass": score_val >= 90 and not errors}

def write_skill_file(skill_path: str, content: str) -> str:
    """Write improved SKILL.md content to disk."""
    with open(skill_path, "w") as f:
        f.write(content)
    return f"Written {len(content)} bytes to {skill_path}"

def create_pull_request(branch: str, title: str, body: str) -> str:
    """Create a git branch, commit changes, and open a PR."""
    import subprocess as sp
    sp.run(["git", "checkout", "-b", branch])
    sp.run(["git", "add", "."])
    sp.run(["git", "commit", "-m", title])
    sp.run(["git", "push", "origin", branch])
    result = sp.run(["gh", "pr", "create", "--title", title, "--body", body], capture_output=True, text=True)
    return result.stdout.strip()

# --- Improvement agent with Google Search grounding ---
improver = LlmAgent(
    name="skill_improver",
    model="gemini-2.5-flash",
    instruction=(
        "You improve a failing SKILL.md. Steps:\n"
        "1. Call diagnose_skill to identify failures.\n"
        "2. Use Google Search to find the latest official docs for the technology.\n"
        "3. Call write_skill_file with the improved content.\n"
        "4. Call diagnose_skill again to validate. If score < 90, repeat.\n"
        "When the score is >= 90, set output_key to the final result."
    ),
    tools=[
        diagnose_skill,
        write_skill_file,
        types.Tool(google_search=types.GoogleSearch()),
    ],
    output_key="improvement_result",
)

# --- Validation check agent ---
class ValidationGate(BaseAgent):
    """Checks if improvement succeeded; escalates to stop the loop."""
    async def _run_async_impl(self, ctx: InvocationContext):
        result = ctx.session.state.get("improvement_result", "{}")
        data = json.loads(result) if isinstance(result, str) else result
        done = data.get("pass", False)
        yield Event(author=self.name, actions=EventActions(escalate=done))

# --- Loop: improve until passing (max 3 attempts) ---
fix_loop = LoopAgent(
    name="fix_validate_loop",
    max_iterations=3,
    sub_agents=[improver, ValidationGate(name="gate")],
)

# --- Final PR creation step ---
pr_agent = LlmAgent(
    name="pr_creator",
    model="gemini-2.5-flash",
    instruction="Call create_pull_request with the improvement results from state.",
    tools=[create_pull_request],
)

pipeline = SequentialAgent(name="skill_improver_pipeline", sub_agents=[fix_loop, pr_agent])

app = agent_engines.AdkApp(agent=pipeline)
remote = client.agent_engines.create(
    agent=app,
    config={
        "requirements": ["google-cloud-aiplatform[agent_engines,adk]"],
        "display_name": "Skill Improver",
    },
)

response = remote.query(
    user_id="ci-improve",
    message="Improve the skill at skills/golang/SKILL.md to score >= 90",
)
print(response["output"])
```

#### Azure AI Agent Service

✅ Code Interpreter for running eval scripts
✅ Bing Grounding for real-time documentation research
✅ Function calling for client-side PR creation
⚠️ Only one instance of each knowledge tool per agent — Bing + File Search cannot be combined on one agent

```python
import os, json, subprocess, time
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.agents.models import (
    CodeInterpreterTool,
    BingGroundingTool,
    FunctionTool,
    FunctionToolDefinition,
    RequiredFunctionToolCall,
    SubmitToolOutputsAction,
    ToolOutput,
)

project = AIProjectClient(
    endpoint=os.environ["PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

# --- Tool definitions ---
code_tool = CodeInterpreterTool()
bing_tool = BingGroundingTool(connection_id=os.environ["BING_CONNECTION_ID"])

create_pr_def = FunctionToolDefinition(
    name="create_pr",
    description="Create a GitHub PR with the improved skill file.",
    parameters={
        "type": "object",
        "properties": {
            "branch": {"type": "string", "description": "Branch name"},
            "title": {"type": "string", "description": "PR title"},
            "body": {"type": "string", "description": "PR body with before/after scores"},
            "skill_path": {"type": "string", "description": "Path to SKILL.md"},
            "content": {"type": "string", "description": "New SKILL.md content"},
        },
        "required": ["branch", "title", "body", "skill_path", "content"],
    },
)
fn_tool = FunctionTool(definitions=[create_pr_def])

# --- Create improver agent ---
agent = project.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="skill-improver",
    instructions=(
        "You improve a failing SKILL.md through a diagnose→research→fix→validate cycle.\n"
        "1. DIAGNOSE: Use Code Interpreter to run scripts/lint-skills.py and "
        "scripts/score-skills.py on the skill path. Parse failures.\n"
        "2. RESEARCH: Use Bing to search for the latest official documentation "
        "for the technology this skill covers.\n"
        "3. FIX: Use Code Interpreter to write an improved SKILL.md.\n"
        "4. VALIDATE: Re-run the eval scripts. If score < target, go to step 1.\n"
        "5. When passing, call create_pr with the improved content."
    ),
    tools=code_tool.definitions + bing_tool.definitions + fn_tool.definitions,
    tool_resources=code_tool.resources,
)

# --- Run improvement cycle ---
thread = project.agents.threads.create()
project.agents.messages.create(
    thread_id=thread.id, role="user",
    content="Improve skills/golang/SKILL.md to score >= 90. Create a PR when done.",
)

run = project.agents.runs.create(thread_id=thread.id, agent_id=agent.id)

# Poll and handle function calls (for create_pr)
while run.status in ("queued", "in_progress", "requires_action"):
    time.sleep(1)
    run = project.agents.runs.get(thread_id=thread.id, run_id=run.id)

    if run.status == "requires_action":
        action = run.required_action
        if isinstance(action, SubmitToolOutputsAction):
            outputs = []
            for call in action.submit_tool_outputs.tool_calls:
                if isinstance(call, RequiredFunctionToolCall) and call.function.name == "create_pr":
                    args = json.loads(call.function.arguments)
                    # Client-side PR creation
                    subprocess.run(["git", "checkout", "-b", args["branch"]])
                    with open(args["skill_path"], "w") as f:
                        f.write(args["content"])
                    subprocess.run(["git", "add", "."])
                    subprocess.run(["git", "commit", "-m", args["title"]])
                    subprocess.run(["git", "push", "origin", args["branch"]])
                    pr = subprocess.run(
                        ["gh", "pr", "create", "--title", args["title"], "--body", args["body"]],
                        capture_output=True, text=True,
                    )
                    outputs.append(ToolOutput(tool_call_id=call.id, output=pr.stdout))
            project.agents.runs.submit_tool_outputs(
                thread_id=thread.id, run_id=run.id, tool_outputs=outputs,
            )

# Print final result
messages = project.agents.messages.list(thread_id=thread.id)
for msg in messages:
    if msg.role == "assistant":
        print(msg.content[0].text.value)
        break

project.agents.threads.delete(thread.id)
project.agents.delete_agent(agent.id)
```

---

### Cloud: 4. GEPA Optimizer (evolutionary search)

**What it does:** Runs Genetic-Pareto optimization: seeds a population from the current SKILL.md, generates mutations guided by Actionable Side Information (ASI from Tier 3 judge reasoning), evaluates fitness in parallel, selects Pareto-dominant candidates, and iterates across generations. This is the most complex workflow — multi-agent parallelism is critical.

#### AWS Bedrock Agents

✅ Multi-agent Supervisor fans out to evaluator collaborators
✅ Session state carries ASI between generations via `sessionAttributes`
⚠️ Collaborators invoked synchronously — true parallel eval requires multiple `invoke_agent` calls from client
⚠️ GEPA library (`pip install gepa`) must run client-side; agent handles mutation + evaluation only

```python
import boto3, json, uuid
from concurrent.futures import ThreadPoolExecutor
# pip install gepa
from gepa import optimize_anything, Individual, Population

bedrock = boto3.client("bedrock-agent")
runtime = boto3.client("bedrock-agent-runtime")

# --- Assume pre-created agents (see §Cloud: 2 for creation pattern) ---
MUTATOR_AGENT_ID = "MUTATOR_AGENT_ID"
MUTATOR_ALIAS_ID = "MUTATOR_ALIAS_ID"
EVALUATOR_AGENT_ID = "EVALUATOR_AGENT_ID"
EVALUATOR_ALIAS_ID = "EVALUATOR_ALIAS_ID"

def invoke_bedrock_agent(agent_id, alias_id, prompt, session_attrs=None):
    """Invoke a Bedrock agent and return the text response."""
    session_id = str(uuid.uuid4())
    kwargs = {
        "agentId": agent_id,
        "agentAliasId": alias_id,
        "sessionId": session_id,
        "inputText": prompt,
    }
    if session_attrs:
        kwargs["sessionState"] = {"sessionAttributes": session_attrs}
    response = runtime.invoke_agent(**kwargs)
    text = ""
    for event in response["completion"]:
        if "chunk" in event:
            text += event["chunk"]["bytes"].decode("utf-8")
    return text

def mutate_skill(skill_content: str, asi: str) -> str:
    """Use the mutator agent to generate a skill variant guided by ASI."""
    return invoke_bedrock_agent(
        MUTATOR_AGENT_ID, MUTATOR_ALIAS_ID,
        f"Mutate this SKILL.md guided by the following ASI feedback.\n\n"
        f"ASI:\n{asi}\n\nSKILL.md:\n{skill_content}",
        session_attrs={"asi": asi},
    )

def evaluate_candidate(candidate_content: str) -> dict:
    """Use the evaluator agent to score a candidate."""
    result = invoke_bedrock_agent(
        EVALUATOR_AGENT_ID, EVALUATOR_ALIAS_ID,
        f"Evaluate this SKILL.md and return JSON with score, lint_pass, "
        f"and asi (actionable side information for improvement):\n\n{candidate_content}",
    )
    return json.loads(result)

# --- GEPA integration: define fitness function for optimize_anything ---
def fitness_fn(individual: Individual) -> dict:
    """Evaluate a SKILL.md candidate. Returns multi-objective scores."""
    result = evaluate_candidate(individual.genome)
    return {
        "content_score": result["score"],
        "lint_pass": 100 if result["lint_pass"] else 0,
    }

# --- Read seed skill ---
with open("skills/golang/SKILL.md") as f:
    seed_content = f.read()

# --- Run GEPA optimization loop ---
asi = ""  # Actionable Side Information — empty for generation 0
population_size = 3
num_generations = 5

population = [seed_content] * population_size

for gen in range(num_generations):
    # Mutate population using ASI from previous generation
    with ThreadPoolExecutor(max_workers=population_size) as pool:
        mutants = list(pool.map(
            lambda content: mutate_skill(content, asi), population
        ))

    # Evaluate all candidates in parallel
    with ThreadPoolExecutor(max_workers=population_size) as pool:
        results = list(pool.map(evaluate_candidate, mutants))

    # Extract ASI from best candidate for next generation
    best = max(results, key=lambda r: r["score"])
    asi = best.get("asi", "")
    print(f"Gen {gen}: best score={best['score']}, asi={asi[:80]}...")

    # Pareto selection: keep candidates that are non-dominated
    scored = list(zip(mutants, results))
    scored.sort(key=lambda x: x[1]["score"], reverse=True)
    population = [content for content, _ in scored[:population_size]]

# Write winner
with open("skills/golang/SKILL.md", "w") as f:
    f.write(population[0])
print(f"GEPA complete. Final score: {results[0]['score']}")
```

#### Google Vertex AI Agent Builder

✅ `ParallelAgent` evaluates all candidates concurrently — native fan-out
✅ `LoopAgent` implements generational iteration with stop condition
✅ Shared session state passes ASI between generations via `output_key`
✅ Best native fit for GEPA among the three platforms

```python
import vertexai, json
from google.adk.agents import (
    Agent, LlmAgent, ParallelAgent, SequentialAgent, LoopAgent, BaseAgent,
)
from google.adk.agents.invocation_context import InvocationContext
from google.adk.events import Event, EventActions
from vertexai import agent_engines

client = vertexai.Client(project="my-project", location="us-central1")

# --- Tool functions ---
def evaluate_skill(skill_content: str) -> dict:
    """Score a SKILL.md candidate. Returns {score, lint_pass, asi}."""
    import subprocess, tempfile, os
    with tempfile.NamedTemporaryFile(mode="w", suffix=".md", delete=False) as f:
        f.write(skill_content)
        tmp = f.name
    try:
        score_out = subprocess.run(
            ["python", "scripts/score-skills.py", tmp], capture_output=True, text=True)
        lint_out = subprocess.run(
            ["python", "scripts/lint-skills.py", tmp], capture_output=True, text=True)
        score = int(score_out.stdout.strip().split(":")[-1])
        errors = [l for l in lint_out.stdout.splitlines() if "FAIL" in l]
        return {"score": score, "lint_pass": len(errors) == 0, "errors": errors}
    finally:
        os.unlink(tmp)

def read_skill_file(skill_path: str) -> str:
    """Read the current SKILL.md content."""
    with open(skill_path) as f:
        return f.read()

def write_skill_file(skill_path: str, content: str) -> str:
    """Write the winning SKILL.md to disk."""
    with open(skill_path, "w") as f:
        f.write(content)
    return f"Written to {skill_path}"

# --- Mutator: generates N variants guided by ASI ---
mutator = LlmAgent(
    name="mutator",
    model="gemini-2.5-pro",
    instruction=(
        "Read 'seed_content' and 'asi' from session state. "
        "Generate 3 distinct SKILL.md mutations that address the ASI feedback. "
        "Store them as a JSON array in output_key."
    ),
    tools=[read_skill_file],
    output_key="candidates",
)

# --- Per-candidate evaluator agents (created dynamically) ---
def make_evaluator(idx: int) -> LlmAgent:
    return LlmAgent(
        name=f"evaluator_{idx}",
        model="gemini-2.5-flash",
        instruction=(
            f"Read candidate index {idx} from 'candidates' in session state. "
            "Call evaluate_skill with its content. Store result in output_key."
        ),
        tools=[evaluate_skill],
        output_key=f"eval_{idx}",
    )

parallel_eval = ParallelAgent(
    name="parallel_eval",
    sub_agents=[make_evaluator(i) for i in range(3)],
)

# --- Selector: picks Pareto-best and updates ASI ---
selector = LlmAgent(
    name="selector",
    model="gemini-2.5-flash",
    instruction=(
        "Read eval_0, eval_1, eval_2 from state. Select the Pareto-dominant "
        "candidate (highest score that also passes lint). Update 'seed_content' "
        "with the winner and 'asi' with actionable improvement feedback. "
        "If the best score >= 95, set 'converged' to 'true'."
    ),
    output_key="selection_result",
)

# --- Convergence gate ---
class ConvergenceGate(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        converged = ctx.session.state.get("converged", "false") == "true"
        yield Event(author=self.name, actions=EventActions(escalate=converged))

# --- One generation = mutate → evaluate → select ---
one_generation = SequentialAgent(
    name="one_generation",
    sub_agents=[mutator, parallel_eval, selector, ConvergenceGate(name="gate")],
)

# --- Loop across generations ---
gepa_loop = LoopAgent(
    name="gepa_loop",
    max_iterations=5,
    sub_agents=[one_generation],
)

# --- Final step: write winner and create PR ---
finalize = LlmAgent(
    name="finalize",
    model="gemini-2.5-flash",
    instruction=(
        "Read 'seed_content' from state — this is the winning SKILL.md. "
        "Call write_skill_file to save it. Report the final score and "
        "number of generations used."
    ),
    tools=[write_skill_file],
    output_key="final_report",
)

pipeline = SequentialAgent(
    name="gepa_optimizer",
    sub_agents=[gepa_loop, finalize],
)

app = agent_engines.AdkApp(agent=pipeline)
remote = client.agent_engines.create(
    agent=app,
    config={
        "requirements": ["google-cloud-aiplatform[agent_engines,adk]"],
        "display_name": "GEPA Optimizer",
        "max_instances": 10,
    },
)

# --- Invoke: seed with initial content and empty ASI ---
with open("skills/golang/SKILL.md") as f:
    seed = f.read()

response = remote.query(
    user_id="gepa",
    message=json.dumps({"seed_content": seed, "asi": "", "converged": "false"}),
)
print(response["output"])
```

#### Azure AI Agent Service

✅ Function calling enables client-side GEPA library integration (`optimize_anything`)
✅ Thread persistence preserves full evolution history for audit
⚠️ No native parallel agent execution — client must parallelize via threads
⚠️ Connected agents limited to depth 2 — sufficient for mutator/evaluator but no deeper nesting
❌ No built-in loop primitive — client code manages the generational loop

```python
import os, json, time
from concurrent.futures import ThreadPoolExecutor
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.agents.models import CodeInterpreterTool

project = AIProjectClient(
    endpoint=os.environ["PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)

code_tool = CodeInterpreterTool()

# --- Create mutator agent ---
mutator = project.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="gepa-mutator",
    instructions=(
        "You generate SKILL.md mutations for evolutionary optimization. "
        "Given a seed SKILL.md and Actionable Side Information (ASI), "
        "produce 3 distinct variants that address the ASI feedback. "
        "Return a JSON array of 3 complete SKILL.md strings."
    ),
    tools=code_tool.definitions,
    tool_resources=code_tool.resources,
)

# --- Create evaluator agent ---
evaluator = project.agents.create_agent(
    model=os.environ["MODEL_DEPLOYMENT_NAME"],
    name="gepa-evaluator",
    instructions=(
        "You evaluate a SKILL.md candidate. Use Code Interpreter to run "
        "scripts/lint-skills.py and scripts/score-skills.py. "
        "Return JSON: {\"score\": int, \"lint_pass\": bool, \"asi\": str} "
        "where asi is actionable feedback for the next generation."
    ),
    tools=code_tool.definitions,
    tool_resources=code_tool.resources,
)

def run_agent_on_thread(agent_id: str, prompt: str) -> str:
    """Create a thread, run an agent, and return the assistant response."""
    thread = project.agents.threads.create()
    project.agents.messages.create(thread_id=thread.id, role="user", content=prompt)
    run = project.agents.runs.create_and_process(thread_id=thread.id, agent_id=agent_id)
    messages = project.agents.messages.list(thread_id=thread.id)
    result = ""
    for msg in messages:
        if msg.role == "assistant":
            result = msg.content[0].text.value
            break
    project.agents.threads.delete(thread.id)
    return result

# --- GEPA loop (client-managed) ---
with open("skills/golang/SKILL.md") as f:
    seed_content = f.read()

asi = ""
population_size = 3
best_score = 0

for gen in range(5):
    # Step 1: Mutate — agent generates candidates
    mutation_prompt = (
        f"Generate {population_size} mutations of this SKILL.md.\n\n"
        f"ASI feedback from previous generation:\n{asi}\n\n"
        f"Current SKILL.md:\n{seed_content}"
    )
    mutants_json = run_agent_on_thread(mutator.id, mutation_prompt)
    candidates = json.loads(mutants_json)

    # Step 2: Evaluate — parallel agent invocations across threads
    def eval_candidate(content):
        return json.loads(run_agent_on_thread(
            evaluator.id,
            f"Evaluate this SKILL.md:\n\n{content}",
        ))

    with ThreadPoolExecutor(max_workers=population_size) as pool:
        results = list(pool.map(eval_candidate, candidates))

    # Step 3: Pareto selection
    scored = sorted(
        zip(candidates, results),
        key=lambda x: (x[1].get("lint_pass", False), x[1].get("score", 0)),
        reverse=True,
    )
    winner_content, winner_result = scored[0]
    best_score = winner_result["score"]
    asi = winner_result.get("asi", "")
    seed_content = winner_content

    print(f"Gen {gen}: best={best_score}, lint={'✅' if winner_result['lint_pass'] else '❌'}")

    if best_score >= 95 and winner_result.get("lint_pass"):
        print("Converged!")
        break

# Write winner
with open("skills/golang/SKILL.md", "w") as f:
    f.write(seed_content)
print(f"GEPA complete. Final score: {best_score}")

# Cleanup agents
project.agents.delete_agent(mutator.id)
project.agents.delete_agent(evaluator.id)
```

---

## Platform Comparison Summary

| Capability | AWS Bedrock | Google Vertex AI | Azure AI Agent Service |
|---|---|---|---|
| **Code execution** | ✅ Code Interpreter | ✅ Code Execution sandbox | ✅ Code Interpreter |
| **Web search** | ⚠️ Lambda action group | ✅ Google Search grounding | ✅ Bing Grounding |
| **Knowledge base** | ✅ S3/OpenSearch RAG | ✅ Vertex AI Search / RAG Engine | ✅ File Search + Azure AI Search |
| **Multi-agent** | ✅ Supervisor/Worker | ✅ ADK hierarchy (Sequential, Parallel, Loop) | ⚠️ Connected Agents (depth 2) |
| **Parallel fan-out** | ⚠️ Client-side threads | ✅ ParallelAgent native | ⚠️ Client-side threads |
| **Loop/retry** | ⚠️ Prompt-driven | ✅ LoopAgent native | ❌ Client-side loop |
| **PR creation** | ✅ Return-of-control | ✅ Python tool function | ✅ Function calling |
| **Best workflow fit** | Skill Lint (simple) | GEPA Optimizer (complex) | Skill Improver (tool-rich) |


## Cloud Platform-Specific Guides

### AWS Bedrock Agents

**What works out of the box:**
- ✅ Fully managed serverless runtime — no infrastructure provisioning
- ✅ Action groups with Lambda-backed or return-of-control tool execution
- ✅ Knowledge bases with hybrid/semantic search, reranking, metadata filtering, and source citations
- ✅ Guardrails (content filters, PII masking, denied topics, contextual grounding, automated reasoning)
- ✅ Built-in code interpreter (`AMAZON.CodeInterpreter`) for sandboxed Python execution
- ✅ Session memory with cross-session summaries (`SESSION_SUMMARY` type, 1–365 day retention)

**What needs adaptation:**
- Skill evaluation scripts need Lambda wrapping (action groups) or `AMAZON.CodeInterpreter` for ad-hoc execution
- Return-of-control pattern requires client-side polling loop to send results back via `sessionState.returnControlInvocationResults`
- Agent lifecycle is multi-step: `create_agent()` → add tools/KBs → `prepare_agent()` → `create_agent_alias()` for production
- Prompt templates use `$placeholder$` syntax — custom orchestration prompts must be adapted to this format

**Platform advantages:**

| Advantage | Detail |
|-----------|--------|
| IAM-native auth | Service roles, resource-based policies, cross-account enforcement — no separate auth layer |
| Multi-agent supervisor | `SUPERVISOR` / `SUPERVISOR_ROUTER` collaboration modes with trace-level caller chains |
| Deep AWS integration | S3 data sources, Lambda action groups, CloudWatch logging, KMS encryption, OpenSearch/Aurora vector stores |
| Guardrails-as-a-service | Standalone `ApplyGuardrail` API — evaluate text without model invocation, cross-region distribution |
| Prompt override system | 6 customizable prompt templates (pre-processing, orchestration, KB response, post-processing, memory summarization, routing) |

**Key limitations:**
- No local dev mode — agents must be deployed to AWS; test via `TSTALIASID` alias
- Lambda cold starts affect action group latency (mitigate with provisioned concurrency)
- Code interpreter limited to 3 regions (us-east-1, us-west-2, eu-central-1), 25 concurrent sessions, 10 MB input
- Only one memory type (`SESSION_SUMMARY`) — no structured long-term memory or user fact extraction
- Two separate boto3 clients required (build-time `bedrock-agent`, runtime `bedrock-agent-runtime`)

**Quick start:**
```python
import boto3

client = boto3.client("bedrock-agent")
runtime = boto3.client("bedrock-agent-runtime")
agent = client.create_agent(agentName="my-agent", foundationModel="anthropic.claude-sonnet-4-20250514", instruction="You are a helpful assistant.", agentResourceRoleArn="arn:aws:iam::123456789012:role/BedrockAgentRole")
client.prepare_agent(agentId=agent["agent"]["agentId"])
# Invoke with: runtime.invoke_agent(agentId=..., agentAliasId="TSTALIASID", sessionId="s1", inputText="Hello")
```

---

### Google Vertex AI Agent Builder

**What works out of the box:**
- ✅ Agent Engine — fully managed hosting with auto-scaling (Cloud Run under the hood, 0–1000 instances)
- ✅ ADK framework with typed agent primitives (`Agent`, `SequentialAgent`, `ParallelAgent`, `LoopAgent`)
- ✅ Google Search grounding with segment-level citations and confidence scores
- ✅ Memory Bank — LLM-driven extraction, consolidation, similarity search, and automatic expiration
- ✅ Multi-framework support: ADK, LangChain, LangGraph, AG2, LlamaIndex, CrewAI
- ✅ A2A (Agent-to-Agent) protocol for cross-framework inter-agent communication

**What needs adaptation:**
- GEPA evaluation scripts need packaging for Agent Engine deployment (specify `requirements`, `staging_bucket`, and optionally source files for CI/CD)
- Agent Engine uses `client.agent_engines.create()` with config dict — not a simple constructor call
- Google Search grounding requires displaying Search Suggestions in production (compliance requirement)
- ADK agents use `async_stream_query` for invocation — adapt synchronous eval scripts to async patterns

**Platform advantages:**

| Advantage | Detail |
|-----------|--------|
| Best-in-class multi-agent | `SequentialAgent`, `ParallelAgent`, `LoopAgent` + LLM-driven delegation — deterministic and dynamic patterns in one framework |
| A2A protocol | Open standard for inter-agent communication; SDKs in Python, Go, JS, Java, C#/.NET |
| Memory Bank | Long-term memory with extraction, consolidation, revisions, multimodal understanding, and per-user scoping |
| Enterprise security | VPC-SC, CMEK, Data Residency, HIPAA, Access Transparency, mTLS-bound agent identity (SPIFFE) |
| Framework flexibility | Deploy ADK, LangChain, LangGraph, AG2, LlamaIndex, or custom agents — same managed runtime |
| Integrated observability | Cloud Trace + Cloud Logging + Cloud Monitoring via OpenTelemetry; DAG visualization in console |

**Key limitations:**
- Newer service — some features in preview (Agent Identity, Example Store, Agent Engine Threat Detection)
- Documentation restructured frequently; several canonical URLs now 404 → redirected paths
- Memory Bank and sessions use `v1beta1` API — expect breaking changes before GA
- Google Search grounding limit: 1M queries/day (contact support for increase)
- Single parent rule: each agent can only have one parent in the hierarchy (ValueError on second assignment)

**Quick start:**
```python
import vertexai
from google.adk.agents import Agent

client = vertexai.Client(project="my-project", location="us-central1")
agent = Agent(model="gemini-2.5-flash", name="my-agent", instruction="You are a helpful assistant.")
from vertexai import agent_engines; app = agent_engines.AdkApp(agent=agent)
remote = client.agent_engines.create(agent=app, config={"requirements": ["google-cloud-aiplatform[agent_engines,adk]"], "display_name": "My Agent"})
# Query with: async for event in remote.async_stream_query(user_id="u1", message="Hello"): print(event)
```

---

### Azure AI Agent Service (Foundry Agent Service)

**What works out of the box:**
- ✅ OpenAI Assistants API-compatible thread/message/run model — familiar to OpenAI users
- ✅ Code interpreter with sandboxed Python execution, file I/O, and chart generation
- ✅ File Search with automatic chunking, embedding, hybrid search, reranking (up to 10K files per vector store)
- ✅ Bing grounding for real-time web search; Azure AI Search for BYO index
- ✅ Multi-SDK support: Python (`azure-ai-projects`), JS/TS (`@azure/ai-agents`), .NET, Java, REST
- ✅ Enterprise-grade: Entra ID auth only, RBAC, audit logs, VNet isolation, Cosmos DB persistence

**What needs adaptation:**
- Thread model is stateful and persistent — differs from stateless skill eval patterns; each eval run should create a fresh thread
- No plugin/skill system — agent behavior is defined entirely via `instructions` parameter and tool definitions
- Function calling requires client-side polling for `requires_action` status and manual result submission
- Project endpoint format changed (May 2025): now `https://<resource>.services.ai.azure.com/api/projects/<project>` — update connection strings

**Platform advantages:**

| Advantage | Detail |
|-----------|--------|
| OpenAI API compatibility | Assistants API surface — easy migration from OpenAI; familiar thread/run/message primitives |
| Semantic Kernel integration | Native integration with Microsoft's orchestration framework; out-of-the-box tracing |
| Enterprise compliance | Entra ID (no API keys), RBAC, audit logs, content filtering, VNet + private endpoints, customer-managed encryption |
| Rich tool catalog | 12+ built-in tools: Code Interpreter, File Search, Bing, Azure AI Search, Logic Apps, OpenAPI, MCP, Fabric, Deep Research, Browser Automation |
| OpenTelemetry tracing | Microsoft-contributed multi-agent semantic conventions; traces to Application Insights, Foundry portal, or local Aspire Dashboard |
| BYO infrastructure | Standard setup: customer-provisioned Cosmos DB, Blob Storage, AI Search — full data control with BCDR |

**Key limitations:**
- Tied to Azure AI Foundry project — requires Foundry Account → Project → Model Deployment hierarchy
- Connected agents (multi-agent) limited to depth of 2; children cannot have sub-agents or use function calling
- Only one instance of each knowledge tool type per agent (work around with connected agents)
- Threads persist until explicitly deleted — no automatic TTL; can accumulate unbounded state
- VNet setup requires Bicep/Terraform (not portal); agent subnet must be dedicated and delegated to `Microsoft.App/environments`

**Quick start:**
```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

client = AIProjectClient(endpoint="https://myresource.services.ai.azure.com/api/projects/myproject", credential=DefaultAzureCredential())
agent = client.agents.create_agent(model="gpt-4o", name="my-agent", instructions="You are a helpful assistant.")
thread = client.agents.threads.create(); client.agents.messages.create(thread_id=thread.id, role="user", content="Hello")
# Run with: run = client.agents.runs.create_and_process(thread_id=thread.id, agent_id=agent.id)
```

---

## Cross-Platform Comparison

| Capability | AWS Bedrock | Vertex AI Agent Builder | Azure AI Agent Service |
|------------|-------------|------------------------|----------------------|
| **Runtime** | Serverless (managed) | Agent Engine (Cloud Run) | Foundry project (managed) |
| **Primary SDK** | `boto3` | `google-cloud-aiplatform` | `azure-ai-projects` |
| **Tool execution** | Lambda / Return-of-Control | Python functions / MCP / OpenAPI | Code Interpreter / Function Calling / OpenAPI |
| **RAG** | Knowledge Bases (6 vector stores) | Google Search + Vertex AI Search + RAG Engine | File Search (vector stores) + Azure AI Search |
| **Multi-agent** | Supervisor/Router (hierarchical) | ADK agents (Sequential, Parallel, Loop, LLM) + A2A | Connected Agents (depth 2) + Workflows (preview) |
| **Memory** | Session summaries (1–365 days) | Memory Bank (extraction, consolidation, similarity) | Threads (persistent until deleted) |
| **Auth** | IAM roles + resource policies | IAM + Agent Identity (SPIFFE) | Entra ID only (no API keys) |
| **Guardrails** | Bedrock Guardrails (6 filter types) | Model Armor + Safety Settings | Azure Content Filters + XPIA detection |
| **Observability** | Traces + CloudWatch logging | Cloud Trace + Cloud Logging (OTel) | Application Insights (OTel) |
| **Local dev** | ❌ No local mode | ✅ ADK local runner | ✅ VS Code AI Toolkit |
| **Open standards** | OpenAPI schemas | A2A protocol, OpenTelemetry, MCP | OpenAI Assistants API, OpenTelemetry, MCP |


---

## Getting Started

Choose your path:

1. **Already using Copilot CLI?** → Follow the [main README](README.md) — everything works out of the box.

2. **Using Claude Code?** → Install the plugin directly (`claude plugin add <url>`), then set up `claude-code-action@v1` for CI/CD using the examples above.

3. **Using Gemini CLI?** → Copy agents and skills to `.gemini/`, convert commands to TOML, set up `run-gemini-cli` for CI/CD.

4. **Using Codex CLI?** → Port instructions to AGENTS.md, set up `codex-action@v1` for CI/CD, consider the TypeScript SDK for GEPA.

5. **Using OpenCode?** → Install with `npm i -g opencode-ai`, run `opencode github install` to add the Action, configure models in `opencode.json`. Native plugin/skill/agent support means minimal conversion needed.

6. **Want the comparison matrix?** → See [AI Coding Agent Comparison Matrix](docs/ai-coding-agent-comparison.md) for a full capability breakdown of all five platforms.

7. **Building cloud agents (AWS)?** → See [Bedrock Agents guide](#aws-bedrock-agents) + [Cloud Comparison Matrix](docs/cloud-agent-comparison.md)

8. **Building cloud agents (GCP)?** → See [Vertex AI guide](#google-vertex-ai-agent-builder) + [Cloud Comparison Matrix](docs/cloud-agent-comparison.md)

9. **Building cloud agents (Azure)?** → See [Azure AI guide](#azure-ai-agent-service-foundry-agent-service) + [Cloud Comparison Matrix](docs/cloud-agent-comparison.md)

10. **Want the cloud comparison matrix?** → See [Cloud-Native Agent Comparison Matrix](docs/cloud-agent-comparison.md)
