# AI Coding Agent Platform Comparison Matrix

> **Last Updated:** February 2026
> **Methodology:** Deep research across official docs, changelogs, GitHub repos, and community sources.

## Quick Summary

| Platform | CLI | Cloud | CI/CD | Hooks | MCP | Plugins | Skills | Multi-Agent | Score |
|----------|:---:|:-----:|:-----:|:-----:|:---:|:-------:|:------:|:-----------:|:-----:|
| **Claude Code** (Anthropic) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 19/26 |
| **Copilot CLI** (GitHub) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 19/26 |
| **Gemini CLI** (Google) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | 20/26 |
| **Codex CLI** (OpenAI) | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ✅ | 14/26 |
| **OpenCode** (Anomaly) | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | 17/26 |

---

## Full Comparison Matrix

### Core Agent Capabilities

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode |
|-----------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| **CLI Agent** | ✅ `claude` | ✅ `copilot` | ✅ `gemini` | ✅ `codex` | ✅ `opencode` |
| **Cloud Agent** | ✅ `--remote` | ✅ Coding agent | ✅ Jules (ext) | ✅ Codex Web | ❌ |
| **CI/CD Action** | ✅ `claude-code-action@v1` | ✅ `gh aw` | ✅ `run-gemini-cli` | ✅ `codex-action@v1` | ✅ `opencode/github@latest` |
| **Headless/SDK** | ✅ `claude -p` | ✅ `copilot -p` | ✅ `gemini -p` | ✅ `codex exec` + SDK | ✅ `opencode run` + `serve` |
| **Code Review** | ✅ Plugin agent | ✅ `@copilot` on PRs | ✅ `/review` | ✅ `/review` | ✅ PR review action |
| **Browser Agent** | ✅ `--chrome` | ❌ | ✅ `browser_agent` | ❌ | ❌ |
| **Web Search** | ❌ | ❌ | ✅ Google Search | ✅ Built-in | ❌ |

### Extensibility & Plugin System

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode |
|-----------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| **Plugin System** | ✅ `.claude-plugin/` | ✅ `plugin.json` | ✅ Extensions | ❌ | ✅ `.opencode/plugins/` |
| **Skills (Knowledge)** | ✅ `SKILL.md` | ✅ `SKILL.md` | ✅ `SKILL.md` | ❌ | ✅ `SKILL.md` |
| **Custom Agents** | ✅ `.claude/agents/` | ✅ `.github/agents/` | ✅ `.gemini/agents/` | ✅ `config.toml` roles | ✅ `.opencode/agents/` |
| **Custom Commands** | ✅ Commands in plugins | ✅ Slash commands | ✅ `/cmd` in `commands/` | ❌ | ✅ `.opencode/commands/` |
| **Context File** | ✅ `CLAUDE.md` | ✅ `copilot-instructions.md` | ✅ `GEMINI.md` | ✅ `AGENTS.md` | ✅ `AGENTS.md` |
| **Hooks (Lifecycle)** | ✅ `hooks.json` | ✅ `hooks.json` | ✅ Hooks | ❌ | ✅ Plugin events (30+) |
| **MCP Integration** | ✅ `.mcp.json` | ✅ GitHub MCP Server | ✅ `gemini-extension.json` | ✅ `config.toml` | ✅ Local + remote MCP |
| **Custom Themes** | ❌ | ❌ | ✅ `themes/` | ❌ | ✅ 12+ built-in |

### Multi-Agent & Orchestration

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode |
|-----------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| **Subagents** | ✅ `agents/*.md` | ✅ Task tool | ✅ `agents/*.md` | ✅ Agent roles | ✅ General + Explore |
| **Multi-Agent Teams** | ✅ Agent Teams | ✅ Orchestrator→Workers | ❌ | ✅ Multi-agent (exp.) | ❌ |
| **Remote Agents** | ❌ | ❌ | ✅ A2A Protocol | ❌ | ✅ ACP Protocol |

