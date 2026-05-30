# Module 7 — Observability and Evals

---

## Tracing

**Status:** ⬜ Not Started

**Definition:** Tracing captures the full execution path of an LLM request — every prompt, tool call, model response, and latency measurement — as a structured trace with parent/child spans. Traces enable debugging of multi-step agent workflows, latency profiling, and cost attribution at the individual request level.

**Key Mental Model:** Tracing is an X-ray of a request's journey — you see every step it took, how long each step took, what went in and what came out, and exactly where something went wrong.

**How It Works:**
- A trace is a tree of spans. The root span represents the entire request (e.g., a user query to an agent). Child spans represent sub-operations: the retrieval step, the LLM call, each tool invocation. Each span records start time, end time, input, output, token counts, and any custom metadata (user ID, feature name, model used).
- Instrumentation is added to the application code via a tracing SDK. Langfuse, LangSmith, and Arize Phoenix provide Python decorators and context managers that automatically create spans. For LangChain and LangGraph, callback-based auto-instrumentation captures all LLM and tool calls with zero manual instrumentation code.
- Spans are created hierarchically by nesting context managers. The outer span creates a trace ID; all inner spans inherit it. When spans are sent to the tracing backend, the trace ID links them into a tree — enabling the "waterfall" view showing which steps ran in sequence vs in parallel and their relative durations.
- Metadata tagging is essential for making traces queryable. Every span should include: session ID (to group multi-turn conversation spans), user ID (for cost attribution), feature name (for per-feature metrics), and model name. Without tags, traces are only useful for debugging individual requests — with tags, they enable aggregate analytics.
- Sampling reduces storage costs in high-volume production systems. Trace 100% of errors and slow requests (latency above p99 threshold), but sample a fraction of normal requests. Head-based sampling decides at trace start; tail-based sampling decides after the trace completes (keeping slow/error traces regardless of the sampling rate).

**Common Misconceptions:**
- Logging prompt/response pairs is sufficient observability — logs lack structure and context; traces capture the causal tree of a multi-step agent execution and enable queries like "show me all requests where the tool call failed".
- Tracing only matters for debugging — traces are essential for performance profiling, cost attribution, regression testing, and demonstrating audit compliance.

**Interview Answer Skeleton:**
- **What it is:** Hierarchical instrumentation that records every operation in an LLM application as a tree of parent/child spans — capturing inputs, outputs, latencies, token counts, and custom metadata for every step from user request to final response.
- **Why it matters / trade-offs:** Without traces, debugging a failing multi-step agent is like debugging a program with no stack trace. Traces also enable cost attribution, latency profiling, and feeding labelled examples into eval datasets. The engineering cost is instrumentation setup and storage — mitigate with sampling for high-volume systems.
- **Example or context:** Instrument a RAG pipeline with three spans: root span (user query), child span for retrieval (captures query vector, top-k chunks, retrieval latency), child span for LLM generation (captures full prompt, response, tokens used, and cache hit status). Query the tracing backend for traces where retrieval latency exceeds 500ms to identify index performance issues.

