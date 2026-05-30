# Module 1 — LLM Fundamentals

---

## Transformers

**Status:** ⬜ Not Started

**Definition:** The Transformer is a neural network architecture that processes sequences using self-attention — a mechanism that lets each token weigh all other tokens in the input to determine contextual relevance. It replaced recurrent networks as the foundation of all modern language models.

**Key Mental Model:** Self-attention is every word in a sentence voting on which other words matter most to understanding its meaning. "Bank" looks at "river" or "money" to resolve its meaning from context.

**How It Works:**
- Each token is projected into three vectors — Query (what am I looking for?), Key (what do I represent?), and Value (what information do I carry?). Attention scores are dot products of Q and K, scaled by the square root of the dimension, then softmaxed into weights that sum to 1 across all positions.
- Attention is computed in parallel across all token pairs simultaneously, not sequentially. This is why transformers can saturate GPUs effectively — the QK^T matrix multiplication maps cleanly to hardware matrix ops.
- Multiple attention heads run in parallel with independent Q/K/V projections, allowing the model to simultaneously attend to syntax, semantics, and long-range dependencies. Outputs are concatenated and projected back down.
- After the attention sublayer, each position passes through an independent feedforward network (two linear transformations with a non-linearity). The feedforward layer is where most of the parameter count lives and where factual associations are believed to be stored.
- Positional encodings (sinusoidal in the original, learned or RoPE in modern variants) are added to token embeddings so the model understands sequence order — raw attention is permutation-invariant without them.

**Common Misconceptions:**
- Transformers process words sequentially like humans read — they process all tokens in parallel using attention; the sequential structure is only in the input format.
- Bigger transformers are always better — scale improves capability but at quadratic attention cost; efficiency innovations (FlashAttention, sparse attention, MQA) matter significantly at deployment.

