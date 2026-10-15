# Framework Integration Patterns: LangGraph, OpenAI Agents, MCP v2 and OpenTelemetry

These examples show **how the same business gateway can be integrated with popular open-source stacks**. The fully tested offline source is in files 08–09. Optional framework adapters depend on separately installed packages and must be compatibility-tested against pinned versions in your environment. The examples deliberately keep **authorization and transaction ownership in the application**, rather than delegating them to an unconstrained model tool.

## A. LangGraph: stateful orchestration around an independently safe step

Source: [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph). `InMemorySaver` is for a local demonstration, not production persistence. The business request state remains in the lab SQLite store; the graph merely calls one bounded step and routes to the next.

```python
# Optional: pip install langgraph
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from harness_demo import Actor, Store, run

DB_PATH = "workshop.sqlite3"

class GraphState(TypedDict):
    request_id: str
    status: str
    stage: str

def take_one_safe_step(state: GraphState) -> dict:
    store = Store(DB_PATH)
    try:
        output = run(
            store, state["request_id"],
            Actor("alice", "acme", "requester"),
            max_steps=1,
        )
        return {"status": output.status, "stage": output.stage}
    finally:
        store.close()

def route(state: GraphState) -> str:
    return "step" if state["status"] == "IN_PROGRESS" else END

builder = StateGraph(GraphState)
builder.add_node("step", take_one_safe_step)
builder.add_edge(START, "step")
builder.add_conditional_edges("step", route)
graph = builder.compile(checkpointer=InMemorySaver())

result = graph.invoke(
    {"request_id": "REQ-1800", "status": "IN_PROGRESS", "stage": "NEW"},
    {"configurable": {"thread_id": "acme:REQ-1800"}},
)
print(result)
```

This expects that `REQ-1800` has been seeded and submitted as described in the runbook. The graph pauses automatically when the business service returns `WAITING_APPROVAL`; resume by invoking a new graph call after `Store.review()`. **Production:** use a durable checkpointer and ensure graph checkpoint replay is coordinated with external effect semantics. Never assume that `InMemorySaver` survives process restarts.

## B. OpenAI Agents SDK: bounded function tool as a gateway façade

