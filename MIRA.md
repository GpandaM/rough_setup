**MIRA: A Platform for Model Invariance and Execution Resilience Analysis in Multi-Agent Workflows**
We introduce MIRA (Model Invariance & Resilience Analysis), a cloud- and framework-agnostic evaluation platform designed to quantify whether an agentic application remains invariant in performance and resilient in execution trajectory when transitioning across underlying LLM providers."


## 1. Executive Summary

Enterprises run many LLM-based agentic applications, each built on a different framework (Google ADK, Microsoft Agent Framework, LangChain/LangGraph, custom code) and each tied to one model. When a cheaper, faster, or more compliant model appears, the question is: “If we swap GPT for Claude or Gemini, does the entire application still work, and at what cost and latency?”

This document proposes a centralized, cloud-agnostic evaluation platform that:

- Connects to heterogeneous agent applications without requiring them to be rewritten.
- Converts every framework’s execution log into one canonical trajectory format.
- Replays the same frozen dataset against a baseline model and 2–3 candidate models.
- Scores results with deterministic checks, LLM-as-judge, human review, and RAG/agent metrics.
- Produces an X vs. Y vs. Z comparison report that shows trade-offs (quality, cost, latency) instead of hiding them in one number.

The platform is informational only: it recommends and never deploys, optimizes prompts automatically, or touches production traffic.

## 2. Problem, Motivation, and the Why / What / How

| Question | Answer |
| --- | --- |
| What are we building? | An evaluation platform that compares candidate LLMs on complete agentic applications, using full execution trajectories. |
| Why is it needed? | Model swaps break agents in non-obvious ways (wrong tool, malformed arguments, broken handoffs) even when the final answer looks fine. Each team also measures differently, so results can’t be compared. |
| How will we do it? | A canonical trace schema, tiered integration (SDK, OpenTelemetry, black-box), a control-plane/execution-plane split, a plugin-based evaluator registry, and multi-objective reporting. |

### 2.1 Four root problems

- Prompt-model coupling. A prompt tuned for GPT may underperform on Claude or Gemini.
- Mid-flow failure. Agents are multi-step programs. A model can pick the wrong tool at step 3 and recover by step 6, so the answer looks fine but the behavior is fragile and expensive.
- Stochasticity. One run proves little. Repeated runs and statistics are required.
- Metric fragmentation. Ragas, DeepEval, and LangSmith each define quality differently, so scores can’t be compared across applications.

## 3. Research Questions and Hypotheses

| RQ | Question | Hypothesis | What in the architecture serves it |
| --- | --- | --- | --- |
| RQ1 | Can a framework-neutral trace schema capture agent trajectories well enough to compare models across different frameworks? | Most spans from ADK, MAF and LangGraph map to one canonical schema with little loss, and one evaluator runs unchanged across frameworks. | Canonical Trace Schema, Normalizer, Integration Layer |
| RQ3 | Do trajectory-level metrics (tool selection, argument accuracy, handoffs) detect migration regressions that final-answer metrics miss? | A measurable share of regressions is invisible to final-answer scoring but caught by trajectory metrics. | Full-trajectory capture, trajectory evaluators, side-by-side trace diff |
| RQ5 | Does multi-objective comparison (quality, cost, latency) support better migration decisions than a single score? | Decision-makers using a Pareto-style report choose more suitable models, and with more confidence, than those given one weighted score. | Aggregator, Pareto analysis, report generator |

Why these three fit together:

- RQ1 is the enabler (a common language for traces).
- RQ3 is the diagnostic (what the common language lets us detect).
- RQ5 is the decision layer (how findings reach a human).

Together they cover the pipeline from raw execution to a migration decision.

Dropped from the research scope: fidelity-tier comparison (RQ2) and judge calibration (RQ4). The platform still uses tiers and judges as engineering features, and the judge’s reliability stays listed as a threat to validity (§17).

## 4. Scope, Assumptions, Constraints

### 4.1 Scope

| In scope | Out of scope |
| --- | --- |
| Offline and preproduction evaluation | Shadow traffic and live production monitoring |
| Baseline vs. 2–3 candidate comparison | CI/CD gates, automatic deployment |
| Full-trajectory evaluation | Automated prompt optimization (DSPy is future work) |
| Deterministic, LLM-judge, human, RAG, and agent metrics | Fine-tuning or training models |
| Admin-driven hyperparameter, prompt, and skill configuration | Runtime model routing |
| Informational recommendation report | Automatic migration |

### 4.2 Assumptions

| ID | Assumption |
| --- | --- |
| A1 | Application teams own and supply the golden dataset and sampled, anonymized production interactions. |
| A2 | The execution team applies hard gates and picks 2–3 candidates. |
| A3 | Candidate models are reachable through APIs or self-hosted endpoints. |
| A4 | Load is about 5–7 concurrent application evaluations. |
| A5 | Full trajectory capture requires application telemetry. Black-box mode captures only input, output, latency, and cost. |
| A6 | Agent tools can be mocked, replayed, or safely run in preproduction. |
| A7 | Where data residency applies, evaluation workers run inside the required boundary. |

### 4.3 Constraints

- No modification of production code is required (but is preferred where allowed).
- Cloud-agnostic: containers on Kubernetes, with no dependency on one cloud’s proprietary services.
- Prompt changes are manual and made by the admin.

## 5. Conceptual Architecture

This section explains the ideas before the boxes.

### 5.1 Five core design principles

| # | Principle | Plain-language meaning | Benefit |
| --- | --- | --- | --- |
| P1 | Canonical trace | Translate every framework’s logs into one shared “language”. | Work scales as frameworks + metrics, not frameworks × metrics. |
| P2 | Control plane / execution plane | One place decides what to run. Workers anywhere do the running. | Governance stays central. Execution can sit inside any cloud or data boundary. |
| P3 | Controlled experiment | Change only the model. Freeze dataset, prompts, tools, and evaluator versions. | Score differences can be attributed to the model. |
| P4 | Honest fidelity | Every result states how much of the trajectory was observable. | Black-box runs never pretend to have tool-level scores. |
| P5 | Trade-offs over verdicts | Present quality, cost, and latency as separate dimensions. | Owners choose based on their priorities (RQ5). |

### 5.2 Conceptual diagram: “what goes in, what comes out”

```text
   INPUTS                      THE PLATFORM                        OUTPUTS
 ┌─────────────┐        ┌──────────────────────────┐        ┌────────────────────┐
 │ Golden      │        │                          │        │ Per-model scores   │
 │ dataset     │───────▶│  1. Replay the whole app │───────▶│ (quality, RAG,     │
 ├─────────────┤        │     with each model      │        │  agent metrics)    │
 │ Sampled prod│        │                          │        ├────────────────────┤
 │ interactions│───────▶│  2. Capture trajectory   │───────▶│ Cost + latency     │
 ├─────────────┤        │     (every step)         │        ├────────────────────┤
 │ Candidate   │        │                          │        │ Trace diffs:       │
 │ models +    │───────▶│  3. Normalize to one     │───────▶│ where did model B  │
 │ parameters  │        │     canonical format     │        │ diverge from A?    │
 ├─────────────┤        │                          │        ├────────────────────┤
 │ Prompts /   │        │  4. Score (deterministic,│        │ Pareto view of     │
 │ skills      │───────▶│     judge, human, RAG)   │───────▶│ trade-offs         │
 ├─────────────┤        │                          │        ├────────────────────┤
 │ Hard gates  │        │  5. Compare statistically│        │ Informational      │
 │ + metrics   │───────▶│     and build report     │───────▶│ recommendation     │
 └─────────────┘        └──────────────────────────┘        └────────────────────┘
```

### 5.3 Conceptual layer model

```text
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 1: APPLICATIONS    (ADK · MAF · LangGraph · Custom · Legacy)│
├──────────────────────────────────────────────────────────────────┤
│  LAYER 2: INTEGRATION     "How do we see inside the app?"        │
│           Tier 1 SDK/Adapter │ Tier 2 OTel/existing traces │ Tier 3 Black-box │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 3: CONTROL PLANE   "What should be evaluated, and how?"   │
│           Console · Registry · Gates · Orchestrator              │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 4: EXECUTION PLANE "Do the work."                         │
│           Replay · Normalize · Evaluate · Human review           │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 5: DATA            "Remember everything, immutably."      │
├──────────────────────────────────────────────────────────────────┤
│  LAYER 6: INSIGHT         Aggregation · Pareto · Report          │
└──────────────────────────────────────────────────────────────────┘
        Cross-cutting: Security · Governance · Lineage · Audit
```

### 5.4 The tiered integration idea

Integration is a spectrum because the same question, “can we see inside your app?”, has different answers across teams.

| Tier | Mechanism | Code change | What we can observe | Typical user |
| --- | --- | --- | --- | --- |
| T1 | Lightweight SDK/adapter for the framework | Small | Full trajectory | New or modifiable apps |
| T2 | Existing telemetry (OpenTelemetry, LangSmith traces, Ragas/DeepEval logs) | None, if already instrumented | Full or near-full | Apps with observability in place |
| T3 | Black-box invocation endpoint | None | Input, output, latency, cost only | Legacy or frozen apps |

Recommendation to your SDK question: treat the SDK as the preferred route and Tier 3 as the safety net. Every result is stamped with its tier, so evaluators that need tool-level data report “not applicable” on T3 runs instead of a false zero.

## 6. Stakeholders and Roles

| Role | Responsibility | Platform actions |
| --- | --- | --- |
| Application Team | Owns the app, dataset, and ground truth | Onboard app, upload golden dataset and sampled production data |
| Execution Team | Owns model-selection policy | Define hard gates, shortlist 2–3 candidate models |
| Admin | Configures and runs experiments | Set hyperparameters, prompts/skills, metrics, weights, trigger runs, review reports, revise manually |
| Human Reviewer (domain expert) | Judges quality where automation is weak | Annotate, correct, and rate sampled outputs |
| Decision Owner | Decides on migration | Reads report and decides (outside the platform) |
| Platform Engineer | Operates the platform | Deploy workers, maintain adapters and the model catalogue |

## 7. Journey: From Application Onboarding to Results

### 7.1 End-to-end journey (swim-lane view)

```text
 App Team          Execution Team        Admin               Platform              Reviewer
    │                    │                 │                     │                     │
    │ 1. Register app    │                 │                     │                     │
    │───────────────────────────────────────────────────────────▶│                     │
    │                    │                 │       2. Choose integration tier          │
    │◀───────────────────────────────────────────────────────────│                     │
    │ 3. Install adapter / connect traces / expose endpoint      │                     │
    │───────────────────────────────────────────────────────────▶│                     │
    │                    │                 │    4. Smoke test: capture sample trace    │
    │◀───────────────────────────────────────────────────────────│                     │
    │ 5. Upload golden dataset + sampled production data         │                     │
    │───────────────────────────────────────────────────────────▶│                     │
    │                    │                 │    6. Validate, anonymize, version        │
    │                    │ 7. Apply hard gates, shortlist 2–3 models                    │
    │                    │────────────────────────────────────▶│                       │
    │                    │                 │ 8. Create experiment: freeze dataset,     │
    │                    │                 │    params, prompts/skills, metrics, weights│
    │                    │                 │────────────────────▶│                     │
    │                    │                 │  9. Run baseline + each candidate (repeated)
    │                    │                 │                     │ 10. Capture, normalize, score
    │                    │                 │                     │ 11. Queue samples ──▶│
    │                    │                 │                     │◀── 12. Human labels──│
    │                    │                 │   13. Aggregate, compare, build report     │
    │                    │                 │◀────────────────────│                     │
    │                    │                 │ 14. Review report. Revise prompts/params?  │
    │                    │                 │     └── yes: new experiment version (loop) │
    │                    │                 │     └── no: hand report to decision owner  │
```

### 7.2 Journey stages in detail

| Stage | Name | Key activities | Output | Gate to next stage |
| --- | --- | --- | --- | --- |
| 1 | Onboard | Register app, owner, framework, cloud, data-sensitivity | App profile | Profile complete |
| 2 | Integrate | Pick tier, connect adapter/traces/endpoint | Working capture | Smoke test passes |
| 3 | Provide data | Upload golden dataset and sampled production interactions | Versioned dataset | Validation and anonymization pass |
| 4 | Select models | Execution team applies hard gates, picks 2–3 candidates | Eligible shortlist | Gates satisfied |
| 5 | Configure | Admin freezes dataset and settings, sets hyperparameters, prompts/skills, metrics, weights | Frozen experiment | Config validated |
| 6 | Execute | Run baseline and each candidate over all tasks with repeats | Raw traces | All jobs finished |
| 7 | Normalize | Convert traces to canonical form | Canonical traces | Schema validation |
| 8 | Evaluate | Run evaluators and queue human review | Scores | Coverage threshold met |
| 9 | Compare | Statistics, Pareto, trace diffs | Analysis | n/a |
| 10 | Report | Generate informational report | Final report | n/a |
| 11 | Decide / Iterate | Admin reviews. Optionally manual revision and re-run | New version or handoff | n/a |

### 7.3 App onboarding flow chart

```text
                        ┌───────────────────────┐
                        │  Register application │
                        └───────────┬───────────┘
                                    ▼
                  ┌─────────────────────────────────────┐
                  │ Can the app emit traces already     │
                  │ (OpenTelemetry/LangSmith/etc.)?     │
                  └─────┬──────────────────────┬────────┘
                    Yes │                      │ No
                        ▼                      ▼
              ┌──────────────────┐   ┌───────────────────────────┐
              │ TIER 2: connect  │   │ Can the team add a         │
              │ existing traces  │   │ lightweight adapter?       │
              └────────┬─────────┘   └──────┬──────────────┬─────┘
                       │                Yes │              │ No
                       │                    ▼              ▼
                       │          ┌────────────────┐ ┌──────────────────────┐
                       │          │ TIER 1: install│ │ TIER 3: expose a     │
                       │          │ framework      │ │ black-box invocation │
                       │          │ adapter        │ │ endpoint             │
                       │          └───────┬────────┘ └──────────┬───────────┘
                       └──────────────────┼─────────────────────┘
                                          ▼
┌─────────────── EVALUATION CONTROL PLANE (central, cloud-agnostic) ─────┐
│ ┌──────────────┐ ┌───────────────────┐ ┌───────────────┐ ┌─────────┐ │
│ │ Admin Console│ │ Registry:         │ │ Candidate     │ │Experiment│ │
│ │ (UI + API)   │ │ datasets, prompts,│ │ model config  │ │Orchestra-│ │
│ │              │ │ skills, versions  │ │ + hard gates  │ │tor       │ │
│ └──────────────┘ └───────────────────┘ └───────────────┘ └─────────┘ │
└──────────────────────────────────┬────────────────────────────────────┘
                                   ▼  jobs
┌─────────────── EVALUATION EXECUTION PLANE (Docker / Kubernetes) ───────┐
│ ┌────────┐ ┌──────────┐ ┌────────────┐ ┌──────────┐ ┌──────┐ ┌─────┐ │
│ │ Full   │ │ Trace    │ │Deterministic│ │LLM-as-   │ │RAG + │ │Human│ │
│ │ app    │ │ normalizer│ │ metrics    │ │Judge     │ │Agent │ │review│ │
│ │ replay │ │          │ │            │ │          │ │metrics│ │     │ │
│ └────────┘ └──────────┘ └────────────┘ └──────────┘ └──────┘ └─────┘ │
└───────────────┬──────────────────────────────────────┬────────────────┘
                ▼                                      ▲ calls
┌──────── DATA LAYER (cloud-agnostic) ────────┐   ┌─────┴───────────────────┐
│ Datasets │ Traces │ Scores │ Versions │      │   │ MODEL PROVIDERS         │
│ Experiment history & lineage                │   │ GPT/Azure OpenAI ·      │
└───────────────┬─────────────────────────────┘   │ Claude · Gemini · self- │
                ▼                                 │ hosted                  │
┌────────── OUTPUT: X vs Y vs Z COMPARISON REPORT ─┴─────────────────────────┐
│ Quality · RAG/agent metrics · Latency · Cost · Trace analysis · Advice    │
└────────────────────────────────────────────────────────────────────────────┘
```

