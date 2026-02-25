# Cloud-Native Agent Platform Comparison Matrix

> **Last Updated:** 2025-07-17
> **Methodology:** Structured research crawl of official documentation — AWS docs, Google Cloud docs, Microsoft Learn — with SDK reference verification.

## Introduction

This matrix compares **cloud-native agent platforms** — fully managed services for building, deploying, and operating AI agents at scale. These platforms differ fundamentally from CLI-first coding agents (covered in [`ai-coding-agent-comparison.md`](./ai-coding-agent-comparison.md)), which focus on developer-facing terminal workflows, local tool use, and code editing.

Cloud-native platforms address production concerns: managed hosting, enterprise security, multi-agent orchestration, persistent memory, and observability at scale. The three platforms compared are:

> **⚠️ Disclaimer:** The cloud-service provider API research in this document and all related code examples were generated from official documentation crawls and have **not been tested against live APIs** due to lack of funding for cloud provider accounts. Treat code snippets as illustrative starting points — verify SDK versions, API signatures, and runtime behavior before production use.

| Platform | Provider | SDK Package | Primary Framework |
|----------|----------|-------------|-------------------|
| **Amazon Bedrock Agents** | AWS | `boto3` (`bedrock-agent`, `bedrock-agent-runtime`) | Bedrock Agent API |
| **Vertex AI Agent Engine** | Google Cloud | `google-cloud-aiplatform[agent_engines,adk]` | Agent Development Kit (ADK) |
| **Foundry Agent Service** | Microsoft Azure | `azure-ai-projects` / `azure-ai-agents` | Assistants API-compatible |

---

## Quick Summary

| Dimension | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|-----------|:---------------------:|:----------------------:|:---------------------------:|
| **Primary Model Ecosystem** | Anthropic Claude, Amazon Titan, Llama, Mistral | Gemini 2.x / 3.x family | GPT-4o, GPT-4, o-series, Llama |
| **Agent Framework** | Proprietary ReAct loop + Lambda orchestration | Open-source ADK (Python, TS, Go, Java) | Assistants API-compatible (OpenAI parity) |
| **Multi-Agent** | Supervisor/Worker (native) | Hierarchical + Workflow agents + A2A protocol | Connected Agents + Workflows (preview) |
| **Memory** | Session summary (cross-session) | Memory Bank (LLM-extracted long-term) | Persistent threads (unlimited retention) |
| **Guardrails** | Bedrock Guardrails (6 filter types) | Model Armor + VPC-SC + IAM deny policies | Azure Content Safety (integrated) |
| **Observability** | Built-in traces + CloudWatch | Cloud Trace + Cloud Logging (OpenTelemetry) | Application Insights (OpenTelemetry) |
| **Key Strength** | Deep AWS integration, Lambda extensibility | Open-source ADK, A2A protocol, Google Search grounding | Multi-SDK (.NET/Java/JS/Python), enterprise VNet |
| **Key Limitation** | No native web search tool | Guardrails less mature (IAM-based, no content filter UI) | Connected agents limited to depth-2 |

---

## Full Comparison Matrix

### Agent Runtime

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 1 | **Agent Creation API** | ✅ `create_agent()` via boto3 with lifecycle (`NOT_PREPARED` → `PREPARED`) | ✅ `client.agent_engines.create()` with ADK `Agent` objects | ✅ `create_agent()` via `AIProjectClient` — Assistants API-compatible |
| 2 | **Model Selection** | ✅ Multi-provider: Claude, Titan, Llama, Mistral, Cohere via `foundationModel` | ✅ Gemini family (2.0 Flash, 2.5 Flash/Pro, 3.0 Flash/Pro); ADK is model-agnostic | ✅ Multi-provider via Azure AI deployments: GPT-4o, Llama, Mistral, Cohere |
| 3 | **Custom Orchestration** | ✅ Lambda-based custom orchestration (`orchestrationType='CUSTOM_ORCHESTRATION'`) | ✅ Custom agents via `BaseAgent` subclass + SequentialAgent/ParallelAgent/LoopAgent | ⚠️ Server-side orchestration is opaque; extensible via Azure Functions + Workflows (preview) |
| 4 | **Streaming** | ✅ `invoke_agent()` returns streaming response with configurable guardrail intervals | ✅ Bidirectional streaming; `async_stream_query()` for real-time events | ✅ SDK polling with streaming support; `onResponse` callback pattern |
| 5 | **Prompt Templates** | ✅ 6 override-able templates (pre-processing, orchestration, KB response, post-processing, memory, routing) | ⚠️ Instructions-based; no formal prompt template system (custom agents provide full control) | ⚠️ `instructions` + `additional_instructions` per run; no template override system |

