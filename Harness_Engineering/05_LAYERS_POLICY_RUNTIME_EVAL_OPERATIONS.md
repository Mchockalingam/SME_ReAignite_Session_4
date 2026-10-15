# Technical Deep Dive: Harness Layers 7–12

## Layer 7 — Independent policy decision and action authorization

**Responsibility:** Enforce *what may happen*, not what the model claims ought to happen. Evaluate actor, tenant, resource, current policy, requested action, tool classification and environment. Keep policy versions stable and traceable through workflow execution.

**Contract:** permit/deny/needs-approval and a non-sensitive decision reason. In this lab, vendor validity, budget availability, currency, transaction cap and approval threshold are checked by deterministic code; precommit checks repeat the critical predicates.

```python
APPROVAL_THRESHOLD_CENTS = 100_000   # illustrative $1,000
MAX_PURCHASE_CENTS = 1_000_000       # illustrative $10,000
need_approval = state["amount_cents"] >= APPROVAL_THRESHOLD_CENTS
valid_policy = state["currency"] == "USD" and 0 < state["amount_cents"] <= MAX_PURCHASE_CENTS
```

**Production choice:** a versioned policy engine (e.g. OPA/Rego or an internal entitlement service), ABAC/RBAC, transaction limits and exception approval chain. Define the semantics of precedence between legal policy, organizational policy, product policy and request-specific decisions. Never encode all policy only in prompts.

**Failure modes:** stale decision cache; prompt-induced approval; rule/version mismatch; privilege escalation; confused deputy. **Tests:** negative authorization matrices and policy property tests. **Telemetry:** deny rates, exception decisions, policy version distribution.

## Layer 8 — Human-in-the-loop (HITL), separation of duties and escalation

**Responsibility:** Stop before high-risk actions, obtain verified human decision, bind the decision to the exact request, and resume only after rechecking authorization and business facts.

**Contract:** immutable intent digest, request ID, policy version, approver identity, decision, timestamp, expiry and permitted scope. The demo computes a SHA-256 fingerprint over core request fields and policy version. The approving actor must differ from the requester. In production add delegation, MFA, escalation, expiry, workflow signing and auditable approval evidence.

```python
approval = db.execute(
    "SELECT request_fingerprint FROM approvals WHERE request_id=? AND tenant_id=?",
    (req, tenant),
).fetchone()
valid_approval = bool(approval and approval[0] == request_fingerprint(state))
```

**Key insight:** Human approval is not a chat message such as “sounds good.” It is a business authorization event attached to a resource and scope. A user-facing UI must display the precise amount, vendor, cost centre, purpose and consequences. In production, bind approver entitlement and approval validity to the decision, not merely to a stored username.

**Failure modes:** self-approval, approval for previous amount, indefinite approval validity, unreviewed delegated action. **Tests:** self-approval denied, no approval → no PO, post-approval request mutation blocked. **Telemetry:** approval age, escalation rate, override frequency, bottleneck time.

## Layer 9 — Durable execution, idempotency and recovery

**Responsibility:** Provide checkpoints and side-effect semantics. Treat network timeouts and worker restarts as normal events. Make external operations retriable only with known idempotency behavior.

**Contract:** state transition journal; effect key derived from tenant+request ID; atomic local `budget update + PO insert + terminal state + audit` transaction. Example:

```python
key = hashlib.sha256(f"{tenant}:{req}".encode()).hexdigest()
# Effect ledger: UNIQUE(request_id), UNIQUE(idempotency_key)
result = db.execute(
    "UPDATE budgets SET available_cents=available_cents-? "
    "WHERE tenant_id=? AND cost_center=? AND available_cents>=?",
    (amount_cents, tenant, cost_center, amount_cents),
)
if result.rowcount != 1:
    # no effect; mark BLOCKED
    ...
```

**Why it works locally:** SQLite provides atomic writes across the demo's budget, PO ledger and audit tables in one database transaction. **What it does not solve:** a remote ERP may commit, then lose the HTTP response. The caller may not know whether the order exists. In production, use an ERP-supported idempotency key; persist a transactional outbox before external dispatch; reconcile status by external reference; move ambiguous effects to `UNKNOWN_EFFECT` for controlled recovery. Never blindly retry irreversible writes without a duplicate-safe contract.

**Failure modes:** duplicated transactions; non-durable approval resume; partial external commits; lost events; concurrent budget oversubscription. **Tests:** completed-run replay, restart after checkpoint, budget contention, injected effect timeout with reconciliation. **Telemetry:** unknown-effect count, redrive count, queue lag, duplicate suppression count.

## Layer 10 — Observability, provenance, cost and audit

**Responsibility:** Produce a correlated narrative of requests, model proposals, execution decisions, policy versions, tool results, human decisions and side effects. Keep diagnostic traces separate from authoritative business audit records.

