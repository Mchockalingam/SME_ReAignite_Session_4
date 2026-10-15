# Trainer Slide Storyboard: 90-Minute Technical Architecture Session

This is a Markdown-only storyboard, **not** a PowerPoint file. It maps to the instructor schedule and can be copied to a preferred enterprise slide template. Each slide has a suggested title, visual and speaker message. Aim for 2–5 minutes per slide; spend the final 20 minutes in terminal demonstrations.

| Slide | Time mark | Title | Visual/asset | One technical message |
|---:|---|---|---|---|
| 01 | 00:00 | AI Harness Architecture: Reliable Agentic Systems | Diagram of request → proposal → gateway → ERP | The harness makes autonomous actions governable |
| 02 | 00:03 | A Fluent Agent Can Still Be Unsafe | Vendor-note injection example | Quality of language ≠ authority to act |
| 03 | 00:07 | Model, Agent Loop, Harness, Runtime, Tool Plane | Six-component taxonomy | Do not confuse the model with the system |
| 04 | 00:11 | What Harness Engineering Changes | Compare prompt-first vs control-first | Make constraints and evidence explicit |
| 05 | 00:15 | The Five Invariants | Five principle matrix | Authority, state, effect, evidence, recovery |
| 06 | 00:18 | Reference Architecture | Mermaid diagram from 03 | Control and data plane relationship |
| 07 | 00:23 | Trust Boundaries and Attack Surfaces | Trust-domain diagram | Retrieved text is data; gateway is authority |
| 08 | 00:28 | State Machine and Effect-Time Recheck | State diagram and stage table | Illegal transition is denied in code |
| 09 | 00:34 | Layers 1–3: Identity, Context, Models | Table and context data contract | Schema, provenance, token budgeting |
| 10 | 00:39 | Layers 4–6: Planner, Orchestrator, Tools | Loop pseudocode | Proposal ≠ execution |
| 11 | 00:45 | Layers 7–9: Policy, Human Approval, Replay | Approval sequence and outbox | The business policy survives model changes |
| 12 | 00:51 | Layers 10–12: OTel, Evals, SRE | Metrics/evidence matrix | Production quality must be measured |
| 13 | 00:57 | Procurement Case and Policy | Business policy table | Thresholds, spend cap, vendor, budget |
| 14 | 01:02 | Code Walkthrough: Interfaces and State | `PurchaseRequest`, `Plan`, `NEXT_ACTION` | Keep tool inputs authoritative |
| 15 | 01:06 | Live Demo: Low-Risk Purchase | Terminal session | Complete without human signoff below threshold |
| 16 | 01:10 | Live Demo: HITL Approval | Terminal session and state | High-risk purchase must pause |
| 17 | 01:14 | Live Demo: Prompt Injection vs Gateway | `AttackPlanner` test | Model can be wrong without acquiring authority |
| 18 | 01:18 | Live Demo: Replays, Race, Audit | Unit tests | Side-effect safety persists across failures |
| 19 | 01:22 | Production Readiness and Eval Gates | SLO and gap checklist | Local test passing is not production approval |
| 20 | 01:27 | Architecture Review: Three Decisions | Prompt questions | Control ownership, approval binding, ERP uncertainty |

## Slide 6 speaker notes — The most important diagram

“Work from left to right. A user or business application enters an authenticated request. A context builder assembles scoped evidence. The model proposes a typed next action, but the harness must not let this proposal become an ERP operation directly. The orchestrator knows the current state. The policy layer knows who is allowed to act, whether the vendor is current, what budget is available, whether approval is required and what to do when external execution status is unknown. The tool gateway is the final enforceable decision point. Traces, audits and evaluation surround the entire chain.”

## Slide 8 speaker notes — State over prose

“A purchase is legal because the business state is `READY`, the current policy permits it, the current budget can cover it and the approval is valid if the threshold requires one. It is not legal merely because an AI assistant says ‘all checks passed.’ The stage machine denies an action even when the proposed action name is recognized. That is the difference between a polished agent demo and a governed enterprise system.”

## Slide 11 speaker notes — Distributed side effects

“The lab is deliberately simple: a single SQLite transaction updates budget, records a purchase and writes audit. But the design is not sufficient for a real ERP. If an ERP order is created and the response is lost, the agent may not know whether retrying would create a duplicate. We use persistent idempotency keys and an outbox/reconciliation process because exactly-once semantics cannot be assumed across a generic HTTP API.”

## Slide 17 speaker notes — Demonstrating attack containment

“Notice that the malicious supplier note is not itself a privileged instruction; it is untrusted document content. The red-team fixture forces the planner to request `create_po` before checking the vendor. The gateway refuses that proposal because it was issued in state `NEW`. This is important: we are not claiming that no model can be fooled. We are proving a narrower security property about the independently enforced workflow state.”

## Slide 19 speaker notes — A realistic readiness statement

“This code is educational and does not claim production readiness. Identity roles are simulated, data and ERP connectors are mocked, the local hash chain is not a WORM log, and the SQLite database is not a high-availability platform. The production blueprint identifies the missing work: IAM, scoped tools, outbox, policy versioning, immutable audit, risk-based approvals, security red-teaming, SRE and per-task economics.”

## Suggested leave-behind material

Give attendees the entire Markdown ZIP; point them to `00_START_HERE.md` and the source/tests in files 08–09. Ask each enterprise team to complete a two-page architecture decision record identifying (1) business actions, (2) authorization owner, (3) effects/idempotency contract, (4) state persistence, (5) human approval, (6) data boundary, (7) SLO/eval gates and (8) incident stop mechanism.