### Tool Use

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 6 | **Function Calling** | ✅ OpenAPI schema + Lambda, or Function Schema + Lambda | ✅ Python functions as tools; OpenAPI tools with OAuth | ✅ Client-side function calling + server-side Azure Functions |
| 7 | **Code Interpreter** | ✅ `AMAZON.CodeInterpreter` — sandboxed Python; 25 concurrent sessions; limited regions | ✅ Managed sandboxed code execution in Agent Engine | ✅ Sandboxed Python; writes/runs code, generates files and charts |
| 8 | **File Search / RAG** | ✅ Knowledge Bases with 6+ vector stores (OpenSearch, Aurora, Pinecone, Kendra, Neptune); hybrid search, reranking, metadata filters | ✅ Vertex AI Search, RAG Engine, Google Search grounding, AlloyDB/Cloud SQL connectors | ✅ File Search with vector stores (auto-chunking, hybrid search, reranking); Azure AI Search integration |
| 9 | **Web Search / Grounding** | ❌ No built-in web search tool | ✅ Google Search grounding with citations, confidence scores, geo-customization, domain exclusion | ✅ Bing Search grounding + Bing Custom Search (scoped domains); Deep Research (preview) |
| 10 | **Return of Control** | ✅ `customControl='RETURN_CONTROL'` — agent returns params to caller for client-side execution | ⚠️ Custom agents can yield control; no formal return-of-control protocol | ✅ `requires_action` run status for function call results; client submits tool outputs |
| 11 | **MCP Support** | ❌ Not documented | ✅ MCP tool servers + MCP Toolbox for Databases | ⚠️ MCP Tool (preview) — endpoint integration |

### Memory & State

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 12 | **Session Persistence** | ✅ Server-side sessions with configurable idle TTL; full CRUD API (`create_session`, `list_sessions`, etc.) | ✅ Managed sessions with 365-day default TTL; full CRUD + event append API | ✅ Persistent threads stored server-side (until deleted); up to 100K messages per thread |
| 13 | **Conversation History** | ✅ Automatic within session; configurable `previousConversationTurnsToInclude` | ✅ Chronological event history per session; shared state via `output_key` across agents | ✅ Thread auto-truncates for context window; full message history persisted |
| 14 | **Long-Term Memory** | ✅ `SESSION_SUMMARY` — async summarization after session end; 1–365 day retention; per-user isolation via `memoryId` | ✅ Memory Bank — LLM-driven extraction, consolidation, similarity search, TTL, revision tracking | ❌ No built-in cross-session memory; threads are the persistence unit |
| 15 | **Knowledge Base** | ✅ Independent KB resources; S3-based data sources; multimodal; NL-to-SQL for structured data | ✅ Vertex AI Search data stores + RAG Engine; Example Store (preview) for few-shot | ✅ Vector stores (auto-created or BYO Azure AI Search); 10K files/store, 512 MB/file |

### Multi-Agent

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 16 | **Native Multi-Agent** | ✅ `agentCollaboration='SUPERVISOR'` or `'SUPERVISOR_ROUTER'`; collaborators via `associate_agent_collaborator()` | ✅ `sub_agents` parameter; hierarchical trees with `find_agent()` navigation | ⚠️ Connected Agents (preview, depth-2 max); Workflows API (newer preview) |
| 17 | **Agent-to-Agent Protocol** | ❌ Proprietary only; no cross-platform protocol | ✅ A2A (Agent2Agent) open protocol — Python, Go, JS, Java, C#/.NET SDKs | ❌ Proprietary only; Semantic Kernel / AutoGen ecosystem |
| 18 | **Supervisor / Worker** | ✅ Native supervisor creates execution plans across collaborators; each collaborator is a full agent | ✅ LLM-driven delegation from parent to sub-agents; workflow agents for deterministic orchestration | ⚠️ Orchestrator agent delegates to connected agents; limited to single-level hierarchy |
| 19 | **Shared State** | ⚠️ `conversationHistory` in `sessionState` for collaborator context; `sessionAttributes` shared within session | ✅ All agents in a hierarchy share `session.state`; `output_key` for inter-agent data passing | ⚠️ Connected agents share thread context; responses visible only to parent agent |