### 5.5 Why the two-plane split matters

| Concern | Control plane | Execution plane |
| --- | --- | --- |
| What it does | Decides, schedules, records, reports | Runs apps, calls models, scores |
| Where it lives | One central place | Anywhere: any cloud, on-prem, or inside a team’s boundary |
| Why separate | One place for governance and audit | Data residency, scale, and failure isolation |
| Analogy | Air traffic control | The aircraft |

### 5.6 Design decisions and rationale

| Decision | Choice | Rationale |
| --- | --- | --- |
| Integration | Tiered (SDK, OTel, black-box) | Honors “cannot modify” while maximizing fidelity |
| Existing tools | Ragas, DeepEval, LangSmith become evaluator plugins and trace sources | Teams keep their tools. The platform adds normalization and comparison |
| Model access | Provider adapter layer | A model swap is configuration, not code |
| Deployment | Containers on Kubernetes | Cloud-agnostic |
| Decision support | Pareto view plus optional weighted score | Supports RQ5 |
| Automation | Evaluate only | No deployment, no auto-optimization |

## 9. Low-Level Design (LLD)

### 9.1 The canonical trajectory model (heart of RQ1)

Described conceptually, as a tree of events per task run.

```text
 Run (one task, one model, one repeat)
  │
  ├── Run metadata: experiment, task, model + parameters, prompt version,
  │                 app version, fidelity tier
  │
  └── Root span (the whole request)
        ├── Agent span: Supervisor
        │     ├── LLM call span: planning (tokens, cost, latency)
        │     ├── Handoff span ──▶ Research Agent
        │     │     ├── LLM call span
        │     │     ├── Tool call span: Search (name, arguments, result)
        │     │     └── Retry span (links to the failed attempt)
        │     ├── Handoff span ──▶ Analysis Agent
        │     │     └── Tool call span: Database query
        │     └── Handoff span ──▶ Response Agent
        │           └── LLM call span → Final answer
        └── Final output
```

Span types: agent, LLM call, tool call, handoff, retrieval, retry.

| Field group | Contents | Why it matters |
| --- | --- | --- |
| Identity | run, experiment, task, span, parent | Rebuild the tree |
| Model context | provider, model, version, parameters | Attribute results to a model |
| Version context | prompt, skill, app, evaluator versions | Reproducibility |
| Content | input, output, tool name, arguments, results | Correctness checks |
| Resource | tokens, cost, latency, context size | Cost/performance comparison |
| Reliability | error, retry link | Retry and failure analysis |
| Fidelity | tier flag per run | Honest reporting |

Mapping rule: each framework adapter maps native concepts (a LangGraph node, an ADK sub-agent, a MAF agent step) onto these span types. RQ1 measures how completely that mapping succeeds.

### 9.2 Component specifications

| Component | Responsibility | Inputs | Outputs |
| --- | --- | --- | --- |
| Admin Console | UI/API for configuration, runs, and reports | Admin actions | Experiment definitions |
| App Registry | App profiles, tier, owner, sensitivity | Onboarding data | App catalogue |
| Dataset Registry | Versioned golden and sampled datasets | Uploaded data | Immutable dataset versions |
| Prompt/Skill Registry | Versioned prompts and skills | Admin edits | Version IDs |
| Model Catalogue | Approved models, capabilities, provider settings | Platform engineers | Candidate metadata |
| Hard-Gate Filter | Excludes ineligible models and gives reasons | Model capabilities, gate rules | Eligible list |
| Experiment Orchestrator | Expands an experiment into jobs and schedules them | Frozen experiment | Job queue, lineage |
| Replay Worker | Runs the whole application per task with the chosen model | Job | Raw trace |
| Model Injector | Makes the application use the candidate model | Model config | Overridden model client/endpoint |
| Normalizer | Converts native traces to canonical form | Raw trace | Canonical trace |
| Evaluator Registry | Plugin evaluators with name, version, and fidelity needs | Canonical trace, task | Scores |
| Human Review Queue | Samples outputs for expert labeling | Sampled runs | Labels, corrections |
| Aggregator | Statistics, Pareto set, trace diffs | Scores | Analysis |
| Report Generator | Builds the comparison report | Analysis | Report |
| Audit & Lineage Service | Records every change and result link | All events | Audit trail |

### 9.3 Model injection: how “replace GPT with Claude” actually happens

This is the most practical risk, so it deserves explanation.

| Tier | Injection mechanism | Requirement on the app team |
| --- | --- | --- |
| T1 | The adapter intercepts the framework’s model client and substitutes the candidate | Minimal (adapter installed) |
| T2 | The application reads its model from configuration that the worker overrides | A model-configuration seam |
| T3 | A separate deployed variant of the app per candidate model | Provide one environment/endpoint per model |

State this limitation openly in the proposal. Without a configuration seam or a variant environment, a fair model swap isn’t possible.

### 9.4 Evaluator framework

Evaluators are plugins with a uniform contract: each declares a name, a version, and the trace fidelity it needs. It receives a canonical trace plus the task, and returns a score with a rationale.

| Evaluator family | Examples | Needs from trace | Cost | Ground truth needed |
| --- | --- | --- | --- | --- |
| Deterministic | Exact match, schema validity, tool-argument check, latency, cost | Varies | Low | Often yes |
| Trajectory (RQ3 focus) | Tool selection correctness, argument accuracy, handoff correctness, unnecessary steps, retry count, plan adherence, context-window endurance | Full trajectory (T1/T2) | Low–medium | Expected tool path helps |
| LLM-as-Judge | Groundedness, relevance, completeness, factual consistency, rubrics | Final output (+ context) | Medium | Optional |
| RAG metrics | Faithfulness, answer relevance, context precision/recall (Ragas, DeepEval) | Retrieval spans | Medium | Partly |
| Human review | Expert rating, correction, preference | Output (+ trace view) | High | None |

Rule of honesty: if an evaluator needs data the trace doesn’t have (for example a tool-argument check on a black-box run), it returns “Not applicable”, never zero.

Judge safeguards: use a judge from a family different from the candidates (or an ensemble), randomize answer order to reduce position bias, use explicit rubrics, and spot-check against human labels.

### 9.5 Aggregation and statistics

| Step | What happens | Why |
| --- | --- | --- |
| 1 | Score each run (task × repeat) | Atomic unit |
| 2 | Average repeats within a task | Smooths randomness |
| 3 | Report pass@k (at least one of k attempts succeeds) and a consistency measure (all k succeed) | Capability vs. reliability |
| 4 | Compare candidate to baseline on the same tasks using paired methods (paired bootstrap confidence intervals, McNemar for pass/fail) | Controls task difficulty and tells real differences from noise |
| 5 | Compute the Pareto front over quality, cost, and latency | Show non-dominated options (RQ5) |
| 6 | Optional weighted score using admin weights | For owners who want one number |
| 7 | Build trace diffs between baseline and candidate | Explain why (RQ3) |

Pareto explained: a model is dominated if another is at least as good on every dimension and better on one. The Pareto front is the set of non-dominated models. Choosing among them is a business judgment, which is why the platform shows trade-offs instead of one verdict.

### 9.6 Conceptual data model

```text
 Application ──< Dataset ──< Task
      │                        │
      │                        │
      └──< Experiment ──< Run >── Task
              │             │
              │             ├──< Span (trace tree)
              │             ├──< Score >── Evaluator (versioned)
              │             └──< HumanReview
              │
              ├── frozen: dataset version, prompt/skill versions,
              │           evaluator versions, hyperparameters
              ├──< Model (baseline + candidates) >── Model Catalogue
              └── Report ──< Comparison Finding
```

All entities are immutable and versioned. A changed prompt creates a new version and a new experiment. Nothing is overwritten, which makes results reproducible and auditable.

### 9.7 Experiment state machine

```text
 Draft ──▶ Validated ──▶ Frozen ──▶ Running ──▶ Evaluating ──▶ Aggregating ──▶ Reported
              │            │           │            │
              ▼            ▼           ▼            ▼
          Rejected     (immutable)   Failed ◀── Partial (resumable)

 Reported ──(manual revision by admin)──▶ New Draft (v+1)
```

### 9.8 Execution flow chart (one experiment)

```text
                     ┌──────────────────────────┐
                     │ Admin triggers frozen    │
                     │ experiment               │
                     └────────────┬─────────────┘
                                  ▼
                     ┌──────────────────────────┐   fail  ┌───────────────┐
                     │ Hard-gate validation of  │────────▶│ Reject model, │
                     │ every candidate          │         │ show reasons  │
                     └────────────┬─────────────┘         └───────────────┘
                                  ▼ pass
                     ┌──────────────────────────┐
                     │ Expand into jobs:        │
                     │ model × task × repeat    │
                     └────────────┬─────────────┘
                                  ▼
              ┌───────────────────────────────────────────┐
              │ FOR EACH job (parallel, queued)           │
              │  1. Inject model + parameters             │
              │  2. Replay full application on the task   │
              │  3. Capture trajectory                    │
              │  4. Normalize to canonical trace          │
              │  5. Run all applicable evaluators         │
              │  6. Store trace and scores                │
              └────────────────────┬──────────────────────┘
                                   ▼
                     ┌──────────────────────────┐   no   ┌────────────────────┐
                     │ All jobs done and        │───────▶│ Retry failed jobs; │
                     │ coverage threshold met?  │        │ mark partial if not│
                     └────────────┬─────────────┘        └────────────────────┘
                                  ▼ yes
                     ┌──────────────────────────┐
                     │ Sample runs for human    │
                     │ review; collect labels   │
                     └────────────┬─────────────┘
                                  ▼
                     ┌──────────────────────────┐
                     │ Aggregate: statistics,   │
                     │ Pareto, trace diffs      │
                     └────────────┬─────────────┘
                                  ▼
                     ┌──────────────────────────┐
                     │ Generate X vs Y vs Z     │
                     │ report                   │
                     └──────────────────────────┘
```

### 9.9 Sequence diagram

```text
Admin   Console   Orchestrator  Gate Filter  Worker   App(+Adapter)  Normalizer  Evaluators  Data   Aggregator
 │ create  │          │            │           │          │              │           │         │        │
 │────────▶│ submit   │            │           │          │              │           │         │        │
 │         │─────────▶│ validate   │           │          │              │           │         │        │
 │         │          │───────────▶│           │          │              │           │         │        │
 │         │          │◀── eligible│           │          │              │           │         │        │
 │         │          │ job(m,t,r) │           │          │              │           │         │        │
 │         │          │───────────────────────▶│ invoke with model m     │           │         │        │
 │         │          │            │           │─────────▶│              │           │         │        │
 │         │          │            │           │◀── native trace + output│           │         │        │
 │         │          │            │           │ normalize│─────────────▶│           │         │        │
 │         │          │            │           │◀──────────────── canonical trace    │         │        │
 │         │          │            │           │ evaluate │              │──────────▶│ scores  │        │
 │         │          │            │           │ persist  │              │           │────────▶│        │
 │         │          │ all jobs complete      │          │              │           │         │───────▶│
 │         │          │◀───────────────────────────────────────────────────────────────────────────────│ report
 │◀────────────────── report ready ──────────────────────────────────────────────────────────────────── │
```

## Addendum: Trajectory Capture, Metrics, Statistics, Libraries, Repo Structure and Coverage Check

These were real gaps in v2.0. The answers below extend that document. Where I name a library or framework hook, check its current docs before building, because agent tooling changes quickly.

### A. How the platform knows which sub-agent or tool fired, and with what payload

#### A.1 The concept

To reconstruct a trajectory you need three facts about every event:

| Question | Fact needed | Where it comes from |
| --- | --- | --- |
| Who acted? | Agent or tool identity | Framework metadata or explicit instrumentation |
| What did they do? | Action type and payload (inputs, arguments, outputs) | The hook that fires around the call |
| In what order and why? | Parent-child and causal links | Trace context propagation (run IDs, parent IDs) |

Every framework produces these facts differently. The adapter converts them into one span with a type, an identity, a payload and a parent.

#### A.2 Capture mechanism per framework

| Source | How events are observed | What you get |
| --- | --- | --- |
| LangChain / LangGraph | A callback handler receives start/end events for chains, graph nodes, LLM calls, tools and retrievers. Run IDs and parent run IDs form the tree. | Tool name, input arguments, output, node name, LLM prompts and responses, token usage |
| Google ADK | Lifecycle callbacks before/after agent, model and tool calls. The event stream records which agent authored each step and which function calls were requested. | Agent identity, tool name and arguments, sub-agent transfers |
| Microsoft Agent Framework | Built-in OpenTelemetry instrumentation for agent invocation, model calls and tool execution (following the evolving OTel GenAI conventions). | Agent and tool spans, optionally with argument and result payloads |
| Custom agents | Thin wrappers (decorators or context managers) around the agent entry point, the tool registry and the model client | Whatever you wrap. Tool registry wrapping is the key point. |
| Any app (gateway) | Point the app’s model base URL at an LLM gateway that logs every request and response | Every LLM call, tool calls the model requested with their arguments, tokens, latency, TTFT |
| Black-box | Only the invocation endpoint is visible | Input, output, latency, cost |

The gateway insight. Model injection already forces you to control where model calls go, so a gateway comes almost for free. It captures what the model decided (for example “call search_db with these arguments”) for apps you cannot instrument. It does not see tool execution results, or agent identity unless the app propagates a header. I would add this as a middle fidelity level, T2-G.

| Level | Sees agents? | Sees tool decisions + args? | Sees tool results? | Sees LLM calls? |
| --- | --- | --- | --- | --- |
| T1 SDK/adapter | Yes | Yes | Yes | Yes |
| T2 OTel / existing traces | Depends on instrumentation | Usually | Usually | Yes |
| T2-G Gateway only | No (unless header) | Yes | No | Yes |
| T3 Black-box | No | No | No | No |

#### A.3 Identity resolution (which sub-agent was it?)

The normalizer resolves the agent for each span in priority order:

1. An explicit agent name attribute (OTel agent name, ADK author, MAF agent name).
2. The framework’s structural unit (LangGraph node name).
3. A declared mapping in the app profile (for example “node research_node = Research Agent”).
4. Fallback: mark the span unattributed rather than guess.

The app profile created at onboarding therefore needs an agent and tool inventory: the expected sub-agents and tools and their roles. This is also what lets the platform detect “expected agent never invoked”.

