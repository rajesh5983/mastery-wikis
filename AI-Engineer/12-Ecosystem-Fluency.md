# Module 12 — Ecosystem Fluency

---

## Open Models

**Status:** ⬜ Not Started

**Definition:** Open models are LLMs released with publicly available weights (Llama, Mistral, Qwen, Gemma, Phi) that can be downloaded and run locally or on your own infrastructure. They range from sub-1B to 405B parameters and span general-purpose, code-specialised, and multimodal variants.

**Mental Model:** Open models are like open-source software — the "source code" (model weights) is publicly available. You can use them as-is, fine-tune for your domain, or study their architecture, without licensing restrictions or per-token costs.

**Common Misconceptions:**
- "Open" models are completely unrestricted — most open models have licenses with commercial use conditions (Meta's Llama licence, for instance, has usage restrictions for large companies); always read the licence.
- Open models are always less capable than frontier models — in 2024–2025, the gap has closed dramatically; smaller fine-tuned open models often outperform larger general frontier models on specific tasks.

**Interview Skeleton:**
- What it is: LLMs with publicly available weights that can be self-hosted, fine-tuned, and deployed without per-token licensing
- Why it matters: enables data residency, custom fine-tuning, cost control at scale, and use in air-gapped environments
- Example: compare Llama 3.1 70B vs Claude Sonnet for a specific enterprise task — what factors determine the choice?

**Free Resources:** https://paperswithcode.com — Papers With Code tracks open model releases, benchmarks, and research that underpins the open model ecosystem

---

## Hugging Face

**Status:** ⬜ Not Started

**Definition:** Hugging Face is the central hub for open AI models, datasets, and ML tools. The Transformers library provides model loading and inference APIs; the Hub hosts 500K+ models and datasets; Spaces hosts live demos; and the Inference API provides hosted endpoints for thousands of open models.

**Mental Model:** Hugging Face is the GitHub of AI — where models and datasets are shared, discovered, and discussed. The Transformers library is like npm for ML — standard packages that work with every open model.

**Common Misconceptions:**
- Hugging Face is only for research, not production — the Inference API, Dedicated Endpoints, and AutoTrain are production-grade; major companies use Hugging Face infrastructure for production workloads.
- You need to understand the full Transformers codebase to use it — most practical uses involve just loading a model with `from_pretrained()` and using a pipeline; deep architecture knowledge is not required.

**Interview Skeleton:**
- What it is: the central platform for open AI models, datasets, and tooling, including the Transformers library
- Why it matters: the standard ecosystem for working with open models; fluency here is essential for any AI engineer working with non-frontier models
- Example: describe how you'd evaluate five open embedding models for a RAG system using Hugging Face tools

**Free Resources:** https://paperswithcode.com — Papers With Code links open model research to Hugging Face model cards for practical implementation

---

## Reading Papers

**Status:** ⬜ Not Started

**Definition:** AI engineering moves faster than any textbook can track; reading research papers is the only way to stay current with new architectures, techniques, and benchmarks. Effective paper reading focuses on Abstract, Introduction, Method, and Results — most papers can be meaningfully consumed in 20–30 minutes at this level.

**Mental Model:** Reading a paper is like reading a product spec from the team that built the thing — you get the motivation, the design decisions, and the evidence that it works, before it becomes common knowledge.

**Common Misconceptions:**
- You need a PhD to read AI papers — most AI engineering papers are readable by practitioners; the math can be skipped on a first pass to understand the idea and practical implications.
- Reading papers is only for researchers — techniques from papers (RAG, CoT, RLHF, Flash Attention) routinely become production engineering tools within months of publication.

**Interview Skeleton:**
- What it is: the practice of reading primary research to stay current with developments that will become engineering tools within months
- Why it matters: being aware of emerging techniques 6–12 months ahead of widespread adoption provides significant competitive advantage
- Example: describe your process for evaluating whether a new technique from a recent paper is worth implementing in a production system

**Free Resources:** https://paperswithcode.com — Papers With Code links research papers to code implementations and benchmark results

---

## Benchmarks

**Status:** ⬜ Not Started

**Definition:** In the context of ecosystem fluency, benchmark awareness means knowing which benchmarks matter for which tasks — MMLU for knowledge, HumanEval/SWE-Bench for code, MATH/AIME for reasoning, LMSYS Chatbot Arena for general preference, and MTEB for embeddings — and understanding their limitations.

**Mental Model:** Benchmarks are the fuel economy ratings of AI models — useful for comparison shopping, but your actual performance depends on your specific driving conditions. Use them to narrow the field, then test on your own data.

**Common Misconceptions:**
- The model at the top of every leaderboard is the right choice for all tasks — leaderboards aggregate across many tasks; for specialised workloads, a model lower on the general leaderboard may dominate.
- New benchmarks are always more meaningful than established ones — new benchmarks may be poorly calibrated, too easy due to model contamination, or not yet validated; established benchmarks with known limitations are often more reliable.

**Interview Skeleton:**
- What it is: knowledge of the benchmark landscape — what each benchmark measures, its limitations, and how to use benchmark data for model selection
- Why it matters: effective model selection requires interpreting benchmarks critically, not just reading rankings
- Example: given a requirement to select an embedding model for a legal document retrieval system, which benchmarks would guide your evaluation and what caveats would you apply?

**Free Resources:** https://paperswithcode.com — Papers With Code hosts the State of the Art leaderboards and benchmark methodology descriptions
