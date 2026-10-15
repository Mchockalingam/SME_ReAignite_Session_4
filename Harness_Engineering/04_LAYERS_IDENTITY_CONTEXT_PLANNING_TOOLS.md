# Technical Deep Dive: Harness Layers 1–6

For each layer, the implementation question is: **Who owns this decision, what is its typed contract, what can fail, how does it fail closed, and what evidence proves the control works?** See `08_COMPLETE_HARNESS_SOURCE.md` for runnable code; small snippets below emphasize boundaries rather than duplicating the entire application.

## Layer 1 — Identity, task intake and tenant boundary

**Responsibility:** Bind a business request to a verified principal, tenant, business resource and permissible intent. Do not let the agent or user prompt determine roles or resource scope.

**Contract:** `Actor(user_id, tenant_id, role)` (lab only) plus `PurchaseRequest(request_id, requester_id, tenant_id, vendor_id, cost_center, amount_cents, ...)`. Inputs need schema, length, currency and business constraints. In production, `Actor` must be derived from verified OIDC/JWT claims and group entitlements, not request fields or CLI switches.

**Invariant:** `actor.tenant_id == request.tenant_id`, requester ownership and role allowed for requested operation. A production resource query should filter the tenant at the DB query layer and preferably enforce DB row-level security as defense in depth.

```python
# Core lab: tenant-aware lookup (does not trust the model's request content)
row = db.execute(
    "SELECT * FROM requests WHERE request_id=? AND tenant_id=?",
    (request_id, actor.tenant_id),
).fetchone()
if row is None:
    raise HarnessError("Request not found in this tenant")
```

**Failure modes:** forged role; user ID taken from prompt; shared cache namespace; changing tenant ID via tool argument. **Tests:** cross-tenant access denied, other requester denied. **Telemetry:** denial counts by tenant and action (without logging raw tokens or notes).

## Layer 2 — Context engineering, retrieval and memory

**Responsibility:** Assemble minimum relevant evidence with access control, provenance and freshness. Separate (a) system policy, (b) authenticated task data, (c) verified tool evidence, (d) untrusted free text. Treat retrieved documents as data even when they contain imperative language.

**Contract:** source ID, tenant, ACL, document version, timestamp, trust label, text excerpt, estimated token budget. **Context budget:** consider `B_context = B_static + B_current_task + B_verified_evidence + B_history`; reject or summarize content before exceeding the model context window.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Evidence:
    source_id: str
    tenant_id: str
    trust: str      # authoritative | verified_tool | untrusted_external
    content: str
    version: str

def build_context(items: list[Evidence], tenant_id: str, max_chars: int = 3000):
    visible = [e for e in items if e.tenant_id == tenant_id]
    return [dict(source_id=e.source_id, trust=e.trust,
                 version=e.version, content=e.content[:max_chars])
            for e in visible]
