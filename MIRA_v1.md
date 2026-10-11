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

The platform is informational only: it produces evidence-backed trade-off statements and never ranks models, deploys, optimizes prompts automatically, or touches production traffic.

## 2. Problem, Motivation, and the Why / What / How

| Question | Answer |
| --- | --- |
| What are we building? | An evaluation platform that compares candidate LLMs on complete agentic applications, using full execution trajectories. |
| Why is it needed? | Model swaps break agents in non-obvious ways (wrong tool, malformed arguments, broken handoffs) even when the final answer looks fine. Each team also measures differently, so results can’t be compared. |
| How will we do it? | A canonical trace schema, two-axis integration (injection mode and fidelity level), a control-plane/execution-plane split, a plugin-based evaluator registry, and multi-objective reporting. |

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

Dropped from the research scope: fidelity-tier comparison (RQ2) and judge calibration (RQ4). The platform still uses tiers and judges as engineering features, and the judge’s reliability stays listed as a threat to validity (§14). Study protocols for RQ1, RQ3 and RQ5 are in §13.

## 4. Scope, Assumptions, Constraints

### 4.1 Scope

| In scope | Out of scope |
| --- | --- |
| Offline and preproduction evaluation | Shadow traffic and live production monitoring |
| Baseline vs. 2–3 candidate comparison | CI/CD gates, automatic deployment |
| Full-trajectory evaluation | Automated prompt optimization (DSPy is future work) |
| Deterministic, LLM-judge, human, RAG, and agent metrics | Fine-tuning or training models |
| Admin-driven hyperparameter, prompt, and skill configuration | Runtime model routing |
| Informational report of evidence-backed trade-off statements | Automatic migration |

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
- Cloud-agnostic: containers with no dependency on one cloud’s proprietary services. The v1 deployment is Docker Compose. Kubernetes is a v2 profile (§12).
- Prompt changes are manual and made by the admin.

## 5. Conceptual Architecture

This section explains the ideas before the boxes.

### 5.1 Five core design principles

| # | Principle | Plain-language meaning | Benefit |
| --- | --- | --- | --- |
| P1 | Canonical trace | Translate every framework’s logs into one shared “language”. | Work scales as frameworks + metrics, not frameworks × metrics. |
| P2 | Control plane / execution plane | One place decides what to run. Workers anywhere do the running. | Governance stays central. Execution can sit inside any cloud or data boundary. |
| P3 | Controlled experiment | Change only the integration bundle (model + framework provider adapter). Freeze dataset, prompts, tools, and evaluator versions. | Score differences can be attributed to the bundle, and the adapter's share is quantified by the adapter-isolation check (§9.3.1). |
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
 │ + metrics   │───────▶│     and build report     │───────▶│ trade-off          │
 │             │        │                          │        │ statements         │
 └─────────────┘        └──────────────────────────┘        └────────────────────┘
```

### 5.3 Conceptual layer model

```text
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 1: APPLICATIONS    (ADK · MAF · LangGraph · Custom · Legacy)│
├──────────────────────────────────────────────────────────────────┤
│  LAYER 2: INTEGRATION     "How do we see inside the app?"        │
│           Injection mode (adapter-intercept · config-seam · gateway-route · variant-deployment) × Fidelity (full · near-full · decisions-only · black-box) │
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

### 5.4 The two-axis integration model

Integration is a spectrum because the same question, “can we see inside your app?”, has different answers across teams.

Integration is described on two independent axes recorded in the App Profile at onboarding.

**Axis A: Injection mode (how we swap the model)**

| Mode | Mechanism | Requirement on the app team |
| --- | --- | --- |
| `adapter-intercept` | MIRA SDK overrides the framework's model client at runtime | Adapter installed |
| `config-seam` | App reads model/provider from env var or config that MIRA overrides per job | A configuration seam |
| `gateway-route` | App's model base URL is repointed to MIRA's LLM gateway | Base URL is configurable |
| `variant-deployment` | Immutable app; one live endpoint per candidate model | Provision and register one endpoint per model |

**Axis B: Fidelity level (what we can observe)**

| Level | Mechanism | Observable |
| --- | --- | --- |
| `full` | Native framework hooks | Agents, tools, prompts, results |
| `near-full` | OTel or existing telemetry | Usually agents, tools, results |
| `decisions-only` | Gateway logging | Prompts and tool-call requests, no tool results or agent identity |
| `black-box` | Invocation endpoint | Input, output, latency, cost |

The hard-gate filter rejects invalid experiment configurations before any compute is spent, for example a trajectory metric on a `black-box` app. The legacy labels T1/T2/T2-G/T3 in later sections map to fidelity levels `full`, `near-full`, `decisions-only`, `black-box`.