**Free Resources:**
- [Langfuse Documentation](https://langfuse.com/docs) — Trace instrumentation, span construction, metadata tagging, and observability setup for LLM applications
- [Arize Phoenix Documentation](https://docs.arize.com/phoenix) — LLM tracing, span-level evaluation, and production monitoring with RAG-specific trace analysis

---

## LLM-as-Judge

**Status:** ⬜ Not Started

**Definition:** LLM-as-judge uses a separate, often more capable LLM to evaluate the output of another LLM. The judge model scores responses on criteria like accuracy, helpfulness, coherence, or safety. This scales evaluation beyond human annotation while being more flexible than rule-based metrics.

**Key Mental Model:** LLM-as-judge is like peer review — a qualified reviewer assesses work quality using defined criteria. It's not perfect but is far more scalable than waiting for expert human review on every sample.

**How It Works:**
- The judge prompt provides the original user question, the LLM response being evaluated, optionally the source context (for grounding checks), and an explicit rubric specifying what to evaluate and how to score it. The judge generates a score (numeric or categorical) and usually a rationale.
- Rubric design is the most important factor in judge reliability. Vague criteria like "is this a good response?" produce unreliable scores. Specific criteria like "does the response answer the user's question without adding information not present in the provided context?" produce more consistent, interpretable scores.
- Judge calibration involves running the judge on a set of examples that have been human-labelled, then computing agreement metrics (Cohen's kappa, Pearson correlation). A well-calibrated judge should have strong agreement (>0.7 kappa) with human labels on the calibration set. If calibration is poor, iterate on the rubric before using the judge at scale.
- Positional bias and length bias are known LLM judge failure modes. Judges tend to prefer responses listed first (when comparing two responses) and longer responses over shorter ones — regardless of quality. Mitigate by randomising the order in comparative evaluations and including length-penalising rubric criteria.
- The judge model should be more capable than the model being evaluated, or at least specialised for evaluation. Using the same model to evaluate itself introduces self-preference bias. Common practice: use Claude Opus or GPT-4o as judges for evaluating Claude Sonnet or smaller model outputs.

**Common Misconceptions:**
- LLM judges are objective — LLM judges have biases (preferring longer responses, their own outputs, or confident tone); calibrate judges against human labels and use structured rubrics to reduce bias.
- Any LLM works as a judge — judge quality depends on the model's ability to follow a rubric and reason about quality; use the strongest available model for judging, even if it's too slow/expensive for production.

**Interview Answer Skeleton:**
- **What it is:** An evaluation pattern where a capable LLM is prompted with a rubric to score another LLM's outputs — enabling scalable automated evaluation across large sample sizes where human annotation is impractical.
- **Why it matters / trade-offs:** LLM-as-judge scales continuous evaluation to 100% of production traffic and enables rapid iteration on prompt changes. Biases (length preference, self-preference, positional bias) require explicit mitigation and calibration against human labels before trusting scores operationally.
- **Example or context:** Judge prompt for RAG faithfulness: "You are evaluating whether the following response is supported by the provided context. Score 1 if every claim in the response can be attributed to the context, 0 if any claim lacks support. Context: [context]. Response: [response]. Score: [1/0]. Rationale: [reason]." Run on 500 examples labelled by a human annotator; target >0.8 kappa before using the judge in CI/CD gates.

**Free Resources:**
- [Langfuse Documentation](https://langfuse.com/docs) — LLM-as-judge evaluation setup, scoring configuration, and rubric templates
- [Arize Phoenix Documentation](https://docs.arize.com/phoenix) — Built-in LLM evaluators with calibration tools and judge performance analytics

---

## Custom Evals

**Status:** ⬜ Not Started

**Definition:** Custom evals are evaluation datasets and metrics designed for a specific application's requirements — not generic benchmarks. They consist of representative input examples, expected outputs or rubrics, and scoring logic (exact match, semantic similarity, LLM judge, or domain-specific rules).

**Key Mental Model:** Custom evals are like acceptance tests for LLM systems — they test the specific behaviour your application requires, not general intelligence. They are the difference between "this model is smart" and "this model works for my users".

**How It Works:**
- A custom eval dataset is built by sampling from production traffic (real user queries), adding adversarial examples (edge cases, known failure modes), and including regression examples (inputs that previously caused failures and must not regress). The dataset should cover the full distribution of tasks the application handles.
- Each eval example consists of: an input (the prompt or query), an expected output or rubric (what constitutes a good response), and a scoring function (how to compute a numeric quality score from the model's response). The scoring function is the most important design decision.
- Scoring functions range from deterministic (exact string match for extraction tasks, unit test execution for code generation) to approximate (embedding cosine similarity to a reference answer) to model-based (LLM-as-judge with a rubric). Use the most deterministic scorer that captures the quality dimension — reserve LLM judges for genuinely subjective dimensions.
- Eval execution runs the full dataset through the current version of the application (not just the LLM in isolation), capturing the aggregate score distribution. Individual low-scoring examples become debugging targets; the aggregate score is the deployment gate metric.
- Dataset maintenance is ongoing: when new failure modes are discovered in production (via user feedback, error monitoring, or drift detection), add examples of those failure modes to the eval set. The eval set is a living document that grows more representative of real failure patterns over time.

**Common Misconceptions:**
- Generic benchmarks are sufficient for evaluating production LLMs — generic benchmarks test general capability; a model that scores well on MMLU may still fail your specific task distribution.
- Evals are a one-time setup — eval datasets must evolve as your application evolves and as you discover new failure modes in production.

**Interview Answer Skeleton:**
- **What it is:** Application-specific test suites pairing representative input examples with expected outputs and domain-appropriate scoring functions — providing a reliable, repeatable quality signal for the specific tasks the application must perform.
- **Why it matters / trade-offs:** Custom evals are the foundation of safe iteration. They quantify quality improvements and regressions in terms that matter for the application, not for generic benchmarks. Building and maintaining them requires ongoing investment but is the only way to move fast without breaking production quality.
- **Example or context:** Custom eval for a legal document summarisation system: 150 examples (100 from production traffic, 30 adversarial with complex clause structures, 20 regression examples from known past failures). Scoring: semantic similarity to expert-written reference summaries for fluency, plus a coverage checker ensuring all key clauses are mentioned. Target: mean similarity > 0.85, coverage > 90%. Run in CI on every prompt change.

**Free Resources:**
- [Langfuse Documentation](https://langfuse.com/docs) — Custom dataset creation, eval run execution, scoring pipeline setup, and dataset versioning
- [Arize Phoenix Documentation](https://docs.arize.com/phoenix) — Eval dataset management, custom metric definition, and production eval integration

---

## Regression Testing

**Status:** ⬜ Not Started

**Definition:** LLM regression testing runs the same eval suite against a new model version, prompt change, or retrieval system update and compares scores against a baseline. This catches quality degradations before they reach production — ensuring that improvements in one area don't silently degrade another.

**Key Mental Model:** LLM regression testing is like a CI/CD test suite for LLM behaviour — every change runs the full eval suite, and you can only deploy if the score doesn't degrade beyond an acceptable threshold.

**How It Works:**
- The regression test runs the full custom eval dataset against the new version of the system (new prompt, new model, new retrieval configuration) and computes the aggregate score. The score is compared to the baseline score stored from the previous approved version.
- A score delta threshold determines pass/fail: if the new score is within -2% of baseline on all metrics, the change is approved. Tighter thresholds (0%) prevent any degradation but slow iteration; looser thresholds allow faster iteration but risk quality creep. Calibrate thresholds to the stakes of the application.
- Beyond aggregate scores, regression testing checks per-category performance. A change might improve the aggregate score by 3% while degrading performance on one query category by 15%. Slice analysis (score broken down by query type, user tier, or topic) catches these hidden regressions that aggregate metrics mask.
- Regression evals run as CI/CD pipeline steps triggered on pull request. The eval runner calls the LLM with the new system configuration, scores each example, computes the aggregate, and posts the score comparison as a PR comment. Deployment is gated on the CI check passing.
- Eval cost management: running 500 eval examples × 2 LLM calls (generation + LLM judge) per PR is expensive. Optimise by running a fast subset (50 high-signal examples) on every PR and the full suite only when the fast subset shows a score change. Cache eval outputs for deterministic runs — only re-run examples affected by the change.

**Common Misconceptions:**
- A manual review of a few responses is sufficient for detecting regression — LLM regressions are often subtle and appear only on specific input patterns; systematic eval coverage is needed.
- Regression testing is only needed for model upgrades — prompt changes, retrieval changes, and context window adjustments can all cause regressions; test every change.

**Interview Answer Skeleton:**
- **What it is:** Automated evaluation pipeline that runs a versioned eval dataset against every system change, computes score deltas versus baseline, and gates deployment on a configurable quality threshold — CI/CD for LLM behaviour.
- **Why it matters / trade-offs:** LLM systems are brittle in ways that unit tests cannot catch. A prompt tweak that improves one category often regresses another. Regression testing is what enables confident iteration. The trade-off is eval cost — optimise with tiered eval suites (fast subset per PR, full suite per merge to main).
- **Example or context:** A RAG chatbot CI/CD pipeline: every PR triggers the fast eval suite (50 examples, ~30 seconds). If fast suite score drops > 2% vs main branch, block merge. On merge to main, run the full suite (500 examples). Weekly, run the full suite against the live production system to detect drift from provider-side model updates.

**Free Resources:**
- [Langfuse Documentation](https://langfuse.com/docs) — Eval runs, score comparison, dataset versioning, and CI/CD integration for LLM regression testing
- [Arize Phoenix Documentation](https://docs.arize.com/phoenix) — Evaluation run management, baseline comparisons, and slice analysis for regression detection

---

## Drift Detection

**Status:** ⬜ Not Started

**Definition:** Drift detection monitors production LLM behaviour over time for gradual quality degradation — caused by model updates (silent model provider changes), input distribution shifts (user behaviour changing), or retrieval index staleness. It triggers alerts when quality metrics drop below thresholds.

**Key Mental Model:** Drift detection is like monitoring a river for changes in water quality over time — the river looks the same day to day, but systematic sampling reveals gradual changes that eventually matter.

**How It Works:**
- The drift detection system continuously samples a fraction of production requests (typically 1–5%), runs them through the quality scoring pipeline (LLM-as-judge or deterministic scorers), and writes the scores to a time-series store. This creates a continuous quality signal at production scale without evaluating every request.
- Rolling averages smooth out sample noise. Compute the 7-day rolling mean quality score for each tracked segment (topic category, user tier, query type). Compare the current day's mean against the 30-day historical baseline for that segment — a drop of more than N standard deviations triggers an alert.
- Input distribution drift is detected separately from output quality drift. Track statistical properties of inputs over time: token length distribution, vocabulary diversity (new terms entering the distribution), and topic cluster shifts (using embedding clustering on samples). Input drift often precedes output quality drift, giving earlier warning.
- Retrieval index staleness causes a specific drift pattern: RAG system quality degrades as the underlying data changes and the index is not updated. Track context relevance scores (from the tracing system) over time — declining relevance indicates the index is falling behind the real-world data it should reflect.
- Alert design requires careful threshold calibration. Too sensitive and the alert fires on normal sample noise; too loose and real degradations go undetected. Use control charts (UCL/LCL based on historical variance) rather than fixed absolute thresholds. Alert only on sustained drops (e.g., 3 consecutive days below threshold) to reduce false positives.

**Common Misconceptions:**
- Production LLM quality is stable once deployed — model providers update models silently; input distributions shift as user bases grow; quality degrades over time without detection.
- A fixed alert threshold on overall quality is sufficient — drift detection needs segmented monitoring (by topic, user type, or task) to catch subtle degradations that average out in aggregate metrics.

**Interview Answer Skeleton:**
- **What it is:** Continuous monitoring of production LLM quality via sampled scoring, time-series aggregation, and statistical drift detection — alerting when quality drops below rolling historical baselines, segmented by query type and user segment.
- **Why it matters / trade-offs:** LLM quality degrades silently in production from provider model updates, input distribution shifts, and retrieval index staleness. Without drift detection, quality regressions go unnoticed until users complain. The engineering cost is sampling infrastructure, scoring pipeline, and alert configuration.
- **Example or context:** A production RAG chatbot: sample 3% of requests, score with a faithfulness LLM judge, write scores to a time-series database (InfluxDB or Prometheus). Alert when the 7-day rolling average faithfulness score drops > 5% below the 30-day baseline. On alert: check provider changelog for silent model updates, check retrieval index freshness, and run the full regression eval suite to localise the degradation.

**Free Resources:**
- [Langfuse Documentation](https://langfuse.com/docs) — Production score monitoring, trend visualisation, and alerting for LLM quality drift
- [Arize Phoenix Documentation](https://docs.arize.com/phoenix) — Input distribution monitoring, output quality drift detection, and retrieval drift analysis for RAG systems