### Deployment

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 20 | **Managed Hosting** | ✅ Fully serverless; no infrastructure provisioning; versioned agents with alias-based deployment | ✅ Managed runtime (Cloud Run-based); auto-scaling 0–1000 instances; CPU/memory resource limits | ✅ Fully managed; model deployments in Foundry project; BYO storage optional |
| 21 | **Regional Availability** | ✅ Multiple AWS regions; Code Interpreter in us-east-1, us-west-2, eu-central-1; cross-region inference profiles | ✅ Multiple GCP regions; VPC-SC limits to 100 instances per resource | ✅ Multiple Azure regions; tool availability varies by region |
| 22 | **VPC / Network Isolation** | ⚠️ Not first-class for agents; Lambda functions and data stores can be VPC-configured | ✅ VPC Service Controls; Private Service Connect with DNS peering; blocks public egress | ✅ Standard setup: customer VNet with private endpoints, dedicated agent subnet, no public egress |

### Security

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 23 | **Auth Model** | ✅ IAM service roles with `sts:AssumeRole`; resource-based policies for Lambda; per-agent role ARN | ✅ Agent Identity (SPIFFE-based mTLS, preview); custom service accounts; IAM deny policies + PAB | ✅ Entra ID only (no API keys); RBAC at project scope; agent gets managed identity on publish |
| 24 | **Guardrails / Safety** | ✅ Bedrock Guardrails: content filters, denied topics, word filters, PII detection/masking, contextual grounding, automated reasoning; versioned | ⚠️ Model Armor (Security Command Center); Vertex AI safety filters; Agent Engine Threat Detection (preview); IAM-based guardrails | ✅ Azure Content Safety: hate/violence/sexual/self-harm categories, prompt injection detection, custom blocklists |
| 25 | **Data Encryption** | ✅ Optional KMS encryption (`customerEncryptionKeyArn`); SSE-KMS for S3 logging | ✅ CMEK via Cloud KMS; data residency zones; HIPAA compliant | ✅ Microsoft-managed or customer-managed encryption keys; BYO Cosmos DB for state |
| 26 | **Audit Logging** | ✅ Model invocation logging to CloudWatch/S3; full request/response capture; CloudTrail for API calls | ✅ Cloud Logging integration; Cloud Audit Logs; Access Transparency | ✅ Enterprise audit trail; Application Insights; Azure Monitor diagnostic logs |

### Observability

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 27 | **Tracing** | ✅ Built-in per-invocation traces (`enableTrace=True`): PreProcessing, Orchestration, PostProcessing, Guardrail, Failure, Routing traces | ✅ Cloud Trace with OpenTelemetry; DAG visualization; span-level inspection with `llm.input_messages` | ✅ OpenTelemetry-based; `AIAgentsInstrumentor`; multi-agent span conventions; Application Insights + Foundry portal |
| 28 | **Metrics & Cost Tracking** | ⚠️ Token counts per trace step; no built-in cost dashboard — use CloudWatch metrics + AWS Cost Explorer | ⚠️ Cloud Monitoring metrics and dashboards; pricing includes free tier; per-query billing for Search grounding | ⚠️ Application Insights metrics; agent playground evaluations (billed); Azure Cost Management at subscription level |
| 29 | **Invocation Step API** | ✅ `list_invocations()`, `list_invocation_steps()`, `get_invocation_step()` — granular step-level inspection and replay | ⚠️ Session event history provides step-level detail; no dedicated invocation step API | ⚠️ Thread run steps visible in playground; run status polling provides step-level visibility |

### SDK & Integration

