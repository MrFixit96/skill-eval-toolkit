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

---

## Full Comparison Matrix

### Core Agent Capabilities

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|-----------|:-----------:|:-----------:|:----------:|:---------:|
| **CLI Agent** | ✅ `claude` | ✅ `copilot` | ✅ `gemini` | ✅ `codex` |
| **Cloud Agent** | ✅ `--remote` | ✅ Coding agent | ✅ Jules (ext) | ✅ Codex Web |
| **CI/CD Action** | ✅ `claude-code-action@v1` | ✅ `gh aw` | ✅ `run-gemini-cli` | ✅ `codex-action@v1` |
| **Headless/SDK** | ✅ `claude -p` | ✅ `copilot -p` | ✅ `gemini -p` | ✅ `codex exec` + SDK |
| **Code Review** | ✅ Plugin agent | ✅ `@copilot` on PRs | ✅ `/review` | ✅ `/review` |
| **Browser Agent** | ✅ `--chrome` | ❌ | ✅ `browser_agent` | ❌ |
| **Web Search** | ❌ | ❌ | ✅ Google Search | ✅ Built-in |

### Extensibility & Plugin System

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|-----------|:-----------:|:-----------:|:----------:|:---------:|
| **Plugin System** | ✅ `.claude-plugin/` | ✅ `plugin.json` | ✅ Extensions | ❌ |
| **Skills (Knowledge)** | ✅ `SKILL.md` | ✅ `SKILL.md` | ✅ `SKILL.md` | ❌ |
| **Custom Agents** | ✅ `.claude/agents/` | ✅ `.github/agents/` | ✅ `.gemini/agents/` | ✅ `config.toml` roles |
| **Custom Commands** | ✅ Commands in plugins | ✅ Slash commands | ✅ `/cmd` in `commands/` | ❌ |
| **Context File** | ✅ `CLAUDE.md` | ✅ `copilot-instructions.md` | ✅ `GEMINI.md` | ✅ `AGENTS.md` |
| **Hooks (Lifecycle)** | ✅ `hooks.json` | ✅ `hooks.json` | ✅ Hooks | ❌ |
| **MCP Integration** | ✅ `.mcp.json` | ✅ GitHub MCP Server | ✅ `gemini-extension.json` | ✅ `config.toml` |
| **Custom Themes** | ❌ | ❌ | ✅ `themes/` | ❌ |

### Multi-Agent & Orchestration

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|-----------|:-----------:|:-----------:|:----------:|:---------:|
| **Subagents** | ✅ `agents/*.md` | ✅ Task tool | ✅ `agents/*.md` | ✅ Agent roles |
| **Multi-Agent Teams** | ✅ Agent Teams | ✅ Orchestrator→Workers | ❌ | ✅ Multi-agent (exp.) |
| **Remote Agents** | ❌ | ❌ | ✅ A2A Protocol | ❌ |

### Security & Sandboxing

| Capability | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|-----------|:-----------:|:-----------:|:----------:|:---------:|
| **Safe Outputs** | ❌ | ✅ Threat detection pipeline | ❌ | ✅ Sandbox modes |
| **Network Allowlist** | ❌ | ✅ `network: domains:` | ❌ | ❌ |
| **Git Worktrees** | ✅ `--worktree` | ❌ | ❌ | ❌ |
| **LSP Integration** | ✅ `.lsp.json` | ❌ | ❌ | ❌ |
| **Structured Output** | ✅ `--json-schema` | ❌ | ✅ `--output-format json` | ❌ |
| **Bot Skip** | ❌ | ✅ `on.skip-bots:` | ❌ | ❌ |
| **Shared Imports** | ❌ | ✅ `imports:` | ❌ | ❌ |

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
| **Cross-machine agents** | Gemini CLI | — | Only platform with A2A remote agent protocol |
| **Browser automation** | Gemini CLI | Claude Code | Chrome DevTools + visual model; Claude has `--chrome` |
| **Knowledge management** | Claude Code | Copilot CLI | SKILL.md auto-invocation; Copilot shares the same format |
| **Cost-sensitive teams** | Copilot CLI | Gemini CLI | Included with Copilot subscription; Gemini has free tier |
| **Open-source preference** | Gemini CLI | Codex CLI | Both Apache 2.0; Gemini has extensions gallery |

