# Module 2 — Prompt and Context Engineering

---

## System Prompts

**Status:** ⬜ Not Started

**Definition:** The system prompt is a privileged instruction block sent before any user content that sets the model's role, behaviour, constraints, and output format for the entire conversation. It is the primary lever for customising a frontier model for a specific application without fine-tuning.

**Key Mental Model:** The system prompt is the employee handbook — it defines the role, the rules, the tone, and what is out of scope. Every response flows from those standing instructions, regardless of what the user asks.

**How It Works:**
- At the API level, the system prompt is passed as a separate `system` parameter (Anthropic) or as a message with `role: "system"` (OpenAI). The model processes it as the highest-priority context during attention computation, placing it before all user and assistant turns.
- The system prompt participates in the KV cache like any other tokens. Because it is the static prefix that changes least frequently across requests, it is the ideal candidate for prompt caching — the system prompt's KV activations are computed once and reused across thousands of calls.
- Models attend back to the system prompt at every generation step. A clear, explicit system prompt persistently biases the conditional probability distribution toward the desired behaviour, even when the user's query does not explicitly reference the instructions.
- In multi-turn conversations, the full conversation history (system + all prior turns + current user message) is packed into the context window. As the conversation grows, the system prompt's relative weight shrinks; models can "drift" away from instructions when the conversation is many turns long.
- Security boundaries enforced by system prompts can be bypassed via prompt injection — crafted user input that attempts to override or ignore system instructions. Critical business rules should be enforced at the application layer, not solely in the system prompt.

**Common Misconceptions:**
- System prompts are always fully followed — models can drift from system prompt instructions, especially in long conversations; important constraints should be reinforced or tested regularly.
- System prompts are secret and protected — in most implementations, a determined user can elicit the system prompt contents through clever prompting; treat them as operational guidance, not security boundaries.

**Interview Answer Skeleton:**
- **What it is:** A privileged instruction block processed before user turns that configures model role, behaviour, format, and constraints for all subsequent responses in the conversation.
- **Why it matters / trade-offs:** System prompts are the lowest-friction way to specialise a general model for a specific application. They interact with prompt caching (significant cost savings), security concerns (prompt injection), and conversation length (drift over many turns).
- **Example or context:** A customer support bot system prompt should declare the persona, scope limitations, tone, escalation policy, and output format. Test it against adversarial inputs that try to make the model discuss out-of-scope topics — models are not inherently bounded by system prompt constraints without guardrail layers.

**Free Resources:**
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Anthropic's canonical documentation on system prompts, roles, and best practices
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Production-ready examples of system prompt patterns for common application types

---

## Few-Shot Prompting

**Status:** ⬜ Not Started

**Definition:** Few-shot prompting provides the model with examples of the desired input-output pattern directly in the prompt, before presenting the actual task. This leverages the model's in-context learning ability — it infers the pattern from examples without any weight updates.

**Key Mental Model:** Few-shot prompting is like showing a new employee three completed expense reports before asking them to fill one out — they infer the format and standard from the examples rather than reading a lengthy manual.

**How It Works:**
- Examples in the prompt shift the conditional probability distribution for the target output. Each example is a (input, output) pair that the model attends to when generating its response; the examples act as soft constraints on format, vocabulary, length, and reasoning style.
- The model processes examples via in-context learning — a capability that emerges from pre-training on diverse data containing many implicit few-shot patterns. The model is not fine-tuned; it is pattern-completing at inference time.
- Example order matters. Later examples in the prompt have greater recency weight in the attention distribution and have disproportionate influence on the response. Placing the most representative or difficult example last tends to improve output quality.
- Label quality in examples matters more than quantity. A single perfect example that precisely matches the target format often outperforms five mediocre examples with inconsistent formatting. Contradictory examples actively hurt performance.
- Few-shot examples consume context window tokens — each example can cost 50–500 tokens depending on task complexity. At large scale, this adds meaningfully to API costs. Dynamic few-shot selection (retrieving the most relevant examples for each query using embedding similarity) optimises both quality and cost.