The SDK is the preferred route and `black-box` is the safety net. Every result is stamped with its injection mode and fidelity level, so evaluators that need tool-level data report “not applicable” on `black-box` runs instead of a false zero.

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
    │                    │                 │       2. Choose injection mode and fidelity level          │
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
| 1 | Onboard | Register app, owner, framework, cloud, data classification, injection mode and fidelity level, agent and tool inventory (each tool classed read-only, idempotent or stateful), dataset provenance and prompt-tuning history | App profile | Profile complete. Stateful tools have a `restore_snapshot` hook, else the app is rejected |
| 2 | Integrate | Pick injection mode and fidelity level, connect adapter/traces/endpoint, run adapter-isolation check (§9.3.1) | Working capture, adapter overhead report | Smoke test passes |
| 3 | Provide data | Upload golden dataset and sampled production interactions | Versioned dataset | Validation and anonymization pass |
| 4 | Select models | Execution team applies hard gates, picks 2–3 candidates, and sets the non-inferiority margin δ before any run | Eligible shortlist, frozen δ | Gates satisfied |
| 5 | Configure | Admin freezes dataset and settings, sets hyperparameters, prompts/skills, metrics, weights. Platform runs the power check and, for `gateway-route`, the translation-fidelity probe (§10.2) | Frozen experiment | Config validated |
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
              │ near-full: connect│   │ Can the team add a         │
              │ existing traces  │   │ lightweight adapter?       │
              └────────┬─────────┘   └──────┬──────────────┬─────┘
                       │                Yes │              │ No
                       │                    ▼              ▼
                       │          ┌────────────────┐ ┌──────────────────────┐
                       │          │ full: install  │ │ black-box: expose a  │
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
│ Quality · RAG/agent metrics · Latency · Cost · Trace analysis · Trade-offs│
└────────────────────────────────────────────────────────────────────────────┘
```

## 8. Architecture Rationale

### 8.1 Why the two-plane split matters

| Concern | Control plane | Execution plane |
| --- | --- | --- |
| What it does | Decides, schedules, records, reports | Runs apps, calls models, scores |
| Where it lives | One central place | Anywhere: any cloud, on-prem, or inside a team’s boundary |
| Why separate | One place for governance and audit | Data residency, scale, and failure isolation |
| Analogy | Air traffic control | The aircraft |

### 8.2 Design decisions and rationale

| Decision | Choice | Rationale |
| --- | --- | --- |
| Integration | Two axes: injection mode and fidelity level (§5.4) | Honors “cannot modify” while maximizing fidelity, and separates how we swap from what we see |
| Existing tools | Ragas, DeepEval, LangSmith become evaluator plugins and trace sources | Teams keep their tools. The platform adds normalization and comparison |
| Model access | Provider adapter layer | A model swap is configuration, not code |
| Deployment | v1 Docker Compose, v2 Kubernetes profile (§12) | Cloud-agnostic and sized to 5–7 concurrent evaluations |
| Decision support | Pareto view plus optional weighted score | Supports RQ5 |
| Automation | Evaluate only | No deployment, no auto-optimization |

## 9. Low-Level Design (LLD)

### 9.1 The canonical trajectory model (heart of RQ1)

Described conceptually, as a tree of events per task run.

```text
 Run (one task, one model, one repeat)
  │
  ├── Run metadata: experiment, task, model + parameters, prompt version,
  │                 fidelity level
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
| Model context | provider, model, version, parameters, provider-adapter package and version (e.g. `langchain-anthropic:0.1.23`) | Attribute results to the integration bundle, not the model alone |
| Version context | prompt, skill, app, evaluator versions | Reproducibility |
| Content | input, output, tool name, arguments, results | Correctness checks |
| Resource | tokens, cost, latency, context size | Cost/performance comparison |
| Reliability | error, retry link | Retry and failure analysis |
| Fidelity | fidelity level and injection mode per run | Honest reporting |

Mapping rule: each framework adapter maps native concepts (a LangGraph node, an ADK sub-agent, a MAF agent step) onto these span types. RQ1 measures how completely that mapping succeeds.

### 9.2 Component specifications

| Component | Responsibility | Inputs | Outputs |
| --- | --- | --- | --- |
| Admin Console | UI/API for configuration, runs, and reports | Admin actions | Experiment definitions |
| App Registry | App profiles, injection mode, fidelity level, data classification, owner | Onboarding data | App catalogue |
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

Injection mode is independent of fidelity level (§5.4). The unit under test depends on the mode:

