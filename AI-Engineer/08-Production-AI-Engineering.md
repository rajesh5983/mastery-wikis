# Module 8 — Production AI Engineering

---

## Streaming Responses

**Status:** ⬜ Not Started

**Definition:** Streaming delivers LLM output token by token (or in chunks) as it is generated, rather than waiting for the full response to complete before returning anything. This dramatically reduces perceived latency and enables progressive UI rendering, making applications feel significantly more responsive.

**Key Mental Model:** Streaming is the difference between watching a video download bar fill up before playback vs. a streaming service that starts playing immediately while buffering the rest. Users experience the content as it arrives.

**How It Works:**
- When streaming is enabled, the provider API returns a chunked HTTP response using Server-Sent Events (SSE) or chunked transfer encoding. Each chunk contains a delta — the newly generated tokens since the last chunk. The client reads chunks from the response stream as they arrive rather than waiting for the connection to close.
- In Python with the Anthropic SDK, streaming is accessed via `client.messages.stream()` context manager. Each iteration of the stream yields an event object with a type (`content_block_delta`, `message_delta`, etc.) and a delta payload containing the new text. The application concatenates deltas to build the full response incrementally.
- For extended thinking (reasoning) models, the stream delivers two block types in sequence: `thinking` block deltas first (internal reasoning tokens), then `text` block deltas (the visible response). The application must track which block type is currently streaming to route deltas correctly to the UI — showing a "thinking..." indicator during the thinking block.
- Error handling mid-stream requires special care. A network error or provider error can occur after partial output is already delivered to the user. The application must handle partial responses gracefully: either show the partial output with an error indicator, or discard it and show a full retry. Unlike non-streaming, you cannot simply retry transparently — the user has already seen partial output.
- Streaming complicates structured output parsing. You cannot parse a partial JSON object. Strategies: buffer the full stream and parse on completion (sacrifices the UX benefit), use a streaming JSON parser (e.g., ijson) that parses incrementally, or design the output schema so that displayable content appears at the end of the JSON structure.

**Common Misconceptions:**
- Streaming reduces total generation time — streaming does not change how long the model takes to generate; it only changes when the client receives the output, improving perceived latency.
- Streaming is always the right choice — streaming complicates error handling, token counting, and structured output parsing; for batch processing or short responses, non-streaming is simpler.

**Interview Answer Skeleton:**
- **What it is:** An HTTP streaming pattern (SSE or chunked transfer) that delivers LLM output deltas to the client as each token is generated, enabling progressive rendering without waiting for the full generation to complete.
- **Why it matters / trade-offs:** Streaming is the primary UX lever for reducing perceived latency in interactive AI applications — a 10-second streamed response feels far better than a 10-second wait followed by instant display. Trade-offs: error handling complexity, inability to transparently retry mid-stream, and complications with structured output parsing.
- **Example or context:** Implementing a streaming chat interface: iterate over the stream, accumulate text deltas into a buffer, and flush to the UI on each delta event. For extended thinking models, switch the UI to "thinking..." display mode when a thinking block starts, then switch to text display mode when the text block starts. Handle stream errors by showing "Response interrupted — please retry" with the partial text hidden rather than displayed.

