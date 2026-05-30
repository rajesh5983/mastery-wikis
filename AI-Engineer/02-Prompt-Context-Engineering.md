# Module 2 — Prompt and Context Engineering

---

## System Prompts

**Status:** ⬜ Not Started

**Definition:** The system prompt is a privileged instruction block sent before any user content that sets the model's role, behaviour, constraints, and output format for the entire conversation. It is the primary lever for customising a frontier model for a specific application without fine-tuning.

**Mental Model:** The system prompt is the employee handbook — it defines the role, the rules, the tone, and what is out of scope. Every response flows from those standing instructions, regardless of what the user asks.

**Common Misconceptions:**
- System prompts are always fully followed — models can drift from system prompt instructions, especially in long conversations; important constraints should be reinforced or tested regularly.
- System prompts are secret and protected — in most implementations, a determined user can elicit the system prompt contents through clever prompting; treat them as operational guidance, not security boundaries.

**Interview Skeleton:**
- What it is: privileged instructions that configure model behaviour, persona, and constraints at the conversation level
- Why it matters: the system prompt is the cheapest and most effective way to specialise a general model for a specific application
- Example: write a system prompt for a customer support bot that is helpful but will not discuss competitors or make pricing commitments

**Free Resources:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering — Anthropic's prompt engineering guide covering system prompts, roles, and best practices

---

## Few-Shot Prompting

**Status:** ⬜ Not Started

**Definition:** Few-shot prompting provides the model with examples of the desired input-output pattern directly in the prompt, before presenting the actual task. This leverages the model's in-context learning ability — it infers the pattern from examples without any weight updates.

**Mental Model:** Few-shot prompting is like showing a new employee three completed expense reports before asking them to fill one out — they infer the format and standard from the examples rather than reading a lengthy manual.

**Common Misconceptions:**
- More examples are always better — examples consume context window tokens; 2–5 well-chosen, diverse examples typically outperform 20 redundant ones.
- Examples must be real data — for many tasks, synthetic examples that clearly illustrate the pattern work as well or better than real data.

**Interview Skeleton:**
- What it is: providing the model with labelled examples in the prompt to guide output format and reasoning pattern
- Why it matters: dramatically improves consistency and quality for tasks with a specific output schema without requiring fine-tuning
- Example: demonstrate few-shot prompting for extracting structured fields from unstructured customer emails

**Free Resources:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering — Anthropic's documentation on few-shot examples and in-context learning

---

## Chain of Thought

**Status:** ⬜ Not Started

**Definition:** Chain of Thought (CoT) prompting instructs the model to produce step-by-step reasoning before arriving at a final answer. This can be elicited with instructions like "think step by step" or by providing CoT examples. It significantly improves performance on reasoning, math, and multi-step tasks.

**Mental Model:** CoT is asking someone to show their work, not just give an answer. The act of writing out each step forces sequential reasoning and reduces the chance of skipping to a wrong conclusion.

**Common Misconceptions:**
- CoT only helps for math problems — CoT improves performance across reasoning, code debugging, logical deduction, and multi-step analysis tasks.
- CoT-produced reasoning chains are always trustworthy — the scratchpad can contain plausible-sounding but incorrect reasoning that leads to a correct answer by coincidence; verify outputs independently.

**Interview Skeleton:**
- What it is: prompting the model to reason step by step before producing a final answer
- Why it matters: improves accuracy on complex reasoning tasks; makes model reasoning transparent and debuggable
- Example: compare a direct-answer prompt vs a CoT prompt for a multi-step data transformation task and explain the difference in reliability

**Free Resources:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering — Anthropic prompt engineering guide including chain-of-thought techniques

---

## XML Structuring

**Status:** ⬜ Not Started

**Definition:** Using XML-style tags (e.g., `<context>`, `<instructions>`, `<examples>`) to clearly delimit sections of a prompt helps models like Claude distinguish between different types of content — instructions vs data vs examples — reducing confusion and improving instruction following.

**Mental Model:** XML tags are dividers in a binder — they tell the reader "this section is instructions, this section is data, this section is examples." Without dividers, the reader has to guess where one thing ends and another begins.