### Security & Sandboxing

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode |
|-----------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| **Safe Outputs** | ❌ | ✅ Threat detection pipeline | ❌ | ✅ Sandbox modes | ❌ |
| **Network Allowlist** | ❌ | ✅ `network: domains:` | ❌ | ❌ | ❌ |
| **Git Worktrees** | ✅ `--worktree` | ❌ | ❌ | ❌ | ❌ |
| **LSP Integration** | ✅ `.lsp.json` | ❌ | ❌ | ❌ | ✅ `opencode.json` |
| **Structured Output** | ✅ `--json-schema` | ❌ | ✅ `--output-format json` | ❌ | ✅ `--format json` |
| **Bot Skip** | ❌ | ✅ `on.skip-bots:` | ❌ | ❌ | ❌ |
| **Shared Imports** | ❌ | ✅ `imports:` | ❌ | ❌ | ❌ |

---

## Detailed Platform Profiles

### 1. Claude Code (Anthropic)

**Install:** `npm install -g @anthropic-ai/claude-code`
**Latest Version:** Active development (Claude Opus 4.6 / Sonnet 4.6)
**License:** Proprietary
**Cost:** Anthropic API billing (per-token); Claude Max subscription alternative

**Unique Strengths:**
- **Agent Teams** — Multiple Claude instances coordinate on a shared task list with inter-agent messaging. Only platform with true local multi-agent parallelism.
- **Git Worktrees** — Each agent works in an isolated git worktree, enabling safe parallel file edits without conflicts.
- **LSP Integration** — Language Server Protocol for code intelligence (TypeScript, Python, Go, Rust). No other CLI agent has native LSP.
- **Plugin Marketplace** — Richest plugin ecosystem: commands, agents, skills, hooks, MCP servers, LSP configs all in one `plugin.json`.
- **Structured Output** — `--json-schema` flag returns validated JSON matching a user-specified schema.

**Cloud Agent:** `claude --remote` creates a session on claude.ai; `--teleport` resumes cloud sessions locally.

**Hooks:** 6 lifecycle events — `PreToolUse`, `PostToolUse`, `Stop`, `NotificationCommand`, `SubagentPreToolUse`, `SubagentStop`. Supports command, prompt, and agent hook types.

**CI/CD:** `claude-code-action@v1` responds to `@claude` mentions, PR events, and cron schedules. Per-token Anthropic API billing.

**Sources:**
- https://docs.anthropic.com/en/docs/claude-code/cli-usage
- https://docs.anthropic.com/en/docs/claude-code/plugins
- https://docs.anthropic.com/en/docs/claude-code/hooks
- https://docs.anthropic.com/en/docs/claude-code/agent-teams
- https://github.com/anthropics/claude-code-action

---

### 2. GitHub Copilot CLI

**Install:** Pre-installed with GitHub Copilot; `gh extension install github/copilot-cli`
**Latest Version:** 0.0.415
**License:** Proprietary (GitHub Copilot subscription)
**Cost:** Included with Copilot subscription (Individual, Business, Enterprise); Agentic Workflows free with Copilot API

**Unique Strengths:**
- **Agentic Workflows (`gh aw`)** — Markdown-to-GitHub-Actions compiler with AI agent runtime. Write workflows in markdown with YAML frontmatter; compile to `.lock.yml`. The most GitHub-native CI/CD integration of any platform.
- **Safe Outputs Pipeline** — Read-only agent → threat detection → scoped write jobs (create-issue, create-pr, add-comment, dispatch-workflow). Strongest security model for CI/CD agents.
- **Network Allowlist** — Per-workflow domain access control using ecosystem identifiers (`go`, `python`, `containers`).
- **Shared Imports** — Reusable knowledge and tool configurations across workflows via `imports:` frontmatter.
- **Bot Skip** — `on.skip-bots:` prevents infinite trigger loops in orchestrator patterns.
- **Copilot Memory** — (Preview) Persistent learning from past interactions across sessions.

**Cloud Agent:** "Copilot coding agent" — assign `@copilot` to any GitHub issue and it creates a PR autonomously. `/delegate` from CLI dispatches to cloud agent.

