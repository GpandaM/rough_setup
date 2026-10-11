# MIRA: Model Invariance and Resilience Analysis for Agentic Applications

**Architecture review document, version 2.** MIRA is a cloud- and framework-agnostic platform that tells an application owner whether swapping the underlying LLM keeps the whole agentic application working, and at what cost and latency.

Reading path: what an app must provide (1), how MIRA is built (2), how apps join (3), how traces are collected (4), what data drives scoring (5), how experiments run (6), how metrics are computed (7, 8), how humans and prompts are governed (9, 10), how results are traced (11), where MIRA runs (12), and what it outputs (13).

---

## 1. What MIRA is and what an app must provide

**Purpose:** State what MIRA does, what it refuses to do, and the minimum an application must offer to be evaluated.

MIRA replays the same frozen tasks through an application using a baseline model and 2-3 candidate models. It captures the full execution trajectory, scores it, and reports trade-offs between quality, cost, and latency.

**What MIRA is NOT**

| MIRA does not | Meaning |
| --- | --- |
| Deploy or migrate | It produces evidence. People decide. |
| Optimize prompts automatically | Prompt changes are manual and made by the admin (Section 10). |
| Touch production traffic | Evaluation is offline or preproduction only. |
| Rank models with one verdict | It shows trade-offs and a Pareto view (Section 13). |

### 1.1 Minimum app contract

| # | Need | Mandatory / optional | Why |
| --- | --- | --- | --- |
| a | Invocation entry point (endpoint, function, or CLI) | Mandatory | MIRA replays every task through it. |
| b | Model-swap seam (config/env, base URL, or variant endpoint) | Mandatory | Without a seam, a fair model swap is impossible (Section 2.3). |
| c | Trace source (callbacks, OTel, DB tables, logs) | Optional | It decides fidelity. No trace source means black-box (defined once in Section 2.4). |
| d | Task set (golden, sampled logs, or generated; Section 5) | Mandatory, with fallback tiers | Replay needs inputs. Labels are needed only for some metrics. |
| e | Tool classification (read-only / idempotent / stateful), plus a `restore_snapshot` hook for stateful tools | Mandatory for stateful tools | Repeated runs must start from the same state. No hook, no evaluation. |
| f | Data classification | Mandatory | Decides which workers, judges, and models may touch the data (Section 12). |

### 1.2 Two data paths

| Path | What happens | Used when |
| --- | --- | --- |
| Replay path (primary) | MIRA re-runs the app with each candidate model and captures fresh traces. | Every experiment. This is how candidates get compared. |
| Import path | Connectors read existing traces from Postgres tables, OTel, LangSmith, logs, or blob storage. | Near-full apps, and bootstrapping tasks from production history (Sections 4, 5). |

### Diagram 1: What goes in, what comes out

```text
   INPUTS                         MIRA                          OUTPUTS
 +----------------+      +------------------------+      +---------------------+
 | App + contract |----->| 1 Replay app per model |----->| Per-model scores    |
 | (1.1 a-f)      |      +------------------------+      +---------------------+
 +----------------+      | 2 Capture trajectory   |      | Cost and latency    |
 | Task set       |----->+------------------------+----->+---------------------+
 | (golden/logs/  |      | 3 Normalize to one     |      | Trace diffs         |
 |  generated)    |      |   canonical trace      |----->+---------------------+
 +----------------+      +------------------------+      | Pareto view         |
 | Baseline + 2-3 |----->| 4 Score (rules, judge, |----->+---------------------+
 | candidates     |      |   human, RAG, agent)   |      | Trade-off statement |
 +----------------+      +------------------------+      | (non-inferiority)   |
 | Prompts, gates,|----->| 5 Compare, build report|      +---------------------+
 | metrics        |      +------------------------+
 +----------------+
```

*Caption: Inputs on the left are supplied by the app team, execution team, and admin. MIRA replays, captures, normalizes, scores, and compares in five steps. The right column lists the five headline outputs of every report.*

**Five headline outputs of the final report**

1. Per-model quality scores (outcome and clean success, RAG and agent metrics).
2. Cost per successful task and latency, with p50 and p95.
3. Trace diffs showing where a candidate diverged from the baseline.
4. A Pareto view of quality, cost, and latency.
5. An informational trade-off statement with a non-inferiority conclusion.

### 1.3 Small end-to-end example

### Diagram 1b: One task through MIRA

```text
 App: claims assistant (Supervisor -> Research -> Analysis -> Response)
 Task T1: "Is claim 4471 covered?"    Baseline: Model A    Candidate: Model B

 ONBOARD (once)  Scan repo -> profile: entry=POST /ask, seam=MODEL_NAME env,
                 tools=search_policy, query_db (read-only). Team confirms.

 RUN (per model, per repeat)
   MIRA sets MODEL_NAME=B -> sends T1 to POST /ask
   App runs by itself:  Supervisor -> Research (search_policy)
                                   -> Analysis (query_db) -> Response
   MIRA only watches: callbacks / OTel / gateway record each step.

 NORMALIZE  Raw records -> canonical trace (agents, tools, arguments, cost, time)

 SCORE (reads the trace only)
                          Model A      Model B
   Final answer correct     yes          yes
   Tool arguments valid     yes          NO (date format), 1 retry
   Cost per run           $0.08        $0.10
   Latency                  6 s          8 s

 REPORT  B matches A on outcome, but needs a retry, costs more, and is slower.
         Verdict is shown as trade-offs, not a single winner.
```

*Caption: MIRA scans the app once, then swaps only the model and sends the task to the entry point. The app runs its own agents and tools while MIRA records them. Scoring reads the trace and shows that Model B gets the right answer through a weaker path.*

**What this means:** An app needs only an entry point, a model seam, a task set, and a data classification to be evaluated. Everything else raises the depth of evidence, not the ability to run. MIRA reports; it never decides.

---

## 2. Architecture: layers and integration axes

**Purpose:** Show every layer, what it contains, how it is configured, and how MIRA sees inside and swaps the model of an app.

### Diagram 2: Layered architecture

```text
+-----------------------------------------------------------------+
| L1 APPLICATIONS   ADK | MAF | LangGraph | custom | legacy       |
+-----------------------------------------------------------------+
| L2 INTEGRATION    Axis A injection mode x Axis B fidelity level |
+-----------------------------------------------------------------+
| L3 CONTROL PLANE  Console | Registries | Gates | Orchestrator   |
+-----------------------------------------------------------------+
| L4 EXECUTION PLANE Replay | Inject | Normalize | Evaluate | Human|
+-----------------------------------------------------------------+
| L5 DATA           Datasets | Traces | Scores | Versions | Lineage|
+-----------------------------------------------------------------+
| L6 INSIGHT        Aggregation | Pareto | Trace diff | Report    |
+-----------------------------------------------------------------+
 Cross-cutting:  Security | Governance | Lineage | Audit
```

*Caption: Six layers stack from the application at the top to the report at the bottom. Control decisions sit in layer 3 and work happens in layer 4. Security, governance, lineage, and audit cut across all layers.*

### 2.1 Layer reference

**L1 Applications**
- Meaning: the agentic apps under test, each tied to one model today.
- Components: orchestrator or router, sub-agents, tools, retrieval, prompts, model client.
- Inputs / outputs: receives tasks through the entry point; returns answers plus whatever telemetry it emits.

| Setting (App Profile) | Allowed values | In practice |
| --- | --- | --- |
| Framework | LangGraph, ADK, MAF, custom | Picks the adapter and trace mapping. |
| Data classification | Org-defined levels (Assumption: public, internal, confidential, restricted) | Limits which workers and judges may process the data. |
| Tool class | read-only, idempotent, stateful | Sets the tool execution policy (Section 6.4). |

