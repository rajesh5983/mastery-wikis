# Module 1 — LLM Fundamentals

---

## Transformers

**Status:** ⬜ Not Started

**Definition:** The Transformer is a neural network architecture that processes sequences using self-attention — a mechanism that lets each token weigh all other tokens in the input to determine contextual relevance. It replaced recurrent networks as the foundation of all modern language models.

**Mental Model:** Self-attention is every word in a sentence voting on which other words matter most to understanding its meaning. "Bank" looks at "river" or "money" to resolve its meaning from context.

**Common Misconceptions:**
- Transformers process words sequentially like humans read — they process all tokens in parallel using attention; the sequential structure is only in the input format.
- Bigger transformers are always better — scale improves capability but at quadratic attention cost; efficiency innovations (FlashAttention, sparse attention, MQA) matter significantly at deployment.

**Interview Skeleton:**
- What it is: the neural architecture underlying all modern LLMs, built on parallel self-attention and feedforward layers
- Why it matters: understanding transformers explains context limits, generation latency, cost, and why certain prompting techniques work
- Example: explain why increasing context length is expensive (quadratic attention complexity) and what FlashAttention or sparse attention does to address this

**Free Resources:** https://huggingface.co/learn/nlp-course — Hugging Face NLP course covering transformer architecture from first principles

---

## Tokenization

**Status:** ⬜ Not Started

**Definition:** Tokenization splits raw text into tokens — sub-word units that the model actually processes as integer IDs. A token is not always a word; common words may be one token, rare words may split into 2–4 tokens. Token count directly determines API cost and context window consumption.

**Mental Model:** Tokens are like Scrabble tiles — common letter combinations get their own tile, rare combinations are built from multiple tiles. "programming" might be one tile; "supercalifragilistic" might be six.

**Common Misconceptions:**
- One token equals one word — on average 1 token ≈ 0.75 words in English; code, URLs, numbers, and non-Latin scripts tokenize very differently.
- Tokenization is a solved, boring problem — multilingual text, code, structured data formats, and mathematical notation all tokenize poorly with models trained primarily on English prose.

**Interview Skeleton:**
- What it is: converting raw text into the integer IDs a model processes, using a learnt vocabulary of sub-word units
- Why it matters: token count drives cost, context window limits, and determines what fits in a single API call
- Example: explain why a prompt that's short in word count might use many tokens if it contains code, tables, or non-English text

**Free Resources:** https://huggingface.co/learn/nlp-course — Hugging Face NLP course with a dedicated tokenisation chapter and interactive tokenizer tool

---

## Context Windows

**Status:** ⬜ Not Started

**Definition:** The context window is the maximum number of tokens a model can process in a single call — input (prompt plus conversation history) and output combined. Beyond this limit, earlier content must be truncated or summarised. Modern frontier models range from 128K to 1M+ tokens.

**Mental Model:** The context window is the model's working memory — it can only see and reason about what is currently on the desk. Anything that scrolled off is gone unless you explicitly bring it back.

**Common Misconceptions:**
- Longer context windows mean better recall of everything in the prompt — models degrade on information in the middle of very long contexts ("lost in the middle" phenomenon); position matters significantly.
- Larger context always improves quality — larger contexts increase latency and cost dramatically; for many tasks, a focused short context outperforms a bloated long one.

**Interview Skeleton:**
- What it is: the token budget for a single model call, shared between input and output
- Why it matters: context limits constrain RAG chunk sizes, conversation history management, and long-document processing strategies
- Example: a 200-page document exceeds your context window — what strategies would you use? (chunking, hierarchical summarisation, RAG)

**Free Resources:** https://huggingface.co/learn/nlp-course — Hugging Face resources on transformer architecture including context and positional encoding

---

## Sampling Strategies

**Status:** ⬜ Not Started