#### A.4 Payload capture policy

| Concern | Policy |
| --- | --- |
| What is stored | Full LLM prompts and responses, tool name, tool arguments (verbatim), tool results |
| Large payloads | Stored in object storage and referenced by hash, so the trace stays light |
| Sensitive data | Redaction (for example with a PII detector) at capture time, before leaving the worker |
| Sampling | None for evaluation runs. Every step of every run is kept. |
| Raw vs. normalized | Keep both: raw native trace for debugging, canonical for evaluation |

#### A.5 Judging observed against expected: arguments and tool selection

The golden dataset can carry an optional expected trajectory spec per task:

- Expected tools, with argument constraints (exact value, pattern, type, or “any”).
- Optional ordering constraints (strict order, partial order or any order).
- Expected agents or handoffs.

| Metric | Definition (conceptual) |
| --- | --- |
| Tool selection precision | Correct tool calls / all tool calls made (penalizes extra or wrong calls) |
| Tool selection recall | Correct tool calls / expected tool calls (penalizes missed calls) |
| Order score | Sequence similarity to the expected path (longest common subsequence or edit distance) |
| Argument accuracy | For each correctly selected tool, matched required arguments / expected arguments |
| Argument validity | Share of calls whose arguments pass the tool’s schema, which needs no ground truth |
| Handoff correctness | Expected agent transfers that occurred, in the right direction |
| Redundancy / retry rate | Repeated or retried calls per task |

Argument matching levels (strictest to loosest): exact, normalized (case, whitespace, date and number formats), schema-valid only, and semantically equivalent (judged by an LLM).

Worked example (illustrative):

| Step | Agent | Tool | Arguments observed | Expected | Verdict |
| --- | --- | --- | --- | --- | --- |
| 1 | Research | search_policy | query="claim 4471 coverage" | query contains claim ID | Correct |
| 2 | Analysis | query_db | date="03/04/2026" | date in ISO format | Argument format error |
| 3 | Analysis | query_db (retry) | date="2026-04-03" | same | Correct, but a retry |
| 4 | Response | none | n/a | n/a | n/a |

The final answer is correct, so final-answer scoring passes. The trajectory shows an argument error and a retry. This is the RQ3 phenomenon.

### B. Computing success and cost, end to end and per sub-task

#### B.1 Three levels of “task”

```text
 LEVEL 2  End-to-end task        "Process this claim question"          (one dataset item)
    │
 LEVEL 1  Sub-task / step        "Research agent retrieves policy",     (an agent's responsibility,
    │                             "Analysis agent queries DB"            or a tool call)
    │
 LEVEL 0  Run                    one execution of one task by one model (one repeat)
```

#### B.2 Defining success

| Level | Success definition | Typical evaluator |
| --- | --- | --- |
| End-to-end, outcome success | Final answer meets the criteria (match, rubric score ≥ threshold) and hard constraints hold (schema, policy) | Deterministic + judge |
| End-to-end, clean success | Outcome success and no trajectory violations (right tools, valid arguments, no excess retries) | Plus trajectory evaluators |
| Sub-task | The step’s own check passes (correct tool, valid arguments, sub-agent output meets its expectation) | Trajectory evaluators, step-level judge |

Report both outcome success and clean success. The gap between them is the hidden regression that RQ3 studies.

Caveat: end-to-end success is not the product of sub-task successes. Agents recover from errors, and a failed sub-task does not always fail the whole task. Keep them as separate measures.

#### B.3 Cost formulas

Cost of one run = sum over its spans of:

- LLM span cost = input_tokens × input_price + output_tokens × output_price (+ cached-token and reasoning-token pricing where the provider differs)
- Tool span cost = declared per-call cost (paid APIs), otherwise 0

Keep a versioned price table in the model catalogue with effective dates, because provider prices change and old experiments must stay reproducible.

For self-hosted models: cost per token ≈ (GPU cost per hour) ÷ (tokens served per hour at the measured load).

Cost per successful end-to-end task (CPST):

$$
CPST(model) = \frac{\text{total cost of ALL runs, including failed ones}}{\text{number of successful runs}}
$$

Failed runs still cost money, so they stay in the numerator. A cheaper-per-run model that fails more often can be more expensive per success.

| Model | Runs | Total cost | Successes | Cost per run | Cost per success |
| --- | --- | ---: | ---: | ---: | ---: |
| A (baseline) | 100 | $15.00 | 75 | $0.150 | $0.200 |
| B (candidate) | 100 | $9.00 | 40 | $0.090 | $0.225 |

B looks 40% cheaper per run but is more expensive per successful task. If a team retries until success, the expected cost is cost-per-attempt ÷ success probability. The same logic applies.

Cost per successful sub-task j:

$$
CPSS_j = \frac{\sum \text{cost of all spans attributed to sub-task } j \text{ across all runs}}{\text{number of runs where sub-task } j \text{ succeeded}}
$$

Use exclusive cost (the node’s own spans, excluding children) to find where money goes, and inclusive cost (with its subtree) to see what a delegation costs.

Validation invariant: the sum of exclusive costs over all sub-tasks must equal the total run cost. If it doesn’t, some spans are unattributed, and the platform flags it.

#### B.4 Latency and TTFT

| Metric | Definition |
| --- | --- |
| End-to-end latency | Request received to final output complete |
| User-perceived TTFT | Request received to first streamed token of the final answer (needs a streaming interface) |
| Per-LLM-call TTFT | Call start to first token of that call, useful for diagnosing slow models |
| Critical-path latency | Longest chain of dependent spans (parallel branches don’t add) |
| Reporting | p50 and p95 across all runs, not just successful ones |

#### B.5 Context-window endurance

Peak context utilization per run = peak context tokens ÷ model limit.

Success by context-length bucket: stratify tasks into short, medium and long contexts and compare the success rate per bucket. A model that holds at 20K tokens but degrades at 100K shows up as a slope.

### C. Statistics: pass@k, pass^k and model comparison

#### C.1 Concept first

Run each task n times. Let c be the number of successes for one task.

| Metric | Question it answers | Typical use |
| --- | --- | --- |
| pass@k | “If the user may try k times, how likely is at least one success?” | Capability (best case) |
| pass^k | “How likely are all k attempts to succeed?” | Reliability and consistency (what production needs) |

For agents that act on real systems, pass^k matters more. A model that succeeds 1 time in 2 has a decent pass@2 but a poor pass^2.

#### C.2 Formulas (per task, then averaged over tasks)

$$
pass@k = 1 - \frac{\binom{n-c}{k}}{\binom{n}{k}}
$$

$$
pass^k = \frac{\binom{c}{k}}{\binom{n}{k}}
$$

Dataset-level value = average of the per-task values over all tasks.

Requires n ≥ k.

If the true per-attempt success probability is p:

$$
pass@k = 1 - (1-p)^k
$$

$$
pass^k = p^k
$$

Worked examples (n = 5 runs per task):

| c (successes) | k | pass@k | pass^k |
| --- | ---: | ---: | ---: |
| 4 | 3 | $1 - \frac{\binom{1}{3}}{\binom{5}{3}} = 1.00$ | $\frac{\binom{4}{3}}{\binom{5}{3}} = 0.40$ |
| 2 | 2 | $1 - \frac{\binom{3}{2}}{\binom{5}{2}} = 0.70$ | $\frac{\binom{2}{2}}{\binom{5}{2}} = 0.10$ |

The second row is the point: a task solved 2 times out of 5 looks decent on pass@2 and terrible on pass^2.

Design cautions:

- Choose k in {1, 3, 5} and n at least as large as the largest k. n of 5 to 10 is a practical range.
- At temperature 0, repeats are nearly identical, so they add little information (provider-side nondeterminism aside). Document the temperature and note that reliability claims need temperature above 0 or a stated rationale.

#### C.3 Comparing a candidate to the baseline

| Need | Method | Why |
| --- | --- | --- |
| Uncertainty on any metric | Cluster (task-level) bootstrap confidence interval: resample tasks, not individual runs | Repeats within a task are correlated, so resampling runs gives falsely narrow intervals |
| Difference baseline vs. candidate | Paired bootstrap on per-task differences | Same tasks, so task difficulty cancels |
| Binary per-task outcome | McNemar’s test | Tests paired pass/fail disagreement |
| Continuous or ordinal scores | Wilcoxon signed-rank (or paired t-test if roughly normal) | Paired comparison |
| 2–3 candidates vs. baseline | Holm correction for multiple comparisons | Controls false positives |
| Proportions | Wilson interval | Behaves better than the normal approximation at small n |
| Effect size | Report the raw difference with its CI, not only a p-value | Decision-relevant |

Non-inferiority framing (strongly recommended). A migration question is rarely “is Claude better than GPT?”. It is “is Claude not meaningfully worse, and is it cheaper or faster?” Define a margin δ in advance (for example 3 percentage points of success rate). Conclude “non-inferior” if the lower bound of the CI on (candidate − baseline) is above −δ. This matches how migration decisions are made.

Judge reliability: agreement with human labels via Cohen’s kappa (two raters) or Spearman correlation (scores). This is reported as a threat to validity, per the scoped-down RQs.

Multi-objective (RQ5):

- Pareto dominance: A dominates B if A is at least as good on every objective and strictly better on one. Keep the non-dominated set.
- Optional weighted score: Σ wᵢ × normalize(metricᵢ), with min-max or baseline-relative normalization. Show it next to the Pareto view, never in place of it.

### D. Library and tool map

| Task | Candidate libraries / tools | Role | Notes |
| --- | --- | --- | --- |
| Trace standard and transport | OpenTelemetry (SDK, Collector) | Ingest and export spans | Vendor-neutral base |
| LLM-specific instrumentation | OpenInference, OpenLLMetry | Auto-instrument model and agent libraries into OTel spans | Reduces adapter code |
| LangChain/LangGraph capture | LangChain callback system; LangSmith trace export | T1/T2 source |  |
| Google ADK capture | ADK callbacks and its built-in tracing | T1 |  |
| Microsoft Agent Framework capture | Built-in OTel telemetry | T1/T2 |  |
| Model gateway / provider abstraction | LiteLLM (or an equivalent proxy) | One interface to GPT, Claude, Gemini and self-hosted models, plus T2-G logging | Central to model injection |
| Trace schema and validation | Pydantic, JSON Schema | Canonical schema enforcement |  |
| RAG metrics | Ragas, DeepEval | Faithfulness, answer relevance, context precision and recall | Wrapped as evaluator plugins |
| Agent and trajectory metrics | DeepEval (tool correctness, task completion), agentevals / openevals (trajectory matching), Ragas agent metrics | Tool and trajectory scoring | Verify current metric names |
| LLM-as-judge | DeepEval G-Eval style rubrics, custom judge prompts | Rubric scoring | Cross-family judge |
| Statistics | NumPy, SciPy (bootstrap, Wilcoxon), statsmodels (McNemar, Wilson intervals, multiple-testing correction) | CIs and significance | pass@k and pass^k are short closed-form functions you implement |
| Pareto and multi-objective | NumPy (non-dominated filter), pymoo (optional) | Pareto front | The filter is trivial to write |
| Workflow orchestration | Argo Workflows, Temporal, or Celery plus a broker | Expand and schedule jobs, retry, resume | Pick one. Argo suits Kubernetes-native deployments |
| Autoscaling | KEDA | Scale workers on queue depth |  |
| Metadata store | PostgreSQL | Experiments, runs, scores, lineage |  |
| Trace and artifact store | MinIO (S3-compatible), Parquet files | Portable payload storage |  |
| Analytical queries | DuckDB | Fast ad-hoc analysis over Parquet | Useful for research analysis |
| Tool mocking / record-replay | VCR-style HTTP recorders, WireMock | Deterministic tool responses | Addresses R2 |
| PII redaction | Microsoft Presidio | Anonymize datasets and traces |  |
| Prompt and skill registry | Git-backed versioned files, or MLflow / Langfuse prompt management | Versioning with history | Git is the simplest for audit |
| Experiment tracking (optional) | MLflow | Run comparison UI | Optional. Don’t duplicate your own lineage |
| API layer | FastAPI | Platform API |  |
| UI and reports | React front end, or Streamlit / Grafana for a fast start | Admin console, Pareto charts, trace viewer |  |
| Secrets and identity | Vault; OIDC-based SSO | Keys and RBAC |  |
| Platform observability | Prometheus, Grafana, OTel | Monitor the platform itself |  |
| Future work | DSPy | Prompt optimization | Extension point only |

### E. Low-level repository structure

A monorepo, layered so that the canonical schema depends on nothing and everything else depends on it.

```text
llm-migration-eval-platform/
├── README.md
├── docs/
│   ├── architecture/              # HLD, LLD, diagrams
│   ├── schema/                    # canonical trace schema spec + mapping tables
│   ├── metrics/                   # metric definitions and formulas
│   ├── research/                  # RQ1/RQ3/RQ5 protocols, study materials
│   └── runbooks/
│
├── packages/
│   ├── core-schema/               # canonical trace, span types, task and score models (no dependencies)
│   ├── stats/                     # pass@k, pass^k, bootstrap, paired tests, non-inferiority, Pareto
│   ├── cost-latency/              # price table logic, cost attribution, TTFT, critical path
│   └── common/                    # config, logging, errors, ids
│
├── integration/
│   ├── adapters/
│   │   ├── langgraph/             # callback-based capture + mapping
│   │   ├── google_adk/            # callback-based capture + mapping
│   │   ├── ms_agent_framework/    # OTel-based capture + mapping
│   │   ├── custom_wrapper/        # decorators for custom agents and tool registries
│   │   └── blackbox_http/         # black-box invocation client
│   ├── otel_ingest/               # collector config, OTel-to-canonical converters
│   ├── trace_importers/           # LangSmith / Ragas / DeepEval log importers
│   └── gateway/                   # LLM gateway config, header propagation, request logging
│
├── control-plane/
│   ├── api/                       # platform REST API
│   ├── console-ui/                # admin console and report viewer
│   ├── registry/
│   │   ├── apps/                  # app profiles, agent and tool inventory
│   │   ├── datasets/              # versioned datasets + expected-trajectory specs
│   │   ├── prompts_skills/        # versioned prompts and skills
│   │   └── models/                # model catalogue, capabilities, price table
│   ├── gates/                     # hard-gate rules and eligibility checks
│   ├── orchestrator/              # experiment freezing, job expansion, state machine, lineage
│   ├── reporting/                 # aggregation, comparison, report generation
│   └── governance/                # RBAC, audit log, approval workflows
│
├── execution-plane/
│   ├── replay_worker/             # runs the full app per job
│   ├── model_injector/            # per-framework model override strategies
│   ├── normalizer/                # raw trace → canonical trace pipeline + validation
│   ├── tool_sandbox/              # record/replay and mocking of tools
│   ├── evaluators/
│   │   ├── registry/              # plugin contract and discovery
│   │   ├── deterministic/
│   │   ├── trajectory/            # tool selection, arguments, order, handoffs, retries
│   │   ├── judge/                 # rubric and cross-family judge logic
│   │   ├── rag/                   # Ragas / DeepEval wrappers
│   │   └── human/                 # review queue, labeling UI backend
│   └── provider_adapters/         # OpenAI/Azure, Anthropic, Google, self-hosted
│
├── data-layer/
│   ├── migrations/                # relational schema migrations
│   ├── repositories/              # data access for runs, scores, lineage
│   └── object_store/              # payload storage by hash
│
├── deploy/
│   ├── containers/                # one image per service and worker type
│   ├── kubernetes/                # base manifests, overlays for any cloud
│   ├── remote-worker/             # package for in-boundary workers
│   └── observability/             # platform dashboards and alerts
│
├── studies/                       # research code kept separate from the platform
│   ├── rq1_mapping_coverage/
│   ├── rq3_regression_detection/  # seeded-regression generator + analysis
│   └── rq5_decision_study/        # report variants, survey, analysis
│
├── examples/
│   ├── sample_app_langgraph/
│   ├── sample_app_adk/
│   ├── sample_app_maf/
│   ├── sample_dataset/
│   └── sample_experiment/
│
└── tests/
    ├── unit/
    ├── contract/                  # every adapter vs. canonical schema; every evaluator vs. plugin contract
    ├── integration/
    └── e2e/                       # full tiny experiment in a local cluster
```