**Hooks:** 6 lifecycle events — `sessionStart`, `sessionEnd`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `errorOccurred`. Same JSON contract as Claude Code. `preToolUse` can approve/deny tool executions.

**MCP:** Ships with GitHub MCP server pre-installed. Supports custom MCP servers, remote MCP, and MCP Registry at github.com/mcp.

**CI/CD:** `gh aw` is the most powerful CI/CD integration — markdown workflows compiled to GitHub Actions with AI agent runtime, schedule shorthand (`weekly`, `daily`), orchestrator→worker dispatch patterns. No per-token API cost.

**Sources:**
- https://github.github.io/gh-aw/introduction/overview/
- https://github.github.io/gh-aw/authoring-workflows/safe-outputs/
- https://docs.github.com/en/copilot/reference/hooks-configuration
- https://docs.github.com/en/copilot/concepts/context/mcp

---

### 3. Gemini CLI (Google)

**Install:** `npm install -g @anthropic-ai/gemini-cli` (actually: `npm install -g @anthropic-ai/gemini-cli` — verify at repo)
**Latest Version:** Active development
**License:** Apache 2.0 (open source)
**Cost:** Free tier with Gemini API key; Google Cloud billing for higher usage

**Unique Strengths:**
- **Google Search Grounding** — Only platform with real-time web search built into the model. Gemini natively grounds responses with live Google Search results.
- **Remote Agents (A2A)** — Agent-to-Agent protocol for connecting to remote subagents. Only platform supporting cross-machine agent collaboration.
- **Browser Agent** — Built-in web browser automation via Chrome DevTools accessibility tree. Visual model support for screenshot analysis with Gemini Computer Use.
- **Extensions Gallery** — Public gallery at geminicli.com with community extensions. `gemini extensions install <url>` for one-command install.
- **Custom Themes** — Color customization for CLI UI.
- **1M Token Context** — Largest context window of any CLI agent (Gemini 3).

**Cloud Agent:** Jules is Google's cloud coding agent, accessible via Gemini CLI extension. Orchestrate cloud tasks from the local CLI.

**Subagents:** Built-in `codebase_investigator`, `cli_help`, `generalist_agent`, `browser_agent`. Custom subagents via `.gemini/agents/*.md` with YAML frontmatter (experimental, requires `enableAgents: true`).

