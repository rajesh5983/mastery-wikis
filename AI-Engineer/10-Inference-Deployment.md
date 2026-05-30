# Module 10 — Inference and Deployment

---

## Frontier APIs

**Status:** ⬜ Not Started

**Definition:** Frontier APIs are cloud-based endpoints for the most capable large language models — Anthropic Claude, OpenAI GPT-4o, Google Gemini — accessed via REST API. They abstract hardware, model loading, and scaling; you pay per token and get access to state-of-the-art capabilities with no infrastructure management.

**Key Mental Model:** Frontier APIs are like renting electricity from the grid — you don't build or maintain the power plant, you just plug in and pay for what you use. The provider handles capacity, reliability, and upgrades.

**How It Works:**
- Each API call is an HTTPS POST to the provider's inference endpoint. The request body contains the model identifier, message array, and generation parameters (temperature, max_tokens, tools). The server routes the request to an available inference worker, processes the prefill (reading the full prompt), then generates output tokens autoregressively until the stop condition is met.
- The provider manages all hardware allocation, model sharding across multiple GPUs (for very large models), batching of concurrent requests onto the same inference workers (continuous batching), and horizontal scaling of worker pools under load. The consumer sees none of this — just an API with latency and throughput characteristics.
- Rate limits are enforced at two dimensions: requests per minute (RPM) and tokens per minute (TPM). Each API key has tier-specific limits. The response headers include current usage against limits (`x-ratelimit-remaining-requests`, `x-ratelimit-remaining-tokens`), enabling proactive rate limit management rather than waiting for 429 responses.
- Provider model versioning matters. Providers sometimes silently update models (e.g., `claude-sonnet-4-5` pointing to an updated weights version). This can cause quality changes in production without a deployment on your side. Pin to versioned model IDs where providers offer them, and monitor for quality drift. See [[AI-Engineer/07-Observability-Evals]].
- Per-token pricing varies by input type: standard input tokens, cached tokens (typically 10% of standard input price for cache hits), and output tokens (typically 3–5× the standard input price). Building accurate cost projections requires measuring your actual input/output token distribution, cache hit rates, and request volumes.

**Common Misconceptions:**
- Frontier APIs are always the most cost-effective for production — frontier models cost 10–100x more than smaller open models; for high-volume, well-defined tasks, fine-tuned smaller models can be more cost-effective.
- Switching between frontier API providers is trivial — each provider has different capability profiles, tool-use formats, and context window behaviour; switching requires testing and likely prompt adjustments.

**Interview Answer Skeleton:**
- **What it is:** Managed REST API access to state-of-the-art LLMs where the provider handles all infrastructure — GPU provisioning, model serving, autoscaling, and hardware maintenance — billed per token with RPM/TPM rate limits.
- **Why it matters / trade-offs:** Frontier APIs are the fastest path to production-quality AI with zero infrastructure investment. The trade-offs versus self-hosting are: per-token cost at scale, data residency limitations, vendor lock-in risk, and silent model updates. These become relevant at scale or in regulated industries.
- **Example or context:** Choosing between Claude, GPT-4o, and Gemini for a document analysis task: run your custom eval suite against all three, compare scores, latency, and cost per query. Model selection should be data-driven, not based on marketing claims. Pin the selected model to a specific version ID and monitor for quality drift from silent provider updates.

