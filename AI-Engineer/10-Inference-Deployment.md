# Module 10 — Inference and Deployment

---

## Frontier APIs

**Status:** ⬜ Not Started

**Definition:** Frontier APIs are cloud-based endpoints for the most capable large language models — Anthropic Claude, OpenAI GPT-4o, Google Gemini — accessed via REST API. They abstract hardware, model loading, and scaling; you pay per token and get access to state-of-the-art capabilities with no infrastructure management.

**Mental Model:** Frontier APIs are like renting electricity from the grid — you don't build or maintain the power plant, you just plug in and pay for what you use. The provider handles capacity, reliability, and upgrades.

**Common Misconceptions:**
- Frontier APIs are always the most cost-effective for production — frontier models cost 10–100x more than smaller open models; for high-volume, well-defined tasks, fine-tuned smaller models can be more cost-effective.
- Switching between frontier API providers is trivial — each provider has different capability profiles, tool-use formats, and context window behaviour; switching requires testing and likely prompt adjustments.

**Interview Skeleton:**
- What it is: managed API access to state-of-the-art LLMs without infrastructure management
- Why it matters: the fastest path to production-grade AI capability; appropriate for most applications that don't require data residency or custom models
- Example: compare Claude, GPT-4o, and Gemini for a specific task — what factors would determine your choice?

**Free Resources:** https://docs.vllm.ai — vLLM documentation for context on self-hosted inference vs managed frontier APIs

---

## Inference Platforms

**Status:** ⬜ Not Started

**Definition:** Inference platforms (Together AI, Fireworks AI, Groq, Replicate) offer hosted inference for open-source models at lower cost and often higher throughput than running models yourself. They handle hardware, batching, and scaling while giving access to models like Llama, Mistral, and Qwen.

**Mental Model:** Inference platforms are like using a shared commercial kitchen instead of building your own — you get professional equipment at a fraction of the ownership cost, and someone else manages maintenance.

**Common Misconceptions:**
- Inference platforms are only for development/prototyping — platforms like Groq and Fireworks AI run production workloads at enterprise scale with SLA commitments.
- All inference platforms support the same models — model availability, quantisation options, and throughput capabilities differ significantly between platforms; verify support for your specific model.

**Interview Skeleton:**
- What it is: managed hosting for open-source model inference, bridging the gap between frontier APIs and full self-hosting
- Why it matters: enables use of open models at production scale without GPU infrastructure expertise or capital expenditure
- Example: when would you choose Groq over Together AI over a frontier API for a specific production workload?

**Free Resources:** https://docs.vllm.ai — vLLM documentation for understanding inference optimisation techniques used by platforms like these

---

## Cloud AI

**Status:** ⬜ Not Started

**Definition:** Cloud AI offerings (AWS Bedrock, Google Vertex AI, Azure AI Foundry) provide frontier and open models through major cloud providers, enabling data residency compliance, private networking, and integration with cloud-native IAM, logging, and billing — important for enterprise deployments.

**Mental Model:** Cloud AI is frontier model access through your existing cloud contract — same models, but with enterprise security controls, private endpoints, and billing through your existing AWS/Azure/GCP account and compliance framework.

**Common Misconceptions:**
- Cloud AI providers are slower than direct API providers — for many models, cloud AI providers are the same infrastructure as the direct APIs (e.g., Claude on Bedrock runs on Anthropic infrastructure).
- Cloud AI is only for compliance-heavy industries — the operational benefits (unified billing, private networking, IAM integration) are valuable for any enterprise regardless of compliance requirements.

**Interview Skeleton:**
- What it is: access to LLMs through major cloud providers with enterprise security, compliance, and integration features
- Why it matters: most enterprise AI deployments require cloud provider integration for security, compliance, and procurement reasons
- Example: describe the trade-offs between calling Claude directly via the Anthropic API vs via AWS Bedrock for an enterprise deployment

**Free Resources:** https://docs.vllm.ai — vLLM documentation provides context on the self-hosted alternative to cloud AI managed services

---

## Self-Hosting

**Status:** ⬜ Not Started

**Definition:** Self-hosting runs open-source LLMs (Llama, Mistral, Qwen) on your own infrastructure using inference servers like vLLM, Ollama, or TGI (Text Generation Inference). This enables data residency control, predictable cost at scale, custom model fine-tuning, and no per-token billing.

**Mental Model:** Self-hosting LLMs is like building your own power generator — higher upfront cost and operational burden, but complete control and predictable costs once operational, especially at scale.

**Common Misconceptions:**
- Self-hosting is only for cost savings — self-hosting is often chosen for data residency requirements, air-gapped deployments, or the ability to fine-tune models on proprietary data.
- Self-hosting is straightforward once you have GPUs — production self-hosting requires expertise in quantisation, batching strategies, model loading, autoscaling, and GPU memory management.

**Interview Skeleton:**
- What it is: running open-source LLMs on owned or rented GPU infrastructure with full control over the inference stack
- Why it matters: enables data residency compliance, custom model deployment, and predictable unit economics at scale
- Example: describe the infrastructure design for self-hosting a 70B parameter model in production, including batching, scaling, and monitoring

**Free Resources:** https://docs.vllm.ai — vLLM documentation covering deployment, configuration, quantisation, and production inference optimisation