**L2 Integration**
- Meaning: how MIRA swaps the model (Axis A) and how much of the run it can see (Axis B). Detailed in Sections 2.3 to 2.5.
- Components: framework adapters, OTel ingest, trace importers, LLM gateway, black-box client.
- Inputs / outputs: takes a model config and task; returns a native trace plus output.

| Setting | Allowed values | In practice |
| --- | --- | --- |
| Injection mode | adapter-intercept, config-seam, gateway-route, variant-deployment | How the model is swapped. |
| Fidelity level | full, near-full, decisions-only, black-box | What evaluators may rely on. |
| Gateway header propagation | on / off | On lets the gateway attribute calls to agents. |

**L3 Control plane**
- Meaning: the single place that decides what runs. Governance stays central.
- Components: Admin Console (UI and API), App Registry, Dataset Registry, Prompt/Skill Registry, Model Catalogue, Hard-Gate Filter, Experiment Orchestrator, Audit and Lineage Service.
- Inputs / outputs: takes admin actions and registry data; emits frozen experiments and job manifests.

| Setting | Allowed values | In practice |
| --- | --- | --- |
| Hard gates | Capability rules: context length, modality, tool calling, residency, throughput | Reject ineligible models before any compute is spent. |
| Non-inferiority margin $\delta$ | Set by Execution Team before the run | The "not meaningfully worse" threshold (Section 7.7). |
| Repeats $n$ and $k$ values | $n \ge \max(k)$, typical $n$ = 5 to 10, $k \in \{1,3,5\}$ | Controls reliability measurement. |
| Metric set and weights | Core fixed; optional chosen (Section 8) | What is scored and how a weighted view is built. |
| Freeze | Dataset, prompts, evaluators, parameters | Nothing changes after freeze. A change makes a new experiment. |

**L4 Execution plane**
- Meaning: workers that do the running, placed anywhere, including inside a team's data boundary.
- Components: Replay Worker, Model Injector, Normalizer, Tool Sandbox, Evaluator Registry and evaluators, Human Review Queue, provider adapters.
- Inputs / outputs: takes signed jobs; returns signed traces and scores.

| Setting | Allowed values | In practice |
| --- | --- | --- |
| Worker location | central, in-boundary remote | Meets data residency (Section 12). |
| Concurrency | Sized for 5-7 concurrent app evaluations (Assumption from the baseline load) | Caps parallel jobs. |
| Tool mode per tool | pure-read-live, recorded-with-strict-miss-fail, sandboxed-stateful | Keeps tool behavior identical across models (Section 6.4). |

**L5 Data**
- Meaning: immutable, versioned memory of everything.
- Components: relational metadata store, object store for payloads by hash, dataset and trace stores.
- Inputs / outputs: takes traces, scores, versions; returns lineage-linked records.

| Setting | Allowed values | In practice |
| --- | --- | --- |
| Retention period | Org policy (Assumption: set per data classification) | Payloads are deleted after the period. |
| Redaction | On at capture (mandatory for non-public data) | PII is removed before leaving the worker. |

**L6 Insight**
- Meaning: turns scores into evidence a person can act on.
- Components: Aggregator, statistics, Pareto filter, trace-diff builder, Report Generator.
- Inputs / outputs: takes scores and traces; returns the comparison report (Section 13).

| Setting | Allowed values | In practice |
| --- | --- | --- |
| Normalization for weighted score | min-max or baseline-relative | Makes dimensions comparable. |
| Report view | Pareto (always), weighted score (optional) | The weighted score never replaces Pareto. |

**Cross-cutting**
- Security: encryption, RBAC, short-lived credentials (Section 12).
- Governance: approvals for models, prompts, and custom metrics (Sections 8, 10).
- Lineage and audit: every result is linked to its inputs (Section 11).

### 2.2 Control plane vs execution plane

| | Control plane | Execution plane |
| --- | --- | --- |
| What it does | Decides, schedules, records, reports | Runs apps, calls models, scores |
| Where it lives | One central place | Any cloud, on-prem, or inside a team boundary |
| Why separate | One place for governance and audit | Data residency, scale, failure isolation |
| Analogy | Air traffic control | The aircraft |

### 2.3 Axis A: injection mode (how the model is swapped)

| Mode | Mechanism | App team must provide | Unit under test |
| --- | --- | --- | --- |
| `adapter-intercept` | MIRA SDK overrides the framework's model client at runtime | Adapter installed | Model + framework provider adapter + unchanged prompts and tools |
| `config-seam` | App reads model and provider from env or config that MIRA overrides per job | A configuration seam | Same as above |
| `gateway-route` | App's model base URL is repointed to MIRA's LLM gateway | Configurable base URL | Model + gateway translation layer (name and version logged) + the framework's baseline adapter + unchanged prompts and tools |
| `variant-deployment` | Immutable app, one live endpoint per candidate model | Provision and register one endpoint per model | Same as `adapter-intercept` |

Swapping a model also swaps its provider adapter (for example `ChatOpenAI` to `ChatAnthropic`), which differs in tool binding, role formatting, and retries. MIRA therefore evaluates the integration bundle (model + provider adapter), not a pure model effect. The adapter package and version are logged on every run, and the adapter-isolation check (Section 3.5) measures the adapter's share.

### 2.4 Axis B: fidelity level (what MIRA can observe)

| Level | Mechanism | Sees agents | Sees tool decisions and arguments | Sees tool results | Sees LLM calls |
| --- | --- | --- | --- | --- | --- |
| `full` | Native framework hooks (SDK adapter) | Yes | Yes | Yes | Yes |
| `near-full` | OTel or existing telemetry | Depends on instrumentation | Usually | Usually | Yes |
| `decisions-only` | Gateway logging | Only if the app propagates a header | Yes | No | Yes |
| `black-box` | Invocation endpoint only | No | No | No | No |

**Black-box (defined once).** An app is black-box when MIRA can call it but see nothing inside. Measurable: input, output, end-to-end latency, and cost (as reported or estimated). Not measurable: agent identity, tool calls, arguments, tool results, handoffs, retries, per-step cost, context size. These metrics return "Not applicable", never zero: tool selection, argument accuracy and validity, handoff correctness, retry metrics, cache-miss rate, per-sub-task cost, context endurance, critical path, per-call TTFT. Black-box is the safety net, and each result states its level.

### 2.5 Axis A x Axis B matrix

### Diagram 3: Valid, risky, and invalid combinations

```text
Legend: V = valid   R = risky (works, with stated caveat)   X = invalid

                     full   near-full   decisions-only   black-box
 adapter-intercept     V        V             X              X
 config-seam           V        V             R              V
 gateway-route         R        R             V              X
 variant-deployment    V        V             R              V

 R notes:
  config-seam + decisions-only   needs a gateway as well as the seam
  gateway-route + full/near-full  gateway translation enters the unit under test
  variant-deployment + decisions-only  gateway must sit in front of each endpoint

 X notes:
  adapter-intercept + decisions-only/black-box  the adapter already sees more
  gateway-route + black-box  a gateway always sees LLM calls
```

*Caption: Rows are how the model is swapped and columns are how much MIRA sees. The hard-gate filter rejects invalid cells before compute is spent and flags risky cells in the report. Results from different injection modes are never pooled or compared.*

**What this means:** The two axes are independent: one asks "how do we swap?", the other asks "what can we see?". Every result carries both stamps. A weak seam or low fidelity limits the evidence, and MIRA says so instead of guessing.

---

## 3. App onboarding

**Purpose:** Turn a registered app into a working, validated integration with a complete App Profile.