**Common Misconceptions:**
- More examples are always better — examples consume context window tokens; 2–5 well-chosen, diverse examples typically outperform 20 redundant ones.
- Examples must be real data — for many tasks, synthetic examples that clearly illustrate the pattern work as well or better than real data.

**Interview Answer Skeleton:**
- **What it is:** Providing labelled input-output examples directly in the prompt to guide the model's output format and reasoning pattern via in-context learning — no weight updates required.
- **Why it matters / trade-offs:** Few-shot prompting dramatically improves output consistency for structured tasks. The cost is token overhead (each example counts against your context budget) and the need to curate high-quality, representative examples.
- **Example or context:** For extracting structured fields from customer emails, provide 3 examples showing the raw email and the expected JSON output. Use dynamic example retrieval to select the 3 examples most semantically similar to the current email, so the examples stay relevant without inflating the context window with a fixed large set.

**Free Resources:**
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Covers in-context learning, example selection, and few-shot formatting best practices
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Explains in-context learning mechanics and how pre-training enables few-shot generalisation

---

## Chain of Thought

**Status:** ⬜ Not Started

**Definition:** Chain of Thought (CoT) prompting instructs the model to produce step-by-step reasoning before arriving at a final answer. This can be elicited with instructions like "think step by step" or by providing CoT examples. It significantly improves performance on reasoning, math, and multi-step tasks.

**Key Mental Model:** CoT is asking someone to show their work, not just give an answer. The act of writing out each step forces sequential reasoning and reduces the chance of skipping to a wrong conclusion.

**How It Works:**
- When the model generates a reasoning step, that step becomes part of the context for all subsequent generation. Each reasoning token conditions the probability distribution for the next token. Writing "therefore X" as an intermediate step shifts subsequent output toward conclusions consistent with X, effectively constraining the solution space.
- CoT works because the model's probability distribution for a correct final answer is much higher when conditioned on a correct intermediate reasoning chain than when generated directly. The reasoning chain acts as a scaffold that the final answer generation can attend back to.
- Zero-shot CoT ("think step by step") works because training data contains vast amounts of problem-solving text that follows this pattern. The instruction activates a reasoning mode latent in the model's weights rather than teaching new behaviour.
- Few-shot CoT (providing example reasoning chains) is generally more reliable than zero-shot CoT for domain-specific tasks. The examples constrain both the reasoning style and the length of the chain, preventing overly verbose or superficial reasoning.
- Self-consistency is a CoT enhancement: sample the same CoT prompt multiple times at temperature > 0, then take a majority vote over final answers. This is especially effective for math and coding tasks where the final answer space is discrete.

**Common Misconceptions:**
- CoT only helps for math problems — CoT improves performance across reasoning, code debugging, logical deduction, and multi-step analysis tasks.
- CoT-produced reasoning chains are always trustworthy — the scratchpad can contain plausible-sounding but incorrect reasoning that leads to a correct answer by coincidence; verify outputs independently.

**Interview Answer Skeleton:**
- **What it is:** A prompting technique that instructs the model to generate explicit intermediate reasoning steps before producing a final answer, leveraging the autoregressive property that each generated step conditions subsequent generation.
- **Why it matters / trade-offs:** CoT improves accuracy on complex reasoning tasks and makes model logic auditable. The cost is increased output token count (and therefore API cost). For production systems, weigh accuracy gains against cost and latency increases.
- **Example or context:** For a multi-step data transformation task, a direct-answer prompt might silently skip a type-conversion step and produce a wrong result. A CoT prompt forces the model to articulate each transformation explicitly, making the type mismatch visible and correctable — either by the model itself or by a human reviewing the output.

