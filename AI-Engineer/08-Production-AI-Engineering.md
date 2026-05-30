# Module 8 — Production AI Engineering

---

## Streaming Responses

**Status:** ⬜ Not Started

**Definition:** Streaming delivers LLM output token by token (or in chunks) as it is generated, rather than waiting for the full response to complete before returning anything. This dramatically reduces perceived latency and enables progressive UI rendering, making applications feel significantly more responsive.

**Mental Model:** Streaming is the difference between watching a video download bar fill up before playback vs. a streaming service that starts playing immediately while buffering the rest. Users experience the content as it arrives.

**Common Misconceptions:**
- Streaming reduces total generation time — streaming does not change how long the model takes to generate; it only changes when the client receives the output, improving perceived latency.
- Streaming is always the right choice — streaming complicates error handling, token counting, and structured output parsing; for batch processing or short responses, non-streaming is simpler.

**Interview Skeleton:**
- What it is: incremental delivery of LLM output as it is generated, rather than waiting for the full response
- Why it matters: critical for interactive applications; a 10-second response feels instant when streamed vs. unacceptable when delivered all-at-once
- Example: implement a streaming chat interface and describe how you'd handle partial JSON output and error mid-stream

**Free Resources:** https://github.com/anthropics/anthropic-cookbook — Anthropic cookbook with streaming implementation examples for Claude

---

## Parallel Tool Calls

**Status:** ⬜ Not Started

**Definition:** Parallel tool calls allow an LLM to request multiple tool executions simultaneously in a single model response, rather than sequentially. When independent tools are called in parallel, total latency equals the slowest single call rather than the sum of all calls.

**Mental Model:** Sequential tool calls are like making three phone calls one after another. Parallel tool calls are like putting three people on calls simultaneously — the total time is determined by the longest call, not the sum of all three.

**Common Misconceptions:**
- Parallel tool calls require special agent architecture — most LLM APIs support parallel function calling natively; the model outputs multiple tool call objects in one response.
- Parallel calls are always safe — parallel tool calls must be genuinely independent; if tool A's output is needed as input to tool B, they must remain sequential.

**Interview Skeleton:**
- What it is: the model requesting multiple independent tool calls in a single generation step to reduce total execution latency
- Why it matters: multi-step agent tasks that use parallel calls are 2–5x faster than purely sequential equivalents
- Example: identify which steps in a multi-tool research agent can be parallelised and implement the parallel call handling logic

**Free Resources:** https://github.com/anthropics/anthropic-cookbook — Anthropic cookbook with parallel tool use examples and implementation patterns

---

## Retries and Rate Limits

**Status:** ⬜ Not Started

**Definition:** LLM API calls can fail due to rate limits (429), transient server errors (500, 503), or timeouts. Production systems must implement retry logic with exponential backoff and jitter, respect provider rate limits (tokens per minute, requests per minute), and implement circuit breakers for sustained outages.

**Mental Model:** Retry logic is like redialling a busy phone — wait a bit, try again. Exponential backoff is waiting progressively longer each time. Jitter prevents all your services from retrying simultaneously and overwhelming the provider.

**Common Misconceptions:**
- Retry immediately on failure for best performance — immediate retries on rate-limit errors return the same 429 and waste quota; exponential backoff with jitter is the correct pattern.
- Rate limits only apply to paid tiers — all API tiers have rate limits; production systems at scale hit them regularly and must be designed to handle them gracefully.

**Interview Skeleton:**
- What it is: resilience patterns for handling LLM API failures, rate limits, and transient errors in production
- Why it matters: production AI applications must handle API unreliability gracefully to maintain SLA commitments
- Example: implement an API client wrapper with exponential backoff, jitter, and circuit breaker logic for Claude API calls

**Free Resources:** https://github.com/anthropics/anthropic-cookbook — Anthropic cookbook with retry and error handling implementation examples

---

## Semantic Caching

**Status:** ⬜ Not Started