| # | Axis | Amazon Bedrock Agents | Vertex AI Agent Engine | Azure Foundry Agent Service |
|---|------|:---------------------:|:----------------------:|:---------------------------:|
| 30 | **SDK Languages** | ✅ Python (boto3), plus all AWS SDK languages (Java, .NET, Go, JS, etc.) | ✅ Python (primary); ADK also in TypeScript, Go, Java | ✅ Python, JavaScript/TypeScript, .NET, Java — all with first-class support |
| 31 | **REST API** | ✅ Full AWS API with Signature V4 auth | ✅ Full REST API (`v1beta1`) for all Agent Engine operations | ✅ Full REST API (`2025-05-01` GA); Assistants API-compatible |
| 32 | **CI/CD Integration** | ⚠️ CloudFormation / Terraform for agent definitions; no first-party CI action | ✅ Deploy from source files (CI/CD friendly); Agent Starter Pack with Cloud Build + Terraform | ⚠️ Bicep/Terraform for infrastructure; no first-party GitHub Action for agents |
| 33 | **Event-Driven Triggers** | ✅ Lambda integration enables EventBridge, SQS, SNS triggers for agent actions | ⚠️ Cloud Run-based; integrable with Pub/Sub, Eventarc, Cloud Functions | ✅ Azure Logic Apps tool integration; Azure Functions for event-driven agent actions |
| 34 | **Framework Ecosystem** | ⚠️ Proprietary only; no third-party framework adapters | ✅ ADK, LangChain, LangGraph, AG2, LlamaIndex, CrewAI — widest framework support | ✅ Semantic Kernel, LangChain/LangGraph, OpenAI Agents SDK, AutoGen |

---

## Detailed Platform Profiles

### Amazon Bedrock Agents

**Service:** Amazon Bedrock Agents
**SDK:** `boto3` with `bedrock-agent` (build-time) and `bedrock-agent-runtime` (runtime) clients
**GA Status:** Generally available

**Unique Strengths:**

- **Lambda-Based Extensibility** — Action groups execute via Lambda functions with full AWS service integration, or return control to the caller for client-side execution. Three tool registration methods (OpenAPI + Lambda, Function Schema + Lambda, Return-of-Control) provide maximum flexibility.
- **Bedrock Guardrails** — The most comprehensive guardrails system: content filters (hate, insults, sexual, violence, misconduct, prompt attack), denied topics, word filters, PII detection/masking, contextual grounding checks, and automated reasoning validation. Guardrails are versioned and support cross-region/cross-account enforcement.
- **Knowledge Base Depth** — Six vector store backends (OpenSearch Serverless, Aurora PostgreSQL, Pinecone, Redis, Kendra, Neptune). Supports hybrid search, reranking with Bedrock models, metadata filtering with rich operators, NL-to-SQL, and multimodal embeddings.
- **Custom Orchestration** — Replace the default ReAct loop entirely with a Lambda-based orchestration engine for full control over reasoning flow.
- **Inline Agents** — `invoke_inline_agent()` runs an agent without pre-creation — useful for ephemeral tasks.

**SDK Snippet:**

```python
import boto3

# Build-time
client = boto3.client('bedrock-agent')
agent = client.create_agent(
    agentName='MyAgent',
    instruction='You are a helpful assistant.',
    foundationModel='anthropic.claude-3-5-sonnet-20241022-v2:0',
    agentResourceRoleArn='arn:aws:iam::123456789:role/bedrock-agent-role',
    guardrailConfiguration={'guardrailIdentifier': 'my-guardrail', 'guardrailVersion': '1'},
    memoryConfiguration={'enabledMemoryTypes': ['SESSION_SUMMARY'], 'storageDays': 30},
)

# Runtime
runtime = boto3.client('bedrock-agent-runtime')
response = runtime.invoke_agent(
    agentId=agent['agent']['agentId'],
    agentAliasId='TSTALIASID',
    sessionId='session-001',
    inputText='Hello!',
    enableTrace=True,
)
```

**Key Sources:**
- https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/agents-create.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/agents-multi-agent-collaboration.html
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
- https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agent.html
- https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agent-runtime.html

---

### Vertex AI Agent Engine

**Service:** Vertex AI Agent Engine (part of Agent Builder)
**SDK:** `google-cloud-aiplatform[agent_engines,adk]` ≥ 1.112
**Framework:** Agent Development Kit (ADK) — open-source, model-agnostic
**GA Status:** Generally available (some features in preview)