**Interview Answer Skeleton:**
- **What it is:** A neural architecture built on scaled dot-product self-attention, allowing every token to directly attend to every other token in the sequence in parallel, followed by position-wise feedforward layers.
- **Why it matters / trade-offs:** Understanding attention mechanics explains context window costs (quadratic in sequence length), generation latency, and why techniques like FlashAttention and KV caching exist. It also explains why prompting techniques like few-shot work.
- **Example or context:** A 128K-token context window requires attention computation over ~128K × 128K token pairs per layer. FlashAttention reorders these computations to stay in GPU SRAM, cutting memory overhead from O(N²) to O(N) while keeping the same mathematical result.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Covers transformer architecture from first principles with interactive code examples
- [Papers With Code — Transformers](https://paperswithcode.com) — Tracks latest transformer variants, benchmarks, and links to implementations

---

## Tokenization

**Status:** ⬜ Not Started

**Definition:** Tokenization splits raw text into tokens — sub-word units that the model actually processes as integer IDs. A token is not always a word; common words may be one token, rare words may split into 2–4 tokens. Token count directly determines API cost and context window consumption.

**Key Mental Model:** Tokens are like Scrabble tiles — common letter combinations get their own tile, rare combinations are built from multiple tiles. "programming" might be one tile; "supercalifragilistic" might be six.

**How It Works:**
- BPE (Byte Pair Encoding), used by GPT models, starts from individual bytes and iteratively merges the most frequent adjacent pairs until the vocabulary target size is reached (typically 50K–100K tokens). The merge table is fixed at training time.
- The tokenizer first normalises text (lowercasing, unicode normalisation), then splits on whitespace and punctuation boundaries, then applies the BPE merge rules greedily from longest match to shortest.
- Each unique token is assigned an integer ID. The model's embedding matrix has one row per token ID; the token ID is the lookup index into this matrix at the start of every forward pass.
- The same string tokenises differently depending on its context — a leading space before a word often produces a different token ID than the word without a space. This matters for few-shot prompt formatting.
- Multilingual models and code tokenisers use larger vocabularies to handle character sets outside the Latin alphabet. Non-Latin scripts (Chinese, Arabic, Thai) often cost 3–5x as many tokens per word as English, directly affecting API costs.

**Common Misconceptions:**
- One token equals one word — on average 1 token ≈ 0.75 words in English; code, URLs, numbers, and non-Latin scripts tokenise very differently.
- Tokenisation is a solved, boring problem — multilingual text, code, structured data formats, and mathematical notation all tokenise poorly with models trained primarily on English prose.

**Interview Answer Skeleton:**
- **What it is:** The process of converting raw text into integer token IDs using a fixed sub-word vocabulary (BPE, WordPiece, or SentencePiece), forming the input representation the model actually processes.
- **Why it matters / trade-offs:** Token count is the billing unit and the context-window unit. Tokenisation artefacts (splitting numbers digit by digit, poor handling of non-Latin scripts) directly cause model failures on arithmetic and multilingual tasks.
- **Example or context:** A JSON payload with deeply nested keys may tokenise at 2–3x the word count because every `{`, `"`, `:`, and indentation space is a separate token. This makes structured prompts expensive and motivates compact serialisation strategies.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Dedicated tokenisation chapter with an interactive tokeniser playground
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Covers how prompt structure and formatting affect token usage and model behaviour

---

## Context Windows

**Status:** ⬜ Not Started

**Definition:** The context window is the maximum number of tokens a model can process in a single call — input (prompt plus conversation history) and output combined. Beyond this limit, earlier content must be truncated or summarised. Modern frontier models range from 128K to 1M+ tokens.

**Key Mental Model:** The context window is the model's working memory — it can only see and reason about what is currently on the desk. Anything that scrolled off is gone unless you explicitly bring it back.

**How It Works:**
- Every token in the context window participates in attention computation with every other token in that window. As the window grows, the KV cache (stored key-value vectors for each attended position) grows linearly in memory, but attention computation scales quadratically without algorithmic optimisations.
- Prompt caching (supported by Anthropic, OpenAI) allows a provider to cache the KV state of a static prefix. On subsequent calls that reuse the same prefix, only the new tokens need to be processed — reducing both latency and cost for repeated system prompts.
- Positional encodings degrade beyond their training distribution. A model trained up to 8K tokens but extended via RoPE scaling to 128K will show degraded recall at positions beyond its original training range unless specifically fine-tuned on long-context data.
- The "lost in the middle" effect means retrieval accuracy is highest for content near the beginning and end of the context; information buried in the middle of a 100K-token context is recalled less reliably than information at the edges.
- Context window size and generation cost are separate concerns. A 128K input with a 500-token output costs much more in prefill compute than a 2K input with a 2K output, because the attention matrix over 128K tokens is enormous even for a single forward pass.

**Common Misconceptions:**
- Longer context windows mean better recall of everything in the prompt — models degrade on information in the middle of very long contexts ("lost in the middle" phenomenon); position matters significantly.
- Larger context always improves quality — larger contexts increase latency and cost dramatically; for many tasks, a focused short context outperforms a bloated long one.

**Interview Answer Skeleton:**
- **What it is:** The maximum token budget shared between input and output in a single model call, setting hard limits on prompt length, conversation history, and retrieved context passed to the model.
- **Why it matters / trade-offs:** Context limits drive architectural decisions in [[AI-Engineer/03-RAG-Systems]] (chunk sizing, retrieval depth), [[AI-Engineer/04-Agentic-Systems]] (state management across turns), and cost optimisation (prompt caching strategies).
- **Example or context:** Summarising a 200-page PDF requires either recursive chunked summarisation, a hierarchical map-reduce pattern, or a model with a sufficiently large context — but even with a 1M-token window, you must decide whether stuffing the full document or using RAG gives better accuracy per dollar.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Covers positional encoding and context scaling in transformer architectures
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Production patterns including prompt caching and long-context document handling

---

## Sampling Strategies

**Status:** ⬜ Not Started

**Definition:** LLMs generate output by sampling from a probability distribution over the next token at each step. Temperature scales the distribution (higher = more random, lower = more deterministic). Top-p (nucleus sampling) limits sampling to the smallest set of tokens whose cumulative probability reaches p. Top-k limits to the k highest-probability tokens.

**Key Mental Model:** Imagine a jar of coloured marbles representing possible next words. Temperature shakes the jar to adjust relative counts. Top-p removes all marbles below a probability threshold before drawing. Greedy decoding always picks the most common marble.

**How It Works:**
- At each generation step, the model's final linear layer produces a logit vector of dimension equal to the vocabulary size (e.g., 50K–128K values). These logits are divided by the temperature value before applying softmax — dividing by a value less than 1 sharpens the distribution, greater than 1 flattens it.
- Top-k sampling sorts all vocabulary tokens by probability, zeroes out all but the top k, then renormalises and samples. This prevents sampling from the extreme tail but has a fixed cutoff regardless of how peaked or flat the distribution is.
- Top-p (nucleus) sampling sorts tokens by descending probability, accumulates them until the running sum hits the p threshold (e.g., 0.9), discards the remainder, then renormalises. This is adaptive — a confident distribution might include only 5 tokens; an uncertain one might include 500.
- Temperature and top-p are applied sequentially: temperature reshapes the distribution first, then top-p truncates the tail of the reshaped distribution. Setting temperature=1.0 and top-p=1.0 is pure multinomial sampling over the full vocabulary.
- At inference time, autoregressive generation runs this sampling process token by token in a loop. Each new token is appended to the context, the full sequence is fed back through the model (using the KV cache to avoid recomputing earlier tokens), and the next token distribution is computed.

**Common Misconceptions:**
- Temperature 0 gives the "correct" answer — fully deterministic output can produce repetitive or locally suboptimal outputs; a small temperature often improves output diversity and quality.
- Top-p and temperature do the same thing — they interact but operate differently: temperature reshapes the distribution, top-p then truncates the tail; combining them requires care.

**Interview Answer Skeleton:**
- **What it is:** A set of hyperparameters (temperature, top-p, top-k) that control how the model draws from its next-token probability distribution at each autoregressive generation step.
- **Why it matters / trade-offs:** Misconfigured sampling causes hallucination (high temperature on factual tasks), repetition loops (temperature too low with greedy decoding), or overly conservative responses. The right settings are use-case specific and should be tuned empirically.
- **Example or context:** For a medical Q&A assistant where accuracy matters, set temperature to 0.1–0.3 and top-p to 0.9. For a creative story generator, temperature of 0.8–1.0 with top-p 0.95 encourages novelty without pure randomness. Always validate against a held-out eval set rather than intuition.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Covers text generation decoding strategies including beam search, nucleus sampling, and contrastive search
- [Papers With Code](https://paperswithcode.com) — Tracks sampling strategy research including speculative decoding and self-consistency methods

---

## Reasoning Models

**Status:** ⬜ Not Started

**Definition:** Reasoning models (Claude's extended thinking, OpenAI o1/o3, DeepSeek R1) are trained or configured to produce explicit intermediate reasoning steps before giving a final answer. This scratchpad approach significantly improves performance on complex, multi-step problems at the cost of higher latency and token usage.

**Key Mental Model:** A reasoning model is a student who shows their working on a test — the scratchpad is visible internally and the process of writing it out often catches errors that direct recall would miss.

**How It Works:**
- Reasoning models are typically trained using reinforcement learning on verifiable outcomes (RLVR) — the model is rewarded for producing correct final answers on math, code, or logic problems, and the training signal propagates back through the intermediate reasoning chain.
- During inference, the model generates a "thinking" or "scratchpad" token sequence before producing the visible response. These thinking tokens are often hidden from the user but consume context window and are billed normally; they can run into the tens of thousands of tokens on hard problems.
- The extended thinking block conditions the final answer generation — the model produces its answer by attending back to the full reasoning chain, making it much harder for the final output to contradict intermediate deductions it has just made.
- Routing logic in production systems typically classifies query complexity first (via a smaller classifier or heuristic), then routes only genuinely complex multi-step queries to the reasoning model. This keeps the cost multiplier manageable. See [[AI-Engineer/05-AI-Gateways-Routing]].
- Reasoning model outputs are more sensitive to problem framing and instruction quality than standard models. Ambiguous or poorly-specified prompts cause the reasoning chain to branch incorrectly early, and the error compounds through subsequent reasoning steps.

**Common Misconceptions:**
- Reasoning models are always better — for simple factual retrieval, reasoning overhead adds cost and latency without benefit; use them selectively for genuinely complex tasks.
- You cannot influence the reasoning model's reasoning — system prompts, problem framing, and explicit instructions on how to reason all meaningfully affect extended thinking output quality.

**Interview Answer Skeleton:**
- **What it is:** Models trained (often via RL on verifiable outcomes) to produce explicit intermediate reasoning chains before giving a final answer, enabling significantly better performance on multi-step problems.
- **Why it matters / trade-offs:** Reasoning models trade latency and token cost (often 10–50x more thinking tokens than output tokens) for accuracy on hard tasks. They require routing logic to avoid using them on simple queries where standard models suffice.
- **Example or context:** A complex SQL debugging task that requires tracing data types across five joins benefits from a reasoning model. A simple "translate this sentence to Spanish" query does not — routing the latter to a fast cheap model keeps p99 latency low while reserving the reasoning model for queries that actually need it.

**Free Resources:**
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Includes examples of extended thinking API usage and routing patterns for reasoning vs standard models
- [Papers With Code](https://paperswithcode.com) — Tracks DeepSeek R1, o1, and other reasoning model benchmarks and open implementations

---

## Benchmarks

**Status:** ⬜ Not Started

**Definition:** Benchmarks are standardised test sets used to compare LLM capabilities across tasks. Common benchmarks include MMLU (multi-task language understanding), HumanEval (code generation), MATH (mathematical reasoning), and GPQA (graduate-level science). Benchmark scores are useful guides but routinely gamed and don't always predict real-world task performance.

**Key Mental Model:** LLM benchmarks are like standardised tests — useful for rough comparison, but a model that aces the SAT isn't guaranteed to succeed at a specific job. Use them for direction, not as ground truth.

**How It Works:**
- Most benchmarks are structured as multiple-choice or free-response question sets. For multiple-choice (e.g., MMLU), the model is typically evaluated by comparing log-probabilities of each answer choice rather than generating free text — this makes evaluation deterministic but doesn't reflect open-ended generation quality.
- HumanEval and MBPP measure code generation by actually running the generated code against a set of unit tests. Pass@k is the standard metric: the probability that at least one of k generated samples passes all tests. This makes them more reliable than text similarity metrics.
- Data contamination occurs when benchmark test examples appear in training data. Models can effectively memorise answers rather than demonstrating generalisation. Newer benchmarks like LiveCodeBench and GPQA use harder or more recent examples to reduce contamination risk.
- Benchmark scores are aggregated averages across heterogeneous task types. A model can score high on MMLU by excelling at easy majority categories while performing poorly on the specific subdomain you care about. Always disaggregate scores by category before making selection decisions.
- Internal "golden set" evals — small hand-curated test sets built from your actual production data — are more predictive of real-world performance than public benchmarks. Running these alongside public benchmarks is standard practice at production AI teams. See [[AI-Engineer/07-Observability-Evals]].

**Common Misconceptions:**
- Higher benchmark score always means better for my use case — benchmarks measure average performance across standardised tasks; your specific use case may differ significantly from benchmark distribution.
- Benchmark scores are objective and trustworthy — models are sometimes trained on benchmark data (data contamination), inflating scores beyond their true generalisation ability.

**Interview Answer Skeleton:**
- **What it is:** Standardised evaluation datasets and protocols used to compare model capabilities across tasks — ranging from multiple-choice knowledge tests (MMLU) to execution-graded code generation (HumanEval) and graduate-level reasoning (GPQA).
- **Why it matters / trade-offs:** Benchmarks are necessary for model selection but insufficient. Contamination, dataset distribution mismatch, and aggregation across heterogeneous tasks all limit their reliability. Production teams complement them with task-specific internal eval sets.
- **Example or context:** Choosing between two models for a code generation task: reference HumanEval/MBPP pass@1 scores for baseline orientation, then build a 50-example internal eval set sampled from your actual codebase to measure the metric that actually matters for your users.

**Free Resources:**
- [Papers With Code](https://paperswithcode.com) — Tracks leaderboards, benchmark descriptions, and links to evaluation datasets and papers
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Covers model evaluation methodology, metrics, and limitations of standardised benchmarks
