# AI Harness Architecture: Designing Reliable Agentic Systems for Production

**Enterprise technical-architecture workshop | 90 minutes | 13 October 2026**  
**Audience:** enterprise architects, AI engineers, platform engineers, security architects, SREs, product owners, and engineering leaders.  
**Level:** intermediate to advanced; Python familiarity helps with the live lab.

## The proposition

An LLM does not become a reliable enterprise agent simply because it can call tools. An **agent harness** is the surrounding software that translates intent into **bounded, authorized, observable, resumable and evaluated execution**. The model can propose an action; the harness must decide whether it is legitimate, safe, affordable and ready to execute. The reference implementation shows that distinction in working code.

This package intentionally uses a **single agent with a bounded action loop** rather than a theatrical swarm of agents. Multiple agents only become justified when specialization and parallelism create measurable benefits; they also increase failure modes, context loss and attack surface.

## Training package map

| File | Purpose | Suggested use |
|---|---|---|
| `00_START_HERE.md` | Orientation, artifact map, setup, extraction | Pre-read |
| `01_90_MINUTE_TEACHING_PLAN.md` | Minute-by-minute delivery, speaker guidance, demonstrations | Trainer |
| `02_HARNESS_ENGINEERING_FOUNDATIONS.md` | Precise terminology and design principles | 0–18 min |
| `03_REFERENCE_ARCHITECTURE.md` | Planes, layers, diagrams, failure paths, trust boundaries | 18–34 min |
| `04_LAYERS_IDENTITY_CONTEXT_PLANNING_TOOLS.md` | Technical specification: layers 1–6 | 34–45 min |
| `05_LAYERS_POLICY_RUNTIME_EVAL_OPERATIONS.md` | Technical specification: layers 7–12 | 45–57 min |
| `06_ENTERPRISE_PROCUREMENT_USE_CASE.md` | Fully defined use case, domain policy, security scenarios | 57–62 min |
| `07_HANDS_ON_RUNBOOK.md` | Hands-on commands and expected outcomes | 62–82 min |
| `08_COMPLETE_HARNESS_SOURCE.md` | **Entire runnable reference implementation** `harness_demo.py` | Lab source |
| `09_COMPLETE_TEST_SUITE.md` | **Entire test suite** `test_harness_demo.py` | Lab verification |
| `10_EVALUATION_REDTEAM_AND_SLOS.md` | Evals, attack cases, scorecards, SLOs and failure drills | 82–87 min |
| `11_FRAMEWORK_ADAPTERS_AND_PATTERNS.md` | LangGraph, OpenAI Agents SDK, MCP v2, OpenTelemetry code | Extend the lab |
| `12_PRODUCTION_READINESS_AND_ADRS.md` | Deployment migration, incident readiness, ADRs and gates | Production planning |
| `13_GITHUB_REPOSITORIES_AND_SOURCES.md` | Multiple verified upstream repositories and original readings | References |
| `14_EXERCISES_QUESTIONS_AND_ANSWER_KEY.md` | Exercises, review questions and answers | 87–90 min / follow-up |
| `15_TRAINER_SLIDE_STORYBOARD.md` | Slide-by-slide talking points mapped to 90 minutes | Presentation authoring |

**All supplied deliverables are Markdown.** The executable Python files are embedded in Markdown pages and can be extracted; this permits a Git-based enterprise learning repository without distributing separate code formats.

## Extract the runnable code from Markdown

Save the package and run this from the directory holding the `.md` files:

```bash
python3 - <<'PY'
from pathlib import Path
import re
root = Path('.').resolve()
pattern = re.compile(r'<!-- extract:([A-Za-z0-9_./-]+) -->\s*```(?:python|yaml|text)\n(.*?)\n```', re.S)
count = 0
for document in root.glob('*.md'):
    for relative_path, content in pattern.findall(document.read_text(encoding='utf-8')):
        target = (root / relative_path).resolve()
        if root not in target.parents:
            raise ValueError(f'Unsafe extraction path: {relative_path}')
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(content + '\n', encoding='utf-8')
        print(f'Extracted {target.relative_to(root)}')
        count += 1
assert count >= 2, f'Expected at least two code files, got {count}'
PY
python3 -m unittest -v
```

The base lab requires **Python 3.11+**, standard library only, no credentials, no internet connection and no ERP access. Its optional OpenAI model adapter requires `pip install openai`, an `OPENAI_API_KEY`, and an explicitly configured `OPENAI_MODEL` with Responses API structured-output support. An external model will incur provider charges. See the source and security notices before enabling it.

## Learning objectives

Participants should leave able to:

1. Distinguish the **model**, **agent loop**, **harness**, **execution runtime**, **tool platform** and **business control plane**.
2. Draw the **model-proposal → policy-decision → approved action → audited evidence** path.
3. Explain each harness layer's interface, invariants, failure modes, test evidence and SLO signals.
4. Implement a bounded agent with validated tool actions, tenant isolation, human approval and idempotency.
5. Demonstrate that a prompt-injected model cannot jump the execution gate.
6. Run regression tests and identify what's still missing to turn a local reference into an enterprise production service.

## Success criteria

The workshop is complete only when the participants can demonstrate: (a) low-risk procurement completes autonomously; (b) high-risk procurement halts for a human; (c) an invalid vendor, insufficient budget or stale approval blocks execution; (d) a repeated run cannot duplicate an order; (e) an adversarial planner proposal cannot bypass the gateway; and (f) the audit trail and test results are inspectable.

**Scope limitation:** The lab simulates identity, budget, vendor records and the ERP side effect using SQLite. It is a technical teaching implementation, **not** a deployment-approved procurement application. Production migration instructions are in `12_PRODUCTION_READINESS_AND_ADRS.md`.