### Diagram 4: Onboarding decision flow

```text
 +-----------------------+
 | Register application  |
 +-----------+-----------+
             v
   Can the app emit traces already (OTel, LangSmith, DB, logs)?
        | Yes                                   | No
        v                                       v
 +--------------------+        Can the team add a lightweight adapter?
 | near-full:         |              | Yes                | No
 | connect traces     |              v                    v
 +---------+----------+     +----------------+   +--------------------+
           |                | full: install  |   | black-box: expose  |
           |                | adapter        |   | invocation endpoint|
           |                +--------+-------+   +----------+---------+
           +-----------------+-------+----------------------+
                             v
        Pick injection mode (2.3) -> smoke test -> adapter-isolation check
```

*Caption: Onboarding asks two questions in order: can the app already emit traces, and can the team add an adapter. The answers set the fidelity level. Every path ends at the same smoke test.*

### 3.1 App Profile fields

| Group | Fields |
| --- | --- |
| Identity | App name, owner, framework, cloud or platform, version |
| Integration | Injection mode, fidelity level, entry point, model seam location |
| Architecture | Agent inventory (name, role), orchestrator or router, tool inventory with class and schema, retrieval components |
| Data | Data classification, task set source and provenance, prompt-tuning history |
| Prompts | Prompt and skill locations and versions |
| Mappings | Node-to-agent mapping for identity resolution (Section 4.4) |

### 3.2 Repo-driven discovery

**Scenario:** the app team gives a repository. MIRA drafts the profile; people confirm it.

### Diagram 5: Repo-driven discovery sequence

```text
 App Team        Scanner           Profile Store      Integrator       Smoke Test
    | share repo    |                    |                 |                |
    |-------------->| 1 static pass:     |                 |                |
    |               |   manifests,       |                 |                |
    |               |   anchors, index   |                 |                |
    |               | 2 trace from entry |                 |                |
    |               |   point, read slice|                 |                |
    |               |------------------->| 3 draft profile |                |
    |<------------------------------------| (with evidence) |                |
    | 4 confirm /   |                    |                 |                |
    |   correct     |------------------->| 5 final profile |                |
    |               |                    |---------------->| 6 pick mode,   |
    |               |                    |                 |   fidelity     |
    |               |                    |                 | 7 instrument / |
    |               |                    |                 |   connect      |
    |               |                    |                 |--------------->| 8 capture
    |               |                    |                 |                |   sample
    |<---------------------------------------------------------------------| 9 isolation
    |               |                    | 10 gates pass?  |                |   check
```

*Caption: The scanner reads a small slice of the repo and drafts the App Profile with file and line evidence. The app team confirms, the integrator wires capture, and the smoke test validates the profile against a real trace. Gates decide whether execution may start.*

**Step by step**

| Step | Action | Output |
| --- | --- | --- |
| a. Scan | Detect framework (LangGraph, ADK, MAF, custom), entry points, agents, orchestrator or router, tools and schemas, prompts, model-client configuration, retrieval components, existing telemetry and logging | Findings with file and line evidence |
| b. Draft profile | Assemble agent inventory, tool inventory with class, model seams, prompt locations | Draft App Profile, confirmed by the app team |
| c. Choose mode | Pick injection mode (2.3) and fidelity level (2.4) with written reasons | Integration decision |
| d. Instrument or connect | SDK adapter, OTel, gateway, or endpoint | Working capture |
| e. Validate | Smoke test captures a sample trace. Adapter-isolation check runs (3.5) | Validation report |
| f. Gates | See 3.4 | Go / no-go |

### 3.3 Scanning a repo that is too large

Do not read the whole repo. Narrow in stages and stop once the App Profile is filled.

1. **Cheap static pass (no LLM).** Parse manifests (`pyproject.toml`, `requirements.txt`, `package.json`) to detect the framework. Use grep or AST patterns for anchors: graph or agent definitions, tool decorators or registries, model-client constructors, `base_url` and env settings, OTel or callback setup, and prompt files.
2. **Build an index.** Map each anchor to file and symbol, and extract the import graph. Skip tests, docs, vendored code, generated files, and lockfiles.
3. **Trace from the entry point.** Start at the invocation entry (API route, main, or CLI). Follow calls only to the orchestrator, agents, tools, and model clients. This slice is usually under 5% of the repo (Assumption).
4. **Targeted LLM reading.** Read only the slice and the anchors, with per-module summaries (map-reduce), so context never holds the full repo. Use embeddings or symbol search only to resolve ambiguities.
5. **Draft with evidence.** Record a file and line reference per finding. Mark anything undetected as "unknown", never guess.
6. **Confirm and fill gaps.** The app team corrects the draft. The smoke test validates it: if a declared agent or tool never appears in captured traces, the profile is wrong or incomplete.

For monorepos, ask the team for the service path first, or detect it from the deployment config (Dockerfile or entry command).

### 3.4 Gates to proceed to execution

| Gate | Condition |
| --- | --- |
| Profile complete | All mandatory fields in 1.1 are filled |
| Stateful tools | Each has a `restore_snapshot` hook, else the app is rejected |
| Smoke test | A sample trace is captured and passes schema validation |
| Inventory match | Declared agents and tools appear in the sample trace, or the gap is explained |
| Task set | Meets at least a tier in Section 5 |

### 3.5 Adapter-isolation check

Run once per framework adapter and model pair. A fixed set of tool-calling probes runs against the model directly (raw API or gateway) and through the framework adapter. MIRA compares tool-call validity, argument fidelity, error and retry rate, and latency overhead. It reports the adapter mutation rate (share of probes where the adapter changes the call) and the latency overhead. The result is stored with the App Profile and cited in reports as the adapter's contribution to observed differences.

**What this means:** Onboarding is mostly automatic discovery plus human confirmation. Fidelity is a result of what the app can emit, not a preference. A profile is trusted only after a real trace confirms it.

---

## 4. Trace collection and normalization

**Purpose:** Gather traces from wherever they live and convert them into one canonical trace that every evaluator can read.

### 4.1 Where traces can live

| Source | Typical content | Fidelity it supports |
| --- | --- | --- |
| Framework callbacks (LangChain/LangGraph, ADK) | Chains, nodes, LLM calls, tools, agent authorship | full |
| OTel collector (MAF, OpenInference, OpenLLMetry) | Agent, model, and tool spans | near-full |
| LangSmith or Langfuse exports | Run trees with inputs and outputs | near-full |
| Application database tables (for example PostgreSQL) | Conversation, step, and tool-call rows | near-full |
| Log files | Free-form or structured logs | near-full or decisions-only |
| Blob storage | Archived payloads | Depends on content |
| Gateway logs | Every model request and response | decisions-only |

### 4.2 Connector model

| Element | Meaning |
| --- | --- |
| Connector | One per source type. Reads and streams raw records. |
| Mapping config | Per app. Maps source table and column (or log field) to a canonical field. |
| Correlation key | Joins records of one run: trace ID, session ID, or a timestamp-window join when no ID exists. |
| Stitching | Merges records from several sources into one trace using the correlation key. |
| Gap flags | Marks missing spans, orphan records, and ambiguous joins. |
| Coverage score | Share of expected agents, tools, and steps found. Reported with every run. |

Timestamp-window joins are less reliable than ID joins (Assumption), so runs joined that way are flagged and their coverage score is shown in the report.

### Diagram 6: Trace pipeline