| Injection mode | Unit under test |
| --- | --- |
| `adapter-intercept`, `config-seam`, `variant-deployment` | Model + framework provider adapter + unchanged prompts and tools |
| `gateway-route` | Model + gateway translation layer (name and version logged) + the framework's baseline adapter + unchanged prompts and tools |

Reports for `gateway-route` experiments carry the warning: "Evaluated via API translation gateway. Trajectory errors may stem from gateway schema translation rather than native model capabilities."

State this limitation openly in the proposal. Without a configuration seam or a variant environment, a fair model swap isn’t possible.

**Unit under test.** MIRA evaluates the integration bundle: model + framework provider adapter + unchanged prompts and tools. Swapping a model also swaps the adapter (for example `ChatOpenAI` to `ChatAnthropic`), which differs in tool binding, role formatting and retry behavior. MIRA does not claim a pure model effect. The adapter package and version are logged on every run (§9.1).

#### 9.3.1 Adapter-isolation check

Run during Integrate (stage 2), once per framework adapter and model pair.

- Run a fixed set of tool-calling probes against the model directly (raw REST API or LiteLLM) and through the framework adapter.
- Compare tool-call validity, argument fidelity, error/retry rate and latency overhead.
- Report the adapter mutation rate (share of probes where the adapter changes the tool call or request) and latency overhead per adapter.
- The result is stored with the app profile and cited in the report as the adapter's contribution to observed differences.

### 9.4 Evaluator framework

Evaluators are plugins with a uniform contract: each declares a name, a version, and the trace fidelity it needs. It receives a canonical trace plus the task, and returns a score with a rationale.

| Evaluator family | Examples | Needs from trace | Cost | Ground truth needed |
| --- | --- | --- | --- | --- |
| Deterministic | Exact match, schema validity, tool-argument check, latency, cost | Varies | Low | Often yes |
| Trajectory (RQ3 focus) | Tool selection correctness, argument accuracy, handoff correctness, unnecessary steps, retry count, plan adherence, context-window endurance | Full trajectory (`full`/`near-full`) | Low–medium | Expected tool path helps |
| LLM-as-Judge | Groundedness, relevance, completeness, factual consistency, rubrics | Final output (+ context) | Medium | Optional |
| RAG metrics | Faithfulness, answer relevance, context precision/recall (Ragas, DeepEval) | Retrieval spans | Medium | Partly |
| Human review | Expert rating, correction, preference | Output (+ trace view) | High | None |

Rule of honesty: if an evaluator needs data the trace doesn’t have (for example a tool-argument check on a black-box run), it returns “Not applicable”, never zero.

Judge safeguards: use a 2-model ensemble from a family outside the candidates, randomize answer order to reduce position bias, and use explicit rubrics.

**Calibration gate.** Judge scores enter the verdict only if the judge ensemble reaches Cohen’s κ ≥ 0.6 against the human-labelled subset. Otherwise they are reported as “Descriptive/Uncalibrated”.

**Boundary-bound calibration.** Calibration runs inside the application’s declared data boundary. Labels come from cleared domain experts on the App Team. The judge must hold the same data-classification clearance as the app (for example a self-hosted model or an isolated endpoint). If no compliant judge exists, the judge evaluator is disabled for that experiment and the platform uses deterministic evaluators and human review only.

**Human review sampling.** Stratified sampling within a configurable labelling budget (10–15% of runs), oversampling runs where deterministic checks and the judge disagree, plus a double-rated subset for inter-rater reliability.

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

**Power and verdicts.** Before freezing, the platform computes the minimum task count (or achievable δ) from pilot variance, δ and α. If the dataset is too small, verdicts are “underpowered”, which is distinct from “inconclusive”. The Execution Team sets δ before the run. Holm correction applies to primary hypotheses only. Subgroup results (capability, document size, context bucket) are descriptive with CIs and carry no verdict.

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

## 10. Experiment Protocol

### 10.1 Experiment types

| Experiment | Meaning | Reported as |
| --- | --- | --- |
| Exp 1: Drop-in Swap | Prompts and tool schemas unchanged | Migration cost floor, not ceiling capability |
| Exp 2: Adapted Swap | Bounded edits for the candidate under a fixed editor protocol | Quality gain per unit of adaptation effort |

Adaptation effort is measured as lines of code changed and tokens changed in prompt templates and tool schemas. The editor may make only syntax, formatting and structural schema changes required by the new provider (for example OpenAI function definitions to Anthropic tool schemas). Adding task logic, examples or behavior changes is forbidden. If an adapted prompt also changes the baseline’s prompt, the baseline is re-run.

