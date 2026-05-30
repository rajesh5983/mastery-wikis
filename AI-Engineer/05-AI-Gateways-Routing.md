# Module 5 — AI Gateways and Routing

---

## Model Routing

**Status:** ⬜ Not Started

**Definition:** Model routing dynamically selects the best model for each request based on task complexity, latency requirements, cost constraints, or capability needs. A simple query might be routed to a fast, cheap model; a complex reasoning task gets routed to a more capable (and expensive) frontier model.

**Key Mental Model:** Model routing is like a triage system at a hospital — a nurse assesses each case and routes it to the right specialist (or handles it directly for minor issues) rather than sending everything to the most senior doctor.

**How It Works:**
- A routing layer sits between the application and the LLM provider. Each incoming request is evaluated by a routing classifier — this can be a rules-based system (query length, keyword presence), a small fast LLM judge ("is this query simple or complex?"), or an ML classifier trained on labelled examples.
- The classifier assigns a route label (e.g., "simple", "moderate", "complex") and the routing layer maps this to a specific model endpoint. The mapping table is configurable — `simple → claude-haiku-3-5`, `complex → claude-sonnet-4-5`, `reasoning → claude-sonnet-4-5 with extended thinking`.
- Routing decisions can also incorporate non-complexity signals: the user's subscription tier (premium users get faster models), the time of day (load-balance across providers during peak hours), or the topic domain (use a domain-fine-tuned model for medical queries).
- The routing layer tracks which route was taken for each request, enabling post-hoc analysis of routing accuracy. When a "simple" routed query produces a low-quality response, that example becomes a training signal for retraining the classifier to route similar queries to a stronger model.
- Routing interacts with fallback chains and cost tracking. If the primary routed model is rate-limited, the fallback must serve a model of comparable capability rather than always falling back to the cheapest option. See [[AI-Engineer/05-AI-Gateways-Routing]] fallback configuration.

**Common Misconceptions:**
- Always use the most capable model for reliability — the most capable model is also the slowest and most expensive; routing to cheaper models for simple tasks reduces cost 60–80% with no quality loss.
- Routing requires complex classification — a simple rules-based classifier or a small LLM judge routing on query complexity is sufficient for most applications.

**Interview Answer Skeleton:**
- **What it is:** A classification layer that evaluates each incoming request and directs it to the appropriate LLM endpoint based on a cost/capability/latency optimisation function — most commonly routing simple queries to cheap fast models and complex queries to more capable ones.
- **Why it matters / trade-offs:** Routing is one of the highest-ROI production optimisations — 60–80% cost reduction on mixed-complexity workloads with zero quality loss on complex queries. The risk is misrouting: a complex query sent to a weak model produces bad output. Classifier quality and fallback logic are the engineering priorities.
- **Example or context:** A customer support application: the router classifies "what are your opening hours?" as simple (→ Claude Haiku, $0.00025/1K tokens, 200ms) and "explain why my international wire transfer was reversed" as complex (→ Claude Sonnet, higher cost, 500ms). Monitor misrouting via user feedback and quality scores, and retrain the classifier on failure cases monthly.