Why this layout:

- `core-schema` and `stats` sit at the bottom, so the thesis’s core contributions are small, testable and reusable on their own.
- `integration/adapters` is isolated per framework, so adding one doesn’t touch the rest. Contract tests enforce the canonical mapping, and RQ1’s mapping-coverage measurement can run directly from these tests.
- `studies/` is separate so research scripts never leak into the platform.

### F. Coverage check: are all the key considerations addressed?

Short answer: not fully. Several are covered, several only partly, and a few are deliberate scope decisions that conflict with your notes. Here is the honest matrix.

#### F.1 Section A and B items (your numbered list)

| Item | Status | Notes |
| --- | --- | --- |
| A1 Data privacy / residency | Covered | Remote in-boundary workers, redaction. Self-hosted vs. API is a catalogue attribute. |
| A2 Context length, modality, tool calling | Covered | Capability gates in the model catalogue |
| A3 Throughput | Partial | Throughput is measured, not declared. Add a load-probe step to gate validation, because vendor limits are not observed throughput. |
| B1 Quality vs. cost per successful task vs. TTFT | Now covered (§B.3, B.4). TTFT and CPST were missing in v2.0. |  |
| B2 Agent steps and context endurance | Now covered (§A.5, B.5) |  |
| B3 Prompt re-adaptation (DSPy) | Deferred by decision | You chose admin-driven manual revision. DSPy is a documented extension point. Flag it as future work in the proposal. |
| B4 pass@k runs | Now covered (§C) |  |
| B5 Shadow deployment | Out of scope by decision | You limited evaluation to offline and preproduction. List it as future work. |

#### F.2 Key considerations

| Consideration | Status | What’s missing or needed |
| --- | --- | --- |
| Framework portability | Covered | KIP set aside per your instruction. “GSP” is still unconfirmed. I assumed a cloud environment, so please confirm. |
| Evaluation data strategy | Partial | Ground truth and sampled interactions are covered. Business outcomes (resolution rate, escalations, conversions) are not. Proposal: allow optional outcome fields on dataset items and report them as secondary metrics. |
| Human-in-the-loop feedback | Partial | Expert review is covered, but the user feedback channel (thumbs, edits, corrections) and explicit vs. implicit weighting are not. Proposal below. |
| Evaluation scope (intermediate steps and component vs. end-to-end) | Now covered (§A, B) |  |
| Prompt management and governance | Mostly covered | Versioned registry and lineage exist. Add a trend view linking prompt versions to results across experiments over time. |
| Quality vs. retry analysis | Partial | Retries are captured as spans but no metric was defined. Proposal below. |
| LLM catalogue and configuration | Partial | A platform-level catalogue with roles and approvals exists. A runtime catalogue exposed inside every application was dropped when you chose evaluation plus selection (no runtime control). State this boundary explicitly. |
| Model traceability | Covered | Every run links to model, version, parameters, prompt, app and evaluator versions |
| Evaluation metrics (standard vs. configurable) | Partial | Metrics are selectable, but I never defined a mandatory core set. Proposal below. |

#### F.3 Section “Design a centralized framework that provides…” (items 1–8)

| # | Requirement | Status |
| --- | --- | --- |
| 1 | Platform-independent deployment | Covered |
| 2 | Intermediate steps + final outputs | Covered |
| 3 | Human-in-the-loop feedback integration | Partial (see below) |
| 4 | Prompt and model version management | Covered |
| 5 | Full lineage and traceability | Covered |
| 6 | Configurable model catalogues and runtime parameters | Covered for evaluation. Application runtime control excluded. |
| 7 | Standardized and application-specific metrics | Partial (see below) |
| 8 | Quality measurement using retries, corrections and feedback | Partial (see below) |

#### B.5 Context-window endurance

Peak context utilization per run = peak context tokens ÷ model limit.

Success by context-length bucket: stratify tasks into short, medium and long contexts and compare the success rate per bucket. A model that holds at 20K tokens but degrades at 100K shows up as a slope.

### C. Statistics: pass@k, pass^k and model comparison

#### C.1 Concept first

Run each task n times. Let c be the number of successes for one task.

| Metric | Question it answers | Typical use |
| --- | --- | --- |
| Mandatory core (all apps) | Outcome success, clean success, pass@k, pass^k, cost per successful task, p50/p95 latency, TTFT, tool-argument validity, retry rate | Platform. Not removable. |
| Standard optional | Ragas/DeepEval RAG metrics, groundedness, tool-selection precision and recall, handoff correctness | App owner selects |
| Application-specific | Custom rubrics, domain checks, business-outcome metrics | App owner defines. Admin approves. |

2. Retry and quality analysis (closes the retry gap)

- First-attempt success rate versus eventual success rate, where the difference is quality that depends on retrying.
- Retries per task and success by attempt number.
- Cost of retries as a share of total cost.
- In production data, a user regenerating a response is an ambiguous signal. It can mean dissatisfaction, curiosity or a UI habit. So treat retry frequency as a weak, triage-level signal that prioritizes samples for review, and never as ground truth.

3. Feedback weighting (closes the human feedback gap)

| Signal | Reliability | Use |
| --- | --- | --- |
| Uncertainty on any metric | Cluster (task-level) bootstrap confidence interval: resample tasks, not individual runs | Repeats within a task are correlated, so resampling runs gives falsely narrow intervals |
| Difference baseline vs. candidate | Paired bootstrap on per-task differences | Same tasks, so task difficulty cancels |
| Binary per-task outcome | McNemar’s test | Tests paired pass/fail disagreement |
| Continuous or ordinal scores | Wilcoxon signed-rank (or paired t-test if roughly normal) | Paired comparison |
| 2–3 candidates vs. baseline | Holm correction for multiple comparisons | Controls false positives |
| Proportions | Wilson interval | Behaves better than the normal approximation at small n |
| Effect size | Report the raw difference with its CI, not only a p-value | Decision-relevant |

Non-inferiority framing (strongly recommended). A migration question is rarely “is Claude better than GPT?”. It is “is Claude not meaningfully worse, and is it cheaper or faster?” Define a margin δ in advance (for example 3 percentage points of success rate). Conclude “non-inferior” if the lower bound of the CI on (candidate − baseline) is above −δ. This matches how migration decisions are made.

Judge reliability: agreement with human labels via Cohen’s kappa (two raters) or Spearman correlation (scores). This is reported as a threat to validity, per the scoped-down RQs.

Multi-objective (RQ5):

- Pareto dominance: A dominates B if A is at least as good on every objective and strictly better on one. Keep the non-dominated set.
- Optional weighted score: Σ wᵢ × normalize(metricᵢ), with min-max or baseline-relative normalization. Show it next to the Pareto view, never in place of it.

## 12. RegDoc worked example

This example shows the evaluation pipeline on a fictional RegDoc system. It answers the question: can we swap the model in an agentic application that processes documents, and how do cost and quality compare?

### 12.1 Worked Example: Running the Evaluation Pipeline on RegDoc

All numbers below are invented to show how the method works. They are not real model results. The RegDoc design details (agents, tools, frameworks) are assumptions, which I list first. Correct them and the example adapts.

#### 0. The scenario and assumptions

Question to answer: RegDoc runs on GPT today. Can Claude or Gemini replace it, at what cost and latency, and where would it break?

Assumed RegDoc design:

| Element | Assumption |
| --- | --- |
| Framework | LangGraph (Tier 1 integration via callbacks) |
| Ingestion | Uploaded Excel, PPT and PDF files are parsed into chunks with metadata (file, page/sheet/slide, cell range) and stored in a vector index |
| Capabilities | (1) Chat over documents, (2) Compliance report from a checklist, (3) Document comparison for discrepancies |
| Sub-agents | Router, Retrieval, Spreadsheet Analyst, Compliance, Comparison, Writer |
| Tools | vector_search, get_sheet_range, run_sheet_query, get_slide, get_page, list_checklist_items, compare_sections, write_report_section, text_to_sql, get_sql_schema, run_sql_query |
| Model seam | The model is set in one config (a chat model object), so the worker can override it |

#### 1. The key design move for RegDoc: freeze ingestion

RegDoc has two stages: ingest (parse and index) and answer (agents and tools). If each candidate model re-parsed the documents, you could not tell whether a score changed because of the model or because of different chunks.

Principle: change only the thing under test.

```text
 Documents ──▶ Parse ──▶ Chunk ──▶ Embed ──▶ INDEX  (built ONCE, frozen, versioned)
                                                  │
                  ┌───────────────────────────────┼───────────────────────────────┐
                  ▼                               ▼                               ▼
          GPT (baseline)                      Claude                          Gemini
          same index, same tools,             same index, same tools,         same index, same tools,
          same prompts                        same prompts                    same prompts
```

If ingestion also uses an LLM (for example a vision model to read slide images), that is a separate migration experiment. Keep it out of this one and note it as future work.

#### 2. Onboarding RegDoc (Stages 1–2 of the journey)

Step 1. Register the app profile. The profile records the owner, the framework (LangGraph), the data sensitivity (regulated documents, so redaction is on and workers run in-boundary), and the integration tier.

Step 2. Agent and tool inventory. This answers the question about how the platform knows which sub-agent or tool should fire. The inventory is declared once:

| Agent | Role | Tools it may use |
| --- | --- | --- |
| Router | Classify request, pick workflow | none |
| Retrieval | Find relevant passages | vector_search, get_page, get_slide |
| Spreadsheet Analyst | Read and query large sheets | get_sheet_range, run_sheet_query, get_sql_schema, run_sql_query, text_to_sql |
| Compliance | Walk the checklist item by item | list_checklist_items, vector_search, run_sql_query |
| Comparison | Align and diff two documents | compare_sections, vector_search |
| Writer | Compose answers and report sections | write_report_section |

The inventory has two uses. It maps framework nodes to agent names (LangGraph node `sheet_node = Spreadsheet Analyst`). It also lets the platform detect expected agent never invoked.

Step 3. Smoke test. Run one question, capture the trace, and validate it against the canonical schema. Check that every span has an agent, a type, a payload, and a parent. If spans come back unattributed, fix the node mapping before continuing.

Step 4. Model seam check. Confirm the worker can override the chat model through config. It can, so RegDoc is onboarded as Tier 1, full trajectory.

#### 3. Building the dataset (the app team’s job)

The dataset must cover all three capabilities and document difficulty, because long-context behavior is where model swaps often fail.

##### 3.1 Document corpus (frozen)

| Bucket | Examples | Why it matters |
| --- | --- | --- |
| Small | 10-page PDF, 5-slide deck, 3-sheet workbook | Easy baseline |
| Medium | 100-page PDF, 60-slide deck, 20-sheet workbook | Typical use |
| Large | 400-page PDF, 200-slide deck, workbook with 100K+ rows | Context endurance and tool-use stress |

##### 3.2 Task mix (illustrative: 120 tasks)

| Capability | Tasks | Example | What “gold” looks like |
| --- | --- | --- | --- |
| Chat / RAG | 50 | “What is the capital adequacy ratio in the Q3 workbook, and does it meet the threshold in the policy PDF?” | Reference answer, evidence locations (file, sheet, cell range, page) |
| Compliance report | 40 | A 25-item checklist over a filing pack | Per-item verdict (compliant / non-compliant / not applicable), evidence location, a “critical” flag on some items |
| Document comparison | 30 | Compare policy v1 and v2 | A list of seeded discrepancies |

Seeding trick for comparison tasks: take a real document, copy it, and inject a known number of changes (a changed figure, a shifted date, a deleted clause, a swapped table row). You now know the correct answer exactly. Without seeding, discrepancy detection has no ground truth.

Include negatives. About 15% of chat tasks should have no answer in the documents. The correct behavior is to say so. This tests hallucination.

##### 3.3 Expected-trajectory spec (optional but powerful)

For a subset of tasks, the app team adds expectations. Example for the capital-adequacy question:

| Expectation | Value |
| --- | --- |
| Agents expected | Router, Retrieval, Spreadsheet Analyst, Writer |
| Tools expected | vector_search (query mentions “capital adequacy”), get_sheet_range (sheet = “Risk”, range covers the ratio row), text_to_sql or run_sql_query if a database-backed workbook is used |
| Ordering | Retrieval and Spreadsheet steps may run in either order, Writer last |
| Argument rules | sheet exact match (case-sensitive), range must include the target row |

Tasks without a spec still get spec-free checks: schema validity of arguments, retry counts, and redundancy.

#### 4. Hard gates (execution team, Stage 4)

| Gate | RegDoc-specific reason | How it’s checked |
| --- | --- | --- |
| Data residency | Regulated documents must stay in-region | Catalogue attribute: in-region API or self-hosted |
| Context length | Large documents need long context | Catalogue attribute plus a measured test |
| Tool calling | The entire agent design depends on it | Catalogue attribute plus a probe (does the model emit valid tool calls?) |
| Modality | Needed only if parsing sends images to the model | Not required here because ingestion is frozen |
| Throughput | Compliance reports make many calls per task | Load probe: measure requests/min and error rate at planned concurrency |

Result: GPT (baseline), Claude and Gemini all pass. A fourth model that failed the context gate is rejected with its reason recorded.

#### 5. Experiment configuration (Admin, Stage 5)

| Setting | Value |
| --- | --- |
| Dataset | regdoc-golden@v1 (120 tasks) plus frozen index idx@v1 |
| Models | GPT (baseline), Claude, Gemini |
| Parameters | Temperature 0.3, same top-p and max tokens for all (documented). Temperature above 0 is needed or repeats add no information |
| Prompts and skills | Unchanged from production (this is “Experiment 1: drop-in swap”) |
| Repeats | n = 5 |
| Total runs | 120 × 3 × 5 = 1,800 |
| Mandatory metrics | Outcome success, clean success, pass@k, pass^k (k = 3), cost per successful task, p50/p95 latency, TTFT, tool-argument validity, retry rate |
| Optional metrics | Ragas faithfulness and context precision/recall, citation accuracy, tool-selection precision/recall, discrepancy precision/recall |
| Judge | A model from a family outside the three candidates, or a two-model ensemble, plus human spot checks |
| Non-inferiority margin δ | 3 percentage points on outcome success |

The admin freezes this configuration. After that it is immutable, and any change creates a new experiment version.