### 10.2 Gateway-route rules

- The baseline passes through the same gateway at the same version as the candidates, so the gateway cancels in the paired difference.
- One native-baseline arm per app measures the gateway’s own latency and error overhead.
- A 50-item translation-fidelity probe (nested schemas, parallel tool calls) runs against each candidate via the gateway before the main experiment. If the gateway mangles the schema, the run fails early.
- Results from different injection modes are never compared or rolled up together.

### 10.3 Run control

- Pin dated model versions and log the returned model identifier and `system_fingerprint`.
- Schedule jobs round-robin across arms (task 1: A, B, C; task 2: A, B, C), never one model’s batch at a time, so time-of-day latency and rate limits apply equally.
- The Model Catalogue holds a per-model parameter-support map. Parameters that are dropped or approximated (for example `seed`, `top_p`) are listed in the report header as unequal treatment.

### 10.4 Tool execution policy

| Tool class | Declared mode | Rule |
| --- | --- | --- |
| Read-only | `pure-read-live` | Called live against a frozen data snapshot |
| Idempotent | `recorded-with-strict-miss-fail` | Replayed from recordings. A cache miss fails that trajectory path. Fuzzy matching is banned |
| Stateful | `sandboxed-stateful` | App Team supplies `restore_snapshot`, run before every run. No hook, no evaluation |

Cache-miss rate is a primary diagnostic and a validity metric for trajectory comparison.

### 10.5 Cost basis

Cost uses a versioned price table with effective dates. Reports show List Cost and Effective Cost (negotiated rates). Self-hosted cost is a range under stated low and high utilization.

## 11. Security, Privacy and Trust

- Each app profile carries a data classification. Residency matching is enforced: a judge or candidate may process an app’s data only if its endpoint meets that classification.
- Payloads are encrypted at rest, redacted before storage, retained for a limited period, and viewable only through RBAC.
- Execution workers use a pull model (polling, no inbound connections). Jobs are signed manifests carrying hashes of frozen artifacts. Workers return signed traces. Vault issues short-lived, job-scoped credentials.

## 12. Deployment Profiles

| Profile | Components | Use |
| --- | --- | --- |
| v1 Minimal Validating Deployment | Docker Compose, PostgreSQL, Celery with Redis, local disk or MinIO | Enough for 5–7 concurrent evaluations and for validating RQ1, RQ3, RQ5 |
| v2 Enterprise Scaling Profile | Kubernetes, Argo Workflows or Temporal, KEDA, Vault, in-boundary remote workers | When scale or isolation requires it |

The library map in the Addendum (§D) lists v2 components. Use only the v1 subset first.

## 13. Study Protocols

### 13.1 RQ1: mapping coverage

Hand-annotate 30 traces per framework (90 total) as ground truth. Pre-registered thresholds: 95% mapping recall for tool spans and 90% for handoff spans, with correct agent identity. Report unmapped residuals by category.

### 13.2 RQ3: regression detection

Seeded-regression dataset: inject known faults (wrong tool, malformed argument, redundant loop) into golden traces and compare detection recall of trajectory metrics against final-answer metrics. Add a small set of natural regressions verified by human adjudication.

### 13.3 RQ5: decision study

Between-subjects study with N ≥ 20 engineers and product managers. Scenarios use seeded data with a known optimal choice, and business priorities are fixed in the scenario text (for example “cut cost by at least 30% while keeping clean success within 2 points of baseline; latency does not matter”). Compare the Pareto report against a single weighted score on decision accuracy, time to decision and confidence.

Fallback, pre-registered: if N ≥ 20 cannot be recruited, run a qualitative pilot (N ≈ 5–10) reporting descriptive accuracy and time, plus structured interviews on cognitive load and confidence.

## 14. Threats to Validity

| Threat | Mitigation |
| --- | --- |
| Judge reliability | Calibration gate and boundary-bound calibration (§9.4) |
| Provider nondeterminism and model drift | Pinned versions, fingerprint logging, round-robin scheduling, repeats (§10.3) |
| Dataset contamination | Provenance and tuning history declared at onboarding, held-out split |
| Prompt-model coupling | Drop-in versus Adapted Swap reported separately (§10.1) |
| Adapter and gateway translation effects | Adapter-isolation check (§9.3.1), native-baseline arm and fidelity probe (§10.2), unit under test stated in every report |
| Tool replay misses | Strict miss-fail and cache-miss rate (§10.4) |
| Trace mapping loss | RQ1 coverage thresholds and unmapped-residual reporting (§13.1) |
| Small sample and multiplicity | Power check, primary-only Holm correction, descriptive subgroups (§9.5) |
| Human labeller variance | Double-rated subset and inter-rater agreement (§9.4) |

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

