# Module 7 — Observability and Evals

---

## Tracing

**Status:** ⬜ Not Started

**Definition:** Tracing captures the full execution path of an LLM request — every prompt, tool call, model response, and latency measurement — as a structured trace with parent/child spans. Traces enable debugging of multi-step agent workflows, latency profiling, and cost attribution at the individual request level.

**Mental Model:** Tracing is an X-ray of a request's journey — you see every step it took, how long each step took, what went in and what came out, and exactly where something went wrong.

**Common Misconceptions:**
- Logging prompt/response pairs is sufficient observability — logs lack structure and context; traces capture the causal tree of a multi-step agent execution and enable queries like "show me all requests where the tool call failed".
- Tracing only matters for debugging — traces are essential for performance profiling, cost attribution, regression testing, and demonstrating audit compliance.

**Interview Skeleton:**
- What it is: structured, hierarchical instrumentation of LLM application executions capturing every prompt, response, tool call, and latency
- Why it matters: without traces, debugging a failing multi-step agent is like debugging a program with no stack trace
- Example: describe the spans you'd instrument in a RAG pipeline and what you'd capture in each span's metadata

**Free Resources:** https://langfuse.com/docs — Langfuse documentation covering trace instrumentation, span types, and observability setup

---

## LLM-as-Judge

**Status:** ⬜ Not Started

**Definition:** LLM-as-judge uses a separate, often more capable LLM to evaluate the output of another LLM. The judge model scores responses on criteria like accuracy, helpfulness, coherence, or safety. This scales evaluation beyond human annotation while being more flexible than rule-based metrics.

**Mental Model:** LLM-as-judge is like peer review — a qualified reviewer assesses work quality using defined criteria. It's not perfect but is far more scalable than waiting for expert human review on every sample.

**Common Misconceptions:**
- LLM judges are objective — LLM judges have biases (preferring longer responses, their own outputs, or confident tone); calibrate judges against human labels and use structured rubrics to reduce bias.
- Any LLM works as a judge — judge quality depends on the model's ability to follow a rubric and reason about quality; use the strongest available model for judging, even if it's too slow/expensive for production.

**Interview Skeleton:**
- What it is: using a capable LLM to evaluate another LLM's outputs according to defined criteria at scale
- Why it matters: enables continuous evaluation at production scale without requiring human annotation for every response
- Example: write a judge prompt for evaluating response faithfulness in a RAG system and describe how you'd validate the judge's reliability

**Free Resources:** https://langfuse.com/docs — Langfuse documentation covering LLM-as-judge evaluation setup and scoring

---

## Custom Evals

**Status:** ⬜ Not Started

**Definition:** Custom evals are evaluation datasets and metrics designed for a specific application's requirements — not generic benchmarks. They consist of representative input examples, expected outputs or rubrics, and scoring logic (exact match, semantic similarity, LLM judge, or domain-specific rules).

**Mental Model:** Custom evals are like acceptance tests for LLM systems — they test the specific behaviour your application requires, not general intelligence. They are the difference between "this model is smart" and "this model works for my users".

**Common Misconceptions:**
- Generic benchmarks are sufficient for evaluating production LLMs — generic benchmarks test general capability; a model that scores well on MMLU may still fail your specific task distribution.
- Evals are a one-time setup — eval datasets must evolve as your application evolves and as you discover new failure modes in production.

**Interview Skeleton:**
- What it is: application-specific test suites that validate LLM behaviour on representative tasks with appropriate quality metrics
- Why it matters: the primary mechanism for making model improvements safely and measuring regression risk
- Example: design a custom eval for a legal document summarisation system, including dataset construction, metrics, and scoring methodology

**Free Resources:** https://langfuse.com/docs — Langfuse documentation covering custom dataset creation, eval runs, and scoring pipelines

---

## Regression Testing

**Status:** ⬜ Not Started

**Definition:** LLM regression testing runs the same eval suite against a new model version, prompt change, or retrieval system update and compares scores against a baseline. This catches quality degradations before they reach production — ensuring that improvements in one area don't silently degrade another.

**Mental Model:** LLM regression testing is like a CI/CD test suite for LLM behaviour — every change runs the full eval suite, and you can only deploy if the score doesn't degrade beyond an acceptable threshold.

**Common Misconceptions:**
- A manual review of a few responses is sufficient for detecting regression — LLM regressions are often subtle and appear only on specific input patterns; systematic eval coverage is needed.
- Regression testing is only needed for model upgrades — prompt changes, retrieval changes, and context window adjustments can all cause regressions; test every change.

**Interview Skeleton:**
- What it is: running a standardised eval suite on every system change to detect quality degradations before production deployment
- Why it matters: LLM systems are fragile; changes that seem unrelated (a prompt tweak) can cause regressions in unexpected behaviours
- Example: describe a CI/CD pipeline for an LLM application that runs regression evals on every PR and gates deployment on a score threshold

**Free Resources:** https://langfuse.com/docs — Langfuse documentation covering eval runs, score comparison, and dataset versioning for regression testing

---

## Drift Detection

**Status:** ⬜ Not Started

**Definition:** Drift detection monitors production LLM behaviour over time for gradual quality degradation — caused by model updates (silent model provider changes), input distribution shifts (user behaviour changing), or retrieval index staleness. It triggers alerts when quality metrics drop below thresholds.

**Mental Model:** Drift detection is like monitoring a river for changes in water quality over time — the river looks the same day to day, but systematic sampling reveals gradual changes that eventually matter.

**Common Misconceptions:**
- Production LLM quality is stable once deployed — model providers update models silently; input distributions shift as user bases grow; quality degrades over time without detection.
- A fixed alert threshold on overall quality is sufficient — drift detection needs segmented monitoring (by topic, user type, or task) to catch subtle degradations that average out in aggregate metrics.

**Interview Skeleton:**
- What it is: continuous monitoring of production LLM quality metrics over time to detect gradual degradation before it becomes a customer-facing problem
- Why it matters: LLM behaviour can change silently due to provider model updates, data drift, or retrieval index decay
- Example: design a drift monitoring system for a production RAG chatbot including what metrics to track and how to alert on drift

**Free Resources:** https://langfuse.com/docs — Langfuse documentation covering production monitoring, score trends, and alerting for LLM drift
