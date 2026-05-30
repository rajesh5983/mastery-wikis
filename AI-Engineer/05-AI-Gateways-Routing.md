# Module 5 — AI Gateways and Routing

---

## Model Routing

**Status:** ⬜ Not Started

**Definition:** Model routing dynamically selects the best model for each request based on task complexity, latency requirements, cost constraints, or capability needs. A simple query might be routed to a fast, cheap model; a complex reasoning task gets routed to a more capable (and expensive) frontier model.

**Mental Model:** Model routing is like a triage system at a hospital — a nurse assesses each case and routes it to the right specialist (or handles it directly for minor issues) rather than sending everything to the most senior doctor.

**Common Misconceptions:**
- Always use the most capable model for reliability — the most capable model is also the slowest and most expensive; routing to cheaper models for simple tasks reduces cost 60–80% with no quality loss.
- Routing requires complex classification — a simple rules-based classifier or a small LLM judge routing on query complexity is sufficient for most applications.

**Interview Skeleton:**
- What it is: the practice of selecting different LLM models for different requests based on a cost/capability/latency trade-off
- Why it matters: routing is one of the most impactful cost optimisations in production AI systems
- Example: design a routing system for a customer support application that routes FAQ-style queries to Haiku and complex escalations to Opus

**Free Resources:** https://docs.litellm.ai — LiteLLM documentation covering model routing, load balancing, and fallback configuration

---

## Fallbacks

**Status:** ⬜ Not Started

**Definition:** Fallback strategies define what happens when a primary model fails — due to rate limits, downtime, or timeout. A fallback hierarchy tries backup providers or models in sequence, ensuring the application continues to serve requests even when the primary endpoint is unavailable.

**Mental Model:** Fallbacks are backup generators — when the main power fails, the generator kicks in automatically without waiting for a human to flip the switch.

**Common Misconceptions:**
- Fallbacks are only needed for unreliable providers — even highly reliable providers experience rate limits and regional outages; fallbacks are standard reliability engineering, not a workaround for bad providers.
- Any model can be a fallback for any other — different models have different capabilities, output formats, and token limits; fallback models must be validated to handle the same workload.

**Interview Skeleton:**
- What it is: automatic failover logic that routes requests to backup models when primary models are unavailable or rate-limited
- Why it matters: ensures application availability meets SLA commitments despite individual provider outages or throttling
- Example: configure a fallback chain: Claude Sonnet → Claude Haiku → GPT-4o-mini, with criteria for when each escalation triggers

**Free Resources:** https://docs.litellm.ai — LiteLLM documentation on fallback configuration and retry strategies

---

## Cost Tracking

**Status:** ⬜ Not Started

**Definition:** Cost tracking monitors LLM API spend at the request, user, feature, and model level. This enables cost attribution (which feature/team/user drives most spend), budget enforcement, anomaly detection (sudden cost spikes), and informed decisions about routing and caching.

**Mental Model:** LLM cost tracking is like a cloud billing dashboard — you need to know which workload is driving spend before you can optimise. Without tracking, you're flying blind and cost surprises arrive at month-end.

**Common Misconceptions:**
- API invoice totals are sufficient for cost management — aggregate totals hide which feature or user is responsible; request-level logging with metadata is essential for attribution and optimisation.
- Cost tracking is a post-launch concern — instrumenting cost tracking at launch is far easier than retrofitting it; it should be built in from the start.

**Interview Skeleton:**
- What it is: observability infrastructure for tracking LLM API spend at granular request and business-unit levels
- Why it matters: production AI systems can accumulate unexpected costs from prompt bloat, model choice, or sudden traffic spikes
- Example: design a cost tracking system for a multi-tenant AI application that can attribute costs by user, feature, and model

**Free Resources:** https://docs.litellm.ai — LiteLLM documentation covering usage tracking, spend logs, and budget controls

---

## Multi-Provider Abstraction

**Status:** ⬜ Not Started

**Definition:** Multi-provider abstraction is a layer that presents a unified API interface to the application while translating calls to multiple backend LLM providers (Anthropic, OpenAI, Google, Mistral, AWS Bedrock). This decouples the application from any single provider, enabling switching, mixing, and comparison without rewriting application code.

**Mental Model:** Multi-provider abstraction is like a universal power adapter — one interface, many plugs. The application speaks one API regardless of which provider is plugged in behind it.

**Common Misconceptions:**
- Abstracting providers means all models are interchangeable — model behaviours, tool calling formats, context windows, and capability profiles differ significantly; abstraction handles the transport layer, not the semantic differences.
- Building a custom abstraction layer is simpler than using an existing tool — libraries like LiteLLM handle dozens of provider quirks, error formats, and auth mechanisms; building from scratch is high-maintenance.

**Interview Skeleton:**
- What it is: a unified interface layer that translates a standard API to multiple LLM provider backends
- Why it matters: avoids vendor lock-in, enables A/B testing across providers, and simplifies routing and fallback logic
- Example: describe how you'd use LiteLLM to abstract over Claude, GPT-4o, and Gemini in a single application codebase

**Free Resources:** https://docs.litellm.ai — LiteLLM documentation covering multi-provider setup, unified interface, and supported providers