#### 6. One run, step by step (Stages 6–8)

Take the chat task “What is the capital adequacy ratio in the Q3 workbook, and does it meet the policy threshold?” with Model A at repeat 1.

##### 6.1 What the platform captures

| # | Agent | Span type | Payload (abridged) | Tokens in/out | Result |
| --- | --- | --- | --- | --- | --- |
| S1 | Router | LLM call | Classifies as “cross-document question” | 1,200 / 80 | OK |
| S2 | Retrieval | LLM call | Decides to search | 2,000 / 120 | OK |
| S3 | Retrieval | Tool call | vector_search(query="capital adequacy threshold", top_k=8, doc="policy.pdf") | n/a | 8 chunks |
| S4 | Retrieval | LLM call | Reads chunks, extracts threshold | 7,000 / 300 | “Minimum 10.5%”, page 41 |
| S5 | Spreadsheet | LLM call | Plans sheet read | 3,000 / 150 | OK |
| S6 | Spreadsheet | Tool call | get_sheet_range(sheet="risk", range="B2:F40") | n/a | Error: sheet not found |
| S7 | Spreadsheet | LLM call (retry, linked to S6) | Fixes the call | 3,400 / 140 | OK |
| S8 | Spreadsheet | Tool call | get_sheet_range(sheet="Risk", range="B2:F40") | n/a | 39 rows returned |
| S9 | Spreadsheet | LLM call | Extracts ratio | 12,000 / 250 | “12.1%” |
| S10 | Writer | LLM call | Composes answer with citations | 9,000 / 400 | “Yes, 12.1% vs 10.5% minimum” |

How the platform knew this. LangGraph’s callbacks fired at each node, LLM call and tool call, carrying run IDs and parent IDs. The adapter turned each into a span. The tool name and arguments came from the tool-call event, and the agent name came from the node-to-agent mapping in the inventory.

##### 6.2 Scoring this one run

| Check | Evaluator | Result |
| --- | --- | --- |
| Final answer matches reference (12.1%, meets threshold) | Deterministic (number match) plus judge | Pass |
| Citation accuracy (cited page 41 and sheet “Risk” contain the evidence) | Deterministic lookup against the frozen index | Pass |
| Faithfulness (answer supported by retrieved context) | Ragas | 0.96 |
| Tool selection (expected vector_search, get_sheet_range) | Trajectory | Precision 1.0, recall 1.0 |
| Argument accuracy | Trajectory | Fail on S6: sheet="risk" does not match the case-sensitive expectation |
| Retry count | Trajectory | 1 |
| Context peak | Computed | 12,000 tokens in the largest call, a small share of the limit |

Outcome success = yes. Clean success = no, because of the argument error and the retry. A final-answer-only evaluation would have recorded this as a pass with no warning. That gap is exactly what RQ3 measures.

##### 6.3 Cost for this run

Illustrative prices: $2.50 per million input tokens, $10 per million output tokens.

| Agent | Input tokens | Output tokens | Exclusive cost |
| --- | ---: | ---: | ---: |
| Router | 1,200 | 80 | $0.0038 |
| Retrieval | 9,000 | 420 | $0.0267 |
| Spreadsheet Analyst | 18,400 | 540 | $0.0514 |
| Writer | 9,000 | 400 | $0.0265 |
| Total | 37,600 | 1,440 | $0.1084 |

Invariant check: the exclusive costs sum to the run total ($0.1084). If they didn’t, some spans would be unattributed and the platform would flag it.

Waste attributable to the retry: span S7 cost about $0.0099, roughly 9% of the run. Aggregated over 1,800 runs, retry cost becomes a headline number.

#### 7. Success at each level for RegDoc’s three capabilities

The three capabilities need different definitions of “the end-to-end task” and “the sub-task”.

| Capability | End-to-end task | Sub-task | Outcome success means | Clean success adds |
| --- | --- | --- | --- | --- |
| Chat / RAG | One question | Each agent step (retrieve, read sheet, write) | Answer correct, citations valid, or correct “not found” | Valid arguments, no excess retries, expected tools used |
| Compliance report | One 25-item checklist | One checklist item | All critical items correct and overall item accuracy ≥ a threshold you set (for example 90%) | No item processed with invalid tool arguments, no skipped items |
| Comparison | One document pair | One seeded discrepancy | Recall of seeded discrepancies ≥ threshold and precision ≥ threshold | Right sections aligned, no repeated alignment retries |

##### 7.1 Compliance report worked example

Gold: 25 items, 5 flagged critical. Model B’s output:

| Measure | Value |
| --- | --- |
| Items with correct verdict | 22 of 25 (88%) |
| Critical items correct | 4 of 5 |
| Evidence citation correct (among correct verdicts) | 20 of 22 |

With the rule “all critical items correct and ≥ 90% overall,” this report fails even though 88% looks close. A regulated setting justifies a hard rule on critical items, and it is your admin’s choice to set.

Cost per successful checklist item = total cost of all item-level spans across runs ÷ number of items judged correct. This is the sub-task version of cost per success.

##### 7.2 Comparison task worked example

12 seeded discrepancies. Model B reports 11 discrepancies, 9 of them correct.

- Recall = 9 / 12 = 0.75
- Precision = 9 / 11 = 0.82
- F1 = 2 × 0.75 × 0.82 / (0.75 + 0.82) ≈ 0.78

Also check location accuracy (did it point to the right page, slide or sheet?) and the number of false discrepancies, which waste reviewer time in practice.

#### 8. Aggregating pass@k and pass^k with RegDoc data

Take one task with n = 5 runs. For three tasks, with k = 3:

| Task | Successes c | pass@3 = 1 − C(n−c,3) / C(n,3) | pass^3 = C(c,3) / C(n,3) |
| --- | ---: | ---: | ---: |
| T-01 | 5 | 1.00 | 1.00 |
| T-02 | 3 | 1 − 0/10 = 1.00 | 1/10 = 0.10 |
| T-03 | 0 | 1 − 1/10 = 0.00 | 0.00 |

Dataset-level pass@3 = (1.00 + 1.00 + 0.00) / 3 = 0.67. Dataset-level pass^3 = (1.00 + 0.10 + 0.00) / 3 = 0.37.

Reading it: T-02 succeeds 3 times out of 5. A user who can retry will probably get an answer, so pass@3 says “fine”. A compliance officer who needs consistency sees a coin flip, and pass^3 shows it. For RegDoc’s compliance and comparison workloads, pass^k is the more honest number.

#### 9. The comparison report (illustrative results)

Headline table (120 tasks, 5 repeats, unchanged prompts):

| Metric | GPT (baseline) | Claude | Gemini |
| --- | ---: | ---: | ---: |
| Outcome success | 78% | 77% | 72% |
| Clean success | 66% | 70% | 58% |
| Gap (outcome − clean) | 12 pts | 7 pts | 14 pts |
| pass@3 | 0.90 | 0.90 | 0.86 |
| pass^3 | 0.61 | 0.63 | 0.52 |
| Tool-argument validity | 94% | 97% | 89% |
| Retry rate (per task) | 0.8 | 0.4 | 1.3 |
| Cost per run | $0.11 | $0.13 | $0.06 |
| Cost per successful task | $0.141 | $0.169 | $0.083 |
| p95 latency | 21 s | 26 s | 14 s |
| p50 TTFT | 1.9 s | 2.4 s | 1.1 s |

By capability (outcome success):

| Capability | GPT | Claude | Gemini |
| --- | ---: | ---: | ---: |
| Chat / RAG | 84% | 83% | 80% |
| Compliance report | 74% | 76% | 68% |
| Comparison (F1) | 0.74 | 0.71 | 0.60 |

By document size (outcome success):

| Size | GPT | Claude | Gemini |
| --- | ---: | ---: | ---: |
| Small | 90% | 88% | 90% |
| Medium | 78% | 78% | 73% |
| Large | 62% | 64% | 48% |

##### 9.1 Statistical reading

Compute the difference (candidate − baseline) per task, then bootstrap by resampling tasks, not runs, since repeats within a task are correlated.

| Comparison | Outcome success difference | 95% CI | Verdict at δ = 3 pts |
| --- | ---: | --- | --- |
| Claude − GPT | −1.0 pt | [−4.2, +2.3] | Inconclusive: lower bound is below −3, so non-inferiority is not shown |
| Gemini − GPT | −6.0 pts | [−10.1, −1.8] | Worse: the entire interval lies below −3 |

The teaching point is that “−1 point” looks like parity, but with 120 tasks the interval is too wide to prove it. You can either state “inconclusive” or add tasks to narrow the interval. Apply Holm correction since you made two comparisons.

##### 9.2 Pareto view (RQ5)

Dimensions: quality (clean success), cost per successful task, p95 latency.

| Model | Quality | Cost/success | Latency | Dominated? |
| --- | ---: | ---: | ---: | --- |
| GPT | 66% | $0.141 | 21 s | Yes: Claude is better on quality and Gemini is better on cost and latency, but neither beats GPT on all three, so GPT is not dominated |
| Claude | 70% | $0.169 | 26 s | No |
| Gemini | 58% | $0.083 | 14 s | No |

Correction: in this invented example, no model dominates another, so all three sit on the Pareto front. That is realistic, and it is precisely why the report shows trade-offs instead of naming a winner. The decision owner weighs “Claude is more reliable but 20% pricier and slower” against “Gemini is far cheaper but fails on large documents”.

##### 9.3 Trace diffs (RQ3)

The report links to concrete divergences, for example:

- Gemini on spreadsheet tasks: frequently passes wrong sheet names or ranges, then retries. Final answers recover, so outcome success hides it. Clean success exposes it.
- Gemini on large documents: retrieves fewer relevant chunks before answering, so the compliance report misses evidence.
- Claude on comparison: the trace shows it aligns sections correctly but reports fewer discrepancies (lower recall, higher precision).

##### 9.4 Informational recommendation

Claude is a plausible drop-in replacement for compliance and chat workloads at higher cost. Gemini is not recommended with the current prompts, but its failures cluster in tool-argument formatting, which suggests a prompt revision might change the picture. Neither claim is established until Experiment 2.

#### 10. Iteration: Experiment 2 (manual revision, the dashed loop)

The admin sees that Gemini’s errors cluster on sheet names. They revise the Spreadsheet Analyst’s tool-usage instructions to stress exact, case-sensitive names, creating prompt v2, and trigger a new experiment.

| Experiment | Prompts | Purpose |
| --- | --- | --- |
| Exp 1 | Unchanged (v1) | Measures a pure drop-in swap |
| Exp 2 | Revised for Gemini (v2) | Measures what a modest prompt adaptation recovers |

Report both and keep them separate. If you only report Exp 2, you hide the migration effort. If you only report Exp 1, you may unfairly reject a model that merely needs different instructions. This also gives you a data point on prompt-model coupling, which supports the DSPy future-work argument.

If prompt v2 for Gemini also changes GPT’s prompt, GPT must be re-run so the baseline stays fair.

#### 11. End-to-end flow of this walk-through

```text
 Onboard RegDoc ─▶ Agent/tool inventory ─▶ Smoke test ─▶ Tier 1 confirmed
        │
        ▼
 Freeze docs + index (idx@v1) ─▶ Dataset v1 (120 tasks, seeded diffs, negatives)
        │
        ▼
 Hard gates (residency, context, tool-call probe, load probe) ─▶ 3 models eligible
        │
        ▼
 Admin freezes Exp 1 ─▶ 1,800 runs ─▶ capture spans ─▶ normalize ─▶ score
        │                                                         │
        │                                          deterministic + trajectory + Ragas + judge
        │                                                         │
        │                                                human review on sampled outputs

# Working Example: Evaluating LLM Migration for RegDoc — An Agentic RAG and SQL-Based Application

## 1. Objective

RegDoc is an enterprise document intelligence application that allows users to:

1. Upload and chat with large PDF, Excel, PowerPoint, and Word documents.
2. Generate regulatory compliance reports using predefined checklists.
3. Compare multiple document versions and identify discrepancies.
4. Extract, aggregate, and analyze structured information from spreadsheets.
5. Query structured data stored in SQL databases.
6. Combine evidence from SQL databases and unstructured documents to answer complex questions.

RegDoc currently uses GPT as its primary LLM.

The evaluation platform must answer:

> Can Claude or Gemini replace GPT in RegDoc without degrading answer quality, agent execution, tool selection, compliance accuracy, latency, or cost efficiency?

The platform evaluates the complete application workflow, not merely isolated LLM responses.

**Important:** All numerical results, costs, traces, schemas, and model scores in this example are illustrative. They do not represent real experiments.

---

## 2. RegDoc Application Architecture

### 2.1 Assumed Agent Architecture

For this walkthrough, assume RegDoc uses LangGraph for orchestration and PostgreSQL for structured data.

These are implementation assumptions, not requirements of the evaluation platform.

| Component | Responsibility |
|---|---|
| Supervisor Agent | Understands user intent and delegates tasks |
| Document Retrieval Agent | Retrieves relevant content from uploaded documents |
| Spreadsheet Analyst Agent | Reads and analyzes large Excel workbooks |
| SQL Analyst Agent | Queries structured relational databases |
| Compliance Agent | Evaluates evidence against regulatory checklists |
| Document Comparison Agent | Identifies discrepancies between document versions |
| Report Writer Agent | Produces the final answer or compliance report |

### 2.2 Available Tools

| Tool | Used By | Purpose |
|---|---|---|
| `vector_search` | Retrieval Agent | Semantic retrieval across indexed documents |
| `get_pdf_page` | Retrieval Agent | Retrieve a particular PDF page |
| `get_slide` | Retrieval Agent | Retrieve PowerPoint content |
| `get_sheet_range` | Spreadsheet Agent | Read Excel cells |
| `run_sheet_query` | Spreadsheet Agent | Query structured spreadsheet data |
| `get_database_schema` | SQL Agent | Discover permitted database tables and columns |
| `execute_readonly_sql` | SQL Agent | Execute validated SQL SELECT queries |
| `list_checklist_items` | Compliance Agent | Retrieve regulatory checklist requirements |
| `compare_sections` | Comparison Agent | Compare corresponding sections of documents |
| `write_report_section` | Writer Agent | Generate a report section |

### 2.3 Application Data Sources

RegDoc works with two primary data categories.

**Unstructured and semi-structured sources**

- Regulatory policy PDFs
- Audit reports
- PowerPoint presentations
- Financial statements
- Excel workbooks
- Compliance checklists

**Structured sources**

A PostgreSQL database with tables such as:

```sql
CREATE TABLE capital_adequacy (
    entity_id INTEGER,
    reporting_period VARCHAR(20),
    tier1_capital NUMERIC(18, 2),
    tier2_capital NUMERIC(18, 2),
    risk_weighted_assets NUMERIC(18, 2),
    reported_car_ratio NUMERIC(8, 4)
);

CREATE TABLE compliance_findings (
    finding_id INTEGER PRIMARY KEY,
    entity_id INTEGER,
    regulation_id VARCHAR(50),
    finding_status VARCHAR(30),
    severity VARCHAR(20),
    observation_date DATE
);