**Unique Strengths:**

- **Open-Source ADK** — The Agent Development Kit is fully open-source with multi-language support (Python, TypeScript, Go, Java). Agents are portable between local development and cloud deployment.
- **A2A Protocol** — The only platform with a cross-framework, cross-platform agent-to-agent communication protocol. Agents built with different frameworks (ADK, LangChain, CrewAI) can interoperate.
- **Widest Framework Support** — First-class integration with ADK, LangChain, LangGraph, AG2, LlamaIndex, and CrewAI. Custom frameworks supported via template pattern.
- **Google Search Grounding** — Native web search with segment-level citations, confidence scores, geo-customization, and domain exclusion. Grounding metadata is structured and verifiable.
- **Memory Bank** — Most advanced long-term memory: LLM-driven extraction, memory consolidation (evolving knowledge), similarity search, revision tracking, and automatic expiration. Scoped to user+agent identity.
- **Workflow Agents** — Deterministic orchestration primitives (SequentialAgent, ParallelAgent, LoopAgent) composable with LLM agents for hybrid workflows.
- **Agent Identity** — SPIFFE-based identity with mTLS binding and certificate-bound tokens (RFC 8705) for non-replayable credentials.

**SDK Snippet:**

```python
import vertexai
from google.adk.agents import Agent
from vertexai import agent_engines

client = vertexai.Client(project="my-project", location="us-central1")

def get_exchange_rate(currency_from: str, currency_to: str) -> dict:
    """Retrieves exchange rate between two currencies."""
    import requests
    return requests.get(f"https://api.frankfurter.app/latest",
                        params={"from": currency_from, "to": currency_to}).json()

agent = Agent(
    model="gemini-2.5-flash",
    name="exchange_agent",
    instruction="You help with currency exchange rates.",
    tools=[get_exchange_rate],
)

app = agent_engines.AdkApp(agent=agent)
remote = client.agent_engines.create(agent=app, config={
    "display_name": "Exchange Agent",
    "requirements": ["google-cloud-aiplatform[agent_engines,adk]"],
})
```

**Key Sources:**
- https://cloud.google.com/agent-builder/agent-engine/overview
- https://cloud.google.com/agent-builder/agent-development-kit/overview
- https://google.github.io/adk-docs/
- https://cloud.google.com/agent-builder/agent-engine/sessions/overview
- https://cloud.google.com/agent-builder/agent-engine/memory-bank/overview
- https://cloud.google.com/agent-builder/agent-engine/agent-identity
- https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/ground-with-google-search
- https://a2a-protocol.org

---

### Azure Foundry Agent Service

**Service:** Microsoft Foundry Agent Service (formerly Azure AI Agent Service)
**SDK:** `azure-ai-projects` (Python), `@azure/ai-agents` (JS/TS), `Azure.AI.Agents.Persistent` (.NET), `azure-ai-agents-persistent` (Java)
**API Version:** `2025-05-01` (GA)
**GA Status:** Generally available

**Unique Strengths:**

- **Broadest SDK Coverage** — First-class SDKs in Python, JavaScript/TypeScript, .NET, and Java. The only platform with production-grade .NET and Java agent SDKs.
- **Assistants API Compatibility** — Thread/Message/Run model compatible with the OpenAI Assistants API, easing migration from OpenAI to Azure.
- **Enterprise VNet Architecture** — Standard setup provides full network isolation: customer VNet, dedicated agent subnet (delegated to `Microsoft.App/environments`), private endpoints for all backing services, no public egress. Deployable via Bicep or Terraform.
- **Richest Tool Catalog** — 12+ built-in tools including Code Interpreter, File Search, Bing Search, Azure AI Search, Logic Apps, OpenAPI, MCP (preview), Microsoft Fabric (preview), Deep Research (preview), and Browser Automation.
- **Entra ID-Only Auth** — No API key authentication; all access via Azure Entra ID with full RBAC, conditional access, and audit trail. Published agents receive their own managed identity.
- **BYO Storage** — Standard setup uses customer-provisioned Azure Cosmos DB (with multi-region failover) for thread/agent state, and customer Azure Blob Storage for file data.
- **Thread Persistence** — Threads persist indefinitely until explicitly deleted, with up to 100,000 messages per thread. No TTL-based expiration by default.