**Free Resources:**
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Covers chain-of-thought elicitation, zero-shot CoT, and production usage patterns
- [Papers With Code](https://paperswithcode.com) — Tracks CoT research including self-consistency, tree-of-thought, and reasoning chain verification methods

---

## XML Structuring

**Status:** ⬜ Not Started

**Definition:** Using XML-style tags (e.g., `<context>`, `<instructions>`, `<examples>`) to clearly delimit sections of a prompt helps models like Claude distinguish between different types of content — instructions vs data vs examples — reducing confusion and improving instruction following.

**Key Mental Model:** XML tags are dividers in a binder — they tell the reader "this section is instructions, this section is data, this section is examples." Without dividers, the reader has to guess where one thing ends and another begins.

**How It Works:**
- XML tags function as explicit boundary markers in the token stream. The model's attention mechanism can learn from pre-training that content between `<instructions>` and `</instructions>` should be treated differently from content between `<user_data>` and `</user_data>`. This is especially important when user data might contain instructions that could be confused with the actual prompt directives.
- Without structural delimiters in complex prompts, the model must infer content boundaries from context clues like paragraph spacing or natural language transitions. XML tags make these boundaries unambiguous, reducing the probability of the model applying the wrong processing to a given section.
- XML tags are also useful for parsing model outputs. If you ask the model to produce `<answer>` and `<reasoning>` sections, you can reliably extract each part with simple string parsing rather than relying on regex or semantic parsing of free-form text.
- Prompts with embedded user-supplied data (RAG context, documents, pasted emails) benefit particularly from XML wrapping. Without clear delimiters, user content can "bleed into" the instruction space — a form of prompt injection. Wrapping retrieved context in `<retrieved_documents>` tags signals to the model that this content is data to reason about, not instructions to follow.
- Claude's training specifically optimises for XML-tagged prompt structure, but the pattern is broadly effective across GPT-4, Gemini, and other frontier models because they have all seen extensive XML in training data.

**Common Misconceptions:**
- XML tags are only for Claude/Anthropic models — XML structure helps most modern models distinguish content types; it is particularly effective with Claude but broadly applicable.
- Using XML makes prompts more complex than needed — for complex prompts with multiple components, XML structure reduces ambiguity and consistently improves reliability.

**Interview Answer Skeleton:**
- **What it is:** Using XML-style tags to structure different logical sections of a prompt (instructions, data, examples, output format), creating unambiguous boundaries that the model can reliably parse.
- **Why it matters / trade-offs:** XML structuring reduces instruction-following errors in complex prompts and provides a natural mitigation against prompt injection when wrapping user-supplied content. The trade-off is slightly increased token count from the tags themselves.
- **Example or context:** A RAG prompt with a long document should wrap it as `<retrieved_context>...</retrieved_context>` and the user's question as `<user_question>...</user_question>`. This prevents the model from treating document content as instructions and makes the structure of the prompt explicit and auditable.

**Free Resources:**
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Anthropic's documentation on XML tagging conventions and prompt structure
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Shows XML structuring patterns in production prompt templates

---

## Prompt Caching

**Status:** ⬜ Not Started

**Definition:** Prompt caching allows you to cache the KV (key-value) activations of a static portion of a prompt — such as a long system prompt or large document — so that repeated API calls reuse the cached computation. This reduces latency and cost dramatically for high-repetition patterns.

**Key Mental Model:** Prompt caching is like pre-cooking a sauce base — the expensive part is done once, and each dish just adds the finishing touches without redoing the base from scratch.

**How It Works:**
- When the API processes a prompt, it computes key-value (KV) attention vectors for every token. Prompt caching saves these KV vectors for the designated static prefix to disk or fast memory on the provider's infrastructure. On the next call with the same prefix, the model skips the prefill computation for those tokens and loads the cached vectors directly.
- The cache is keyed on exact token sequence match up to the breakpoint. Any change to the static prefix (even one token) invalidates the cache for everything after that change. This means the cacheable portion must be truly static across calls — a system prompt plus a fixed document, not a dynamic field.
- Anthropic's prompt caching requires adding `"cache_control": {"type": "ephemeral"}` markers to the prompt structure. Cache TTL is 5 minutes; if a cached prefix is not reused within 5 minutes, it expires and must be recomputed on the next call. High-frequency applications naturally stay warm.
- Cost economics: cached token reads are billed at ~10% of the standard input token price. Writing to the cache costs ~125% of the standard price for that call. The break-even point is typically the second call — any application making the same large-prefix call more than twice per cache window benefits.
- Structuring prompts for maximum cache effectiveness means placing all static content (system prompt, reference documents, few-shot examples) before any dynamic content (user query, session-specific data). The cache breakpoint should be placed at the boundary between static and dynamic content.

**Common Misconceptions:**
- Prompt caching is only for identical prompts — you can cache static prefixes (system prompt, shared context) and vary only the dynamic suffix (user query) across calls.
- Caching is automatic and free — prompt caching must be explicitly enabled via API parameters; cached tokens are billed at a lower rate but not at zero cost.

**Interview Answer Skeleton:**
- **What it is:** An API feature that persists the computed KV attention vectors for a static prompt prefix, allowing subsequent calls to skip prefill computation for those tokens and load the cached state instead.
- **Why it matters / trade-offs:** For applications with large system prompts or shared reference documents, prompt caching reduces per-call cost by 80–90% and cuts time-to-first-token latency by similar margins. The trade-off is the need to structure prompts with strict static/dynamic separation and manage cache TTL expiry.
- **Example or context:** A legal document analysis application sends a 50K-token contract as context with every user query. Without caching, each query pays full prefill cost. With caching, the contract is written to cache once and the 50K tokens cost 10% on every subsequent call — a 10x cost reduction for the dominant cost component.

**Free Resources:**
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Covers prompt caching implementation, breakpoint placement, and cost calculation
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Practical examples of prompt caching in document analysis and multi-turn conversation applications

---

## Structured Outputs

**Status:** ⬜ Not Started

**Definition:** Structured outputs constrain the model to produce responses in a specific format — JSON, XML, or a defined schema — rather than free-form text. This is achieved through JSON mode, tool calling with schemas, or explicit format instructions, making downstream parsing reliable and avoiding regex hacks.

**Key Mental Model:** Structured outputs are a standardised form rather than a blank page — the model fills in defined fields, making the output predictable and machine-readable.

**How It Works:**
- Tool calling (function calling) is the most reliable mechanism for structured outputs. You define a JSON Schema for the expected output; the model generates a structured argument object that conforms to the schema rather than free-form text. The API guarantees the output is valid against the schema before returning it.
- JSON mode (available in OpenAI and Anthropic APIs) constrains the model to produce syntactically valid JSON using constrained decoding — at each token step, only tokens that could appear in valid JSON given the current generation state are allowed. This prevents malformed JSON without requiring a schema.
- Pydantic models in Python can be converted to JSON Schema and passed directly to the API's tool-calling interface. Libraries like Instructor wrap this pattern, letting you define a Pydantic class and receive a validated Python object back from the API call — the parsing and validation are handled transparently.
- Schema design affects output quality. Overly complex nested schemas with many optional fields cause models to omit or hallucinate values. Flat schemas with clear field descriptions and examples in the schema's `description` properties produce more reliable outputs.
- For streaming applications, structured output is harder to work with — you cannot parse a partial JSON object. Strategies include streaming the raw text and parsing when the stream completes, or designing schemas where each field is self-contained and can be extracted incrementally using a streaming JSON parser.

**Common Misconceptions:**
- Instructing the model to produce JSON is enough to guarantee valid JSON — models can produce malformed JSON, especially for complex nested structures; use JSON mode or tool-calling for guaranteed validity.
- Structured output reduces model quality — for well-designed schemas, structured output maintains quality and makes the output significantly more useful for downstream processing.

**Interview Answer Skeleton:**
- **What it is:** Mechanisms (tool calling with JSON Schema, JSON mode, constrained decoding) that constrain model output to a machine-parseable format, eliminating brittle text parsing and enabling reliable downstream processing.
- **Why it matters / trade-offs:** Structured outputs are essential for integrating LLM responses into production data pipelines. Tool calling with schemas gives hard guarantees; JSON mode gives syntactic validity but not semantic schema conformance. Well-designed schemas also improve output quality by reducing the model's generation freedom.
- **Example or context:** For extracting product attributes from review text, define a Pydantic model with fields for product name, sentiment, and a list of feature mentions. Use the Instructor library to pass this as a tool schema — you get back a validated Python object with zero parsing code and automatic retries on schema validation failures.

**Free Resources:**
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Covers tool use, structured output, and schema design for reliable LLM responses
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Shows structured output patterns including Pydantic integration and tool-calling examples

---

## Extended Thinking

**Status:** ⬜ Not Started

**Definition:** Extended thinking (Claude's implementation of reasoning/scratchpad models) allows the model to generate internal reasoning tokens that are not shown to the user before producing the final visible response. This is distinct from visible CoT — the thinking is internal and can be more exploratory and self-correcting.

**Key Mental Model:** Extended thinking is a chess player thinking through multiple moves silently before committing to one. The opponent sees only the move, not the consideration of every alternative that preceded it.

**How It Works:**
- Extended thinking is enabled by passing a `thinking` parameter in the API call with a `budget_tokens` value. This allocates a token budget for the internal reasoning phase. The model generates a `thinking` block (visible in the API response object but typically not shown to end users) followed by the standard `text` response block.
- The thinking tokens are generated autoregressively like any other tokens, but they exist in a separate block. The model can use this space to explore hypotheses, work through calculations, consider and reject alternatives, and self-correct before committing to a final response.
- The thinking block attends to the full input context (system prompt, conversation history, user message). The final response block then attends to both the input context and the completed thinking block. This two-pass structure is what makes the final answer more accurate — it conditions on the completed reasoning rather than generating reasoning and final answer simultaneously.
- Budget token sizing matters: too small a budget truncates reasoning mid-thought and can actually degrade performance below non-thinking baselines. For most hard reasoning tasks, a budget of 5K–16K thinking tokens is appropriate; very complex tasks (long proofs, multi-file code generation) may warrant 32K+.
- Extended thinking interacts with streaming — the thinking block arrives first as a stream of tokens, then the final text block starts. For UX, you can show a "thinking..." indicator while the thinking block streams, then display the text response. See [[AI-Engineer/08-Production-AI-Engineering]] for streaming implementation patterns.

**Common Misconceptions:**
- Extended thinking is the same as prompting for chain-of-thought — extended thinking uses a fundamentally different token budget and may not be visible in the response; CoT prompting generates visible reasoning in the output.
- Extended thinking always improves every task — for simple tasks, extended thinking adds latency and cost without benefit; it is most valuable for genuinely complex, multi-step reasoning.

**Interview Answer Skeleton:**
- **What it is:** An API capability that allocates a dedicated token budget for internal reasoning generation before the user-visible response, enabling the model to explore, self-correct, and reason more deeply without exposing the scratchpad to end users.
- **Why it matters / trade-offs:** Extended thinking produces measurably better results on complex reasoning tasks — math, multi-step planning, code architecture. The trade-offs are higher latency (thinking tokens stream first), higher cost (full token rate for thinking tokens), and the need to gate its use to queries that genuinely require it.
- **Example or context:** A complex SQL debugging query with nested CTEs and multiple joins benefits from extended thinking — the model can trace data types and cardinality through each step before suggesting a fix. Gate extended thinking behind a complexity classifier: route simple queries to a standard model call, route queries the classifier scores as complex to the extended thinking endpoint.

**Free Resources:**
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering) — Official documentation on extended thinking API parameters, budget sizing, and response structure
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Working code examples for extended thinking with streaming, budget management, and complexity-based routing