**Free Resources:**
- [LiteLLM Documentation](https://docs.litellm.ai) — Multi-provider routing, model selection configuration, and load balancing patterns
- [Eugene Yan's Blog](https://eugeneyan.com) — Applied ML posts on model routing strategies, classifier design, and production cost optimisation

---

## Fallbacks

**Status:** ⬜ Not Started

**Definition:** Fallback strategies define what happens when a primary model fails — due to rate limits, downtime, or timeout. A fallback hierarchy tries backup providers or models in sequence, ensuring the application continues to serve requests even when the primary endpoint is unavailable.

**Key Mental Model:** Fallbacks are backup generators — when the main power fails, the generator kicks in automatically without waiting for a human to flip the switch.

**How It Works:**
- The gateway wraps each LLM API call in an exception handler. When the primary call raises a rate limit error (HTTP 429), a timeout, or a provider-specific error, the handler catches the exception and immediately retries the request against the next provider in the fallback chain — without waiting for the primary to recover.
- LiteLLM's fallback configuration is declarative: you define a list of models in priority order and the framework handles exception catching, provider translation, and retry automatically. The application code makes a single `litellm.completion()` call; the framework resolves which provider actually handles it.
- Exponential backoff with jitter is used for retrying the same provider after transient errors (e.g., a 503 that indicates temporary overload). The retry waits 2^n seconds plus a random jitter value (to avoid thundering herd) before retrying — n increments with each retry attempt, up to a configured maximum.
- Fallback models must be pre-validated for compatibility. If the primary model supports tool calling with a specific schema and the fallback model uses a different tool calling format, the fallback will fail at the application layer even if the API call succeeds. Test fallback paths in staging with your actual prompt and schema.
- Circuit breaker patterns extend simple fallbacks: after N consecutive failures to a provider, the circuit "opens" and all traffic bypasses that provider for a cooldown period (e.g., 60 seconds). This prevents timeout cascades from a failing provider slowing all requests while the gateway waits for each primary timeout to expire.

**Common Misconceptions:**
- Fallbacks are only needed for unreliable providers — even highly reliable providers experience rate limits and regional outages; fallbacks are standard reliability engineering, not a workaround for bad providers.
- Any model can be a fallback for any other — different models have different capabilities, output calling formats, and token limits; fallback models must be validated to handle the same workload.

**Interview Answer Skeleton:**
- **What it is:** Automatic failover logic that catches provider errors (rate limits, timeouts, outages) and retries the request against a pre-configured sequence of backup providers — using exponential backoff for transient errors and immediate failover for provider-level failures.
- **Why it matters / trade-offs:** Fallbacks are standard reliability infrastructure for any production AI system with an SLA. Without them, a single provider outage takes down the application. The trade-off is fallback model compatibility — different models behave differently, and a fallback to a weaker model may produce lower-quality responses.
- **Example or context:** Primary: Claude Sonnet. Fallback 1: Claude Haiku (same provider, lower tier, rarely rate-limited). Fallback 2: GPT-4o-mini (different provider, insulates against Anthropic-wide outages). Validate all three against your prompt suite before deploying. Set timeouts aggressively (e.g., 10s primary timeout) to prevent slow providers from blocking the fallback.

**Free Resources:**
- [LiteLLM Documentation](https://docs.litellm.ai) — Fallback configuration, retry logic, exponential backoff, and circuit breaker implementation
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Reliability patterns for production LLM applications including fallback and retry strategies

---

## Cost Tracking

**Status:** ⬜ Not Started

**Definition:** Cost tracking monitors LLM API spend at the request, user, feature, and model level. This enables cost attribution (which feature/team/user drives most spend), budget enforcement, anomaly detection (sudden cost spikes), and informed decisions about routing and caching.

**Key Mental Model:** LLM cost tracking is like a cloud billing dashboard — you need to know which workload is driving spend before you can optimise. Without tracking, you're flying blind and cost surprises arrive at month-end.

**How It Works:**
- Per-request cost is calculated at the gateway layer from the token counts returned in the API response metadata. Input tokens × input price + output tokens × output price = request cost. For models with tiered pricing (e.g., cached vs non-cached tokens), the calculation uses the appropriate rate for each token category.
- Request metadata — user ID, feature name, model used, routing path, prompt cache hit/miss, input/output token counts, and calculated cost — is written to a cost log (e.g., a Postgres table or a data warehouse table) synchronously or via an async fire-and-forget write.
- Budget enforcement works by querying the cost log in real-time. Before processing a request, the gateway checks the user's or tenant's spend in the current billing period. If the spend exceeds the configured budget threshold, the gateway returns a budget-exceeded error rather than making the API call. This prevents runaway spend from a single tenant or a cost-anomalous workflow.
- Anomaly detection runs as a background job (e.g., hourly) that computes the rolling average cost per user/feature and alerts when the current period deviates significantly from the baseline. Common causes of cost spikes: a new code path that sends large untruncated documents, a prompt that entered a repetition loop, or a user systematically trying to extract maximum output.
- Cost attribution to product features requires tagging every LLM call with a `feature_name` metadata field at call time. At reporting time, this enables SQL aggregations like "cost per feature per week" that reveal which product areas are cost-intensive and whether optimisations (caching, routing, prompt compression) are working.

**Common Misconceptions:**
- API invoice totals are sufficient for cost management — aggregate totals hide which feature or user is responsible; request-level logging with metadata is essential for attribution and optimisation.
- Cost tracking is a post-launch concern — instrumenting cost tracking at launch is far easier than retrofitting it; it should be built in from the start.

**Interview Answer Skeleton:**
- **What it is:** Infrastructure that calculates per-request LLM cost from token usage metadata, logs it with business-context tags (user, feature, model, route), and provides real-time budget enforcement and anomaly alerting.
- **Why it matters / trade-offs:** Production AI systems can accumulate unexpected costs from prompt bloat, model misrouting, or user abuse. Granular cost tracking is what enables the optimisation loop: identify the expensive workloads, target them with caching, routing, and prompt compression, and measure the actual cost savings.
- **Example or context:** A multi-tenant SaaS with 1,000 users: tag every call with `{user_id, feature, model, cache_hit}`. After two weeks, query reveals that 5% of users drive 40% of cost because they are using a "summarise document" feature with large PDFs. The fix: add prompt caching for the document context and route PDF summaries to a cheaper model. Re-query two weeks later to confirm the cost reduction.

**Free Resources:**
- [LiteLLM Documentation](https://docs.litellm.ai) — Usage tracking, spend logging, budget controls, and per-user cost attribution
- [Langfuse Documentation](https://langfuse.com/docs) — LLM observability with cost tracking, token analytics, and model usage dashboards

---

## Multi-Provider Abstraction

**Status:** ⬜ Not Started

**Definition:** Multi-provider abstraction is a layer that presents a unified API interface to the application while translating calls to multiple backend LLM providers (Anthropic, OpenAI, Google, Mistral, AWS Bedrock). This decouples the application from any single provider, enabling switching, mixing, and comparison without rewriting application code.

**Key Mental Model:** Multi-provider abstraction is like a universal power adapter — one interface, many plugs. The application speaks one API regardless of which provider is plugged in behind it.

**How It Works:**
- The abstraction layer (LiteLLM being the standard open-source choice) exposes a single `completion(model, messages, **kwargs)` function. Internally, it maps the model identifier string to the corresponding provider SDK, translates the unified message format to the provider-specific format, and handles auth headers and endpoint URLs automatically.
- Provider-specific differences that the abstraction layer normalises include: message role naming conventions, tool calling schema formats (OpenAI's `functions` vs Anthropic's `tools` format), streaming chunk structure, error code conventions, and token count field names in the response object.
- The abstraction layer does not normalise semantic differences between models. A prompt that works well with Claude's XML structuring may not work equally well with GPT-4o which has different formatting preferences. True provider portability requires testing prompts against each target model, not just swapping the model string.
- LiteLLM can be deployed as a proxy server — a self-hosted HTTP service that the application calls instead of calling provider APIs directly. This centralises API key management, adds logging, enables rate limiting, and allows routing and fallback configuration to be updated without application deploys.
- Model identifier strings in LiteLLM follow the pattern `provider/model-name` (e.g., `anthropic/claude-sonnet-4-5`, `openai/gpt-4o`, `bedrock/anthropic.claude-3-5-sonnet`). This explicit namespacing makes the provider visible in code and avoids ambiguity when two providers offer similarly-named models.

**Common Misconceptions:**
- Abstracting providers means all models are interchangeable — model behaviours, tool calling formats, context windows, and capability profiles differ significantly; abstraction handles the transport layer, not the semantic differences.
- Building a custom abstraction layer is simpler than using an existing tool — libraries like LiteLLM handle dozens of provider quirks, error formats, and auth mechanisms; building from scratch is high-maintenance.

**Interview Answer Skeleton:**
- **What it is:** A unified API translation layer (typically LiteLLM) that normalises provider-specific request/response formats, auth mechanisms, and error conventions into a single interface — decoupling application code from any specific LLM provider.
- **Why it matters / trade-offs:** Prevents vendor lock-in and enables A/B testing across providers, fallback hierarchies, and cost-based routing without application code changes. The trade-off is that transport-layer abstraction does not solve semantic model differences — prompts still need to be validated per model.
- **Example or context:** Using LiteLLM, the application calls `litellm.completion(model="anthropic/claude-sonnet-4-5", messages=messages)`. Swapping to GPT-4o is a one-character change to the model string. To A/B test both in production, configure LiteLLM's routing to split 50% of traffic to each model and log quality metrics per provider to compare outputs objectively.

**Free Resources:**
- [LiteLLM Documentation](https://docs.litellm.ai) — Full multi-provider setup guide, proxy server deployment, routing configuration, and supported providers
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Multi-provider patterns and provider comparison examples with LiteLLM integration
