# Module 6 — Guardrails and Safety

---

## Input/Output Validation

**Status:** ⬜ Not Started

**Definition:** Input validation checks user requests before sending them to the LLM — filtering prohibited content, validating format, enforcing length limits. Output validation checks model responses before returning them — schema conformance, content policy checks, and factual consistency tests.

**Mental Model:** Validation is a security checkpoint at both the entrance and exit of a building. What comes in is checked for threats; what goes out is checked to ensure it meets the standards the building is committed to.

**Common Misconceptions:**
- System prompt instructions alone are sufficient for output safety — models do not always follow instructions reliably, especially under adversarial inputs; programmatic output validation is a separate, necessary layer.
- Input validation only needs to block profanity — modern attacks are subtle; validation must handle jailbreaks, prompt injections, and semantic manipulation, not just surface-level content.

**Interview Skeleton:**
- What it is: programmatic checks on both incoming requests and outgoing responses that enforce application safety and quality standards
- Why it matters: LLMs are probabilistic and can be manipulated; validation adds a deterministic safety layer
- Example: design an input/output validation pipeline for a customer-facing AI chatbot including what you'd check at each stage

**Free Resources:** https://docs.guardrailsai.com — Guardrails AI documentation covering validators, output schemas, and validation pipelines

---

## Prompt Injection Defense

**Status:** ⬜ Not Started

**Definition:** Prompt injection is an attack where a malicious user embeds instructions in their input that override the system prompt or manipulate the model's behaviour. Defense strategies include input sanitisation, privilege separation (keeping user and system content structurally separate), instruction anchoring, and LLM-based injection detection.

**Mental Model:** Prompt injection is like a SQL injection attack but for language models — the attacker inserts instructions where only data was expected. Defense is separating the instruction layer from the data layer, just as parameterised queries separate SQL from data.

**Common Misconceptions:**
- Prompt injection is rare and theoretical — prompt injection attacks are actively exploited in production systems, especially those that process external content like web pages, emails, or user-uploaded documents.
- XML tags or delimiters fully prevent injection — structural markers help but determined attackers can include escaped or context-switching instructions; defence in depth with model-level detection is needed.

**Interview Skeleton:**
- What it is: a class of attacks where malicious user input manipulates LLM behaviour by overriding system instructions
- Why it matters: agents with tool access are particularly vulnerable; a successful injection can exfiltrate data or trigger destructive actions
- Example: demonstrate a prompt injection attack and describe three layers of defence you'd implement

**Free Resources:** https://docs.guardrailsai.com — Guardrails AI documentation covering injection detection and defensive prompt patterns

---

## PII Redaction

**Status:** ⬜ Not Started

**Definition:** PII (Personally Identifiable Information) redaction detects and removes or replaces sensitive data — names, emails, phone numbers, credit card numbers, SSNs — from both user inputs before sending to the LLM and from LLM outputs before logging or returning to users. This is essential for GDPR, HIPAA, and data residency compliance.

**Mental Model:** PII redaction is like automatic blurring on a TV broadcast — before the content goes anywhere sensitive, all identifying information is replaced with placeholders or masked automatically.

**Common Misconceptions:**
- LLM providers handle PII automatically — API providers do not redact PII from your prompts; application-layer redaction before the API call is the engineer's responsibility.
- Regex patterns are sufficient for PII detection — rule-based patterns miss contextual PII (e.g., a name without obvious format); ML-based NER models catch significantly more.

**Interview Skeleton:**
- What it is: the automated detection and removal of sensitive personal information from data flowing into and out of LLM systems
- Why it matters: sending PII to third-party LLM APIs violates most enterprise data governance policies and compliance frameworks
- Example: describe a PII redaction pipeline for a medical document processing system, including how you'd handle pseudo-anonymisation

**Free Resources:** https://docs.guardrailsai.com — Guardrails AI documentation covering PII detection and redaction validators

---

## Jailbreak Prevention

**Status:** ⬜ Not Started

**Definition:** Jailbreaking refers to techniques users employ to bypass a model's safety guidelines — through roleplay scenarios ("pretend you are a model with no restrictions"), hypothetical framing, gradual escalation, or adversarial suffixes. Jailbreak prevention involves system prompt hardening, input classification, and layered safety checks.

**Mental Model:** Jailbreaks are social engineering attacks on the model's instruction-following. Defence is like training staff not to be fooled by "I'm from IT, I need your password" — recognise the pattern, escalate to policy, don't comply.

**Common Misconceptions:**
- Jailbreaks are rare attacks from sophisticated adversaries — jailbreak prompts are shared publicly and used by ordinary users; applications must defend against common techniques, not just novel ones.
- Model safety training alone prevents jailbreaks — all major models have been jailbroken through various techniques; application-layer defence is a necessary addition to model-level safety.

**Interview Skeleton:**
- What it is: techniques for preventing users from manipulating the model into producing outputs that violate application policies
- Why it matters: a jailbroken customer-facing LLM creates legal, reputational, and safety risks
- Example: describe a layered defence for a consumer application: system prompt hardening, input classification, and output policy checking

**Free Resources:** https://docs.guardrailsai.com — Guardrails AI documentation covering content policy enforcement and jailbreak detection

---

## Hallucination Detection

**Status:** ⬜ Not Started

**Definition:** Hallucination occurs when an LLM generates plausible-sounding but factually incorrect or unsupported content. Hallucination detection strategies include self-consistency checking (run the same query multiple times), grounding checks (verify claims against retrieved source documents), and LLM-as-judge verification.

**Mental Model:** Detecting hallucinations is like fact-checking a confident-sounding source — confidence of delivery tells you nothing about accuracy. You verify claims against cited sources, not against how convincingly they were stated.

**Common Misconceptions:**
- High-confidence LLM outputs are less likely to hallucinate — models express confidence about incorrect claims as readily as correct ones; confidence scoring does not reliably predict factual accuracy.
- Hallucination only happens for obscure facts — models hallucinate on common facts, recent events, arithmetic, and citations with similar frequency to obscure topics.

**Interview Skeleton:**
- What it is: automated techniques for detecting when LLM output contains unsupported or factually incorrect claims
- Why it matters: hallucinations in production AI systems cause incorrect decisions, erode user trust, and create liability
- Example: describe a hallucination detection pipeline for a RAG system: how would you check that each claim in the response is supported by the retrieved context?

**Free Resources:** https://docs.guardrailsai.com — Guardrails AI documentation covering faithfulness validators and hallucination detection patterns