**Common Misconceptions:**
- XML tags are only for Claude/Anthropic models — XML structure helps most modern models distinguish content types; it is particularly effective with Claude but broadly applicable.
- Using XML makes prompts more complex than needed — for complex prompts with multiple components, XML structure reduces ambiguity and consistently improves reliability.

**Interview Skeleton:**
- What it is: using XML-style tags to structure different logical sections of a prompt for clear parsing by the model
- Why it matters: reduces ambiguity in complex prompts and improves instruction-following and content isolation
- Example: restructure a messy prompt with instructions, context, and examples into clearly tagged sections and explain the improvement

**Free Resources:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering — Anthropic's documentation on prompt structure and XML tagging

---

## Prompt Caching

**Status:** ⬜ Not Started

**Definition:** Prompt caching allows you to cache the KV (key-value) activations of a static portion of a prompt — such as a long system prompt or large document — so that repeated API calls reuse the cached computation. This reduces latency and cost dramatically for high-repetition patterns.

**Mental Model:** Prompt caching is like pre-cooking a sauce base — the expensive part is done once, and each dish just adds the finishing touches without redoing the base from scratch.

**Common Misconceptions:**
- Prompt caching is only for identical prompts — you can cache static prefixes (system prompt, shared context) and vary only the dynamic suffix (user query) across calls.
- Caching is automatic and free — prompt caching must be explicitly enabled via API parameters; cached tokens are billed at a lower rate but not at zero cost.

**Interview Skeleton:**
- What it is: reusing computed model activations for repeated static prompt prefixes to reduce cost and latency
- Why it matters: applications with large system prompts or shared context documents see 80–90% cost reduction for cache hits
- Example: describe how you'd structure an application prompt to maximise cache hit rate using prefix caching

**Free Resources:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering — Anthropic documentation covering prompt caching implementation and best practices

---

## Structured Outputs

**Status:** ⬜ Not Started

**Definition:** Structured outputs constrain the model to produce responses in a specific format — JSON, XML, or a defined schema — rather than free-form text. This is achieved through JSON mode, tool calling with schemas, or explicit format instructions, making downstream parsing reliable and avoiding regex hacks.

**Mental Model:** Structured outputs are a standardised form rather than a blank page — the model fills in defined fields, making the output predictable and machine-readable.

**Common Misconceptions:**
- Instructing the model to produce JSON is enough to guarantee valid JSON — models can produce malformed JSON, especially for complex nested structures; use JSON mode or tool-calling for guaranteed validity.
- Structured output reduces model quality — for well-designed schemas, structured output maintains quality and makes the output significantly more useful for downstream processing.

**Interview Skeleton:**
- What it is: constraining model output to a machine-parseable format using API-level mechanisms
- Why it matters: eliminates brittle parsing code and makes LLM output directly usable by downstream systems
- Example: design a schema for extracting product attributes from review text and implement it using tool calling

**Free Resources:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering — Anthropic documentation on structured outputs and tool use for format control

---

## Extended Thinking

**Status:** ⬜ Not Started

**Definition:** Extended thinking (Claude's implementation of reasoning/scratchpad models) allows the model to generate internal reasoning tokens that are not shown to the user before producing the final visible response. This is distinct from visible CoT — the thinking is internal and can be more exploratory and self-correcting.

**Mental Model:** Extended thinking is a chess player thinking through multiple moves silently before committing to one. The opponent sees only the move, not the consideration of every alternative that preceded it.

**Common Misconceptions:**
- Extended thinking is the same as prompting for chain-of-thought — extended thinking uses a fundamentally different token budget and may not be visible in the response; CoT prompting generates visible reasoning in the output.
- Extended thinking always improves every task — for simple tasks, extended thinking adds latency and cost without benefit; it is most valuable for genuinely complex, multi-step reasoning.

**Interview Skeleton:**
- What it is: an API feature that gives the model internal reasoning capacity before generating the user-visible response
- Why it matters: improves accuracy on hard reasoning tasks at the cost of higher latency and token usage
- Example: describe which types of tasks in a production AI system would benefit from extended thinking, and how you'd gate its use

**Free Resources:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering — Anthropic documentation on extended thinking capabilities and usage guidance