Source: [openai/openai-agents-python](https://github.com/openai/openai-agents-python). The following minimal agent can *request* that the business harness advance one verified step. It cannot call arbitrary SQL or directly approve a purchase. The `@function_tool` wrapper provides schema extraction; it does not itself replace business policy.

```python
# Optional: pip install openai-agents
from agents import Agent, Runner, function_tool
from harness_demo import Actor, Store, run

@function_tool
def advance_approved_workflow_step() -> str:
    """Advance one policy-gated step for a fixed demo request."""
    store = Store("workshop.sqlite3")
    try:
        # For a production tool the tenant/user must come from verified
        # server-side session claims, not from free-form LLM arguments.
        outcome = run(store, "REQ-1800", Actor("alice", "acme", "requester"), max_steps=1)
        return f"status={outcome.status}; stage={outcome.stage}"
    finally:
        store.close()

agent = Agent(
    name="Procurement workflow assistant",
    instructions=(
        "You may only request the next safe workflow step. "
        "Never claim approval or completion without the tool result."
    ),
    tools=[advance_approved_workflow_step],
)
result = Runner.run_sync(agent, "Advance the procurement request by one safe step.")
print(result.final_output)
```

For a real application, the tool must bind to a verified actor and task server-side and use SDK tool guardrails where supported. Read the [SDK guardrails lifecycle](https://github.com/openai/openai-agents-python/blob/main/docs/guardrails.md): an agent-level input/output guardrail alone does not guarantee prechecks for every function tool or all hosted tools. Treat instructions as a soft defense; the `Store` gateway is the hard authorization boundary.

## C. MCP Python SDK v2: expose only a read-only business tool

Source: [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk). **As of the verified October 2026 repository, v2 uses `MCPServer` rather than v1 `FastMCP`.** This sample uses stdio and a fixed synthetic actor for local training. It does not implement production OAuth, so it must not be exposed over a network with real data.

```python
# Optional: pip install 'mcp>=2,<3'
from mcp.server import MCPServer
from harness_demo import Actor, Store

server = MCPServer("TrainingProcurementReadOnly")

@server.tool()
def request_status(request_id: str) -> dict:
    """Read status of the current training principal's request."""
    store = Store("workshop.sqlite3")
    try:
        row = store.get(request_id, Actor("alice", "acme", "requester"))
        return {"request_id": row["request_id"],
                "status": row["status"], "stage": row["stage"]}
    finally:
        store.close()

if __name__ == "__main__":
    server.run(transport="stdio")
```

In production, MCP over Streamable HTTP must validate bearer tokens and scopes on each request; also bind tenant/resource ownership and output redaction. See [MCP SDK authorization](https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/run/authorization.md). The protocol advertises tool capability; it does **not** grant a model independent authority to use every advertised tool.

## D. OpenTelemetry: structured tracing without prompt leakage

Source: [open-telemetry/opentelemetry-python](https://github.com/open-telemetry/opentelemetry-python). This is a minimal local console-export example. The enterprise replacement exports to a managed collector under suitable filtering, retention, authorization and sampling controls.

```python
# Optional: pip install opentelemetry-api opentelemetry-sdk
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor, ConsoleSpanExporter
from harness_demo import Actor, Store, run

provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("enterprise.harness.procurement", "1.0.0")

store = Store("workshop.sqlite3")
try:
    with tracer.start_as_current_span("procurement.workflow") as span:
        span.set_attribute("workflow.request_id", "REQ-1800")
        span.set_attribute("workflow.policy_version", "procurement-v1")
        outcome = run(store, "REQ-1800", Actor("alice", "acme", "requester"))
        span.set_attribute("workflow.status", outcome.status)
        span.set_attribute("workflow.model_input_tokens", outcome.model_input_tokens)
        span.set_attribute("workflow.model_output_tokens", outcome.model_output_tokens)
finally:
    store.close()
```

Do not emit vendor notes, full prompts, secrets or financial personal data as tracing attributes. Consider hashed tenant IDs and separately protected audit records. Instrument model, gateway, approval and effect stages with one correlated trace ID.

## E. Temporal: when business workflows outlive the agent process

Source: [temporalio/sdk-python](https://github.com/temporalio/sdk-python). Temporal offers durable workflow orchestration and retryable activities. The agent is not a substitute for the workflow engine; place the *business side effect* in an activity that uses an ERP idempotency key, not unprotected direct model execution.

```python
# Optional sketch: pip install temporalio
from datetime import timedelta
from temporalio import workflow, activity
from harness_demo import Actor, Store, run

@activity.defn
async def take_one_step(request_id: str) -> dict:
    store = Store("workshop.sqlite3")
    try:
        r = run(store, request_id, Actor("alice", "acme", "requester"), max_steps=1)
        return {"status": r.status, "stage": r.stage}
    finally:
        store.close()

@workflow.defn
class SafeProcurementWorkflow:
    @workflow.run
    async def run(self, request_id: str) -> dict:
        while True:
            result = await workflow.execute_activity(
                take_one_step, request_id,
                start_to_close_timeout=timedelta(seconds=30),
            )
            if result["status"] != "IN_PROGRESS":
                return result
```

This is the workflow and activity *definition*, not a complete Temporal server/worker deployment. For a full deployment, register both with a worker and use a Temporal client, activity timeouts, retry policies and workflow signals/updates for manager approval; consult the repository's runnable samples. Never put I/O, random effects or a direct ERP call in deterministic workflow code. Approval resumption and unknown-effect reconciliation need explicit design.

## F. Which framework to choose?

| Requirement | Strong candidates | Do not confuse with |
|---|---|---|
| Explicit graph transitions and interrupts | LangGraph; Microsoft Agent Framework | Business policy enforcement |
| Lightweight function tools, handoffs, tracing | OpenAI Agents SDK | Enterprise IAM/financial approval |
| Long-horizon context, subagents, skills | Deep Agents; Claude Agent SDK | Transaction integrity |
| Durable long-running business processes | Temporal; durable graph runtime | End-to-end exactly-once ERP effects |
| Standardized agent-to-tool integration | MCP SDK | Permission to perform every tool action |
| Telemetry | OpenTelemetry SDK | Immutable financial audit |
| Prompt/model attack suites | Promptfoo; Inspect AI | Deterministic business correctness |

**Recommended enterprise pattern:** select one orchestration runtime, one model-provider gateway abstraction and one governed tool plane. Retain a separate policy/authorization layer. Avoid mixing multiple agent orchestration frameworks in the critical execution path without a compelling architectural reason and measurable benefit.