### Hook Lifecycle Events Comparison

| Event | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|-------|:-----------:|:-----------:|:----------:|:---------:|
| Session Start | ✅ | ✅ `sessionStart` | ✅ | ❌ |
| Session End | ✅ | ✅ `sessionEnd` | ✅ | ❌ |
| User Prompt | ✅ | ✅ `userPromptSubmitted` | ✅ | ❌ |
| Pre-Tool Use | ✅ `PreToolUse` | ✅ `preToolUse` | ✅ | ❌ |
| Post-Tool Use | ✅ `PostToolUse` | ✅ `postToolUse` | ✅ | ❌ |
| Error | ✅ | ✅ `errorOccurred` | ✅ | ❌ |
| Stop/Completion | ✅ `Stop` | via sessionEnd | ✅ | ❌ |
| Subagent Events | ✅ `SubagentPreToolUse` | ❌ | ❌ | ❌ |

### Context File Comparison

| Feature | `CLAUDE.md` | `copilot-instructions.md` | `GEMINI.md` | `AGENTS.md` |
|---------|:-----------:|:-------------------------:|:-----------:|:-----------:|
| Auto-loaded | ✅ | ✅ | ✅ | ✅ |
| Hierarchical | ✅ (parent dirs) | ✅ (.github/) | ✅ | ✅ |
| Org-level | ✅ | ✅ | ❌ | ❌ |
| Memory/Learning | ❌ | ✅ (Copilot Memory) | ❌ | ❌ |
| Agent Config | ❌ (separate file) | ❌ (separate file) | ❌ (separate file) | ✅ (inline) |

### Plugin/Extension Primitives

| Primitive | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|-----------|:-----------:|:-----------:|:----------:|:---------:|
| Commands | ✅ `commands/*.md` | ✅ `commands/*.md` | ✅ `commands/*.toml` | ❌ |
| Agents | ✅ `agents/*.md` | ✅ `agents/*.md` | ✅ `agents/*.md` | ❌ |
| Skills | ✅ `skills/*/SKILL.md` | ✅ `skills/*/SKILL.md` | ✅ `skills/*/SKILL.md` | ❌ |
| Hooks | ✅ `hooks.json` | ✅ `hooks.json` | ✅ In extension | ❌ |
| MCP Servers | ✅ `.mcp.json` | ✅ MCP config | ✅ `gemini-extension.json` | ✅ `config.toml` |
| LSP | ✅ `.lsp.json` | ❌ | ❌ | ❌ |
| Themes | ❌ | ❌ | ✅ `themes/` | ❌ |
| Manifest | `plugin.json` | `plugin.json` | `gemini-extension.json` | N/A |
| Install Method | `claude plugin add` | `plugin install <url>` | `gemini extensions install` | N/A |
| Marketplace | ✅ (emerging) | ✅ (emerging) | ✅ geminicli.com | ❌ |

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

---

## Cost Model Comparison

| Factor | Claude Code | Copilot CLI | Gemini CLI | Codex CLI |
|--------|:-----------:|:-----------:|:----------:|:---------:|
| **Base Cost** | API per-token | Copilot subscription | Free tier / API | API per-token |
| **CI/CD Runs** | Anthropic API billing | Free (Copilot API) | Google API billing | OpenAI API billing |
| **Cloud Agent** | API billing | Included in subscription | Free trial / billing | ChatGPT Plus/Pro |
| **Subscription Option** | Claude Max ($100-200/mo) | Copilot ($10-39/mo) | Google One AI | ChatGPT Pro ($200/mo) |
| **Open Source** | ❌ | ❌ | ✅ Apache 2.0 | ✅ Apache 2.0 |

---

## Data Sources (28 total)

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