```text
 SOURCES                CONNECTORS          NORMALIZER           CANONICAL STORE
 +----------------+    +-----------+      +------------+       +----------------+
 | Callbacks      |--->| Callback  |--+   |            |       | Raw traces     |
 | OTel collector |--->| OTel      |  |   | Apply      |------>| (native form,  |
 | LangSmith/     |--->| Export    |  +-->| mapping    |       |  for debugging)|
 |  Langfuse      |    | importer  |  |   | config     |       +----------------+
 | DB tables      |--->| SQL       |  |   | Resolve    |       +----------------+
 | Log files      |--->| Log parser|  |   | identity   |------>| Canonical      |
 | Blob storage   |--->| Blob      |  |   | Stitch by  |       | traces (for    |
 | Gateway logs   |--->| Gateway   |--+   | correlation|       |  evaluation)   |
 +----------------+    +-----------+      | key        |       +----------------+
                                          | Flag gaps  |       Both kept, linked
                                          +------------+       by run ID
```

*Caption: Every source type has one connector. The normalizer applies the app's mapping config, resolves identity, stitches records, and flags gaps. Raw and canonical traces are both stored and linked.*

### 4.3 The canonical trace

A tree of events per run (one task, one model, one repeat).

**Span types:** agent, LLM call, tool call, handoff, retrieval, retry.

| Field group | Contents | Why it matters |
| --- | --- | --- |
| Identity | run, experiment, task, span, parent | Rebuild the tree |
| Model context | provider, model, version, parameters, provider-adapter package and version | Attribute results to the integration bundle |
| Version context | prompt, skill, app, evaluator versions | Reproducibility (Section 11) |
| Content | input, output, tool name, arguments, results | Correctness checks |
| Resource | tokens, cost, latency, context size | Cost and performance (Section 7) |
| Reliability | error, retry link | Retry and failure analysis |
| Fidelity | fidelity level and injection mode | Honest reporting |

Each framework adapter maps native concepts (a LangGraph node, an ADK sub-agent, a MAF agent step) to these span types.

### 4.4 Where the orchestrator routed

| Question | How MIRA answers |
| --- | --- |
| Which agent acted? | Identity is resolved in this order: (1) explicit agent attribute (OTel agent name, ADK author, MAF agent name); (2) the framework's structural unit (LangGraph node name); (3) the App Profile mapping (for example "node `research_node` = Research Agent"); (4) otherwise "unattributed". MIRA never guesses. |
| Who handed off to whom? | Handoff spans record source agent, target agent, and direction. |
| Why did the router pick that path? | The router's LLM call span holds its tool-call decision (for example "transfer to Research Agent"). |
| What is the order? | Parent-child links and timestamps. |
| Was an expected agent skipped? | The agent inventory in the App Profile is compared with observed agents. |

### 4.5 Which arguments went to which tool

| Fidelity level | Capture point | Argument evidence |
| --- | --- | --- |
| full | Framework hook around each tool call | Arguments and results, verbatim |
| near-full | Telemetry spans or DB rows | Usually arguments and results, if the app emits them |
| decisions-only | Gateway sees the model's tool-call request | Arguments as requested, no results |
| black-box | None | Not applicable (Section 2.4) |

Arguments are stored verbatim. Each call is validated against the tool schema from the App Profile. Where the task has an expected trajectory spec (Section 5), arguments are also compared with expected values.

### Diagram 7: Canonical trace tree with routing and tool arguments

```text
 Run r17  (task T42, model B, repeat 2, fidelity=full, mode=config-seam)
 +-- Root span: "Is claim 4471 covered?"
     +-- Agent: Supervisor [identity: explicit attribute]
     |   +-- LLM call: router  tokens 820/45  decision: transfer(Research)
     |   +-- Handoff ---> Research Agent
     |   |   +-- LLM call
     |   |   +-- Tool call: search_policy
     |   |   |     args: {"query":"claim 4471 coverage"}   schema: valid
     |   |   +-- Retrieval: policy docs (3 chunks)
     |   +-- Handoff ---> Analysis Agent
     |   |   +-- Tool call: query_db
     |   |   |     args: {"date":"03/04/2026"}   schema: INVALID (expects ISO)
     |   |   +-- Retry of query_db
     |   |         args: {"date":"2026-04-03"}   schema: valid
     |   +-- Handoff ---> Response Agent
     |       +-- LLM call --> Final answer
     +-- Unattributed spans: 0    Coverage score: 1.00
```

*Caption: One run is a tree. The router's decision appears as an LLM call, handoffs show routing, and each tool call carries its verbatim arguments and a schema check. The invalid date and its retry are visible even though the final answer is correct.*

### 4.6 Payload policy

| Concern | Policy |
| --- | --- |
| Sensitive data | Redaction at capture, before leaving the worker |
| Large payloads | Stored in the object store, referenced by hash |
| Sampling | None. Every step of every evaluation run is kept. |
| Raw vs canonical | Both kept (Diagram 6) |

**What this means:** Evaluators never need to know where a trace came from. Connectors and mappings absorb source differences, and gaps are flagged, not hidden. What cannot be attributed is labelled "unattributed".

---

## 5. Evaluation data strategy and golden-dataset dependency

**Purpose:** Define what data MIRA uses, how much it depends on labelled data, and what is still possible without it.

### 5.1 Data sources

| Source | Used for |
| --- | --- |
| Golden set (labelled tasks with expected outputs) | Correctness metrics, judge calibration, non-inferiority verdicts |
| Sampled production interactions (anonymized) | Realistic task inputs, bootstrap of task sets, baseline behavior |
| Ground truth (expected answers, expected trajectory spec: tools, argument constraints, order, handoffs) | Trajectory and correctness scoring |
| System telemetry (traces, cost, latency) | Cost, latency, trajectory, and retry metrics |
| User interactions (edits, thumbs, regenerations) | Triage and sampling (Section 9) |
| Business outcomes (resolution rate, escalations, conversions) | Optional secondary metrics, reported separately |

### 5.2 How much does MIRA depend on a golden dataset?

| Metric family | Dependency |
| --- | --- |
| Cost, latency, TTFT, tokens | Reference-free |
| Schema validity, tool-argument validity, retry rate, cache-miss rate | Reference-free |
| Baseline-vs-candidate trace diff and agreement | Reference-free (uses the baseline as comparator) |
| Exact match, outcome success | Needs golden |
| Tool selection precision and recall, order score, argument accuracy, handoff correctness | Needs golden (expected trajectory spec) |
| LLM-as-judge rubrics (groundedness, relevance, completeness) | Optional (rubric-only is possible; calibration needs human labels) |
| RAG metrics (faithfulness, answer relevance) | Optional (partly reference-free; context recall needs golden) |
| pass@k, pass^k, non-inferiority | Needs a success definition, so golden or an approved proxy (tiers below) |

**Short answer:** MIRA can always run and always report cost, latency, and behavior. A quality verdict that a board can rely on needs labels, from a golden set or a human-labelled seed.

### 5.3 If the app team has no golden dataset

| Tier | Approach | Strongest verdict possible |
| --- | --- | --- |
| 1 | Reference-free metrics: schema validity, tool-argument validity, retry rate, cost, latency, trace diff and agreement vs baseline | Descriptive only. No quality verdict. |
| 2 | Bootstrap tasks from sampled production logs with PII redaction (import path) | Descriptive with agreement rates and confidence intervals |
| 3 | Synthetic tasks generated from repo and tool schemas, reviewed by the app team | Conditional: "synthetic-task" non-inferiority, labelled as such |
| 4 | Baseline model as pseudo-reference, with human adjudication of disagreements | Non-inferiority relative to the baseline, bounded by baseline correctness |
| 5 | Minimum human-labelled seed set. Suggested size: 150 tasks (Assumption; the power check in Section 6 overrides it) | Full non-inferiority verdict on the labelled seed |