**Multi-Agent:** NOT native — subagent calls are sequential (issue #14963). Community proposals exist (Maestro) but no built-in multi-agent orchestration.

**CI/CD:** `google-github-actions/run-gemini-cli` with `@gemini-cli` mentions, dispatch workflows, PR review, and issue triage patterns. `/setup-github` command for easy workflow setup.

**Sources:**
- https://github.com/google-gemini/gemini-cli
- https://geminicli.com/docs/core/subagents/
- https://geminicli.com/docs/core/remote-agents/
- https://geminicli.com/extensions/
- https://github.com/google-github-actions/run-gemini-cli

---

### 4. Codex CLI (OpenAI)

**Install:** `npm install -g @openai/codex`
**Latest Version:** 0.104.0 (Rust rewrite)
**License:** Apache 2.0 (open source)
**Cost:** OpenAI API billing (per-token); ChatGPT Plus/Pro includes Codex Web

**Unique Strengths:**
- **TypeScript SDK** — `@openai/codex-sdk` with `thread.run()` API and resume. Most programmable SDK of any platform for embedding in applications.
- **Sandbox Modes** — Fine-grained security: `read-only`, `workspace-write`, `danger-full-access` per agent role. Sub-agents automatically inherit parent's sandbox restrictions.
- **Multi-Agent Roles** — `config.toml` defines agent roles (worker, explorer, monitor, custom) with per-role model, sandbox, and instructions overrides. `max_threads` and `max_depth` control parallelism.
- **Codex Web** — Cloud-based sandboxed agent at chatgpt.com/codex. Create coding tasks from web, get PRs. Unique SaaS-first approach.
- **Official GitHub Action** — `openai/codex-action@v1` with safety strategies (`drop-sudo`, `unprivileged-user`, `unsafe`) and prompt file conventions (`.github/codex/prompts/`).

**No Hooks:** Feature request exists (Reddit) but not implemented. No lifecycle hook system.

**No Plugins/Skills:** No native plugin system or SKILL.md support. SKILL.md can be ported manually (blog post) but there's no auto-invocation mechanism.

**CI/CD:** `openai/codex-action@v1` wraps `codex exec` with safety controls, sandbox modes, and output-file parameter. Solid but less integrated than `gh aw` or Claude Code Action.

**Sources:**
- https://github.com/openai/codex
- https://developers.openai.com/codex
- https://developers.openai.com/codex/multi-agent
- https://developers.openai.com/codex/sdk
- https://developers.openai.com/codex/changelog/

---

### 5. OpenCode (Anomaly)

**Install:** `npm install -g opencode-ai` or `brew install anomalyco/tap/opencode` or `curl -fsSL https://opencode.ai/install | bash`
**Latest Version:** Active development (110K+ GitHub stars)
**License:** Apache 2.0 (open source)
**Cost:** BYO API key — 75+ models from any provider (OpenAI, Anthropic, Google, Groq, AWS Bedrock, Azure, OpenRouter, GitHub Copilot); OpenCode Zen curated model service

**Unique Strengths:**
- **Provider-Agnostic** — 75+ models from ANY provider; not locked to one vendor. Supports OpenAI, Anthropic, Google, Groq, AWS Bedrock, Azure, OpenRouter, and GitHub Copilot.
- **Richest Hook System** — 30+ plugin events (most of any platform): `tool.execute.before/after`, `session.*`, `file.edited`, `lsp.*`, `permission.*`, `message.*`, `tui.*`. JS/TS plugins with full SDK; `tool.execute.before` can deny tool calls.
- **SKILL.md Compatibility** — Reads from `.opencode/skills/`, `.claude/skills/`, AND `.agents/skills/` paths. Most compatible skill system across platforms.
- **HTTP API Server** — `opencode serve` exposes full functionality as REST API; `opencode web` adds a web UI. Unique among CLI agents.
- **ACP Protocol** — Agent Client Protocol for agent-to-agent communication via stdin/stdout nd-JSON. Different approach from Gemini's A2A.
- **Dual LSP Support** — Native LSP integration via `opencode.json`. Only Claude Code and OpenCode have LSP among all platforms.

**Cloud Agent:** No persistent cloud agent.

**Plugins:** `.opencode/plugins/` with JS/TS modules; 30+ event hooks; custom tool definitions; npm distribution; `@opencode-ai/plugin` TypeScript types.

**Skills:** Native `SKILL.md` with lazy loading; per-skill permissions (allow/deny/ask); on-demand via `skill` tool. Compatible with Claude Code and Copilot CLI skill paths.

**Subagents:** General + Explore built-in; @mention to invoke; child sessions with parent/child navigation; per-subagent model/tools/permissions. Sequential only (no multi-agent parallelism).

**CI/CD:** `anomalyco/opencode/github@latest` with `/opencode` or `/oc` mentions in issues/PRs; schedule and workflow_dispatch triggers. `opencode github install` for easy setup.

**Hooks:** 30+ plugin events — `tool.execute.before/after`, `session.created/compacted/deleted/idle/error/status`, `file.edited`, `lsp.*`, `permission.*`, `message.*`, `shell.env`, `tui.*`. `tool.execute.before` can DENY tool calls; compaction hooks for context management.

**Custom Themes:** 12+ built-in themes (tokyonight, catppuccin, gruvbox, nord, etc.); custom JSON themes; per-project or global configuration.

**Sources:**
- https://opencode.ai/docs/
- https://opencode.ai/docs/plugins/
- https://opencode.ai/docs/agents/
- https://opencode.ai/docs/skills/
- https://opencode.ai/docs/github/
- https://opencode.ai/docs/cli/
- https://opencode.ai/docs/mcp-servers/
- https://opencode.ai/docs/themes

---

## Cross-Platform Feature Analysis

### Which platform for which use case?

| Use Case | Best Platform | Runner-Up | Why |
|----------|:------------:|:---------:|-----|
| **GitHub-native CI/CD** | Copilot CLI | Gemini CLI | `gh aw` compiles markdown→Actions; safe-outputs; no API cost |
| **Plugin ecosystem** | Claude Code | Copilot CLI | Richest plugin system; marketplace; same format works in Copilot |
| **Multi-agent parallelism** | Claude Code | Codex CLI | Agent Teams with git worktree isolation; truly parallel |
| **Web research tasks** | Gemini CLI | Codex CLI | Google Search grounding built into model |
| **Embedded in applications** | Codex CLI | Claude Code | TypeScript SDK with thread.run(); structured API |
| **Security-critical CI/CD** | Copilot CLI | Codex CLI | Safe-outputs pipeline + network allowlist; read-only by default |
| **Cross-machine agents** | Gemini CLI | OpenCode | Gemini A2A protocol; OpenCode ACP protocol |
| **Browser automation** | Gemini CLI | Claude Code | Chrome DevTools + visual model; Claude has `--chrome` |
| **Knowledge management** | Claude Code | OpenCode | SKILL.md auto-invocation; OpenCode reads 3 skill paths |
| **Cost-sensitive teams** | Copilot CLI | Gemini CLI | Included with Copilot subscription; Gemini has free tier |
| **Open-source preference** | Gemini CLI | OpenCode | All three Apache 2.0; Gemini has extensions gallery, OpenCode has richest hooks |
| **Provider flexibility** | OpenCode | — | 75+ models from any provider; no vendor lock-in |
| **HTTP API / REST server** | OpenCode | Codex CLI | `opencode serve` REST API; Codex has TypeScript SDK |

### Hook Lifecycle Events Comparison

| Event | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode |
|-------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| Session Start | ✅ | ✅ `sessionStart` | ✅ | ❌ | ✅ `session.created` |
| Session End | ✅ | ✅ `sessionEnd` | ✅ | ❌ | ✅ `session.deleted` |
| User Prompt | ✅ | ✅ `userPromptSubmitted` | ✅ | ❌ | ✅ `message.*` |
| Pre-Tool Use | ✅ `PreToolUse` | ✅ `preToolUse` | ✅ | ❌ | ✅ `tool.execute.before` |
| Post-Tool Use | ✅ `PostToolUse` | ✅ `postToolUse` | ✅ | ❌ | ✅ `tool.execute.after` |
| Error | ✅ | ✅ `errorOccurred` | ✅ | ❌ | ✅ `session.error` |
| Stop/Completion | ✅ `Stop` | via sessionEnd | ✅ | ❌ | ✅ `session.idle` |
| Subagent Events | ✅ `SubagentPreToolUse` | ❌ | ❌ | ❌ | ❌ |

### Context File Comparison

| Feature | `CLAUDE.md` | `copilot-instructions.md` | `GEMINI.md` | `AGENTS.md` (Codex) | `AGENTS.md` (OpenCode) |
|---------|:-----------:|:-------------------------:|:-----------:|:-------------------:|:----------------------:|
| Auto-loaded | ✅ | ✅ | ✅ | ✅ | ✅ |
| Hierarchical | ✅ (parent dirs) | ✅ (.github/) | ✅ | ✅ | ✅ |
| Org-level | ✅ | ✅ | ❌ | ❌ | ❌ |
| Memory/Learning | ❌ | ✅ (Copilot Memory) | ❌ | ❌ | ❌ |
| Agent Config | ❌ (separate file) | ❌ (separate file) | ❌ (separate file) | ✅ (inline) | ❌ (separate file) |

### Plugin/Extension Primitives

| Primitive | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode |
|-----------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| Commands | ✅ `commands/*.md` | ✅ `commands/*.md` | ✅ `commands/*.toml` | ❌ | ✅ `commands/*.md` |
| Agents | ✅ `agents/*.md` | ✅ `agents/*.md` | ✅ `agents/*.md` | ❌ | ✅ `agents/*.md` |
| Skills | ✅ `skills/*/SKILL.md` | ✅ `skills/*/SKILL.md` | ✅ `skills/*/SKILL.md` | ❌ | ✅ `skills/*/SKILL.md` |
| Hooks | ✅ `hooks.json` | ✅ `hooks.json` | ✅ In extension | ❌ | ✅ Plugin events |
| MCP Servers | ✅ `.mcp.json` | ✅ MCP config | ✅ `gemini-extension.json` | ✅ `config.toml` | ✅ `opencode.json` |
| LSP | ✅ `.lsp.json` | ❌ | ❌ | ❌ | ✅ `opencode.json` |
| Themes | ❌ | ❌ | ✅ `themes/` | ❌ | ✅ JSON themes |
| Manifest | `plugin.json` | `plugin.json` | `gemini-extension.json` | N/A | `plugin.json` (npm) |
| Install Method | `claude plugin add` | `plugin install <url>` | `gemini extensions install` | N/A | npm install |
| Marketplace | ✅ (emerging) | ✅ (emerging) | ✅ geminicli.com | ❌ | ❌ |

---

## Architecture Diagrams

### Claude Code Agent Teams
```
┌─────────────┐
│  Main Agent  │──── Shared Task List ────┐
└──────┬──────┘                           │
       │ spawn                            │
  ┌────┴─────────────────────┐            │
  │        │        │        │            │
┌─┴──┐ ┌──┴─┐ ┌──┴──┐ ┌──┴──┐          │
│ A1 │ │ A2 │ │ A3  │ │ A4  │◀─────────┘
│    │ │    │ │     │ │     │  inter-agent
│wkt1│ │wkt2│ │wkt3 │ │wkt4 │  messaging
└────┘ └────┘ └─────┘ └─────┘
 (each in isolated git worktree)
```

### Copilot Agentic Workflows
```
┌──────────────────┐    compile     ┌──────────────┐
│  workflow.md      │──────────────▶│ .lock.yml    │
│  (YAML frontmatter│               │ (GitHub      │
│   + markdown body)│               │  Actions)    │
└──────────────────┘               └──────┬───────┘
                                          │ runs
                                   ┌──────┴───────┐
                                   │  AI Agent    │
                                   │  (read-only) │
                                   └──────┬───────┘
                                          │ safe-outputs
                               ┌──────────┼──────────┐
                               │          │          │
                         create-PR  create-issue  dispatch
```

### Gemini CLI A2A Protocol
```
┌──────────────┐   A2A    ┌──────────────┐
│  Local Agent │◀────────▶│ Remote Agent │
│  (gemini)    │ protocol │ (any A2A     │
│              │          │  compatible) │
└──────┬───────┘          └──────────────┘
       │ local
  ┌────┴──────────┐
  │       │       │
┌─┴──┐ ┌─┴──┐ ┌─┴──────────┐
│code│ │help│ │browser_agent│
│inv.│ │    │ │(Chrome CDP) │
└────┘ └────┘ └─────────────┘
```

### Codex Multi-Agent Roles
```
┌───────────────────┐
│   Main Thread     │
│   (config.toml)   │
└───────┬───────────┘
        │ spawn (max_threads)
   ┌────┼────────┐
   │    │        │
┌──┴──┐│    ┌───┴────┐
│work-││    │monitor │
│er   ││    │(watch, │
│     ││    │ alert) │
└─────┘│    └────────┘
  ┌────┴───┐
  │explorer│
  │(read-  │
  │ only)  │
  └────────┘
  Each role: own model, sandbox, instructions
```

### OpenCode Client/Server Architecture
```
┌──────────────────┐     ┌──────────────┐
│  opencode TUI    │     │ opencode web │
│  (Bubble Tea)    │     │   (Web UI)   │
└───────┬──────────┘     └──────┬───────┘
        │                       │
        └───────────┬───────────┘
                    │
           ┌────────┴────────┐
           │  opencode serve │──── REST API ──▶ External Apps
           │  (HTTP Server)  │
           └────────┬────────┘
                    │
       ┌────────────┼────────────┐
       │            │            │
  ┌────┴────┐  ┌───┴───┐  ┌────┴─────┐
  │  LLM    │  │  MCP  │  │ Plugins  │
  │ 75+ any │  │Servers│  │ (JS/TS)  │
  │ provider│  │       │  │ 30+ hooks│
  └─────────┘  └───────┘  └──────────┘
  Supports: OpenAI, Anthropic, Google,
  Groq, Bedrock, Azure, OpenRouter, etc.
```

---

## Cost Model Comparison

| Factor | Claude Code | Copilot CLI | Gemini CLI | Codex CLI | OpenCode |
|--------|:-----------:|:-----------:|:----------:|:---------:|:--------:|
| **Base Cost** | API per-token | Copilot subscription | Free tier / API | API per-token | BYO API key (any provider) |
| **CI/CD Runs** | Anthropic API billing | Free (Copilot API) | Google API billing | OpenAI API billing | Depends on provider |
| **Cloud Agent** | API billing | Included in subscription | Free trial / billing | ChatGPT Plus/Pro | N/A |
| **Subscription Option** | Claude Max ($100-200/mo) | Copilot ($10-39/mo) | Google One AI | ChatGPT Pro ($200/mo) | OpenCode Zen |
| **Open Source** | ❌ | ❌ | ✅ Apache 2.0 | ✅ Apache 2.0 | ✅ Apache 2.0 |

---

## Key Relationship Insights

Analysis of cross-platform relationships reveals structural patterns in how these platforms converge and differentiate.

### Compatibility Triangle

Three platforms form a **strong compatibility cluster** — they can share configuration files with minimal or no conversion:

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

| Platform A | Platform B | Shared Formats |
|-----------|-----------|---------------|
| Claude Code | Copilot CLI | `plugin.json`, SKILL.md spec, same plugin system |
| OpenCode | Claude Code | Reads `.claude/skills/` and `CLAUDE.md` natively |
| OpenCode | Copilot CLI | Reads `.agents/skills/`; `AGENTS.md` standard |
| Gemini CLI | OpenCode | `GEMINI.md` ↔ `AGENTS.md` conceptually similar (medium compatibility) |

> **Implication:** A plugin built for Copilot CLI works on Claude Code with zero changes, and OpenCode can consume skills from both without conversion.

### Capability Dependency Chain

Capabilities aren't independent — they form an **enables chain** where foundational primitives unlock higher-order features:

```
CLI Agent ──→ Headless/SDK Mode ──→ CI/CD Action ──→ Scheduled Eval
                                        │
Skills ──→ Custom Agents ──→ Subagents ──→ Multi-Agent Orchestration
                                        │
Plugin System ──→ Hooks (Lifecycle) ──→ Custom Themes
                                        │
MCP Integration ──→ Web Search, Browser Agent (gap-fill via servers)
```

**Key insight:** Every platform has the full CLI → Headless → CI/CD → Scheduled Eval chain (universal baseline). The differentiation happens in the **extensibility** and **orchestration** branches — platforms without Plugin System also lack Hooks, and platforms without Subagents cannot offer Multi-Agent Orchestration.

### Platform Leadership by Capability

Each platform leads in a specific domain — no single platform dominates all:

| Domain | Leader | Why |
|--------|--------|-----|
| **Hook System** | OpenCode | 30+ plugin events — most granular of any platform (vs 6 on Claude/Copilot, ~8 on Gemini) |
| **Multi-Agent** | Claude Code | Agent Teams with parallel dispatch + git worktree isolation — only platform with true parallel multi-agent |
| **Remote Agents** | Gemini CLI | Agent-to-Agent (A2A) protocol — most standardized inter-agent communication |
| **Security Sandbox** | Copilot CLI | AWF safe-outputs pipeline — all writes go through auditable output channels, never direct filesystem |

### Capability Tiers

| Tier | Count | Capabilities |
|------|:-----:|-------------|
| **Universal** (5/5) | 10 | CLI Agent, CI/CD Action, Headless/SDK, MCP, Custom Agents, Subagents, Skills, Code Review, Context File, Scheduled Eval |
| **Near-Universal** (4/5) | 4 | Cloud Agent, Custom Commands, Hooks, Plugin System |
| **Common** (3/5) | 2 | Multi-Agent Orchestration, Structured Output |
| **Differentiating** (2/5) | 6 | Browser Agent, Custom Themes, LSP Integration, Remote Agents, Safe Outputs, Web Search |
| **Unique** (1/5) | 4 | Bot Skip (Copilot), Git Worktrees (Claude), Network Allowlist (Copilot), Shared Imports (Copilot) |

> **Trend:** The universal tier grew from ~6 to 10 capabilities in 2025-2026 as platforms rapidly converged on MCP, Skills, and Subagents. The differentiating tier is where platform identity lives — choose based on which 2/5 capabilities matter most to your workflow.

### MCP as the Great Equalizer

MCP Integration is universal (5/5), and it **fills gaps** for platforms missing native capabilities:

| Missing Capability | Platforms Without It | MCP Gap-Fill |
|-------------------|---------------------|-------------|
| Web Search | Claude Code, Copilot CLI, OpenCode | `@anthropic/web-search`, `@opencode/web-search`, or custom MCP server |
| Browser Agent | Copilot CLI, Codex CLI, OpenCode | Playwright MCP server provides browser automation |
| LSP Integration | Copilot CLI, Gemini CLI, Codex CLI | Language server MCP bridges (emerging) |

> **Implication:** MCP reduces the effective gap between platforms from 26 capabilities to ~20 — the remaining differences are in orchestration, security, and developer experience rather than raw functionality.

---

## Data Sources (36 total)

### Anthropic (7 sources)
1. https://docs.anthropic.com/en/docs/claude-code/cli-usage
2. https://docs.anthropic.com/en/docs/claude-code/github-actions
3. https://docs.anthropic.com/en/docs/claude-code/plugins
4. https://docs.anthropic.com/en/docs/claude-code/sub-agents
5. https://docs.anthropic.com/en/docs/claude-code/hooks
6. https://docs.anthropic.com/en/docs/claude-code/agent-teams
7. https://github.com/anthropics/claude-code-action

### GitHub Copilot (7 sources)
1. https://github.github.io/gh-aw/introduction/overview/
2. https://github.github.io/gh-aw/authoring-workflows/frontmatter/
3. https://github.github.io/gh-aw/authoring-workflows/safe-outputs/
4. https://github.github.io/gh-aw/security/overview/
5. https://docs.github.com/en/copilot/reference/hooks-configuration
6. https://docs.github.com/en/copilot/concepts/context/mcp
7. https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-hooks

### Google Gemini (7 sources)
1. https://github.com/google-gemini/gemini-cli
2. https://geminicli.com/docs/extensions/
3. https://geminicli.com/docs/extensions/writing-extensions
4. https://github.com/google-github-actions/run-gemini-cli
5. https://cloud.google.com/products/gemini/code-assist
6. https://geminicli.com/docs/core/subagents/
7. https://geminicli.com/docs/core/remote-agents/

### OpenAI Codex (7 sources)
1. https://github.com/openai/codex
2. https://developers.openai.com/codex
3. https://developers.openai.com/codex/cli
4. https://developers.openai.com/codex/multi-agent
5. https://developers.openai.com/codex/sdk
6. https://developers.openai.com/codex/changelog/
7. https://blog.fsck.com/2025/10/27/skills-for-openai-codex/

### OpenCode / Anomaly (8 sources)
1. https://opencode.ai/docs/
2. https://opencode.ai/docs/plugins/
3. https://opencode.ai/docs/agents/
4. https://opencode.ai/docs/skills/
5. https://opencode.ai/docs/github/
6. https://opencode.ai/docs/cli/
7. https://opencode.ai/docs/mcp-servers/
8. https://opencode.ai/docs/themes