The gateway insight. Model injection already forces you to control where model calls go, so a gateway comes almost for free. It captures what the model decided (for example “call search_db with these arguments”) for apps you cannot instrument. It does not see tool execution results, or agent identity unless the app propagates a header. I would add this as the `decisions-only` fidelity level (formerly T2-G).

| Level | Sees agents? | Sees tool decisions + args? | Sees tool results? | Sees LLM calls? |
| --- | --- | --- | --- | --- |
| `full` (SDK/adapter) | Yes | Yes | Yes | Yes |
| `near-full` (OTel / existing traces) | Depends on instrumentation | Usually | Usually | Yes |
| `decisions-only` (gateway) | No (unless header) | Yes | No | Yes |
| `black-box` | No | No | No | No |

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
| LangChain/LangGraph capture | LangChain callback system; LangSmith trace export | `full`/`near-full` source |  |
| Google ADK capture | ADK callbacks and its built-in tracing | `full` |  |
| Microsoft Agent Framework capture | Built-in OTel telemetry | `full`/`near-full` |  |
| Model gateway / provider abstraction | LiteLLM (or an equivalent proxy) | One interface to GPT, Claude, Gemini and self-hosted models, plus `decisions-only` logging | Central to model injection |
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
| Human-in-the-loop feedback | Partial | Expert review is covered, but the user feedback channel (thumbs, edits, corrections) and explicit vs. implicit weighting are not. Resolved in §F.4. |
| Evaluation scope (intermediate steps and component vs. end-to-end) | Now covered (§A, B) |  |
| Prompt management and governance | Mostly covered | Versioned registry and lineage exist. Add a trend view linking prompt versions to results across experiments over time. |
| Quality vs. retry analysis | Partial | Retries are captured as spans but no metric was defined. Resolved in §F.4. |
| LLM catalogue and configuration | Partial | A platform-level catalogue with roles and approvals exists. A runtime catalogue exposed inside every application was dropped when you chose evaluation plus selection (no runtime control). State this boundary explicitly. |
| Model traceability | Covered | Every run links to model, version, parameters, prompt, app and evaluator versions |
| Evaluation metrics (standard vs. configurable) | Partial | Metrics are selectable, but I never defined a mandatory core set. Resolved in §F.4. |

#### F.3 Section “Design a centralized framework that provides…” (items 1–8)

| # | Requirement | Status |
| --- | --- | --- |
| 1 | Platform-independent deployment | Covered |
| 2 | Intermediate steps + final outputs | Covered |
| 3 | Human-in-the-loop feedback integration | Covered (§F.4) |
| 4 | Prompt and model version management | Covered |
| 5 | Full lineage and traceability | Covered |
| 6 | Configurable model catalogues and runtime parameters | Covered for evaluation. Application runtime control excluded. |
| 7 | Standardized and application-specific metrics | Covered (§F.4) |
| 8 | Quality measurement using retries, corrections and feedback | Covered (§F.4) |

#### F.4 Resolutions for the partial items

**Mandatory metric core**

| Class | Metrics | Rule |
| --- | --- | --- |
| Mandatory core (all apps) | Outcome success, clean success, pass@k, pass^k, cost per successful task, p50/p95 latency, TTFT, tool-argument validity, retry rate, cache-miss rate | Platform. Not removable. |
| Standard optional | Ragas/DeepEval RAG metrics, groundedness, tool-selection precision and recall, handoff correctness | App owner selects |
| Application-specific | Custom rubrics, domain checks, business-outcome metrics | App owner defines. Admin approves. |

**Retry and quality analysis**

- First-attempt success rate versus eventual success rate. The difference is quality that depends on retrying.
- Retries per task and success by attempt number.
- Cost of retries as a share of total cost.
- In production data, a user regenerating a response is ambiguous (dissatisfaction, curiosity, UI habit). Treat retry frequency as a weak triage signal for prioritizing review samples, never as ground truth.

**Feedback weighting**

| Signal | Reliability | Use |
| --- | --- | --- |
| Expert rating or correction | High | Ground truth and judge calibration |
| User edit of an output | Medium | Candidate correction, needs review before use |
| Thumbs up or down | Low to medium | Triage and sampling priority |
| Regeneration or retry | Low | Triage only |

Business-outcome fields (resolution rate, escalations, conversions) are optional on dataset items and reported as secondary metrics. The platform has no runtime model catalogue inside applications: it evaluates and reports, and runtime control is out of scope.