Tiers combine: tier 1 always runs, and higher tiers add stronger verdicts. If the dataset is too small for the chosen $\delta$, the verdict is "underpowered", which is distinct from "inconclusive".

### Diagram 8: Golden-dataset dependency tiers

```text
 Verdict strength
   ^
   |                                              +-------------------+
   |                                              | T5 Labelled seed  |
   |                                              | full non-inferior.|
   |                                +-------------+-------------------+
   |                                | T4 Baseline as pseudo-reference |
   |                                | + human adjudication            |
   |                  +-------------+---------------------------------+
   |                  | T3 Synthetic tasks, app-team reviewed          |
   |                  | conditional verdict                            |
   |     +------------+------------------------------------------------+
   |     | T2 Bootstrap from production logs (redacted)                |
   |     | descriptive + CIs                                           |
   | +---+-------------------------------------------------------------+
   | | T1 Reference-free metrics: cost, latency, schema, retries, diff |
   | | descriptive only                                                |
   +-+-----------------------------------------------------------------> Labelling effort
```

*Caption: Each tier adds labelling effort and a stronger verdict. Tier 1 needs no labels and always runs. Only tier 5 supports a full non-inferiority verdict, and tiers 3 and 4 are labelled as conditional.*

**What this means:** No golden dataset does not block evaluation, it caps the verdict. MIRA states the tier on every report so no one reads a descriptive result as proof of equivalence.

---

## 6. Experiment execution

**Purpose:** Run baseline and candidates under controlled, identical conditions.

### 6.1 Experiment types

| Experiment | Meaning | Reported as |
| --- | --- | --- |
| Drop-in Swap | Prompts and tool schemas unchanged | Migration cost floor, not ceiling capability |
| Adapted Swap | Bounded edits for the candidate under a fixed editor protocol | Quality gain per unit of adaptation effort |

Adaptation effort is lines of code changed plus tokens changed in prompt templates and tool schemas. Only syntax, formatting, and structural schema changes required by the new provider are allowed. Adding task logic, examples, or behavior changes is forbidden. If an adapted prompt also changes the baseline's prompt, the baseline is re-run.

### Diagram 9: Execution flow and state machine

```text
 EXECUTION FLOW
 Admin triggers frozen experiment
   -> Hard gates per candidate ----fail----> reject model, show reasons
   -> Power check and (gateway-route) translation-fidelity probe
   -> Expand jobs: model x task x repeat
   -> FOR EACH job (parallel, queued, round-robin across models):
        1 inject model + parameters   2 replay app on the task
        3 capture trajectory          4 normalize to canonical trace
        5 run applicable evaluators   6 store trace + scores
   -> All jobs done and coverage threshold met?
        no: retry failed jobs, mark Partial        yes: continue
   -> Human review sampling (Section 9) -> aggregate -> report

 STATE MACHINE
 Draft -> Validated -> Frozen -> Running -> Evaluating -> Aggregating -> Reported
   |          |          |          |            |
 Rejected  (immutable)   +--------> Failed <---- Partial (resumable)
 Reported --(manual revision by admin)--> New Draft (v+1)
```

*Caption: The flow expands one frozen experiment into jobs for every model, task, and repeat. Each job runs the same six steps. The state machine shows that a frozen experiment is immutable, failed jobs are resumable, and a revision creates a new version.*

### 6.2 Run control

| Control | Rule |
| --- | --- |
| Pinned model versions | Dated model versions pinned. The returned model identifier and `system_fingerprint` are logged. |
| Round-robin scheduling | Task 1 on A, B, C, then task 2 on A, B, C. Never one model's batch at a time, so time-of-day latency and rate limits apply equally. |
| Parameter-support map | The Model Catalogue lists which parameters each model supports. Dropped or approximated ones (for example `seed`, `top_p`) are listed in the report header as unequal treatment. |
| Temperature | Documented per experiment. At temperature 0, repeats add little information, so reliability claims need a stated rationale or temperature above 0. |

### 6.3 Power check

Before freezing, MIRA computes the minimum task count (or the achievable $\delta$) from pilot variance, $\delta$, and significance level $\alpha$. Too few tasks gives the verdict "underpowered".

### 6.4 Tool execution policy

| Tool class | Declared mode | Rule |
| --- | --- | --- |
| Read-only | `pure-read-live` | Called live against a frozen data snapshot |
| Idempotent | `recorded-with-strict-miss-fail` | Replayed from recordings. A cache miss fails that trajectory path. Fuzzy matching is banned. |
| Stateful | `sandboxed-stateful` | App team supplies `restore_snapshot`, run before every run. No hook, no evaluation. |

Cache-miss rate is a primary diagnostic and a validity measure for trajectory comparison.

### 6.5 Gateway-route rules

- The baseline passes through the same gateway at the same version as the candidates, so the gateway cancels in the paired difference.
- One native-baseline arm per app measures the gateway's own latency and error overhead.
- A 50-item translation-fidelity probe (nested schemas, parallel tool calls) runs per candidate before the main experiment. If the gateway mangles the schema, the run fails early.

**What this means:** The experiment changes only the integration bundle. Dataset, prompts, tools, and evaluator versions are frozen, so score differences can be attributed to the bundle. Scheduling and tool policy remove noise that is not about the model.

---

## 7. Metric computation

**Purpose:** Define how each metric is computed, at which stage, and from which data.

Evaluators read only the canonical trace (Section 4.3), never the app code. This is why MIRA works without knowing the app's architecture. Architecture knowledge comes from the App Profile (Section 3.1) and the trace structure. Unknowns become "unattributed" (Section 4.4) or "Not applicable" (Section 2.4).

### 7.1 Stage at which each metric is computed

| Stage | Unit | Example metrics | Data needed |
| --- | --- | --- | --- |
| Span | One LLM or tool call | Tokens, span cost, span latency, argument validity | Single span |
| Step / sub-task | An agent's responsibility or a tool call | Tool selection correctness, handoff correctness, sub-task success, exclusive and inclusive cost | Subtree of spans, expected spec if available |
| Run | One task, one model, one repeat | Outcome success, clean success, end-to-end latency, TTFT, critical path, run cost, retries, peak context | Full trace of the run |
| Task | One task across repeats | First-attempt vs eventual success, pass@k, pass^k, mean cost | All repeats of the task |
| Experiment | All tasks, one model, vs baseline | CPST, p50/p95, paired difference, non-inferiority, Pareto | All task results |

### 7.2 Success

| Level | Definition |
| --- | --- |
| Outcome success | Final answer meets the criteria (match, or rubric score above a threshold) and hard constraints hold (schema, policy) |
| Clean success | Outcome success, and no trajectory violations: right tools, valid arguments, retries within limit $r_{max}$ (Assumption: default 0 for argument errors) |
| Sub-task success | The step's own check passes (correct tool, valid arguments, sub-agent output meets expectation) |

$$
\text{Success rate} = \frac{\text{successful runs}}{\text{all runs}}
$$

The gap between outcome and clean success is the hidden regression. End-to-end success is not the product of sub-task successes, because agents recover from errors. Keep them as separate measures. Worked example: Diagram 7 has outcome success (correct answer) but not clean success (invalid argument, one retry).

### 7.3 Overall task result including retries

| Measure | Definition |
| --- | --- |
| First-attempt success | The task succeeded with no retry span in the run |
| Eventual success | The task succeeded after any retries |
| Retries per task | Count of retry spans per run, averaged over repeats |
| Success by attempt number | Share of tasks first succeeding on attempt 1, 2, 3 |
| Cost of retries | Cost of spans flagged as retries, as a share of run cost |

$$
\text{Retry cost share} = \frac{\text{cost of retry spans}}{\text{total run cost}}
$$