**Definition:** LLMs generate output by sampling from a probability distribution over the next token at each step. Temperature scales the distribution (higher = more random, lower = more deterministic). Top-p (nucleus sampling) limits sampling to the smallest set of tokens whose cumulative probability reaches p. Top-k limits to the k highest-probability tokens.

**Mental Model:** Imagine a jar of coloured marbles representing possible next words. Temperature shakes the jar to adjust relative counts. Top-p removes all marbles below a probability threshold before drawing. Greedy decoding always picks the most common marble.

**Common Misconceptions:**
- Temperature 0 gives the "correct" answer — fully deterministic output can produce repetitive or locally suboptimal outputs; a small temperature often improves output diversity and quality.
- Top-p and temperature do the same thing — they interact but operate differently: temperature reshapes the distribution, top-p then truncates the tail; combining them requires care.

**Interview Skeleton:**
- What it is: hyperparameters controlling how a model selects each output token from its probability distribution at generation time
- Why it matters: wrong settings cause hallucinations, repetitive loops, or overly conservative responses depending on the use case
- Example: for a medical Q&A assistant, what temperature and sampling settings would you choose and why?

**Free Resources:** https://huggingface.co/learn/nlp-course — Hugging Face course covering text generation, decoding strategies, and beam search

---

## Reasoning Models

**Status:** ⬜ Not Started

**Definition:** Reasoning models (Claude's extended thinking, OpenAI o1/o3, DeepSeek R1) are trained or configured to produce explicit intermediate reasoning steps before giving a final answer. This scratchpad approach significantly improves performance on complex, multi-step problems at the cost of higher latency and token usage.

**Mental Model:** A reasoning model is a student who shows their working on a test — the scratchpad is visible internally and the process of writing it out often catches errors that direct recall would miss.

**Common Misconceptions:**
- Reasoning models are always better — for simple factual retrieval, reasoning overhead adds cost and latency without benefit; use them selectively for genuinely complex tasks.
- You cannot influence the reasoning model's reasoning — system prompts, problem framing, and explicit instructions on how to reason all meaningfully affect extended thinking output quality.

**Interview Skeleton:**
- What it is: models that generate explicit intermediate reasoning before producing final answers, trading latency for accuracy on hard problems
- Why it matters: dramatically improves performance on math, code, and multi-step reasoning tasks; changes the cost/latency/quality trade-off calculation
- Example: when would you route a query to a reasoning model vs a standard model? How would you measure whether it improves output quality?

**Free Resources:** https://huggingface.co/learn/nlp-course — Hugging Face resources on reasoning-enhanced models and chain-of-thought evaluation

---

## Benchmarks

**Status:** ⬜ Not Started

**Definition:** Benchmarks are standardised test sets used to compare LLM capabilities across tasks. Common benchmarks include MMLU (multi-task language understanding), HumanEval (code generation), MATH (mathematical reasoning), and GPQA (graduate-level science). Benchmark scores are useful guides but routinely gamed and don't always predict real-world task performance.

**Mental Model:** LLM benchmarks are like standardised tests — useful for rough comparison, but a model that aces the SAT isn't guaranteed to succeed at a specific job. Use them for direction, not as ground truth.

**Common Misconceptions:**
- Higher benchmark score always means better for my use case — benchmarks measure average performance across standardised tasks; your specific use case may differ significantly from benchmark distribution.
- Benchmark scores are objective and trustworthy — models are sometimes trained on benchmark data (data contamination), inflating scores beyond their true generalisation ability.

**Interview Skeleton:**
- What it is: standardised evaluation datasets used to compare model capabilities across tasks
- Why it matters: benchmarks guide model selection but must be interpreted with scepticism about contamination, recency, and relevance to your workload
- Example: choosing between two models for a code generation task — which benchmarks would you reference, and what additional evaluation would you run yourself?

**Free Resources:** https://huggingface.co/learn/nlp-course — Hugging Face NLP course covering model evaluation, benchmarks, and limitations