**Recommended span/event attributes:** `workflow_id`, `tenant_hash`, `stage`, `model_id`, `model_calls`, `token_in`, `token_out`, `action`, `policy_version`, `decision`, `approval_id`, `tool_duration_ms`, `idempotency_key_hash`, `downstream_request_id`, `error_class`. Avoid raw prompts, vendor notes, credentials, PII, unrestricted model chain-of-thought or secret-bearing tool results.

```python
# The demo creates metadata-only hash-chain events, not a WORM audit log.
self.audit(db, tenant, req, "PO_COMMITTED", {"po_number": po_number})
```

For OpenTelemetry, instrument `model.request`, `planner.propose`, `tool.authorize`, `tool.execute`, `human.approve`, `workflow.checkpoint`. Emit traces/metrics to a governed collector. For compliance-grade audit, export to access-controlled immutable storage with retention and verification; a SHA-256 chain inside a modifiable SQLite DB is **tamper-evident only against naive edits**.

**Failure modes:** missing trace, raw customer data exfiltration into logs, disconnected approval event, aggregate metrics without denominator. **Tests:** end-to-end trace IDs, redaction inspection, audit tamper detection. **Telemetry:** trace completeness, span error rate, cost per completed task, audit export lag.

## Layer 11 — Evaluation, adversarial testing and release qualification

**Responsibility:** Quantify agent correctness and control effectiveness before release and continuously after deployment. Separate **semantic output quality** from **workflow/business correctness** and **security-policy adherence**.

Four evaluation sets:

1. **Task success** — valid vendor, permitted budget, right status, correct human routing, correct effect.
2. **Safety/authorization** — injection, stolen tools, other-tenant data, unauthorized approvals, policy-bypass attempts.
3. **Reliability** — timeouts, concurrent writes, resumed workflows, stale approvals, service restart and uncertain ERP responses.
4. **Economics** — model calls, tokens, task completion cost, avoidable tool requests, human handling time.

Use deterministic fixtures to assert exact outcomes, then model-in-the-loop datasets for uncertain interpretation and multi-step reasoning. [Promptfoo](https://github.com/promptfoo/promptfoo) and [Inspect AI](https://github.com/UKGovernmentBEIS/inspect_ai) can augment, not replace, the enterprise's deterministic business assertions. Reference the [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) when building adversarial coverage.

**Failure modes:** only happy-path evals; no tool trace grading; learned-to-the-test prompts; ignoring rare high-severity events. **Tests:** 15 supplied unit/integration tests and additional offline red team corpus. **Telemetry:** pass rate with confidence intervals, critical-policy regressions, attack success rate, production-vs-eval drift.

## Layer 12 — Runtime operations, containment, change and incident response

**Responsibility:** Keep the agent service isolated, within its declared scope and economically bounded in real operations. Controls include infrastructure identity, secrets, egress policy, allowlisted tools, feature flags, rate limits, circuit breakers, rollbacks, runbooks and on-call ownership.

**Production contract:** per-tenant isolation, no standing high-value credentials in prompts, network egress allowlists, secret rotation, SLO-backed alerts, provider outage plan, dependency version pinning, supply-chain attestation and deterministic rollback of agent/config/policy bundles.

A minimal kill switch must be able to block all *new privileged effects* at the gateway, independent of the model's preference. Decide separately how in-flight transactions, outstanding approvals and the outbox are drained or quarantined. When the switch is active, preserve evidence and allow read-only status checks where policy permits.

**Failure modes:** model vendor outage; runaway loops; connector rate limit; expired secrets; prompt/version drift; mass approvals; cascading incident from a shared tool. **Tests:** kill switch, egress denial, credential scoping, disaster recovery exercise. **Telemetry:** availability, P95 latency, queue depth, error budget burn, effect rate spike, per-tenant costs.

## Cross-cutting control ownership

| Control | Business owner | Technical enforcer | Verification |
|---|---|---|---|
| Spend threshold | Finance/procurement | Policy service + gateway | Unit and audit test |
| Who may approve | Business/IAM | Identity + approval service | Separation-of-duties test |
| Tenant isolation | Security/privacy | API + DB + retrieval filters | Negative multitenancy suite |
| Model provider data controls | Security/legal | Model gateway | Config review and data-flow test |
| Tool authorization | Service owner | Execution gateway | Per-action policy matrix |
| Recovery semantics | Platform/ERP owner | Outbox + reconciliation worker | Chaos/timeout drill |
| Model quality | AI engineering/product | Evaluator + release pipeline | Golden set and model regression |
| Incident stop | SRE/security | Gateway kill switch | Exercise and response-time test |

## A note on “guardrails”

Prompt-based guardrails can improve behavior; structured output can improve parsability; moderation can catch some harmful content; test-time and runtime verification can detect classes of errors. **None alone substitutes for a deterministic business authorization check at the moment of a privileged tool call.** Read [OpenAI Agents SDK tool guardrails](https://github.com/openai/openai-agents-python/blob/main/docs/guardrails.md) and be specific about *which* lifecycle stages a chosen SDK's guardrails actually cover.