**Counting rules.** A retried tool call is a retry span linked to the failed span. At step level it is attempt 2 of that step. At task level the run is still one run: it counts as one eventual success and, because a retry occurred, zero first-attempt success. A regenerated answer in replay (app-internal re-ask) is also a retry inside the run. A user regeneration in imported production data is a separate interaction, linked to the earlier one, and is used only for triage (Section 9). All attempts count in cost and latency.

### 7.4 Cost

| Measure | Formula |
| --- | --- |
| LLM span cost | $\text{in\_tokens} \times p_{in} + \text{out\_tokens} \times p_{out}$ (plus cached and reasoning token pricing where applicable) |
| Tool span cost | Declared per-call cost, otherwise 0 |
| Run cost | Sum of span costs, including failed attempts and retries |
| Self-hosted token cost | $\approx \dfrac{\text{GPU cost per hour}}{\text{tokens served per hour at measured load}}$ (reported as a range under low and high utilization) |

$$
CPST = \frac{\text{total cost of ALL runs, including failed ones}}{\text{number of successful runs}}
$$

$$
CPSS_j = \frac{\sum \text{cost of spans attributed to sub-task } j}{\text{runs where sub-task } j \text{ succeeded}}
$$

Exclusive cost counts the node's own spans. Inclusive cost adds its subtree. **Invariant:** the sum of exclusive costs over all sub-tasks equals the run cost. If not, some spans are unattributed and MIRA flags it.

Costs use a versioned price table with effective dates. Reports show List Cost and Effective Cost (negotiated rates).

| Model | Runs | Total cost | Successes | Cost per run | CPST |
| --- | ---: | ---: | ---: | ---: | ---: |
| A (baseline) | 100 | $15.00 | 75 | $0.150 | $0.200 |
| B (candidate) | 100 | $9.00 | 40 | $0.090 | $0.225 |

B is 40% cheaper per run but costs more per success.

### 7.5 Latency

| Measure | Definition | Retries and parallel branches |
| --- | --- | --- |
| End-to-end latency | Request received to final output complete | Includes every retry (wall-clock) |
| User-perceived TTFT | Request received to first streamed token of the final answer | Needs a streaming interface |
| Per-LLM-call TTFT | Call start to first token of that call | Diagnoses slow models |
| Critical path | Longest chain of dependent spans | Parallel branches count as their maximum, not their sum |
| p50 / p95 | Percentiles across all runs, including failed runs | Retried runs count once, at their total time |

$$
T_{critical} = \max_{\text{paths}} \sum_{\text{spans on path}} t_{span}
$$

### 7.6 Reliability and context endurance

$$
pass@k = 1 - \frac{\binom{n-c}{k}}{\binom{n}{k}} \qquad pass^k = \frac{\binom{c}{k}}{\binom{n}{k}}
$$

Here $n$ is runs per task, $c$ is successes, and $n \ge k$. Dataset values average the per-task values. With success probability $p$ per attempt: $pass@k = 1-(1-p)^k$ and $pass^k = p^k$.

| $c$ (of $n=5$) | $k$ | pass@k | pass^k |
| ---: | ---: | ---: | ---: |
| 4 | 3 | 1.00 | 0.40 |
| 2 | 2 | 0.70 | 0.10 |

pass@k measures capability (best case). pass^k measures reliability, which agents acting on real systems need.

**Context endurance.** Peak utilization is $\text{peak context tokens} / \text{model limit}$. Tasks are stratified into short, medium, and long context buckets, and success is compared per bucket. A model that holds at 20K tokens but degrades at 100K shows up as a slope.

### 7.7 Statistics

| Need | Method |
| --- | --- |
| Uncertainty on any metric | Task-level (cluster) bootstrap confidence interval. Resample tasks, not runs, because repeats are correlated. |
| Candidate vs baseline | Paired bootstrap on per-task differences |
| Binary per-task outcome | McNemar's test |
| Continuous or ordinal scores | Wilcoxon signed-rank |
| Multiple candidates | Holm correction on primary hypotheses only |
| Proportions | Wilson interval |
| Subgroups (capability, document size, context bucket) | Descriptive with CIs, no verdict |

**Non-inferiority.** The Execution Team sets margin $\delta$ before any run (for example 3 percentage points). With $D = \text{candidate} - \text{baseline}$ and the lower confidence bound $L_D$:

$$
\text{Non-inferior} \iff L_D > -\delta
$$

Example: $D = -1.0$ points, 95% CI $[-2.4, +0.4]$, $\delta = 3$. Since $-2.4 > -3$, the candidate is non-inferior.

**Pareto dominance.** Model A dominates B if A is at least as good on every objective and strictly better on one. The Pareto front is the non-dominated set. An optional weighted score $\sum_i w_i \cdot \text{normalize}(m_i)$ is shown beside it, never instead of it.

### Diagram 10: Metric computation with retries (worked numbers)

```text
 Run r17, task T42   (retry budget r_max = 0)
 Span (attempt)             Cost    Time   Notes
 Router (Supervisor)        $0.01   1 s
 Research (search_policy)   $0.02   3 s   parallel branch A
 Retrieval check            $0.01   2 s   parallel branch B
 Analysis query_db  try 1   $0.02   1 s   invalid date  -> FAIL
 Analysis query_db  try 2   $0.03   1 s   RETRY span    -> OK
 Response                   $0.01   2 s
 ------------------------------------------------------------------
 Run cost          = 0.01+0.02+0.01+0.02+0.03+0.01     = $0.10
 Exclusive sum     = 0.10                              -> invariant holds
 Retry cost share  = 0.03 / 0.10                       = 30%
 Critical path     = 1 + max(3,2) + (1+1) + 2          = 8 s
 Sum of spans      = 1+3+2+1+1+2                       = 10 s (not the latency)
 Outcome success = 1   Clean success = 0   First-attempt success = 0
 Retries = 1   Step attempt: query_db succeeded on attempt 2
 Task T42 (5 repeats: 2 succeeded, 2 clean)  -> pass^2 = 0.10 for c=2
```

*Caption: One run shows how all attempts add to cost, how parallel branches shorten the critical path, and how a retry lowers clean and first-attempt success while the outcome still succeeds. The task-level reliability line shows how repeats feed pass^k.*

**What this means:** Every metric is computed at one named stage from the canonical trace, so each number can be traced to spans. Retries are never hidden: they raise cost and latency and lower clean success. Quality, cost, and latency stay separate until the Pareto view.

---

## 8. Metrics catalogue

**Purpose:** List every metric with its class, the lowest fidelity it needs, its dependency on golden data, and where it is computed.

| Metric | Class | Minimum fidelity | Golden data | Stage |
| --- | --- | --- | --- | --- |
| Outcome success | Mandatory core | black-box | Needs (or approved proxy, Section 5.3) | Run |
| Clean success | Mandatory core | near-full | Optional | Run |
| pass@k, pass^k | Mandatory core | black-box | Needs | Task |
| Cost per successful task (CPST) | Mandatory core | black-box | Needs (success definition) | Experiment |
| End-to-end latency, p50/p95 | Mandatory core | black-box | None | Run, Experiment |
| TTFT | Mandatory core | black-box (streaming needed) | None | Run |
| Tool-argument validity | Mandatory core | decisions-only | None | Span |
| Retry rate | Mandatory core | near-full | None | Run |
| Cache-miss rate | Mandatory core | full | None | Run |
| First-attempt vs eventual success | Standard optional | near-full | Needs | Task |
| Tool selection precision and recall | Standard optional | decisions-only | Needs (expected spec) | Step |
| Argument accuracy | Standard optional | decisions-only | Needs | Step |
| Order score | Standard optional | near-full | Needs | Step |
| Handoff correctness | Standard optional | near-full | Needs | Step |
| Sub-task success, per sub-task cost | Standard optional | full | Optional | Step |
| Context endurance | Standard optional | near-full | None | Run, Task |
| Groundedness, relevance, completeness (judge) | Standard optional | black-box | Optional | Run |
| Faithfulness, answer relevance, context precision and recall (Ragas, DeepEval) | Standard optional | near-full (retrieval spans) | Partly | Run |
| Human expert rating | Standard optional | black-box | None | Run |
| Business outcomes (resolution, escalation, conversion) | Application-specific | n/a | Optional | Experiment |
| Custom rubrics and domain checks | Application-specific | Declared by metric | Declared by metric | Declared by metric |