**Free Resources:**
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Streaming implementation examples including extended thinking streaming and error handling patterns
- [FastAPI Documentation](https://fastapi.tiangolo.com) — Server-side streaming with StreamingResponse and SSE for serving LLM output to web frontends

---

## Parallel Tool Calls

**Status:** ⬜ Not Started

**Definition:** Parallel tool calls allow an LLM to request multiple tool executions simultaneously in a single model response, rather than sequentially. When independent tools are called in parallel, total latency equals the slowest single call rather than the sum of all calls.

**Key Mental Model:** Sequential tool calls are like making three phone calls one after another. Parallel tool calls are like putting three people on calls simultaneously — the total time is determined by the longest call, not the sum of all three.

**How It Works:**
- When the model determines it needs multiple independent pieces of information, it generates a response containing multiple `tool_use` content blocks in a single API response rather than one at a time. The application receives all tool call requests together and can dispatch them concurrently.
- On the application side, parallel execution is implemented with `asyncio.gather()` (Python async) or a thread pool. Each tool call is dispatched as an independent coroutine or thread; all results are awaited concurrently. The total elapsed time is max(individual call latencies) rather than sum.
- All tool results are then bundled into a single `tool_result` message sent back to the model. The model receives all results simultaneously and uses them together to generate the next response. This is structurally different from sequential tool calling where each result triggers a new model generation step.
- The model decides whether to use parallel tool calls based on its assessment of the tool call dependencies. You can influence this through the tool descriptions and few-shot examples in the system prompt. Explicitly stating "these tools are independent and can be called simultaneously" can encourage parallel calling behaviour.
- Not all tool calls can be parallelised. Sequential ordering is required when tool B needs tool A's output as its input (dependency chain). Parallel calling when there is an implicit dependency produces incorrect results — the model receives the wrong input to tool B because it used a guess rather than A's actual output.

**Common Misconceptions:**
- Parallel tool calls require special agent architecture — most LLM APIs support parallel function calling natively; the model outputs multiple tool call objects in one response.
- Parallel calls are always safe — parallel tool calls must be genuinely independent; if tool A's output is needed as input to tool B, they must remain sequential.

**Interview Answer Skeleton:**
- **What it is:** An LLM API capability where the model requests multiple independent tool executions in a single response, which the application dispatches concurrently using async or threading — reducing multi-tool agent latency from sum-of-calls to max-of-calls.
- **Why it matters / trade-offs:** For research or data-gathering agents with many independent lookups, parallel tool calling can cut latency by 3–5x compared to sequential execution. The constraint is that only genuinely independent tool calls can be parallelised — dependency chains must remain sequential.
- **Example or context:** A financial analysis agent asked "compare AAPL, GOOG, and MSFT's Q3 earnings." The model generates three parallel `get_earnings_report` tool calls in one response. The application dispatches them concurrently with `asyncio.gather()`, waits for all three, bundles the results, and sends them back in a single `tool_result` message. Total latency is the slowest single earnings API call, not 3×.

**Free Resources:**
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Parallel tool use examples with async implementation and latency comparison
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Parallel agent execution patterns and fan-out/fan-in graph nodes for multi-tool agents

---

## Retries and Rate Limits

**Status:** ⬜ Not Started

**Definition:** LLM API calls can fail due to rate limits (429), transient server errors (500, 503), or timeouts. Production systems must implement retry logic with exponential backoff and jitter, respect provider rate limits (tokens per minute, requests per minute), and implement circuit breakers for sustained outages.

**Key Mental Model:** Retry logic is like redialling a busy phone — wait a bit, try again. Exponential backoff is waiting progressively longer each time. Jitter prevents all your services from retrying simultaneously and overwhelming the provider.

**How It Works:**
- Exponential backoff computes the retry wait time as `min(cap, base * 2^attempt)`. Starting from a 1-second base with a 60-second cap, the sequence is: 1s, 2s, 4s, 8s, 16s, 32s, 60s, 60s... This prevents hammering the provider while still recovering quickly from short transients.
- Jitter adds a random value (uniformly distributed between 0 and the computed wait time) to the backoff delay. This desynchronises retry storms — if 100 clients hit a rate limit simultaneously, full jitter spreads their retries across the wait window rather than all retrying at the same second.
- Rate limit (429) responses from the provider often include a `Retry-After` header specifying the exact wait time in seconds. Respecting this header is more efficient than exponential backoff — it waits exactly the right amount rather than guessing. Parse and use this header when present.
- Tokens-per-minute (TPM) rate limits require pre-call budgeting in high-throughput systems. Track tokens consumed in a sliding 60-second window; if the next request would exceed the TPM limit, delay it until the window clears. This is more efficient than letting the 429 happen and retrying — it avoids wasted API call overhead.
- Circuit breakers add a state machine on top of retry logic: CLOSED (normal operation) → OPEN (stop all calls after N consecutive failures) → HALF_OPEN (allow one probe call to test recovery) → CLOSED (if probe succeeds). This prevents cascading failures when a provider is experiencing a sustained outage — without a circuit breaker, every request blocks for the full retry timeout.

**Common Misconceptions:**
- Retry immediately on failure for best performance — immediate retries on rate-limit errors return the same 429 and waste quota; exponential backoff with jitter is the correct pattern.
- Rate limits only apply to paid tiers — all API tiers have rate limits; production systems at scale hit them regularly and must be designed to handle them gracefully.

**Interview Answer Skeleton:**
- **What it is:** A resilience pattern combining exponential backoff with full jitter for transient errors, `Retry-After` header compliance for rate limits, and circuit breakers for sustained provider outages — implemented as a wrapper around LLM API calls.
- **Why it matters / trade-offs:** LLM APIs are not perfectly reliable — rate limits, brief outages, and transient errors are normal. Without retry logic, these translate directly into user-visible errors. Backoff with jitter is safe; aggressive immediate retries make the situation worse by consuming rate limit quota faster.
- **Example or context:** Implementing an API client wrapper: on 429, read `Retry-After` header, sleep that duration (or fall back to exponential backoff if header absent), retry. After 5 consecutive failures, open the circuit breaker and immediately failover to the backup provider in the fallback chain rather than blocking on retry attempts. Log all retry events with attempt number for monitoring.

**Free Resources:**
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Retry logic and error handling implementation examples for production Claude API usage
- [LiteLLM Documentation](https://docs.litellm.ai) — Built-in retry and fallback configuration with exponential backoff and circuit breaker support

---

## Semantic Caching

**Status:** ⬜ Not Started

**Definition:** Semantic caching stores LLM responses and retrieves them for semantically similar (not necessarily identical) future queries, using embedding similarity to determine cache hits. Unlike exact-match caching, it handles paraphrased or reformulated versions of the same question, significantly improving cache hit rates.

**Key Mental Model:** Semantic caching is like a librarian who recognises that "how do I make pasta?" and "what is the pasta cooking process?" are asking the same thing, and retrieves the same answer for both without going back to the kitchen.

**How It Works:**
- When a query arrives, it is embedded using a fast, cheap embedding model (e.g., text-embedding-3-small). The resulting query vector is compared against all cached query vectors using ANN search (HNSW index in Redis or pgvector). If the nearest cached query has a cosine similarity above the configured threshold (typically 0.92–0.97), the cached response is returned immediately without an LLM call.
- On a cache miss, the LLM call proceeds normally. After the response is received, the query embedding and response are stored in the cache: the embedding is inserted into the ANN index, the response text is stored in a key-value store keyed by the embedding's vector ID.
- Similarity threshold selection is the critical tuning parameter. Too low (e.g., 0.85) and semantically distinct queries match each other, returning wrong cached answers. Too high (e.g., 0.99) and the cache only hits on near-duplicate queries, losing most of the value. Start at 0.95 and tune down by testing against your actual query distribution.
- Cache invalidation is harder for semantic caches than exact-match caches. When the underlying data or model changes, cached responses may be stale. Implement time-to-live (TTL) expiry for queries about time-sensitive topics. For knowledge base updates, use topic-based invalidation: tag cached responses with the knowledge base sections they reference, and expire those tags when the source data changes.
- Cache warm-up for new deployments: pre-populate the cache with the most frequent query patterns from production logs, embedded and stored before launch. This gives the cache immediate high hit rates rather than waiting for organic cache population.

**Common Misconceptions:**
- Semantic caching is only useful for high-traffic applications — even moderate-traffic applications with repetitive query patterns benefit significantly; Q&A bots and support systems have highly repetitive query distributions.
- Semantic cache hits are always appropriate — semantic similarity doesn't guarantee the cached answer is appropriate for the new query; set similarity thresholds carefully and validate edge cases.

**Interview Answer Skeleton:**
- **What it is:** A caching layer that embeds incoming queries, performs ANN similarity search against cached query embeddings, and returns stored responses when similarity exceeds a threshold — serving identical or paraphrased queries without LLM API calls.
- **Why it matters / trade-offs:** Semantic caching can reduce LLM API costs 30–70% for applications with repetitive query distributions (support bots, FAQ systems). The risks are incorrect cache hits (similarity threshold too low) and stale responses (TTL and invalidation must match data freshness requirements).
- **Example or context:** A customer support bot with 10K daily queries about the same 500 FAQ topics: embed incoming queries with text-embedding-3-small (< 1ms), ANN search against cached queries in Redis (< 5ms), return cached answer at similarity > 0.94. Cache miss rate after warm-up is ~20% — 80% of queries served from cache at 1/100th the cost of an LLM call. Monitor hit/miss rates and cached response quality via a sample of cache hits reviewed by the LLM judge.

**Free Resources:**
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Caching patterns including semantic caching implementation and cost optimisation strategies
- [Langfuse Documentation](https://langfuse.com/docs) — Tracking cache hit rates and cost savings from caching in LLM observability dashboards

---

## Cost Optimisation

**Status:** ⬜ Not Started

**Definition:** LLM cost optimisation systematically reduces API spend without proportionally degrading quality. Key levers include prompt compression (removing redundant context), model routing (cheapest model that meets quality bar), caching (prompt and semantic), batching (async batch APIs at lower rates), and output length control.

**Key Mental Model:** Cost optimisation is like home energy management — you don't turn everything off (that defeats the purpose), you identify what's consuming the most, eliminate waste, and switch to more efficient alternatives for low-priority tasks.

**How It Works:**
- The first step is cost attribution: break down spend by feature, model, and prompt component (system prompt tokens vs user tokens vs output tokens). Often one feature or one large prompt prefix accounts for a disproportionate share of cost — this is the highest-leverage target. See [[AI-Engineer/05-AI-Gateways-Routing]] for attribution infrastructure.
- Prompt compression removes redundant, verbose, or irrelevant content from prompts without degrading output quality. Techniques: remove filler text and verbal padding from system prompts, truncate conversation history to the most recent N turns, compress retrieved context with a lightweight summarisation pass, and use shorter variable names in structured data. Measure quality before and after every compression change.
- Model routing shifts simple queries to cheaper, faster models while preserving expensive models for queries that require their capability. A well-calibrated routing classifier can route 60–80% of queries to a model that is 5–20x cheaper with no detectable quality difference on those queries.
- Batch APIs (Anthropic's Message Batches API, OpenAI's Batch API) process requests asynchronously at 50% of standard pricing. For non-real-time workloads (nightly data enrichment, report generation, eval runs), batching cuts cost in half with no quality change and with latency measured in minutes rather than seconds.
- Output length control via explicit max_tokens limits and output format instructions prevents runaway long responses. For structured output tasks, instruct the model to be concise: "respond in at most 3 sentences" or "use bullet points, no prose." Output tokens are typically billed at 3–5x the rate of input tokens — controlling output length has an outsized cost impact.

**Common Misconceptions:**
- Cost optimisation always degrades quality — most applications have significant headroom between current cost and minimum viable quality; routing simple queries to cheaper models rarely degrades user experience.
- Token reduction always helps — truncating context that the model actually needs degrades quality and may cause regressions that are more expensive to fix than the tokens saved.

**Interview Answer Skeleton:**
- **What it is:** A systematic approach to reducing LLM API spend using attribution analysis to identify high-cost workloads, then applying targeted levers (model routing, prompt caching, semantic caching, batch APIs, output length control) in order of ROI.
- **Why it matters / trade-offs:** Unoptimised LLM applications can have unsustainable unit economics at scale. Cost engineering is a first-class production concern, not a post-launch afterthought. Every optimisation requires an eval gate — confirm that cost reduction does not degrade quality before shipping.
- **Example or context:** A $10K/month application: attribution reveals 60% of cost comes from a 15K-token system prompt sent on every call. Fix: implement prompt caching (→ 10% of input token cost on cache hits, saves $5K/month). Remaining 40% is split between two features; route the simpler one to a cheaper model (→ saves additional $2K/month). Total: 70% cost reduction, validated against regression eval suite. Track monthly via cost dashboard.

**Free Resources:**
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Cost optimisation patterns including prompt caching, batch APIs, and model routing examples
- [LiteLLM Documentation](https://docs.litellm.ai) — Cost tracking, model routing, and multi-provider optimisation for production LLM applications

---

## Latency Budgets

**Status:** ⬜ Not Started

**Definition:** A latency budget allocates the acceptable end-to-end response time for an AI feature across its components: retrieval, reranking, LLM inference, post-processing, and network. Each component gets a budget; exceeding it degrades user experience. Latency budgets force explicit trade-offs between quality and speed.

**Key Mental Model:** A latency budget is like a project timeline broken into phases — if one phase runs over, you know immediately which trade-offs to make rather than discovering the problem only when the whole pipeline exceeds SLA.

**How It Works:**
- The latency budget starts with the user-facing SLA target (e.g., p95 < 2000ms end-to-end). This is decomposed into component allocations: network overhead (~50ms), query embedding (~30ms), ANN retrieval (~50ms), reranking (~100ms), LLM inference (time-to-first-token + streaming duration = ~800ms at p95), post-processing (~50ms). Total: ~1080ms with ~920ms of headroom for variance.
- Time-to-first-token (TTFT) is the most user-perceived latency metric for streaming interfaces. It measures the time from request send to receiving the first token in the response. It is driven primarily by model size and input prompt length (prefill time). Strategies to reduce TTFT: shorter input prompts, prompt caching to skip prefill for static content, and using faster models for the first-token response.
- Per-component budget enforcement uses timeout parameters. Set connection and read timeouts on every external call (vector database, reranker API, LLM API) explicitly. Without timeouts, a slow component blocks the entire pipeline. When a component exceeds its budget, log the breach and either return a degraded result or fail fast rather than waiting for the component to complete.
- p99 vs p95 vs average: average latency is a poor SLA metric because it hides tail behaviour. A p95 of 2000ms means 5% of users wait longer than 2 seconds — often acceptable. A p99 of 10000ms means 1% of users wait 10 seconds — usually unacceptable. Measure and alert on p95 and p99 separately. LLM inference has high latency variance (long outputs or high-load periods cause p99 to spike well above p95).
- Latency/quality trade-off decisions: when a component chronically exceeds its budget, the choices are: (1) replace it with a faster alternative (smaller reranker model, faster ANN algorithm), (2) reduce the workload (fewer retrieval candidates, shorter prompts), or (3) accept the budget overrun and raise the SLA. Evaluate each option against the quality impact using the eval suite.

**Common Misconceptions:**
- Latency only matters for real-time chat interfaces — background processing, batch analytics, and report generation also have latency SLAs; stakeholders notice when reports arrive late.
- Faster models are always better — faster models are usually less capable; the goal is to meet the latency budget with the highest quality model that fits, not to minimise latency at all costs.

**Interview Answer Skeleton:**
- **What it is:** The practice of explicitly decomposing an end-to-end latency SLA across each pipeline component, setting per-component timeout budgets, measuring actual p95/p99 per component, and making explicit quality-vs-speed trade-offs when components breach their allocations.
- **Why it matters / trade-offs:** Latency budgets make trade-off decisions explicit before they become fires in production. Without component-level budgets, the pipeline is a black box and optimisation is guesswork. The discipline of measuring p95/p99 per component — not just end-to-end average — is what reveals where the bottlenecks actually are.
- **Example or context:** RAG chatbot with a 2-second p95 SLA: embed (30ms) + ANN search (50ms) + rerank top-50 (120ms) + LLM TTFT (400ms) + first-token to complete stream (800ms) + network (50ms) = 1450ms budget consumed, 550ms headroom. Monitor each component via traces. When reranking spikes to 350ms at p99, switch to a smaller reranker model for the p99 cases or reduce candidate count from 50 to 20.

**Free Resources:**
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Latency profiling patterns and component-level performance optimisation for production AI systems
- [Langfuse Documentation](https://langfuse.com/docs) — Span-level latency measurement and p95/p99 analytics for LLM pipeline performance monitoring