CREATE TABLE regulatory_thresholds (
    regulation_id VARCHAR(50),
    metric_name VARCHAR(100),
    threshold_value NUMERIC(12, 4),
    effective_date DATE
);
```

The SQL Analyst is allowed to query approved views and read-only tables.

It cannot modify production data.

---

# 3. The Controlled Evaluation Strategy

RegDoc has multiple processing stages:

```text
                     USER DOCUMENT UPLOAD
                              |
                              v
                    DOCUMENT INGESTION
                              |
                 +------------+------------+
                 |            |            |
                 v            v            v
             PDF Parser   Excel Parser  PPT Parser
                 |            |            |
                 +------------+------------+
                              |
                              v
                        CHUNKING
                              |
                              v
                         EMBEDDING
                              |
                              v
                      VECTOR INDEX v1
                              |
                              |
                  FROZEN FOR EXPERIMENT
                              |
                              v
                       REGDOC AGENTS
                              |
          +-------------------+------------------+
          |                   |                  |
          v                   v                  v
        GPT                CLAUDE             GEMINI
      Baseline            Candidate 1         Candidate 2
          |                   |                  |
          v                   v                  v
       SAME DATA           SAME DATA          SAME DATA
       SAME TOOLS          SAME TOOLS         SAME TOOLS
       SAME INDEX          SAME INDEX         SAME INDEX
       SAME PROMPTS        SAME PROMPTS       SAME PROMPTS
          |                   |                  |
          +-------------------+------------------+
                              |
                              v
                     EVALUATION PLATFORM
                              |
                              v
                    COMPARISON REPORT
```

### 3.1 What Must Remain Constant?

To isolate the effect of changing the LLM, the experiment freezes:

| Artifact | Example Version |
|---|---|
| RegDoc application | `regdoc@v2.1` |
| Golden dataset | `regdoc-golden@v1` |
| Parsed documents | `corpus@v1` |
| Vector index | `idx@v1` |
| Database snapshot | `postgres-snapshot@v1` |
| Database schema | `schema@v1` |
| Tool implementations | `tools@v1` |
| Prompts | `prompts@v1` |
| Agent definitions | `agents@v1` |
| Evaluation rubrics | `eval-rubrics@v1` |

SQL queries must execute against the same database snapshot.

Otherwise, a result could change because of the database rather than the LLM.

### 3.2 What Changes?

Only the model configuration changes during Experiment 1.

```yaml
experiment:
  id: regdoc-exp-001

  application: regdoc@v2.1
  dataset: regdoc-golden@v1

  candidates:
    - id: baseline
      provider: openai
      model: gpt-baseline

    - id: candidate-1
      provider: anthropic
      model: claude-candidate

    - id: candidate-2
      provider: google
      model: gemini-candidate

  model_parameters:
    temperature: 0.3
    max_output_tokens: 4096

  repeats: 5

  isolation:
    vector_index: idx@v1
    database_snapshot: postgres-snapshot@v1
    prompts: prompts@v1
    tools: tools@v1

  evaluation:
    deterministic: true
    trajectory: true
    rag_metrics: true
    llm_judge: true
    human_review: true
```

Provider-specific parameter support must be validated before execution.

If a parameter is unsupported or semantically different across providers, the platform records that difference rather than silently assuming equivalence.

---

# 4. Application Onboarding

## 4.1 Register RegDoc

The application team registers RegDoc with the platform.

```json
{
  "application_id": "regdoc",
  "application_name": "RegDoc",
  "framework": "langgraph",
  "application_version": "2.1",
  "integration_mode": "sdk",
  "environment": "preproduction",
  "data_classification": "confidential",
  "model_injection_supported": true
}
```

### 4.2 Register Agent Inventory

```json
{
  "agents": [
    {
      "name": "supervisor",
      "framework_node": "supervisor_node"
    },
    {
      "name": "retrieval_agent",
      "framework_node": "retrieval_node"
    },
    {
      "name": "spreadsheet_agent",
      "framework_node": "spreadsheet_node"
    },
    {
      "name": "sql_agent",
      "framework_node": "sql_node"
    },
    {
      "name": "compliance_agent",
      "framework_node": "compliance_node"
    },
    {
      "name": "comparison_agent",
      "framework_node": "comparison_node"
    },
    {
      "name": "writer_agent",
      "framework_node": "writer_node"
    }
  ]
}
```

The inventory identifies which framework node corresponds to which logical agent.

It also supports trajectory validation.

For example, if a SQL-based question requires the SQL Analyst but that agent never executes, the evaluator can flag the missing step.

---

# 5. SDK Integration and Trace Capture

The application integrates with a lightweight evaluation SDK.

Conceptually:

```python
from regdoc_eval import EvalTracer

tracer = EvalTracer(
    application_id="regdoc",
    environment="preproduction"
)

tracer.instrument_langgraph(graph)

result = graph.invoke(
    {"question": user_question},
    config={
        "callbacks": [tracer.callback_handler],
        "configurable": {
            "model": selected_model
        }
    }
)
```

This illustrates the desired SDK interface, not an existing production package.

The SDK must capture:

- Agent invocations
- Agent handoffs
- LLM requests and responses
- Tool calls and arguments
- Tool execution results
- Retrieval events
- SQL queries
- SQL execution errors
- Retry attempts
- Token utilization
- Model latency
- Cost
- Final application response

A crucial distinction is that **observability and model replacement are separate capabilities**.

Installing tracing callbacks does not automatically make it possible to replace GPT with Claude.

RegDoc must provide an explicit model-configuration seam, provider adapter, or separately configured application deployment.

If no supported replacement mechanism exists, the platform must report that the migration experiment cannot be executed without integration changes.

---

# 6. Building the RegDoc Evaluation Dataset

The application team owns dataset preparation.

## 6.1 Dataset Distribution

Assume 150 evaluation tasks.

| Capability | Number of Tasks |
|---|---:|
| Document chat and RAG | 40 |
| Compliance report generation | 35 |
| Document comparison | 25 |
| SQL-based analysis | 25 |
| Hybrid SQL and RAG analysis | 25 |
| **Total** | **150** |

The evaluation dataset must cover small, medium, and large document collections.

It should also include difficult SQL queries, ambiguous questions, missing evidence, malformed queries, and cases where the correct behavior is to refuse an unsupported conclusion.

## 6.2 Golden Dataset Structure

```json
{
  "task_id": "REGDOC-HYBRID-001",
  "category": "hybrid_sql_rag",
  "question": "What was the capital adequacy ratio for Bank A in Q3 2026, and was it compliant with the regulatory minimum specified in the uploaded policy?",

  "expected_answer": {
    "capital_adequacy_ratio": 12.1,
    "minimum_required": 10.5,
    "compliant": true
  },

  "expected_evidence": [
    {
      "source": "sql",
      "table": "capital_adequacy",
      "reporting_period": "2026-Q3"
    },
    {
      "source": "document",
      "file": "capital_policy.pdf",
      "page": 41
    }
  ],

  "expected_agents": [
    "supervisor",
    "retrieval_agent",
    "sql_agent",
    "writer_agent"
  ],

  "expected_tools": [
    "vector_search",
    "get_database_schema",
    "execute_readonly_sql"
  ]
}
```

The expected trajectory is optional.

When no expected trajectory is available, the platform can still measure:

- Tool schema validity
- SQL safety
- Execution errors
- Retries
- Redundant tool invocations
- Latency
- Cost
- Final-answer correctness

It cannot reliably claim that the exact agent sequence was correct without an appropriate reference specification or evaluator.

---

# 7. Working Example A: Hybrid RAG and SQL Question

## 7.1 User Question

> What was Bank A's capital adequacy ratio in Q3 2026, and did it satisfy the regulatory threshold mentioned in the uploaded policy document?

This question requires combining:

1. A numerical value from PostgreSQL.
2. A regulatory threshold from an uploaded PDF.
3. A compliance calculation.
4. Evidence citations.

## 7.2 Golden Answer

The reference data contains:

```text
Capital adequacy ratio: 12.1%

Regulatory minimum: 10.5%

Difference: +1.6 percentage points

Verdict: Compliant
```

Expected response:

> Bank A reported a capital adequacy ratio of 12.1% in Q3 2026. The applicable regulatory minimum was 10.5%, according to page 41 of capital_policy.pdf. Therefore, Bank A exceeded the requirement by 1.6 percentage points and was compliant.

---

# 8. Baseline GPT Execution

## 8.1 Captured Agent Trajectory

| Step | Agent | Operation | Output |
|---|---|---|---|
| S1 | Supervisor | Classify request | Hybrid SQL + document question |
| S2 | Supervisor | Handoff | SQL Agent |
| S3 | SQL Agent | `get_database_schema` | Schema returned |
| S4 | SQL Agent | Generate SQL | Valid SELECT query |
| S5 | SQL Agent | `execute_readonly_sql` | Ratio = 12.1 |
| S6 | Supervisor | Handoff | Retrieval Agent |
| S7 | Retrieval Agent | `vector_search` | Relevant policy chunks |
| S8 | Retrieval Agent | `get_pdf_page` | Page 41 |
| S9 | Retrieval Agent | Extract threshold | 10.5 |
| S10 | Supervisor | Handoff | Writer Agent |
| S11 | Writer Agent | Generate final answer | Correct compliance verdict |

## 8.2 SQL Generated by GPT

```sql
SELECT
    reported_car_ratio
FROM capital_adequacy
WHERE entity_id = 101
  AND reporting_period = '2026-Q3';
```

Tool execution:

```json
{
  "tool_name": "execute_readonly_sql",
  "arguments": {
    "query": "SELECT reported_car_ratio FROM capital_adequacy WHERE entity_id = 101 AND reporting_period = '2026-Q3'"
  },
  "result": {
    "rows": [
      {
        "reported_car_ratio": 12.1
      }
    ],
    "row_count": 1,
    "status": "success"
  }
}
```

## 8.3 RAG Retrieval

```json
{
  "tool": "vector_search",
  "arguments": {
    "query": "minimum capital adequacy ratio regulatory requirement",
    "document_filter": "capital_policy.pdf",
    "top_k": 5
  }
}
```

Retrieved evidence:

```json
{
  "document": "capital_policy.pdf",
  "page": 41,
  "text": "The minimum required capital adequacy ratio is 10.5%."
}
```

## 8.4 Final Answer

GPT generates:

> Bank A's capital adequacy ratio for Q3 2026 was 12.1%. The regulatory minimum was 10.5%. Therefore, Bank A complied with the requirement, exceeding the minimum by 1.6 percentage points.

All required evidence is correctly referenced.

---

# 9. Claude Execution: Same Task

Claude receives the same user question, tool definitions, database snapshot, vector index, and application prompts.

Assume Claude performs the following actions.

| Step | Agent | Operation | Result |
|---|---|---|---|
| S1 | Supervisor | Classify | Hybrid question |
| S2 | SQL Agent | Inspect schema | Success |
| S3 | SQL Agent | Generate SQL | Success |
| S4 | SQL Agent | Execute SQL | 12.1 |
| S5 | Retrieval Agent | Search policy | Success |
| S6 | Retrieval Agent | Read page | 10.5 |
| S7 | Writer Agent | Compose answer | Correct |

The generated SQL is:

```sql
SELECT
    reported_car_ratio
FROM capital_adequacy
WHERE reporting_period = '2026-Q3'
  AND entity_id = 101
LIMIT 1;
```

The query is syntactically different from GPT's SQL but produces the same result.

**Important evaluation principle: SQL correctness should not be based on exact string equality.**

Equivalent SQL statements may differ in:

- Predicate ordering
- Aliases
- Whitespace
- Common table expressions
- Join structure
- Subqueries

The evaluator should check whether the query is permitted, logically correct, and returns the expected result against the frozen database.

---

# 10. Gemini Execution: Same Task

Assume Gemini makes an error when generating SQL.

## 10.1 Initial SQL

```sql
SELECT
    capital_ratio
FROM capital_adequacy
WHERE entity_id = 101
  AND reporting_period = '2026-Q3';
```

The `capital_ratio` column does not exist.

Tool response:

```json
{
  "status": "error",
  "error_type": "UndefinedColumn",
  "message": "column capital_ratio does not exist"
}
```

Gemini receives the tool error and generates a corrected query.

```sql
SELECT
    reported_car_ratio
FROM capital_adequacy
WHERE entity_id = 101
  AND reporting_period = '2026-Q3';