**Rules**
- Core metrics are not removable.
- Optional metrics are chosen by the app owner.
- Application-specific metrics are defined by the owner and approved by the admin.
- Evaluator plugin contract: every evaluator declares a name, a version, and the fidelity it needs. It receives a canonical trace plus the task. It returns a score with a rationale.
- "Not applicable, never zero": if the trace lacks the data an evaluator needs, it returns "Not applicable".

**What this means:** The core set makes every app comparable. Optional and custom metrics add depth without breaking comparability. The fidelity column tells owners what extra integration buys.

---

## 9. Human feedback and retry-quality analysis

**Purpose:** Use human signals at the right weight, protect judge reliability, and treat retry counts honestly.

### 9.1 Feedback channels

| Signal | Reliability | How it feeds evaluation |
| --- | --- | --- |
| Expert review (rating, correction, preference) | High | Ground truth and judge calibration |
| User edit of an output | Medium | Candidate correction. Needs review before use. |
| Thumbs up or down | Low to medium | Triage and sampling priority |
| Regenerate or retry | Low | Triage only |

### 9.2 Judge safeguards and calibration gate

| Safeguard | Rule |
| --- | --- |
| Judge ensemble | 2 models from a family outside the candidates |
| Bias control | Answer order randomized. Explicit rubrics. |
| Calibration gate | Judge scores enter the verdict only if Cohen's $\kappa \ge 0.6$ against the human-labelled subset. Otherwise they are "Descriptive/Uncalibrated". |
| Boundary-bound calibration | Runs inside the app's data boundary, with cleared domain experts as labellers. The judge must hold the app's data-classification clearance. If no compliant judge exists, the judge is disabled and only deterministic evaluators and human review are used. |

### 9.3 Human review sampling

Stratified sampling within a labelling budget of 10-15% of runs (configurable). Runs where deterministic checks and the judge disagree are oversampled. A double-rated subset measures inter-rater agreement.

### 9.4 Quality vs retry

| Question | Answer |
| --- | --- |
| How does quality relate to retries? | Compare first-attempt success and eventual success (Section 7.3). The difference is quality that depends on retrying. |
| Can retry frequency indicate satisfaction? | Weak signal only. A regeneration may mean dissatisfaction, curiosity, or UI habit. |
| How is it used? | Triage and review sampling priority. Never ground truth. |
| Caveats | Retries depend on UI design. Production regenerations are not comparable across apps. Replay-path retries (agent-internal) are measured, user regenerations are not. |

**What this means:** Experts set the standard, and everything weaker only points the experts to where to look. A judge is trusted only after it agrees with humans. Retries are evidence of fragility, not proof of dissatisfaction.

---

## 10. Prompt governance and LLM catalogue

**Purpose:** Keep prompts and model settings flexible for experimentation and controlled for audit.

### 10.1 Prompts as versioned assets

| Capability | Meaning |
| --- | --- |
| Versioning | Every prompt or skill edit makes a new immutable version |
| History | Who changed what, when, and why |
| Approvals | The admin approves versions before use in an experiment |
| Audit trail | Every change is logged (Section 11) |
| Trend view | Prompt version vs evaluation results across experiments over time |
| Change rule | A changed prompt means a new experiment. A changed adapted prompt re-runs the baseline (Section 6.1). |

### 10.2 LLM catalogue: flexibility with governance

| Aspect | Configurable (by admin per experiment) | Governed (by platform engineers and policy) |
| --- | --- | --- |
| Model | Choose among approved models | Approved list, versions, owners |
| Temperature, top-p, max tokens | Set within range | Allowed ranges per model |
| Provider settings | Endpoint, region within policy | Residency and classification matching |
| Pricing | Select price table version | Versioned price table with effective dates |
| Approval | Request a new model | Approval roles (Assumption: Platform Engineer proposes, Execution Team approves) |

### Roles

| Role | Responsibility |
| --- | --- |
| Application Team | Owns the app, dataset, ground truth |
| Execution Team | Defines gates, shortlists 2-3 candidates, sets $\delta$ |
| Admin | Configures and runs experiments, approves prompts and custom metrics |
| Human Reviewer | Judges quality where automation is weak |
| Decision Owner | Decides on migration, outside MIRA |
| Platform Engineer | Operates the platform, maintains adapters and the catalogue |

**Boundary.** MIRA's catalogue is for evaluation. Exposing a catalogue inside each running app is an optional integration, not a core component (Assumption).

**What this means:** Admins can try settings freely inside governed ranges. Every setting that reaches a run is approved, versioned, and logged. The catalogue never controls production runtime.

---

## 11. Lineage and traceability

**Purpose:** Make every result reproducible and auditable.

### 11.1 Lineage record attached to every result

| Item | Recorded value |
| --- | --- |
| Model | Provider and name |
| Model version and fingerprint | Pinned version and returned `system_fingerprint` |
| Parameters | Temperature, top-p, max tokens, others |
| Prompt version | Prompt and skill version IDs |
| App version | Application build or commit |
| Adapter version | Provider adapter package and version |
| Evaluator version | Name and version of each evaluator |
| Dataset version | Immutable dataset version |
| Injection mode | Axis A value |
| Fidelity level | Axis B value |

### Diagram 11: Lineage entity-relationship

```text
 Application 1---* Dataset 1---* Task
      |                             |
      1                             |
      |                             |
      *                             *
  Experiment 1---* Run *---1 Task
      |            |
      |            +---* Span          (trace tree)
      |            +---* Score *---1 Evaluator (versioned)
      |            +---* HumanReview
      |
      +-- frozen: dataset version, prompt/skill versions,
      |           evaluator versions, hyperparameters
      +---* Model (baseline + candidates) *---1 Model Catalogue entry
      +---1 Report 1---* Comparison Finding

 Every box is immutable and versioned. Nothing is overwritten.
```

*Caption: An application owns datasets and tasks and runs experiments. An experiment holds runs, and each run holds spans, scores, and human reviews. The experiment freezes its versions, so a changed prompt creates a new experiment.*

### 11.2 How lineage is used

| Use | How |
| --- | --- |
| Audit | Trace any report finding back to runs, spans, and versions. See who approved each input. |
| Debugging | Find the first span where a candidate diverged, with the exact prompt, parameters, and adapter. |
| Evaluation | Prove two scores are comparable (same dataset, evaluators, and injection mode) before pairing them. |

**What this means:** A result without its lineage record is not shown. This is what lets a review board trust, repeat, and challenge any number in a report.

---

## 12. Portability and deployment

**Purpose:** Run one design on different platforms and clouds without changing the architecture.

### 12.1 Platform assumptions

| Term | Assumption - to be confirmed |
| --- | --- |
| KIP | Assumed to be the organization's internal container application platform, with managed network boundaries and platform-provided services. Not defined in the source. |
| GCP | Assumed to be Google Cloud Platform. The source wrote "GSP" without definition. |
| Azure | Microsoft Azure |

