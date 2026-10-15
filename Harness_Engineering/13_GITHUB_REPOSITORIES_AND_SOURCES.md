# Verified GitHub Repositories and Primary Technical References

**Verified against public repository/documentation pages on 13 October 2026.** Repository main branches and interfaces may continue to change. Always pin package versions and review release notes before writing deployment code. The course's 12-layer classification and reference system design are our instructional synthesis—not vendor-endorsed standards.

## A. Model/agent harness and general agent architecture

| Repository or reading | Relevance to harness design | Suggested code/research starting point |
|---|---|---|
| [openai/openai-agents-python](https://github.com/openai/openai-agents-python) | Agents, function tools, handoffs, sessions, human involvement and tracing | `docs/`, `examples/` and [tool guardrails](https://github.com/openai/openai-agents-python/blob/main/docs/guardrails.md) |
| [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) | Low-level, durable graph orchestration with state and interrupts | `libs/langgraph/`; [graph architecture](https://github.com/langchain-ai/docs/blob/main/src/oss/langgraph/graph-api.mdx) |
| [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents) | Opinionated harness on top of LangChain and LangGraph, with tools, files and long-horizon context | [`libs/ARCHITECTURE.md`](https://github.com/langchain-ai/deepagents/blob/main/libs/ARCHITECTURE.md) |
| [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python) | Programmable long-horizon agent harness and sessions | SDK readme and usage samples |
| [anthropics/claude-quickstarts](https://github.com/anthropics/claude-quickstarts) | Concrete long-running coding harness demonstrating initializer and work sessions | [`autonomous-coding/`](https://github.com/anthropics/claude-quickstarts/tree/main/autonomous-coding) |
| [microsoft/agent-framework](https://github.com/microsoft/agent-framework) | Multi-language enterprise agents and workflows, connectors and orchestration | [`python/samples/01-get-started/`](https://github.com/microsoft/agent-framework/tree/main/python/samples/01-get-started) |
| [microsoft/autogen](https://github.com/microsoft/autogen) | Historical multi-agent research and implementation patterns | **Maintenance mode as of verification**; new work should evaluate Microsoft Agent Framework |

## B. Durable workflows and protocol/tool interfaces

| Repository | Primary capability | What it does not automatically provide |
|---|---|---|
| [temporalio/sdk-python](https://github.com/temporalio/sdk-python) | Durable long-running workflows, activities, retries and recovery | A guarantee that an arbitrary external ERP API is duplicate-safe |
| [temporalio/samples-python](https://github.com/temporalio/samples-python) | Runnable worker, activity, replay, encryption, failure and orchestration examples | Organization-specific identity and financial authority |
| [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) | Model Context Protocol clients/servers, typed tools, resources and prompts | Business authorization just because a tool is advertised |
| [modelcontextprotocol/python-sdk v2 migration](https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/migration.md) | Details Python SDK v2 changes: `FastMCP` → `MCPServer` | Backward compatibility for v1 code without migration |
| [OpenAI Agents API architecture](https://developers.openai.com/api/docs/guides/agents-api/architecture) | Distinction between hosted harness, execution environment and application server | Enterprise-specific business approval policies |

## C. Observability, evaluation and adversarial security

| Repository / source | Use in production engineering |
|---|---|
| [open-telemetry/opentelemetry-python](https://github.com/open-telemetry/opentelemetry-python) | Cross-service spans/metrics and correlation infrastructure |
| [open-telemetry/opentelemetry-python-genai](https://github.com/open-telemetry/opentelemetry-python-genai) | Provider/framework GenAI instrumentation; check API stability per package |
| [promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) | Model/app evaluation and automated prompt-injection/red-team testing |
| [UKGovernmentBEIS/inspect_ai](https://github.com/UKGovernmentBEIS/inspect_ai) | Reproducible LLM evaluations, scorers and task configuration |
| [OWASP/secure-agent-playbook](https://github.com/OWASP/secure-agent-playbook) | Agent-specific threat modeling, security review procedures and evidence templates |
| [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) | Agentic threat families and mitigations for review |

## D. Primary technical essays / articles to assign as pre-reading

1. **OpenAI (11 February 2026)** — [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/). Key lesson: constrain architectures and make environments, repository instructions, feedback and app behavior legible to coding agents. It is about software development agents, so this course extends its discipline to business transactions.
2. **OpenAI (23 January 2026)** — [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/). Useful for understanding interaction between model inference, tool calls and loop orchestration.
3. **OpenAI (2026)** — [Unlocking the Codex harness / App Server](https://openai.com/index/unlocking-the-codex-harness/). Useful for persistence, auth/config, tool execution and agent-thread lifecycle.
4. **OpenAI (8 May 2026)** — [Running Codex safely at OpenAI](https://openai.com/index/running-codex-safely/). Focus on security boundaries, approvals, isolation and agent telemetry.
5. **OpenAI (15 April 2026)** — [The next evolution of the Agents SDK](https://openai.com/index/the-next-evolution-of-the-agents-sdk/). Distinction between agent harness and isolated compute environment.
6. **Anthropic (26 November 2025)** — [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents). Persistent context, initialization, artifacts and progress across sessions.
7. **LangChain** — [Deep Agents architecture](https://github.com/langchain-ai/deepagents/blob/main/libs/ARCHITECTURE.md). Distinguishes opinionated harness, agent abstraction and graph runtime.
8. **MCP Python SDK** — [Authorization](https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/run/authorization.md) and [v2 migration](https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/migration.md). Tool interface vs resource-server authorization.

## Recommended repository walkthrough (15–30-minute extension)

**Walkthrough 1: Deep Agents vs LangGraph.** Compare middleware/backends/skills in [`deepagents/libs/ARCHITECTURE.md`](https://github.com/langchain-ai/deepagents/blob/main/libs/ARCHITECTURE.md) against LangGraph's state/checkpoint semantics. Ask which pieces are product-level convenience and which are durable runtime primitives.

**Walkthrough 2: OpenAI Agents SDK guardrails.** Read `docs/guardrails.md` and identify which checks occur at input/output and which apply per tool. Discuss what still requires downstream transaction validation.

**Walkthrough 3: MCP v2 tool server.** Compare v1 `FastMCP` imports with current v2 `MCPServer`, then identify how a network MCP server verifies bearer tokens and enforces per-tool scopes.

**Walkthrough 4: Temporal transaction recovery.** Study a sample with retries, signals and idempotent activities. Ask what happens if an ERP commits just before a worker times out.

**Walkthrough 5: Agentic security.** Use OWASP's agentic threat taxonomy to tag each adversarial test in `09_COMPLETE_TEST_SUITE.md`, then identify threats not covered by the synthetic lab.

## Repository selection criteria for an enterprise

Assess each candidate on the actual target use case rather than GitHub popularity: state durability, human pause/resume, action authorization extension points, external tool schema reliability, model-provider independence, audit event model, deployment control, observability, production SLA, API stability and security maintenance. Proof of concept is not a substitute for an adapter contract test with your real ERP, ITSM or data platform.