```

**Important:** Truncation and trust labels are not a prompt-injection defense on their own; authorization must remain downstream. In production, add document ACL at query time, data classification, retention/expiry, versioned retrieval indexes, poisoning tests and citations in final explanatory outputs. A vector database **is not** an authorization system.

**Failure modes:** stale policy excerpt; corpus poisoning; irrelevant chunks; context overflow; hidden instructions inside a vendor email; cross-tenant embeddings. **Tests:** retrieval ACL assertions, outdated-policy rejection, injection corpus. **Telemetry:** context bytes/tokens, retrieval hit rate, provenance coverage and redacted source IDs.

## Layer 3 — Model gateway and budget governance

**Responsibility:** Provide a replaceable inference provider interface, model capability checks, structured outputs, retry limits, per-call timeout, rate quota and token/cost observability. No business system should require a specific model name to preserve its authorization invariant.

**Contract:** `propose(state) -> Plan(action, input_tokens, output_tokens)`. In `OpenAIPlanner`, the Responses API is asked to return a strict JSON object with a single enum field `action`. Even valid JSON must be checked against current stage, policy and permission downstream.

```python
schema = {
    "type": "object",
    "properties": {"action": {"type": "string", "enum": [
        "check_vendor", "check_budget", "read_policy", "assess", "create_po"]}},
    "required": ["action"], "additionalProperties": False,
}
# See OpenAIPlanner in the full code for the API request and parse checks.
```

**Budget mechanics:** max turns, provider request timeout, max generated output, input context cap, per-tenant rate quota and observed usage. For financial planning, compute provider expense as `input_tokens × input_unit_price + output_tokens × output_unit_price`, with actual provider prices/versioning; apply cached-token rates where relevant. Usage returned after a model call cannot prevent that already-issued call from consuming its tokens: use pre-request limits plus post-call circuit breaking.

**Failure modes:** schema refusal, provider outage, unpredictable model switching, silently expensive context, sensitive prompt export. **Tests:** malformed response, timeout, token-cap stop. **Telemetry:** model/version, input/output/cache token counts, cost per successful workflow, P95 model latency, fallback frequency.

## Layer 4 — Planner and bounded decision loop

**Responsibility:** Allow a model to propose the next *eligible* action while preventing unbounded loops and goal drift. Explicitly distinguish planning from executing.

**Contract:** model proposes a typed `Plan`; it does not supply privileged financial values, credentials, role changes or approval decisions. In the lab, `RulePlanner` follows `NEXT_ACTION` and `OpenAIPlanner` uses the exact same execution gateway.

```python
NEXT_ACTION = {
    "NEW": "check_vendor", "VENDOR_OK": "check_budget",
    "BUDGET_OK": "read_policy", "POLICY_OK": "assess",
    "READY": "create_po",
}
# The model can propose a different value; it will not pass the gateway.
```

**Engineering choices:** single-step decision vs plan-then-execute, reflection loops, specialized planners, parallel plan candidates, supervisor routing. Default to the simplest controller that meets quality targets. An agent should stop on terminal, denied, waiting, expired budget or ambiguity requiring human input.

**Failure modes:** infinite retries, goal hijack, hallucinated action names, plan races. **Tests:** malicious planner, model budget, unexpected tool selection. **Telemetry:** steps/run, unexpected-plan frequency, token-to-useful-outcome ratio.

## Layer 5 — Orchestration, graph state and checkpoint

**Responsibility:** Own the authoritative workflow state; guard stage transitions; checkpoint results; resume after interruption without repeating successful irreversible steps. Distinguish **workflow state** from informal conversational memory.

**Contract:** `stage`, `status`, version, request snapshot, terminal reason, and event ledger. The demonstrated state machine has `NEW → VENDOR_OK → BUDGET_OK → POLICY_OK → READY/WAITING_APPROVAL → DONE/BLOCKED`.

```python
expected = NEXT_ACTION.get(state["stage"])
if state["status"] != "IN_PROGRESS" or action != expected:
    # audit DENIED_ACTION; reject the proposal after recording it
    raise HarnessError("Action denied by the execution gateway")
```

The production equivalent may use LangGraph for graph state and interrupts or Temporal for long-running durable business workflows. **Do not** assume a framework checkpoint makes an external ERP side effect exactly once; wrap effects in idempotent activities and reconcile unknown outcomes.

**Failure modes:** partially written stage, checkpoint replay, concurrent actor racing over same request, stale handoff. **Tests:** restart after two steps, replay after completion, two requests competing for budget. **Telemetry:** transition counts, stuck states, pause-to-resume latency, abandoned workflows.

## Layer 6 — Tool registry, typed contracts and system integration

**Responsibility:** Expose narrowly scoped functions with explicit input/output schemas, timeout, scoped credentials and categorized effects. Tools should be categorized into `read`, `reversible_write`, `irreversible_write`, and `privileged_admin`. Disallow privileged tool discovery by default.

**Contract:** the planner chooses the **action name only**. The server obtains `vendor_id`, `cost_center`, `amount_cents`, `request_id` and `tenant_id` from an authenticated persisted request, not model-produced tool arguments. The source implementation's `execute_action()` maps the action to a finite code branch.

```python
if action == "check_vendor":
    vendor = db.execute(
        "SELECT active, verified FROM vendors WHERE tenant_id=? AND vendor_id=?",
        (actor.tenant_id, state["vendor_id"]),
    ).fetchone()
    allowed = bool(vendor and vendor["active"] and vendor["verified"])
```

When using MCP, expose tools through a protocol but still perform server-side authentication and authorization. **MCP is a transport/interface standard, not automatically a business policy engine.** The 2026 MCP Python SDK v2 renamed `FastMCP` to `MCPServer`; examples using the older import may be incompatible with current 2.x packages.

**Failure modes:** SSRF through URL tools, command injection through shell tools, exfiltration through overbroad connectors, implicit privilege on a generic “execute SQL” tool, stale API schema. **Tests:** forbidden resource attempts, wrong-tenant arguments, schema fuzzing, tool timeouts, downstream failure injection. **Telemetry:** tool invocation by category, timeout rate, denied action rate, per-tool cost and downstream SLA.

## Review exercise

Look at a proposed action `{"action":"create_po","amount_cents":1,"tenant_id":"other"}`. Which fields may the model determine? **Only the action proposal.** The canonical amount, tenant and request ID must come from authoritative persisted intent; unrecognized fields are discarded or rejected. If state is `NEW`, the proposal is denied even if it uses a valid action name.