MIRA uses containers and a thin abstraction layer, so KIP and non-KIP apps and GCP and Azure share one design.

### 12.2 Abstraction table

| Capability | MIRA depends on | Swapped per platform |
| --- | --- | --- |
| Compute | Container runtime | Docker Compose (v1), Kubernetes (v2), on KIP, GCP, or Azure |
| Queue | Job queue (Assumption: Celery with Redis in v1) | Managed or self-hosted queue behind the same interface |
| Storage | Relational store plus S3-compatible object store | PostgreSQL and MinIO, or the cloud's equivalents |
| Secrets | Secret store with short-lived credentials | Vault, or the platform's secret manager |
| Identity | OIDC single sign-on and RBAC | Enterprise identity provider |
| Model endpoints | Provider adapter layer and gateway | OpenAI/Azure OpenAI, Anthropic, Google, self-hosted |

### Diagram 12: Deployment portability

```text
 +------------- CONTROL PLANE (central) -------------+
 | Console | Registries | Gates | Orchestrator | DB  |
 +----------------------+---------------------------+
                        | signed job manifests (workers pull; no inbound)
          +-------------+-----------------+----------------+
          v                               v                v
 +-------------------+   +-----------------------+  +-----------------+
 | Worker: KIP       |   | Worker: GCP boundary  |  | Worker: Azure   |
 | (in-boundary)     |   | (in-boundary)         |  | boundary        |
 | app + replay +    |   | app + replay +        |  | app + replay +  |
 | normalize + eval  |   | normalize + eval      |  | normalize + eval|
 +---------+---------+   +-----------+-----------+  +--------+--------+
           | signed traces, scores   |                       |
           +-------------------------+-----------------------+
                                     v
                 Data layer (metadata + object store)
 Short-lived, job-scoped credentials from the secret store.
 Residency matching: judge and candidate endpoints must meet the app's classification.
```

*Caption: One central control plane issues signed job manifests. Workers inside each boundary (KIP, GCP, or Azure) pull jobs, so no inbound connection to the app network is needed. Workers return signed traces and scores using short-lived credentials.*

### 12.3 Security controls

| Control | Rule |
| --- | --- |
| Residency matching | A judge or candidate may process an app's data only if its endpoint meets that classification |
| Pull model | Workers poll for jobs. No inbound connections. |
| Signed manifests | Jobs carry hashes of frozen artifacts. Workers return signed traces. |
| Credentials | Vault issues short-lived, job-scoped credentials |
| Data handling | Encrypted at rest, redacted before storage, limited retention, RBAC-only viewing |

### 12.4 Deployment profiles

| Profile | Components | Use |
| --- | --- | --- |
| v1 Minimal Validating Deployment | Docker Compose, PostgreSQL, Celery with Redis, local disk or MinIO | 5-7 concurrent evaluations. Use this subset first. |
| v2 Enterprise Scaling Profile | Kubernetes, Argo Workflows or Temporal, KEDA autoscaling, Vault, in-boundary remote workers | When scale or isolation requires it |

**What this means:** The architecture is fixed and the platform pieces are swappable. Sensitive data never leaves its boundary, because the worker moves to the data. Start with v1 and add v2 only when needed.

---

## 13. Report, requirement traceability, scope, risks, glossary

**Purpose:** Show what MIRA delivers, prove every requirement is covered, and state limits and risks.

### 13.1 Report contents

| Element | Content | Defined in |
| --- | --- | --- |
| X vs Y vs Z comparison | Baseline and candidates across mandatory core and chosen metrics | Sections 7, 8 |
| Pareto view | Non-dominated models over quality, cost, latency. Optional weighted score beside it. | 7.7 |
| Non-inferiority statement | Per candidate, against the pre-set $\delta$, with CI and data tier | 5.3, 7.7 |
| Trace diff | Where each candidate diverged from the baseline | 4 |
| Stamps | Injection mode, fidelity level, data tier, coverage score | 2, 4, 5 |
| Unequal-treatment header | Parameters dropped or approximated per model | 6.2 |
| Gateway warning | Shown for `gateway-route`: "Evaluated via API translation gateway. Trajectory errors may stem from gateway schema translation rather than native model capabilities." | 2.3, 6.5 |
| Unit under test | Integration bundle and adapter mutation rate | 2.3, 3.5 |

### 13.2 Requirement traceability

**Capabilities**

| # | Capability | Section | Status |
| --- | --- | --- | --- |
| 1 | Platform-independent deployment across KIP, non-KIP, GCP, Azure | 12 | Covered (KIP and GCP definitions are Assumptions) |
| 2 | Intermediate and final output evaluation | 4, 7, 8 | Covered |
| 3 | Human-in-the-loop | 9 | Covered |
| 4 | Prompt and model version management | 10, 11 | Covered |
| 5 | Lineage and traceability | 11 | Covered |
| 6 | Configurable model catalogue and runtime parameters | 10 | Partial (evaluation only; app runtime catalogue is optional) |
| 7 | Standard and application-specific metrics | 8 | Covered |
| 8 | Quality using retries, corrections, feedback | 7.3, 9 | Covered |

**Key considerations**

| Consideration | Section | Status |
| --- | --- | --- |
| Framework portability | 2, 4, 12 | Covered |
| Evaluation data strategy | 5 | Covered |
| Human feedback | 9 | Covered |
| Evaluation scope (intermediate vs end-to-end, component vs application) | 7.1, 7.2 | Covered |
| Prompt governance | 10 | Covered |
| Quality vs retry | 7.3, 9.4 | Covered |
| LLM catalogue | 10 | Partial (platform catalogue only) |
| Model traceability | 11 | Covered |
| Evaluation metrics | 7, 8 | Covered |

### 13.3 Out of scope

| Item | Note |
| --- | --- |
| Shadow traffic, live production monitoring | Future work |
| CI/CD gates, automatic deployment, automatic migration | Not MIRA's role |
| Automated prompt optimization (for example DSPy) | Future extension point |
| Fine-tuning or training models | Out of scope |
| Runtime model routing and in-app catalogue | Optional integration (Assumption) |

### 13.4 Top 5 risks

| Risk | Mitigation |
| --- | --- |
| Adapter or gateway translation distorts results | Adapter-isolation check, native-baseline arm, translation-fidelity probe, unit under test in every report |
| No or small golden dataset | Data tiers with capped verdicts, power check, "underpowered" status |
| Unreliable LLM judge | Calibration gate, cross-family ensemble, human review |
| Tool replay misses and stateful side effects | Strict miss-fail, cache-miss rate, mandatory `restore_snapshot` |
| Provider nondeterminism and drift | Pinned versions, fingerprint logging, round-robin scheduling, repeats |

### 13.5 Glossary

| Term | Meaning |
| --- | --- |
| App Profile | Record of an app's integration, agents, tools, prompts, and data class (3.1) |
| Integration bundle | Model plus its framework provider adapter, the unit under test (2.3) |
| Injection mode | How the model is swapped (Axis A) |
| Fidelity level | How much of the run MIRA observes (Axis B) |
| Canonical trace | One framework-neutral trace format (4.3) |
| Clean success | Correct outcome with no trajectory violations (7.2) |
| CPST | Cost per successful task (7.4) |
| pass^k | Probability that all k attempts succeed (7.6) |
| Non-inferiority margin $\delta$ | Largest acceptable quality loss, set before runs (7.7) |
| Pareto front | Set of models not dominated on quality, cost, and latency (7.7) |

**What this means:** Every requirement maps to a section, with two partial items stated openly. The report states its own limits. Known risks each have a named control.