```

The second query succeeds.

Gemini retrieves the regulatory minimum from the policy PDF and produces the correct final answer.

## 10.2 Gemini Trajectory

| Step | Agent | Operation | Result |
|---|---|---|---|
| S1 | Supervisor | Classification | Correct |
| S2 | SQL Agent | Schema inspection | Correct |
| S3 | SQL Agent | Generate SQL | Invalid column |
| S4 | SQL Agent | Execute SQL | Error |
| S5 | SQL Agent | Interpret error | Retry |
| S6 | SQL Agent | Correct SQL | Valid |
| S7 | SQL Agent | Execute SQL | 12.1 |
| S8 | Retrieval Agent | Vector search | Correct |
| S9 | Retrieval Agent | Extract threshold | 10.5 |
| S10 | Writer Agent | Final response | Correct |

### Why This Matters

All three models answer the user correctly.

A final-answer evaluator would give each model a pass.

However, trajectory evaluation reveals that Gemini:

- Generated a SQL query referencing a nonexistent column.
- Caused a database execution error.
- Required an additional LLM invocation.
- Required an additional SQL tool call.
- Incurred additional latency and cost.

This demonstrates why complete trajectory evaluation provides information that final-answer evaluation misses.

---

# 11. Canonical Trace Representation

Framework-specific traces must be normalized into one common format.

For example:

```json
{
  "experiment_id": "regdoc-exp-001",
  "task_id": "REGDOC-HYBRID-001",
  "run_id": "gemini-run-001",
  "model": "gemini-candidate",
  "framework": "langgraph",
  "fidelity": "full_trajectory",

  "spans": [
    {
      "span_id": "span-001",
      "parent_span_id": null,
      "agent": "supervisor",
      "span_type": "agent",
      "operation": "classify_request"
    },
    {
      "span_id": "span-002",
      "parent_span_id": "span-001",
      "agent": "sql_agent",
      "span_type": "tool_call",
      "tool_name": "execute_readonly_sql",
      "arguments": {
        "query": "SELECT capital_ratio FROM capital_adequacy WHERE entity_id = 101"
      },
      "status": "error",
      "error_type": "UndefinedColumn"
    },
    {
      "span_id": "span-003",
      "parent_span_id": "span-001",
      "agent": "sql_agent",
      "span_type": "tool_call",
      "tool_name": "execute_readonly_sql",
      "retry_of": "span-002",
      "arguments": {
        "query": "SELECT reported_car_ratio FROM capital_adequacy WHERE entity_id = 101"
      },
      "status": "success"
    }
  ]
}
```

This is an abbreviated example.

A production trace should additionally include:

- Trace ID and parent IDs
- Start and end timestamps
- Framework-specific event metadata
- Agent and model identity
- Model parameters
- Prompt and skill versions
- Inputs and outputs or secure payload references
- Token counts
- Cache usage
- Exclusive and inclusive cost
- Errors and retry linkage
- Tool-result hashes
- Context utilization
- Correlation IDs

Sensitive payloads should remain within their authorized data boundary.

---

# 12. Evaluating the Hybrid SQL + RAG Task

## 12.1 Final Answer Metrics

| Metric | GPT | Claude | Gemini |
|---|---|---|---|
| Numerical answer correct | Pass | Pass | Pass |
| Regulatory threshold correct | Pass | Pass | Pass |
| Compliance verdict correct | Pass | Pass | Pass |
| Evidence citation correct | Pass | Pass | Pass |
| Final-answer outcome | Pass | Pass | Pass |

## 12.2 Trajectory Metrics

| Metric | GPT | Claude | Gemini |
|---|---|---|---|
| Correct agent selection | Pass | Pass | Pass |
| Correct tool selection | Pass | Pass | Pass |
| SQL syntax validity | Pass | Pass | Pass |
| SQL schema validity on first attempt | Pass | Pass | Fail |
| SQL execution success on first attempt | Pass | Pass | Fail |
| Unnecessary SQL retries | 0 | 0 | 1 |
| Final SQL result correct | Pass | Pass | Pass |
| Evidence retrieved | Pass | Pass | Pass |
| Final answer correct | Pass | Pass | Pass |
| Clean execution | Pass | Pass | Fail |

Here, SQL syntax validity and SQL schema validity are separate metrics.

Gemini's first query is syntactically valid SQL, but it references a nonexistent column.

An evaluator that checks only SQL syntax would miss this failure.

## 12.3 SQL Evaluation Techniques

The evaluation platform should support multiple levels of SQL evaluation.

### Level 1: Syntax Validation

Can the query be parsed?

Example:

```sql
SELECT FROM capital_adequacy;
```

Result: Invalid syntax.

### Level 2: Schema Validation

Do referenced tables and columns exist?

Example:

```sql
SELECT capital_ratio
FROM capital_adequacy;
```

Result: Invalid column reference.

### Level 3: Security Validation

Is the query permitted?

Example:

```sql
DELETE FROM capital_adequacy;
```

Result: Rejected.

The evaluation environment should enforce read-only database credentials, approved schemas, query timeouts, row limits, and SQL validation independently of the LLM.

### Level 4: Execution Correctness

Does the query execute successfully against the frozen database?

### Level 5: Result Correctness

Does the execution result match the reference result?

### Level 6: Semantic Correctness

Does the SQL represent the user's intended question?

For example, a query may return a plausible value but apply the wrong reporting period.

### Level 7: Efficiency

Measure:

- Database execution latency
- Rows scanned
- Query execution plan
- Number of tool calls
- Retry count
- Redundant queries

Not every SQL query needs an exact reference SQL statement.

For many tasks, result comparison and semantic constraints are more robust than string matching.

---

# 13. Working Example B: Compliance Report Generation

## 13.1 User Request

> Generate a regulatory compliance report for Bank A using the uploaded checklist, financial workbook, policy document, and SQL compliance findings database.

This task combines:

1. Checklist extraction.
2. Evidence retrieval from PDF files.
3. Financial data from Excel.
4. Compliance observations from SQL.
5. Per-item regulatory judgment.
6. Report writing.

## 13.2 Expected Execution

```text
                     USER REQUEST
                           |
                           v
                    SUPERVISOR AGENT
                           |
              +------------+-------------+
              |            |             |
              v            v             v
         COMPLIANCE     RETRIEVAL     SQL AGENT
           AGENT          AGENT
              |            |             |
              v            v             v
         CHECKLIST     POLICY PDF     DATABASE
              |            |             |
              +------------+-------------+
                           |
                           v
                   SPREADSHEET AGENT
                           |
                           v
                     FINANCIAL DATA
                           |
                           v
                    COMPLIANCE AGENT
                           |
                           v
                     ITEM VERDICTS
                           |
                           v
                       WRITER
                           |
                           v
                   COMPLIANCE REPORT
```

## 13.3 SQL Tool Example

The Compliance Agent asks the SQL Analyst to retrieve critical open findings.

```sql
SELECT
    finding_id,
    regulation_id,
    severity,
    finding_status
FROM compliance_findings
WHERE entity_id = 101
  AND finding_status = 'OPEN'
  AND severity = 'CRITICAL';
```

Suppose the result contains three critical findings.

The Compliance Agent must incorporate these into its assessment.

A model that produces a polished report but ignores those findings should fail relevant compliance checks.

## 13.4 Expected Compliance Output

Assume a checklist contains 25 items, including five critical items.

The reference report includes:

```json
{
  "entity": "Bank A",
  "reporting_period": "2026-Q3",
  "checklist_items": 25,
  "critical_items": 5,
  "expected_critical_verdicts": {
    "C01": "compliant",
    "C02": "non_compliant",
    "C03": "compliant",
    "C04": "non_compliant",
    "C05": "compliant"
  }
}
```

## 13.5 Compliance Evaluation

Suppose Claude generates:

| Evaluation Dimension | Result |
|---|---:|
| Correct checklist verdicts | 22 / 25 |
| Correct critical verdicts | 4 / 5 |
| Correct citations among correct verdicts | 20 / 22 |
| Skipped checklist items | 0 |
| Incorrect SQL tool calls | 1 |

The Admin configures the following acceptance criteria:

```yaml
compliance_success:
  minimum_item_accuracy: 0.90
  require_all_critical_items_correct: true
  require_evidence_citations: true
```

Claude's item accuracy is:

```text
22 / 25 = 88%
```

The compliance report fails because:

1. Overall accuracy is below 90%.
2. One critical item is incorrect.

This illustrates why the platform needs task-specific success criteria.

A generic response-quality score is insufficient for regulatory reporting.

---

# 14. Working Example C: Document Comparison

## 14.1 User Request

> Compare Policy Version 1 and Policy Version 2 and identify all changes affecting regulatory capital requirements.

For this task, the application team prepares a reference set with 12 known discrepancies.

The differences include:

- Changed numerical thresholds.
- Removed clauses.
- Modified effective dates.
- Updated definitions.
- Changes in exception handling.
- Altered reporting requirements.

## 14.2 Expected Workflow

```text
            USER REQUEST
                  |
                  v
             SUPERVISOR
                  |
                  v
           COMPARISON AGENT
                  |
         +--------+--------+
         |                 |
         v                 v
      POLICY V1         POLICY V2
         |                 |
         +--------+--------+
                  |
                  v
            ALIGN SECTIONS
                  |
                  v
           COMPARE CONTENT
                  |
                  v
        IDENTIFY DISCREPANCIES
                  |
                  v
          VALIDATE EVIDENCE
                  |
                  v
             FINAL REPORT
```

## 14.3 Illustrative Model Result

Suppose Gemini identifies 11 discrepancies.

Nine are genuine discrepancies.

There are 12 discrepancies in the reference set.

```text
True positives = 9

False positives = 2

False negatives = 3
```

Precision:

```text
9 / 11 = 0.818
```

Recall:

```text
9 / 12 = 0.750
```

F1:

```text
2 × Precision × Recall / (Precision + Recall)

F1 ≈ 0.783
```

Additional evaluation dimensions should include:

- Correct document section alignment.
- Correct page and paragraph references.
- Whether numerical changes were preserved.
- Whether material regulatory changes were missed.
- Whether false discrepancies were introduced.
- Whether the agent unnecessarily repeated retrieval.

A high-level answer-quality judge alone cannot reliably measure discrepancy coverage.

---

# 15. Working Example D: SQL-Only Analytical Question

## 15.1 User Question

> How many critical compliance findings remained open for Bank A in Q3 2026?

This task requires SQL analysis without document retrieval.

The Supervisor should select the SQL Analyst.

A vector-search call is unnecessary unless RegDoc's application logic explicitly requires supplementary document evidence.

## 15.2 Reference SQL

```sql
SELECT COUNT(*) AS critical_open_findings
FROM compliance_findings
WHERE entity_id = 101
  AND severity = 'CRITICAL'
  AND finding_status = 'OPEN'
  AND observation_date >= DATE '2026-07-01'
  AND observation_date < DATE '2026-10-01';
```

Assume the reference result is:

```json
{
  "critical_open_findings": 3
}
```

## 15.3 Candidate Behavior

| Behavior | GPT | Claude | Gemini |
|---|---|---|---|
| Select SQL Agent | Yes | Yes | Yes |
| Generate valid SQL | Yes | Yes | Yes |
| Apply correct date filter | Yes | Yes | No |
| Execute successfully | Yes | Yes | Yes |
| Return correct count | 3 | 3 | 7 |

Here Gemini generates syntactically valid SQL, but omits the reporting-period filter.

This is a semantic SQL error rather than a syntax or execution error.

The SQL tool itself reports success.

Only result-based evaluation against the reference data detects the wrong answer.

This is why SQL validation should evaluate both **execution validity** and **query intent correctness**.

---

# 16. RAG Evaluation Metrics

The platform should support RAG-specific evaluation independently of SQL evaluation.

| Metric | Question |
|---|---|
| Faithfulness | Is the answer supported by retrieved evidence? |
| Answer relevance | Does the response address the user's question? |
| Context precision | How much of the retrieved context is relevant? |
| Context recall | Was the necessary reference evidence retrieved? |
| Citation accuracy | Does each citation point to supporting content? |
| Retrieval hit rate | Was relevant evidence present among retrieved results? |
| Groundedness | Are factual claims supported by available sources? |

For RegDoc, citation validation should support:

```text
PDF:
    file + page + text span

PowerPoint:
    file + slide + object/section

Excel:
    workbook + worksheet + cell range

SQL:
    snapshot + query + result + source table
```

A citation to an irrelevant page should not pass simply because the document title is correct.

---

# 17. Agentic Evaluation Metrics

Agent trajectories introduce additional evaluation dimensions.

| Metric | Definition |
|---|---|
| Agent selection accuracy | Whether appropriate agents executed |
| Handoff correctness | Whether delegation went to the correct agent |
| Tool selection precision | Relevant tool calls divided by all tool calls |
| Tool selection recall | Required tool calls successfully covered |
| Tool argument accuracy | Correctness of supplied arguments |
| Tool schema validity | Whether arguments satisfy tool schemas |
| SQL execution validity | Whether SQL calls execute successfully |
| SQL semantic accuracy | Whether SQL represents the intended question |
| Retry rate | Number of retries per task |
| Redundant calls | Unnecessary repeated operations |
| Step efficiency | Work performed relative to a suitable reference |
| Context utilization | Peak context consumption |
| Recovery success | Whether a failed step is subsequently recovered |
| Clean success | Successful result without disallowed trajectory violations |

Care is needed with trajectory matching.

There can be multiple valid paths to the same answer.

For example:

```text
Path A:
SQL Agent -> Retrieval Agent -> Writer

Path B:
Retrieval Agent -> SQL Agent -> Writer
```

Both paths may be acceptable.

Therefore, the expected trajectory should support partial ordering rather than requiring one exact sequence unless the application genuinely requires it.

---

# 18. Evaluation Execution at Scale

Assume:

```text
150 evaluation tasks

3 models:
    GPT
    Claude
    Gemini

5 repeated runs per task
```

Total application executions:

```text
150 × 3 × 5 = 2,250
```

Each execution runs the complete RegDoc agent workflow.

A single application execution may generate multiple:

- Agent spans
- LLM calls
- Database queries
- Retrieval operations
- Tool calls
- Retry spans

Therefore, 2,250 application executions do not mean 2,250 LLM API calls.

The number of LLM calls may be substantially higher.

The platform should record total cost from all applicable operations.

## 18.1 Worker Execution

```python
for model in candidate_models:

    for task in evaluation_dataset:

        for repeat in range(5):

            result = run_regdoc_application(
                model=model,
                task=task,
                frozen_environment=environment
            )

            canonical_trace = normalize_trace(
                result.trace
            )

            scores = evaluate(
                task=task,
                result=result,
                trace=canonical_trace
            )

            store_result(
                model=model,
                task=task,
                repeat=repeat,
                trace=canonical_trace,
                scores=scores
            )
```

This is simplified orchestration pseudocode.

A real implementation should distribute jobs across workers, preserve task isolation, control provider-specific concurrency, retry infrastructure failures, and maintain immutable execution records.

---

# 19. Cost Evaluation

For one run:

```text
Total Cost =
    LLM inference cost
  + paid tool/API cost
  + allocated infrastructure cost
```

The platform should distinguish provider-billed cost from infrastructure estimates.

### 19.1 Cost Per Successful Task

```text
Cost Per Successful Task =
    Total Cost of All Runs
    ----------------------
    Number of Successful Runs
```

Failed executions remain in the numerator.

Example:

| Metric | GPT | Claude | Gemini |
|---|---:|---:|---:|
| Average cost per run | $0.11 | $0.13 | $0.06 |
| Outcome success | 78% | 77% | 72% |
| Approx. cost per successful task | $0.141 | $0.169 | $0.083 |

These are illustrative values.

A model with a lower cost per run may be less economical if it requires frequent retries or produces unusable outputs.

### 19.2 SQL-Specific Cost Attribution

The platform should separately capture:

```text
SQL Agent:
    LLM query-generation cost
    LLM retry cost
    SQL execution cost
    Database execution latency
```

For a tool call that fails and requires regeneration, both the failed operation and recovery steps count toward the total task cost.

To avoid double counting, aggregate exclusive span costs across the trace.

---

# 20. Repeated Runs and Reliability

Running a task once does not measure consistency.

Suppose one task is executed five times.

| Model | Successful Runs |
|---|---:|
| GPT | 5/5 |
| Claude | 4/5 |
| Gemini | 3/5 |

We can calculate two complementary metrics.

## 20.1 pass@k

Measures the probability that at least one of k selected attempts succeeds.

For n observed runs and c successes:

```text
pass@k = 1 - C(n-c, k) / C(n, k)
```

## 20.2 pass^k

Measures the probability that all k selected attempts succeed.

```text
pass^k = C(c, k) / C(n, k)
```

For k = 3:

| Model | pass@3 | pass^3 |
|---|---:|---:|
| GPT | 1.00 | 1.00 |
| Claude | 1.00 | 0.40 |
| Gemini | 1.00 | 0.10 |

For these observed results, every model has a perfect pass@3 estimate because at least one success is guaranteed among any three selected attempts.

But their consistency differs substantially.

For compliance workflows, this distinction is important.

The platform should report both capability and repeatability.

---

# 21. Aggregated Model Comparison

Assume the platform completes the illustrative experiment.

## 21.1 Overall Results

| Metric | GPT Baseline | Claude | Gemini |
|---|---:|---:|---:|
| Outcome success | 78% | 77% | 72% |
| Clean success | 66% | 70% | 58% |
| Tool argument validity | 94% | 97% | 89% |
| Average retries per task | 0.8 | 0.4 | 1.3 |
| Average cost per run | $0.11 | $0.13 | $0.06 |
| Cost per successful task | $0.141 | $0.169 | $0.083 |
| p95 latency | 21 sec | 26 sec | 14 sec |

## 21.2 Results by Capability

| Capability | GPT | Claude | Gemini |
|---|---:|---:|---:|
| Document RAG success | 84% | 83% | 80% |
| Compliance report success | 74% | 76% | 68% |
| Document comparison F1 | 0.74 | 0.71 | 0.60 |
| SQL analysis success | 86% | 87% | 73% |
| Hybrid SQL + RAG success | 76% | 78% | 65% |

These are hypothetical capability-level results, not measured observations.

For the SQL Agent, detailed trace analysis might reveal that Gemini produces more:

- Missing-column queries.
- Incorrect date filters.
- Unnecessary joins.
- Query retries.
- Incorrect aggregation logic.

For the Compliance Agent, the platform might reveal that Claude produces more consistent critical-item judgments.

Those interpretations must be validated against captured traces rather than inferred from aggregate scores alone.

---

# 22. Statistical Comparison

The dataset contains repeated measurements of the same tasks across models.

These observations are not independent.

For a fair comparison, use paired task-level analysis.

## 22.1 Paired Bootstrap

For each task:

```text
Task-level difference =
    Candidate success rate
  - GPT success rate