**Definition:** Semantic caching stores LLM responses and retrieves them for semantically similar (not necessarily identical) future queries, using embedding similarity to determine cache hits. Unlike exact-match caching, it handles paraphrased or reformulated versions of the same question, significantly improving cache hit rates.

**Mental Model:** Semantic caching is like a librarian who recognises that "how do I make pasta?" and "what is the pasta cooking process?" are asking the same thing, and retrieves the same answer for both without going back to the kitchen.

**Common Misconceptions:**
- Semantic caching is only useful for high-traffic applications — even moderate-traffic applications with repetitive query patterns benefit significantly; Q&A bots and support systems have highly repetitive query distributions.
- Semantic cache hits are always appropriate — semantic similarity doesn't guarantee the cached answer is appropriate for the new query; set similarity thresholds carefully and validate edge cases.

**Interview Skeleton:**
- What it is: caching LLM responses using embedding similarity to serve cached results for semantically equivalent queries
- Why it matters: can reduce LLM API costs 30–70% for applications with repetitive query patterns
- Example: design a semantic caching layer for a customer support bot, including similarity threshold selection and cache invalidation strategy

**Free Resources:** https://github.com/anthropics/anthropic-cookbook — Anthropic cookbook covering caching patterns and cost optimisation strategies

---

## Cost Optimisation

**Status:** ⬜ Not Started

**Definition:** LLM cost optimisation systematically reduces API spend without proportionally degrading quality. Key levers include prompt compression (removing redundant context), model routing (cheapest model that meets quality bar), caching (prompt and semantic), batching (async batch APIs at lower rates), and output length control.

**Mental Model:** Cost optimisation is like home energy management — you don't turn everything off (that defeats the purpose), you identify what's consuming the most, eliminate waste, and switch to more efficient alternatives for low-priority tasks.

**Common Misconceptions:**
- Cost optimisation always degrades quality — most applications have significant headroom between current cost and minimum viable quality; routing simple queries to cheaper models rarely degrades user experience.
- Token reduction always helps — truncating context that the model actually needs degrades quality and may cause regressions that are more expensive to fix than the tokens saved.

**Interview Skeleton:**
- What it is: a set of engineering techniques for reducing LLM API cost without proportionally reducing application quality
- Why it matters: unoptimised LLM applications can have unsustainable unit economics; cost engineering is a first-class concern
- Example: given a production application spending $10K/month on LLM APIs, walk through your optimisation investigation and the levers you'd pull first

**Free Resources:** https://github.com/anthropics/anthropic-cookbook — Anthropic cookbook with cost optimisation patterns including prompt caching and batching

---

## Latency Budgets

**Status:** ⬜ Not Started

**Definition:** A latency budget allocates the acceptable end-to-end response time for an AI feature across its components: retrieval, reranking, LLM inference, post-processing, and network. Each component gets a budget; exceeding it degrades user experience. Latency budgets force explicit trade-offs between quality and speed.

**Mental Model:** A latency budget is like a project timeline broken into phases — if one phase runs over, you know immediately which trade-offs to make rather than discovering the problem only when the whole pipeline exceeds SLA.

**Common Misconceptions:**
- Latency only matters for real-time chat interfaces — background processing, batch analytics, and report generation also have latency SLAs; stakeholders notice when reports arrive late.
- Faster models are always better — faster models are usually less capable; the goal is to meet the latency budget with the highest quality model that fits, not to minimise latency at all costs.

**Interview Skeleton:**
- What it is: the practice of explicitly allocating acceptable latency across each component of an AI pipeline
- Why it matters: latency budgets make trade-off decisions explicit and provide a framework for optimisation prioritisation
- Example: define a latency budget for a RAG chatbot with a 2-second P95 SLA and describe how you'd measure and enforce each component's allocation

**Free Resources:** https://github.com/anthropics/anthropic-cookbook — Anthropic cookbook covering latency profiling and optimisation patterns for production AI systems