**SDK Snippet:**

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.agents.models import CodeInterpreterTool

client = AIProjectClient(
    endpoint="https://my-foundry.services.ai.azure.com/api/projects/my-project",
    credential=DefaultAzureCredential(),
)

code_tool = CodeInterpreterTool()
agent = client.agents.create_agent(
    model="gpt-4o",
    name="analyst",
    instructions="You analyze data and create visualizations.",
    tools=code_tool.definitions,
    tool_resources=code_tool.resources,
)

thread = client.agents.threads.create()
client.agents.messages.create(thread_id=thread.id, role="user", content="Analyze this dataset.")
run = client.agents.runs.create_and_process(thread_id=thread.id, agent_id=agent.id)
```

**Key Sources:**
- https://learn.microsoft.com/azure/ai-services/agents/overview
- https://learn.microsoft.com/azure/ai-services/agents/quickstart
- https://learn.microsoft.com/azure/ai-services/agents/how-to/tools/overview
- https://learn.microsoft.com/azure/ai-services/agents/concepts/threads-runs-messages
- https://learn.microsoft.com/azure/ai-services/agents/how-to/connected-agents
- https://learn.microsoft.com/azure/ai-services/agents/how-to/virtual-networks
- https://learn.microsoft.com/azure/ai-services/agents/concepts/tracing
- https://learn.microsoft.com/python/api/overview/azure/ai-projects-readme

---

## Cross-Platform Analysis

### Leadership by Domain

| Domain | Leader | Why |
|--------|--------|-----|
| **Guardrails & Content Safety** | 🥇 **Bedrock** | Six filter types with versioning, cross-region enforcement, standalone API, automated reasoning — most comprehensive by far |
| **Multi-Agent Orchestration** | 🥇 **Vertex AI** | Workflow agents (Sequential, Parallel, Loop) + LLM delegation + A2A open protocol; most flexible orchestration primitives |
| **Long-Term Memory** | 🥇 **Vertex AI** | Memory Bank with LLM-driven extraction, consolidation, revision tracking, similarity search — far beyond session summaries |
| **Enterprise Networking** | 🥇 **Azure** | Customer VNet with private endpoints, dedicated subnet, no public egress, Bicep/Terraform-deployable — production-grade from day one |
| **Framework Ecosystem** | 🥇 **Vertex AI** | ADK + LangChain + LangGraph + AG2 + LlamaIndex + CrewAI — widest framework support; ADK is open-source |
| **Tool Catalog Breadth** | 🥇 **Azure** | 12+ tools including Bing Search, Logic Apps, Fabric, Deep Research, Browser Automation — richest built-in catalog |
| **Web Search / Grounding** | 🥇 **Vertex AI** | Google Search grounding with segment-level citations and confidence scores; Azure's Bing is close but less structured |
| **SDK Breadth** | 🥇 **Azure** | First-class Python, JS/TS, .NET, Java — only platform with production .NET and Java SDKs |
| **Observability** | 🥇 **Azure** | OpenTelemetry-native with multi-agent semantic conventions; Application Insights integration; Foundry portal thread logs |
| **Serverless Simplicity** | 🥇 **Bedrock** | True serverless with zero infrastructure; alias-based deployment; provisioned throughput option |

### Key Trade-Offs

| Trade-Off | Bedrock | Vertex AI | Azure |
|-----------|---------|-----------|-------|
| **Open vs. Proprietary** | Proprietary API; no framework adapters | Open-source ADK; A2A protocol; multi-framework | Assistants API-compatible; Semantic Kernel ecosystem |
| **Flexibility vs. Simplicity** | Lambda-based extensibility = maximum flexibility, more wiring | Workflow agents + LLM delegation = structured flexibility | Server-side orchestration = simplest to start, less control |
| **Memory Model** | Session summaries (good for many use cases, limited depth) | Memory Bank (powerful but requires LLM processing costs) | Persistent threads (simple, no cross-session intelligence) |
| **Network Isolation** | Lambda/data stores in VPC; agents themselves not VPC-first | VPC-SC perimeter; no public egress when configured | Full VNet integration with dedicated subnet and private endpoints |
| **Multi-Agent Depth** | Supervisor/collaborator with full agent capabilities per collaborator | Unlimited hierarchy depth; parallel/sequential/loop patterns | Depth-2 max for connected agents; Workflows API emerging |

### Key Differentiators

1. **Bedrock's Return-of-Control** — Unique pattern where the agent returns parameters to the caller for client-side execution, enabling hybrid architectures where sensitive operations stay on-premises.

2. **Vertex AI's A2A Protocol** — The only cloud platform with an open, cross-framework agent-to-agent communication standard. Agents built with ADK, LangChain, or CrewAI can discover and communicate with each other.

3. **Azure's Assistants API Compatibility** — Direct compatibility with the OpenAI Assistants API model (threads, messages, runs) means existing OpenAI codebases migrate with minimal changes.

4. **Bedrock's Custom Orchestration** — Replace the entire ReAct reasoning loop with a Lambda function. No other platform allows this level of orchestration customization.

5. **Vertex AI's Agent Starter Pack** — Production-ready templates with Terraform infrastructure, CI/CD pipelines, observability, and interactive playground out of the box.

6. **Azure's BYO Infrastructure** — Standard setup provisions customer-owned Cosmos DB, Blob Storage, and AI Search, giving full data sovereignty and BCDR control.

---

## Hybrid Integration Patterns: CLI ↔ Cloud

CLI-first coding agents and cloud-native agent platforms serve complementary roles. The following patterns show how they combine for end-to-end workflows.

| Pattern | CLI Tool Role | Cloud Agent Role | Example |
|---------|--------------|-----------------|---------|
| **Dev → Prod Pipeline** | Local development, iteration, and testing with interactive agent | Production runtime with managed hosting, scaling, and monitoring | Develop and debug agent logic with Claude Code locally → deploy to Bedrock Agents with `create_agent()` + Lambda action groups |
| **CI → Cloud Dispatch** | CI/CD action triggers evaluation or orchestration | Cloud agent executes complex multi-step tasks at scale | `gh aw` or `claude-code-action` triggers CI → dispatches workload to Vertex AI Agent Engine for production inference |
| **Skill Portability** | Author `SKILL.md` files with domain knowledge using CLI agent | Upload skills as knowledge base documents for cloud RAG | Author `SKILL.md` with Copilot CLI → ingest into Bedrock Knowledge Base (S3 data source) or Vertex AI Search data store |
| **Eval from Cloud** | Cloud agent executes evaluations with code interpreter | Results scored by local evaluation framework | Azure Foundry Agent + Code Interpreter runs `gepa_evaluator.py` → scores returned to CLI for aggregation and reporting |
| **Local Prototype → Cloud Scale** | Rapid prototyping with local tools, MCP servers, and file access | Scale validated prototype to multi-user, multi-region deployment | Build agent with Gemini CLI + MCP tools → deploy identical ADK agent to Agent Engine with `client.agent_engines.create()` |
| **Hybrid RAG Pipeline** | CLI agent curates, indexes, and tests knowledge base content | Cloud agent serves RAG queries at scale with managed vector search | OpenCode + grep/glob curates docs → synced to Azure AI Search index → Foundry Agent serves file search at scale |
| **Multi-Agent Handoff** | CLI agent handles developer-facing tasks (code review, refactoring) | Cloud agent handles user-facing tasks (customer support, data analysis) | Copilot CLI reviews PR and generates summary → summary sent as context to Bedrock supervisor agent for customer-facing response |
| **Observability Bridge** | CLI agent generates traces and logs during development | Cloud observability stack aggregates and visualizes across environments | Local Claude Code traces → exported via OpenTelemetry → Application Insights (Azure) or Cloud Trace (GCP) for unified dashboards |

### Pattern Details

**Dev → Prod Pipeline:** The most common pattern. CLI agents provide rapid iteration with direct file access, terminal integration, and interactive debugging. Once agent logic is validated locally, the same instructions, tool definitions, and knowledge base content deploy to a cloud platform for production use. Bedrock's alias-based versioning and Vertex AI's source-file deployment make this CI/CD-friendly.

**Skill Portability:** `SKILL.md` files authored with CLI agents (Copilot CLI, Claude Code, Gemini CLI) contain structured domain knowledge. These files can be directly ingested as documents into cloud knowledge bases:
- **Bedrock**: Upload to S3 → create data source → `start_ingestion_job()`
- **Vertex AI**: Upload to GCS → create data store → index
- **Azure**: Upload to Blob Storage → add to vector store → `create_and_poll()`

**Eval from Cloud:** Cloud code interpreters provide sandboxed, reproducible execution environments. Evaluation scripts (like `gepa_evaluator.py`) run in cloud sandboxes with consistent dependencies, while results flow back to the CLI environment for local analysis, comparison, and iteration.

**Observability Bridge:** Both CLI and cloud agents can emit OpenTelemetry traces. By configuring the same OTLP endpoint, development and production traces appear in a single observability backend — enabling end-to-end visibility from local development through production deployment.

---

## Data Sources

All research data was gathered via structured documentation crawls in July 2025.

### Amazon Bedrock Agents
| Source | URL |
|--------|-----|
| Agent overview | https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html |
| Create agent | https://docs.aws.amazon.com/bedrock/latest/userguide/agents-create.html |
| Action groups | https://docs.aws.amazon.com/bedrock/latest/userguide/agents-action-create.html |
| Knowledge bases | https://docs.aws.amazon.com/bedrock/latest/userguide/agents-kb-add.html |
| Code interpreter | https://docs.aws.amazon.com/bedrock/latest/userguide/agents-code-interpretation.html |
| Multi-agent | https://docs.aws.amazon.com/bedrock/latest/userguide/agents-multi-agent-collaboration.html |
| Memory | https://docs.aws.amazon.com/bedrock/latest/userguide/agents-memory.html |
| Guardrails | https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html |
| Trace events | https://docs.aws.amazon.com/bedrock/latest/userguide/trace-events.html |
| Model invocation logging | https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html |
| boto3 bedrock-agent | https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agent.html |
| boto3 bedrock-agent-runtime | https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-agent-runtime.html |

### Vertex AI Agent Engine
| Source | URL |
|--------|-----|
| Agent Builder overview | https://cloud.google.com/agent-builder/overview |
| ADK overview | https://cloud.google.com/agent-builder/agent-development-kit/overview |
| ADK open-source docs | https://google.github.io/adk-docs/ |
| Agent Engine overview | https://cloud.google.com/agent-builder/agent-engine/overview |
| Deploy agent | https://cloud.google.com/agent-builder/agent-engine/deploy |
| Sessions | https://cloud.google.com/agent-builder/agent-engine/sessions/overview |
| Memory Bank | https://cloud.google.com/agent-builder/agent-engine/memory-bank/overview |
| Agent identity | https://cloud.google.com/agent-builder/agent-engine/agent-identity |
| Tracing | https://cloud.google.com/agent-builder/agent-engine/manage/tracing |
| Grounding with Search | https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/ground-with-google-search |
| Multi-agent (ADK) | https://google.github.io/adk-docs/agents/multi-agents/ |
| A2A Protocol | https://a2a-protocol.org |
| Agent Starter Pack | https://github.com/GoogleCloudPlatform/agent-starter-pack |
| Python SDK reference | https://cloud.google.com/python/docs/reference/aiplatform/latest |
| Pricing | https://cloud.google.com/vertex-ai/pricing#vertex-ai-agent-engine |

### Azure Foundry Agent Service
| Source | URL |
|--------|-----|
| Agent overview | https://learn.microsoft.com/azure/ai-services/agents/overview |
| Quickstart | https://learn.microsoft.com/azure/ai-services/agents/quickstart |
| Tools overview | https://learn.microsoft.com/azure/ai-services/agents/how-to/tools/overview |
| Threads, runs, messages | https://learn.microsoft.com/azure/ai-services/agents/concepts/threads-runs-messages |
| Connected agents | https://learn.microsoft.com/azure/ai-services/agents/how-to/connected-agents |
| File search | https://learn.microsoft.com/azure/ai-services/agents/how-to/tools/file-search |
| Virtual networks | https://learn.microsoft.com/azure/ai-services/agents/how-to/virtual-networks |
| Tracing | https://learn.microsoft.com/azure/ai-services/agents/concepts/tracing |
| Python SDK | https://learn.microsoft.com/python/api/overview/azure/ai-projects-readme |