```

Resample tasks rather than individual repeated runs.

This preserves the correlation structure of repeated executions.

## 22.2 Non-Inferiority Margin

Assume the execution team defines:

```text
Allowed quality degradation = 3 percentage points
```

The platform tests whether a candidate can be considered non-inferior to GPT within that margin.

Example:

| Comparison | Estimated Difference | 95% CI | Interpretation |
|---|---:|---|---|
| Claude - GPT | -1.0 pt | [-4.2, +2.3] | Non-inferiority not established |
| Gemini - GPT | -6.0 pts | [-10.1, -1.8] | Inferior at the 3-point margin |

These hypothetical results must be interpreted with the specified statistical procedure and multiple-comparison handling.

A point estimate close to GPT does not prove equivalence.

If the confidence interval is too wide, the platform should report an inconclusive result.

---

# 23. Multi-Objective Decision Making

Quality is not the only migration consideration.

The decision owner may care about:

1. Outcome correctness.
2. Clean execution reliability.
3. Cost per successful task.
4. End-to-end latency.
5. Data residency.
6. SQL reliability.
7. Regulatory compliance accuracy.

## 23.1 Pareto Analysis

Consider three dimensions:

```text
Maximize:
    Clean success rate

Minimize:
    Cost per successful task
    p95 latency
```

Illustrative values:

| Model | Clean Success | Cost Per Success | p95 Latency |
|---|---:|---:|---:|
| GPT | 66% | $0.141 | 21 sec |
| Claude | 70% | $0.169 | 26 sec |
| Gemini | 58% | $0.083 | 14 sec |

No model dominates another across all three dimensions.

Claude provides better clean success but has higher cost and latency.

Gemini offers lower cost and latency but reduced reliability.

GPT occupies an intermediate position.

All three lie on the Pareto frontier in this example.

The platform should display these trade-offs instead of hiding them behind one weighted score.

---

# 24. Trace-Level Comparison

The most valuable output is not simply:

```text
GPT: 78%

Claude: 77%

Gemini: 72%
```

The platform should also explain where execution paths diverged.

## 24.1 Example Trace Diff

```text
TASK: REGDOC-HYBRID-001

EXPECTED:
    SQL retrieval
    Document retrieval
    Compliance calculation
    Final answer


GPT:
    SQL Agent
      -> Valid SQL
      -> Successful execution

    Retrieval Agent
      -> Correct evidence

    Writer Agent
      -> Correct answer


CLAUDE:
    SQL Agent
      -> Valid SQL
      -> Successful execution

    Retrieval Agent
      -> Correct evidence

    Writer Agent
      -> Correct answer


GEMINI:
    SQL Agent
      -> Invalid column reference
      -> SQL execution error
      -> Retry
      -> Correct SQL
      -> Successful execution

    Retrieval Agent
      -> Correct evidence

    Writer Agent
      -> Correct answer
```

## 24.2 Diagnostic Finding

```text
Finding ID: SQL-TOOL-004

Affected Model:
    Gemini

Issue:
    Frequent SQL schema-reference errors

Observed Consequence:
    More SQL retries

Potential Impact:
    Higher latency
    Additional inference cost
    Greater workflow fragility

Recommendation:
    Review SQL Agent instructions,
    schema grounding, and tool argument constraints.

Confidence:
    Requires aggregation over the SQL evaluation subset.
```

This supports RQ3: trajectory analysis can detect regressions that final-answer evaluation misses.

---

# 25. Experiment 2: Manual Prompt Adaptation

Experiment 1 evaluated an unchanged prompt.

Suppose the Admin finds that Gemini's SQL errors frequently involve incorrect column names.

The Admin creates a new SQL Agent prompt version.

## 25.1 Original Prompt

```text
Generate SQL to answer the user question.

Use the available database tools.
```

## 25.2 Revised Prompt

```text
You are a SQL Analyst Agent.

Before generating a SQL query:

1. Inspect the available database schema.
2. Use only existing tables and columns.
3. Use explicit reporting-period filters.
4. Prefer parameterized queries where supported.
5. Generate read-only SQL.
6. Never invent column names.
7. Verify column references before tool invocation.

If a query fails:
    Inspect the tool error.
    Correct the query.
    Retry only when appropriate.
```

The Admin creates a new experiment:

```yaml
experiment:
  id: regdoc-exp-002

  parent_experiment: regdoc-exp-001

  target_model: gemini-candidate

  prompt_version: sql-agent-prompt@v2

  dataset: regdoc-golden@v1

  database_snapshot: postgres-snapshot@v1

  vector_index: idx@v1
```

This experiment measures how much prompt adaptation improves Gemini.

## 25.3 Why Separate Experiments?

Experiment 1 answers:

> How well does Gemini work as a drop-in replacement with unchanged application prompts?

Experiment 2 answers:

> How much can Gemini improve through manual prompt adaptation?

These are different questions.

The platform must preserve both results rather than overwriting Experiment 1.

If a shared prompt changes for multiple models, the affected baseline and candidate configurations should be rerun under the same new prompt condition.

Automated prompt optimization is outside the present scope.

---

# 26. Human Evaluation

Some regulatory assessments require expert review.

For example:

> Does the explanation adequately justify the compliance verdict?

An LLM judge may provide a useful preliminary score, but subjective and high-stakes regulatory interpretations require human validation.

Human reviewers should inspect:

- Final report content.
- Supporting citations.
- Critical checklist verdicts.
- SQL-derived evidence.
- Agent execution trajectory, when available.
- Disagreements between deterministic metrics and LLM judges.

Example human-review output:

```json
{
  "task_id": "REGDOC-COMPLIANCE-017",
  "model": "claude-candidate",

  "review": {
    "regulatory_accuracy": 4,
    "evidence_quality": 5,
    "completeness": 3,
    "overall_acceptable": false
  },

  "reason": "A critical exception was omitted from the report."
}
```

The platform can compare automatic evaluator results against sampled human judgments to identify unreliable scoring patterns.

---

# 27. RegDoc-Specific Engineering Safeguards

## 27.1 SQL Safety

Use:

- Read-only database credentials.
- Isolated database snapshots.
- Query validation.
- Query timeouts.
- Resource limits.
- Approved schema access.
- Row-level authorization where applicable.

Never depend solely on an LLM instruction to prevent unsafe database operations.

## 27.2 Retrieval Consistency

Freeze:

- Document parsing outputs.
- Chunk boundaries.
- Embedding versions.
- Vector index contents.
- Retrieval tool configuration.

Validate that the reference evidence actually exists in the frozen corpus.

## 27.3 Tool Response Replay

For deterministic evaluation, external tool responses may be recorded and replayed.

For SQL workflows, use database snapshots when the generated query itself is under evaluation.

Replaying only a fixed SQL response would hide query-generation errors.

## 27.4 Model Injection

The framework must distinguish:

```text
Trace Capture:
    Observe application behavior

Model Injection:
    Replace the LLM used by the application
```

A tracing SDK alone cannot guarantee that every application supports model replacement.

RegDoc must expose a supported model configuration or provider abstraction.

## 27.5 Confidentiality

Full traces may contain regulated information.

Apply:

- Encryption.
- Redaction.
- Access controls.
- Secure payload references.
- Regional worker placement.
- Audit logging.
- Retention policies.

Only capture intermediate reasoning content that the application or model API explicitly exposes. Internal model reasoning that is not available through the interface cannot be reconstructed from traces.

---

# 28. End-to-End Evaluation Workflow

```text
                      APPLICATION TEAM
                              |
                              v
                       REGISTER REGDOC
                              |
                              v
                       INTEGRATE SDK
                              |
                              v
                     CAPTURE SMOKE TRACE
                              |
                              v
                 VERIFY MODEL INJECTION
                              |
                              v
                       UPLOAD DATASET
                              |
                              v
                   FREEZE ENVIRONMENT
                              |
                  +-----------+-----------+
                  |                       |
                  v                       v
             VECTOR INDEX            SQL SNAPSHOT
                  |                       |
                  +-----------+-----------+
                              |
                              v
                       EXECUTION TEAM
                              |
                              v
                     APPLY MODEL GATES
                              |
                              v
                    SELECT CANDIDATES
                              |
                              v
                            ADMIN
                              |
                              v
                   CONFIGURE EXPERIMENT
                              |
                              v
                     FREEZE EXPERIMENT
                              |
                              v
                    EXPERIMENT SCHEDULER
                              |
           +------------------+------------------+
           |                  |                  |
           v                  v                  v
       GPT WORKER        CLAUDE WORKER      GEMINI WORKER
           |                  |                  |
           v                  v                  v
       RUN REGDOC         RUN REGDOC         RUN REGDOC
           |                  |                  |
           v                  v                  v
      CAPTURE TRACE      CAPTURE TRACE      CAPTURE TRACE
           |                  |                  |
           +------------------+------------------+
                              |
                              v
                      TRACE NORMALIZER
                              |
                              v
                     CANONICAL TRAJECTORY
                              |
                              v
                      EVALUATOR ENGINE
                              |
         +-------------+------+------+-------------+
         |             |             |             |
         v             v             v             v
    DETERMINISTIC    SQL EVAL     RAG EVAL     AGENT EVAL
         |             |             |             |
         +-------------+------+------+-------------+
                              |
                              v
                       LLM-AS-JUDGE
                              |
                              v
                       HUMAN REVIEW
                              |
                              v
                        AGGREGATION
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          QUALITY            COST            LATENCY
             |                |                |
             +----------------+----------------+
                              |
                              v
                    STATISTICAL ANALYSIS
                              |
                              v
                       PARETO ANALYSIS
                              |
                              v
                    TRAJECTORY DIFFS
                              |
                              v
                       FINAL REPORT
                              |
                              v
                         ADMIN REVIEW
                              |
                +-------------+-------------+
                |                           |
                v                           v
           REVISE PROMPTS             HAND OFF REPORT
                |                     TO DECISION OWNER
                v
         CREATE EXPERIMENT V2
                |
                v
           RE-EVALUATE
```

---

# 29. Mapping the Example to Research Questions

## RQ1: Framework-Neutral Trajectory Representation

**Research question:**

Can one canonical execution-trace schema represent trajectories from different agent frameworks sufficiently well to support common evaluation?

RegDoc initially uses LangGraph.

The platform converts LangGraph agent and tool events into a canonical trace.

To validate framework neutrality, equivalent agent workflows should also be tested using other supported frameworks, such as Google ADK and Microsoft Agent Framework.

The same evaluator implementations should run on their canonical traces without framework-specific changes.

Potential measurement dimensions:

- Span mapping coverage.
- Agent attribution completeness.
- Tool argument preservation.
- Parent-child relationship preservation.
- Handoff preservation.
- Normalization failures.

One LangGraph application alone is not enough to validate the cross-framework hypothesis.

## RQ3: Trajectory Evaluation Versus Final-Answer Evaluation

**Research question:**

Can trajectory-level evaluation identify migration regressions that final-answer evaluation fails to detect?

The Gemini SQL example illustrates the hypothesis.

Gemini returns the correct final answer but initially uses an invalid database column.

Final-answer evaluation:

```text
PASS
```

Trajectory evaluation:

```text
Final answer:
    PASS

SQL first-attempt validity:
    FAIL

Retry efficiency:
    FAIL

Clean success:
    FAIL
```

The experiment should measure how frequently this pattern occurs across the evaluation dataset.

## RQ5: Multi-Objective Model Migration Decisions

**Research question:**

Does multi-objective reporting improve migration decisions compared with a single quality score?

The platform reports:

- Outcome success.
- Clean success.
- SQL and RAG reliability.
- Cost per successful task.
- Latency.
- Statistical uncertainty.
- Pareto-optimal candidates.

Decision-makers can assess competing trade-offs.

The study can compare decisions made using a single weighted score with decisions made using the richer multi-objective report.

The example motivates RQ5 but does not, by itself, prove improved decision quality.

---

# 30. Final Expected Output of the Evaluation Platform

The platform generates a model migration assessment containing:

## A. Executive Comparison

```text
Application:
    RegDoc

Baseline:
    GPT

Candidates:
    Claude
    Gemini

Dataset:
    regdoc-golden@v1

Execution Count:
    2,250 application runs

Experiment:
    regdoc-exp-001
```

## B. Quality Assessment

```text
Compare:

    Final answer quality
    Clean success
    RAG faithfulness
    Citation accuracy
    SQL query correctness
    Tool argument validity
    Agent execution reliability
    Compliance report correctness
    Document comparison F1
```

## C. Operational Assessment

```text
Compare:

    Average cost per run
    Cost per successful task
    Retry cost
    p50 latency
    p95 latency
    Context utilization
    SQL execution overhead
    Provider failure rate
```

## D. Migration Risks

```text
Identify:

    SQL schema errors
    Incorrect tool selection
    Invalid tool arguments
    Missing agent handoffs
    Incomplete retrieval
    Missed regulatory evidence
    Incorrect checklist judgments
    Excessive retries
    Context-window failures
```

## E. Statistical Evidence

```text
Include:

    Paired model differences
    Confidence intervals
    Non-inferiority analysis
    Repeatability
    pass@k
    pass^k
```

## F. Multi-Objective Comparison

```text
Show:

    Quality vs. cost
    Quality vs. latency
    Cost vs. reliability
    Pareto frontier
    Non-dominated candidates
```

## G. Informational Recommendation

For this illustrative experiment:

**Claude**

A plausible candidate for compliance and hybrid SQL/RAG workloads, with stronger clean execution but higher cost and latency. Statistical non-inferiority remains unproven in the hypothetical results.

**Gemini**

Potentially attractive for cost-sensitive workloads, but its illustrative SQL reliability and large-document weaknesses warrant further investigation before migration.

**GPT**

Remains the baseline, with intermediate cost and reliability characteristics.

No automatic application migration is performed.

The decision owner evaluates the report and decides whether a model replacement is appropriate.

---

# 31. Key Architectural Insight

The evaluation platform should not treat RegDoc as a single text-generation endpoint.

It must understand the application as an observable execution system:

```text
User Request
    |
    v
Agent Decisions
    |
    v
Tool Selection
    |
    v
Tool Arguments
    |
    v
Tool Execution
    |
    v
Retrieved Evidence
    |
    v
Intermediate Outputs
    |
    v
Final Response
```

Changing the LLM can affect every stage.

A successful migration therefore means more than generating a correct-looking final answer.

The candidate model must demonstrate that it can operate the application's agents and tools reliably, produce grounded outputs, satisfy domain-specific requirements, and deliver an acceptable cost-latency profile.

**The central principle is to evaluate the complete application behavior under a controlled model substitution, then make the quality, reliability, cost, and latency trade-offs explicit.**
  