**Free Resources:**
- [vLLM Documentation](https://docs.vllm.ai) — Context on self-hosted inference mechanics, providing a basis for comparing managed vs self-hosted trade-offs
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Production patterns for frontier API usage including cost management, rate limiting, and model versioning

---

## Inference Platforms

**Status:** ⬜ Not Started

**Definition:** Inference platforms (Together AI, Fireworks AI, Groq, Replicate) offer hosted inference for open-source models at lower cost and often higher throughput than running models yourself. They handle hardware, batching, and scaling while giving access to models like Llama, Mistral, and Qwen.

**Key Mental Model:** Inference platforms are like using a shared commercial kitchen instead of building your own — you get professional equipment at a fraction of the ownership cost, and someone else manages maintenance.

**How It Works:**
- Inference platforms run GPU clusters pre-loaded with popular open-source model weights. When a request arrives, the platform routes it to a worker that already has the model loaded in GPU memory — avoiding the multi-minute model loading latency of cold starts. Workers use continuous batching to process multiple concurrent requests on the same GPU simultaneously.
- Groq's hardware differentiator is the Language Processing Unit (LPU) — a custom chip optimised for the sequential, memory-bandwidth-bound operations of autoregressive token generation. LPUs can generate tokens 5–10x faster than equivalent GPU deployments, making Groq the platform of choice for latency-critical applications (real-time voice, interactive agents).
- Quantisation is applied to open models on inference platforms to reduce memory requirements and increase throughput. INT8 and INT4 quantisation compress model weights from FP16 (2 bytes/param) to 1 byte or 0.5 bytes/param respectively. A 70B parameter model in FP16 requires ~140GB VRAM; in INT4, it fits in ~35GB. Quality degradation from quantisation is task-dependent and should be validated on your eval set.
- The OpenAI-compatible API format has become the de facto standard for inference platforms — Together AI, Fireworks, and Groq all expose a `/v1/chat/completions` endpoint. This means code written against the OpenAI Python SDK works against these platforms with just a `base_url` change, reducing switching friction significantly.
- Pricing is typically charged per million tokens (input and output separately), often at 5–20x lower rates than frontier API providers for equivalent capability tiers. The trade-off is that open models, even large ones, have capability gaps compared to frontier models on certain task categories.

**Common Misconceptions:**
- Inference platforms are only for development/prototyping — platforms like Groq and Fireworks AI run production workloads at enterprise scale with SLA commitments.
- All inference platforms support the same models — model availability, quantisation options, and throughput capabilities differ significantly between platforms; verify support for your specific model.

**Interview Answer Skeleton:**
- **What it is:** Managed hosting services for open-source model inference that provide GPU clusters pre-loaded with popular models (Llama, Mistral, Qwen), accessible via OpenAI-compatible APIs — bridging the gap between frontier managed APIs and full self-hosting.
- **Why it matters / trade-offs:** Inference platforms enable open model deployment without GPU infrastructure expertise, at 5–20x lower per-token cost than frontier APIs. Trade-offs: open model capability gaps on complex tasks, less predictable quality (model updates, platform issues), and data residency still depends on the platform's cloud provider.
- **Example or context:** A real-time customer support application with strict latency requirements: Groq (LPU-based) for the primary inference path because it generates tokens 5–10x faster than GPU inference. Use Llama-3.1-70B-Instruct — adequate quality for well-structured support queries at a fraction of frontier API cost. Validate quality against your custom eval suite before launching, and maintain a fallback to a frontier API for queries the open model scores poorly on.

**Free Resources:**
- [vLLM Documentation](https://docs.vllm.ai) — Understanding inference optimisation techniques (continuous batching, quantisation) that inference platforms implement under the hood
- [Papers With Code](https://paperswithcode.com) — Open model benchmarks and capability comparisons to inform platform and model selection decisions

---

## Cloud AI

**Status:** ⬜ Not Started

**Definition:** Cloud AI offerings (AWS Bedrock, Google Vertex AI, Azure AI Foundry) provide frontier and open models through major cloud providers, enabling data residency compliance, private networking, and integration with cloud-native IAM, logging, and billing — important for enterprise deployments.

**Key Mental Model:** Cloud AI is frontier model access through your existing cloud contract — same models, but with enterprise security controls, private endpoints, and billing through your existing AWS/Azure/GCP account and compliance framework.

**How It Works:**
- AWS Bedrock wraps multiple model providers (Anthropic, Meta, Mistral, AI21) under a unified AWS API. API calls authenticate using AWS IAM roles rather than provider-specific API keys, enabling integration with existing AWS security policies, CloudTrail audit logs, and VPC private endpoints that never leave AWS infrastructure.
- Private endpoints (AWS VPC Endpoints, Azure Private Link, GCP Private Service Connect) route LLM API traffic over the cloud provider's private backbone rather than the public internet. This satisfies data-in-transit residency requirements and reduces exposure to public internet threats.
- Cloud providers offer provisioned throughput options in addition to on-demand pricing. AWS Bedrock's Provisioned Throughput reserves a fixed model inference capacity (measured in Model Units) at a fixed hourly rate, providing guaranteed throughput and latency for production workloads with predictable traffic — versus on-demand which is usage-based but subject to rate limits.
- Data logging and compliance features are built in: CloudTrail logs every Bedrock API call with request metadata (but not prompt content by default), enabling audit trails for compliance frameworks (SOC2, HIPAA, GDPR). Guardrail features (AWS Bedrock Guardrails, Vertex AI safety filters) add provider-managed content filtering that applies before the model response is returned.
- Fine-tuning and custom model hosting are available on cloud AI platforms. You can fine-tune Claude or Llama models on your own data (stored in S3/GCS) without the data leaving your cloud account, then deploy the fine-tuned weights to a managed endpoint. This combines the compliance of self-hosting with the operational simplicity of managed inference.

**Common Misconceptions:**
- Cloud AI providers are slower than direct API providers — for many models, cloud AI providers are the same infrastructure as the direct APIs (e.g., Claude on Bedrock runs on Anthropic infrastructure).
- Cloud AI is only for compliance-heavy industries — the operational benefits (unified billing, private networking, IAM integration) are valuable for any enterprise regardless of compliance requirements.

**Interview Answer Skeleton:**
- **What it is:** Managed LLM access through major cloud providers (AWS, GCP, Azure) that adds enterprise security controls — IAM authentication, private networking, audit logging, data residency guarantees, and provisioned throughput — on top of frontier model capabilities.
- **Why it matters / trade-offs:** Enterprise AI deployments almost always require cloud provider integration for procurement (existing contracts), security (IAM, private networking), and compliance (audit trails, data residency). The trade-off is slightly higher cost and latency compared to direct provider APIs, and a dependency on the cloud provider's model availability roadmap.
- **Example or context:** An enterprise financial services firm building an internal AI assistant: deploy Claude via AWS Bedrock with VPC private endpoints (data never leaves AWS), IAM role-based access (no shared API keys), CloudTrail audit logging (demonstrates compliance), and provisioned throughput (predictable latency for internal SLAs). All of these would require custom engineering if calling Anthropic directly.

**Free Resources:**
- [vLLM Documentation](https://docs.vllm.ai) — Self-hosting context that clarifies when cloud AI managed services are preferable to running your own inference
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Deployment patterns including cloud provider integration for enterprise Claude deployments

---

## Self-Hosting

**Status:** ⬜ Not Started

**Definition:** Self-hosting runs open-source LLMs (Llama, Mistral, Qwen) on your own infrastructure using inference servers like vLLM, Ollama, or TGI (Text Generation Inference). This enables data residency control, predictable cost at scale, custom model fine-tuning, and no per-token billing.

**Key Mental Model:** Self-hosting LLMs is like building your own power generator — higher upfront cost and operational burden, but complete control and predictable costs once operational, especially at scale.

**How It Works:**
- vLLM is the production-standard open-source inference server. Its core innovation is PagedAttention — a memory management technique that stores KV cache in non-contiguous GPU memory pages, similar to how an OS manages virtual memory pages. This prevents the memory fragmentation that occurs with fixed contiguous KV cache allocation, enabling much higher GPU utilisation and batch sizes.
- Continuous batching (implemented by vLLM and TGI) dynamically adds new requests to the current batch as generation steps complete. In a naive batching approach, all sequences in a batch must finish before new ones can start — short sequences block long ones from releasing their GPU compute. Continuous batching inserts new requests at each generation step, keeping GPU utilisation near 100%.
- Tensor parallelism shards the model across multiple GPUs. A 70B parameter model in FP16 requires ~140GB of VRAM — far exceeding a single A100 80GB. Tensor parallelism splits each weight matrix across N GPUs and uses NVLink all-reduce operations to synchronise activations at each layer. 2-GPU tensor parallel is common for 70B models.
- Quantisation (GPTQ, AWQ, GGUF) reduces model weight precision from FP16 to INT8 or INT4, halving or quartering VRAM requirements. vLLM supports loading pre-quantised models and running INT8 activation quantisation (bitsandbytes). Quality impact is model and task dependent — always validate with your eval suite after quantising.
- KV cache sharing (via vLLM's prefix caching feature) caches the KV tensors for identical prompt prefixes across requests. When multiple requests share the same system prompt, the prefix is computed once and the KV cache is reused. This is the self-hosted equivalent of Anthropic's prompt caching, and it reduces per-request prefill compute and latency for high-traffic shared-prefix workloads.

**Common Misconceptions:**
- Self-hosting is only for cost savings — self-hosting is often chosen for data residency requirements, air-gapped deployments, or the ability to fine-tune models on proprietary data.
- Self-hosting is straightforward once you have GPUs — production self-hosting requires expertise in quantisation, batching strategies, model loading, autoscaling, and GPU memory management.

**Interview Answer Skeleton:**
- **What it is:** Running open-source LLMs on owned or rented GPU infrastructure using inference servers (vLLM, TGI) that implement production optimisations — PagedAttention for memory efficiency, continuous batching for GPU utilisation, tensor parallelism for multi-GPU sharding, and prefix caching for repeated system prompts.
- **Why it matters / trade-offs:** Self-hosting enables full data residency control, no per-token billing (predictable costs at scale), and the ability to fine-tune on proprietary data. The trade-offs are significant: GPU procurement or rental cost, operational engineering burden (model management, autoscaling, hardware failures), and the capability gap between open models and frontier models on complex tasks.
- **Example or context:** Self-hosting Llama-3.1-70B on 2× A100-80GB with vLLM: tensor parallel 2 (model sharded across both GPUs), INT8 quantisation (70B model fits in ~70GB total VRAM), continuous batching enabled, prefix caching for the shared system prompt (cached after first request, reused on all subsequent). Monitor GPU utilisation (target > 85%), queue depth (requests waiting for batch slot), and TTFT latency. Scale out by adding GPU pairs and a load balancer.

**Free Resources:**
- [vLLM Documentation](https://docs.vllm.ai) — Complete documentation for production vLLM deployment including PagedAttention, continuous batching, tensor parallelism, and quantisation configuration
- [Papers With Code](https://paperswithcode.com) — Open model capability benchmarks and quantisation research to inform self-hosting model and precision selection
