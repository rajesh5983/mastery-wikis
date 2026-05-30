# Module 11 — Multimodal Integration

---

## Vision Models

**Status:** ⬜ Not Started

**Definition:** Vision models (Claude, GPT-4o, Gemini, LLaVA) can process images alongside text, enabling applications like document understanding, chart analysis, screenshot-to-code, visual Q&A, and image-based data extraction. Images are typically passed as base64-encoded data or URLs in the API request.

**Key Mental Model:** Vision models give the LLM eyes — it can now look at a screenshot, diagram, or photo and reason about it in the same call as a text prompt, without requiring a separate computer vision pipeline.

**How It Works:**
- Images are preprocessed by a vision encoder (typically a ViT — Vision Transformer) before entering the language model. The vision encoder divides the image into a grid of fixed-size patches (e.g., 16×16 or 32×32 pixels), flattens each patch, and projects it into an embedding vector of the same dimension as the language model's token embeddings.
- The sequence of image patch embeddings is prepended to (or interleaved with) the text token embeddings. The combined sequence is then processed by the language model's attention layers — each text token can attend to image patch embeddings and vice versa, enabling cross-modal reasoning in the standard transformer attention mechanism.
- In the API, images are passed either as base64-encoded bytes (the full image data encoded as a string) or as a URL that the provider fetches. Most providers impose image size limits (e.g., max 5MB per image, max 5 images per request) and recommended resolutions — images outside the optimal range are resized internally, which can degrade quality for fine-detail tasks.
- Image token counts are significant. A 1024×1024 image may be encoded as 1,000–1,600 image tokens depending on the model's patch size and encoding strategy. This directly affects API cost and context window consumption. Large images in multi-image requests can quickly consume the context budget.
- Vision model accuracy is domain-dependent. OCR of printed text, diagram interpretation, and screenshot analysis perform well. Medical imaging (X-rays, pathology slides), highly technical engineering drawings, and low-resolution or blurry images perform substantially worse. Always run domain-specific evals before assuming a vision model can handle your image types.

**Common Misconceptions:**
- Vision models are primarily for consumer applications — enterprise use cases (invoice processing, document digitisation, visual data extraction, engineering diagram analysis) are some of the highest-value vision applications.
- Vision models understand all image types equally well — vision models perform better on document/screenshot content than on medical imaging, highly technical diagrams, or low-resolution photographs.

**Interview Answer Skeleton:**
- **What it is:** LLMs augmented with a vision encoder that projects image patches into the language model's embedding space — enabling joint text-image reasoning in a single forward pass using the standard attention mechanism.
- **Why it matters / trade-offs:** Vision models eliminate separate CV pipeline stages (OCR, layout parsing, object detection) for many document and image understanding tasks. The trade-offs are high image token costs (affects context budget and pricing), domain-specific accuracy gaps for specialised image types, and the need to validate performance on your specific image distribution.
- **Example or context:** Invoice data extraction: pass the invoice as a base64-encoded image with a structured output schema (vendor name, invoice date, line items, total). The model reads the invoice layout including tables and extracts fields directly. Compare against AWS Textract for your specific invoice formats — specialised document AI tools may outperform general vision LLMs on structured forms at scale.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Covers multimodal models, vision-language architectures, and image tokenisation mechanics
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Vision API usage examples including document extraction, chart analysis, and screenshot-to-code patterns

---

## Image Generation

**Status:** ⬜ Not Started

**Definition:** Image generation models (DALL-E 3, Stable Diffusion, Flux, Midjourney) produce images from text prompts. In AI engineering, they are integrated via API for product visualisation, content generation, marketing assets, and UI mockup generation. Prompt engineering for image models requires different patterns from text prompting.

**Key Mental Model:** Image generation is the reverse of vision models — instead of reading images, it writes them. The model translates a text description into pixels by learning the statistical relationship between captions and images.

**How It Works:**
- Diffusion models (Stable Diffusion, Flux, DALL-E 3) work by learning to reverse a noise process. During training, the model learns to predict the noise added to an image at each step of a fixed forward noising process. At inference, the model starts from pure random noise and iteratively removes noise (reverse diffusion) over N steps (typically 20–50), guided by the text prompt's conditioning, until a coherent image emerges.
- Text conditioning is achieved via cross-attention. The text prompt is encoded by a text encoder (CLIP or T5) into a sequence of embeddings. At each denoising step, the diffusion model's attention layers cross-attend to these text embeddings, guiding the denoising direction toward the described content.
- DALL-E 3 (OpenAI's API-accessible model) uses ChatGPT to first expand short user prompts into detailed captions, then generates images from the expanded caption. This automatic prompt expansion is why DALL-E 3 responds well to simple prompts — but it also means the model sometimes adds elements the user did not specify.
- Image generation APIs typically accept: the text prompt, negative prompts (what to avoid), resolution (512×512, 1024×1024), number of inference steps (more steps = higher quality but slower), guidance scale (how closely to follow the prompt vs creative freedom), and a random seed (for reproducibility). Higher guidance scale produces more literal prompt adherence; lower guidance scale allows more creative interpretation.
- For consistent product/brand imagery at scale, fine-tuning or LoRA (Low-Rank Adaptation) adapters train the model on examples of a specific subject, style, or brand asset. A product photography pipeline might fine-tune on 20 images of a product to enable generating that product in arbitrary settings with consistent appearance.

**Common Misconceptions:**
- Image generation models can produce any image from any prompt — safety filters, model biases, and training data gaps mean all current models have significant blind spots and refusals.
- All image generation APIs produce consistent quality — quality, style, consistency, and prompt adherence vary dramatically between models and even between API calls with the same prompt.

**Interview Answer Skeleton:**
- **What it is:** Diffusion-based models that iteratively reverse a noise process guided by CLIP/T5 text encodings to generate images — accessible via API with parameters for resolution, steps, guidance scale, and seed.
- **Why it matters / trade-offs:** Image generation APIs enable scalable visual content creation without creative staff for templated use cases. The trade-offs are inconsistency across generations (same prompt produces different results), safety filter limitations, and the significant engineering needed for brand-consistent output (fine-tuning or LoRA).
- **Example or context:** An e-commerce product photography pipeline: use a fine-tuned Stable Diffusion model (fine-tuned on 20 product photos) to generate the product in various lifestyle settings — "on a wooden table with coffee mug", "in a minimalist white studio." Fine-tuning cost: a few hours of GPU compute. Value: eliminates a photoshoot for each setting. Quality gate: human review before each image goes live.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Covers diffusion model architectures, text conditioning, and image generation pipelines
- [Papers With Code](https://paperswithcode.com) — Tracks image generation benchmarks (FID, CLIP score), model comparisons, and open implementation releases

---

## Voice Agents

**Status:** ⬜ Not Started

**Definition:** Voice agents combine speech-to-text (Whisper, Deepgram), LLM reasoning, and text-to-speech (ElevenLabs, Azure TTS) into a real-time conversational loop. Latency is critical — users perceive pauses above 500ms as unnatural. The pipeline must stream both ASR output and TTS input for responsive conversation.

**Key Mental Model:** A voice agent is a telephone with an LLM inside — sound goes in, text comes out of ASR, the LLM thinks, text goes into TTS, sound comes out — all fast enough to feel like a natural conversation.

**How It Works:**
- The pipeline has three sequential stages, each with its own latency budget: ASR (speech-to-text), LLM generation, and TTS (text-to-speech). End-of-utterance detection (VAD — Voice Activity Detection) determines when the user has finished speaking and triggers the ASR transcription. VAD latency (how quickly the system detects silence-after-speech) adds to the perceived response lag.
- Streaming ASR returns partial transcripts as the user speaks rather than waiting for them to finish. These partial transcripts are speculatively fed to the LLM as the user is still talking — when the final transcript is confirmed, the LLM generation is either confirmed or restarted. This speculative execution reduces the gap between user finishing speaking and LLM response beginning.
- LLM responses are streamed character by character to the TTS engine. The TTS engine doesn't wait for the full response before starting speech synthesis — it starts generating audio from the first sentence as the LLM streams the second. This pipeline overlap is the primary technique for achieving sub-500ms first-audio latency.
- Interruption handling is essential for natural conversation. When the user starts speaking while the agent is still talking (barge-in), the pipeline must: (1) immediately stop TTS audio playback, (2) run VAD on the new audio, (3) wait for the interruption to complete and transcribe it, (4) provide the LLM the context of what it had said and what the user interrupted with, and (5) restart generation from the interruption point.
- Provider selection for each stage is optimised separately: Deepgram Nova-2 or Whisper Large-v3 for ASR (low WER, streaming capable), a fast model on Groq or a frontier model via streaming API for LLM generation, ElevenLabs or Cartesia for TTS (low latency, natural prosody, voice cloning). Total latency target: ASR final (200ms) + LLM TTFT (150ms) + TTS first-audio (150ms) = 500ms.

**Common Misconceptions:**
- Voice agents are just text chatbots with audio wrappers — voice has fundamentally different UX requirements: interruption handling, turn-taking detection, filler responses for perceived thinking time, and prosody matching.
- High-quality STT is all you need — recognition accuracy is only one component; end-of-speech detection, latency, and handling of accents and background noise are equally important in production.

**Interview Answer Skeleton:**
- **What it is:** A real-time pipeline combining streaming ASR (speech-to-text), LLM generation with streaming output, and streaming TTS (text-to-speech) with VAD-based turn detection and interruption handling — optimised for sub-500ms first-audio latency.
- **Why it matters / trade-offs:** Voice agents enable phone-based customer service, accessibility tools, and hands-free interfaces where text input is impractical. The engineering challenge is the strict latency budget — every component must be streaming-capable, and pipeline overlap is required to hit perceptual responsiveness thresholds.
- **Example or context:** A customer service voice agent targeting sub-500ms first-response: Deepgram streaming ASR → speculative LLM start on partial transcript → ElevenLabs streaming TTS starts on first LLM sentence. VAD detects barge-in → stop playback → transcribe interruption → inject interruption context into LLM. LLM system prompt includes: "If interrupted, acknowledge the interruption and address what the user said."

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Covers ASR models, speech processing, and audio-to-text architectures
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Voice agent architecture examples with streaming LLM integration and latency optimisation patterns

---

## Video Generation

**Status:** ⬜ Not Started

**Definition:** Video generation models (Sora, Runway Gen-3, Kling) produce short video clips from text prompts or image sequences. AI engineering applications include product demo generation, training data synthesis, and automated content creation. The field is rapidly evolving and most production use cases involve API-based integration.

**Key Mental Model:** Video generation extends image generation into time — instead of one frame, the model generates a coherent sequence of frames where objects, lighting, and motion are physically consistent across time.

**How It Works:**
- Video diffusion models extend 2D image diffusion to 3D (spatial + temporal) by adding a temporal attention dimension. Instead of denoising a single image, the model denoises a sequence of frames simultaneously, attending across both spatial positions and time steps. This is what enables temporal coherence — objects maintain consistent appearance and motion trajectories across frames.
- Video generation inference is computationally intensive: generating a 5-second clip at 24fps produces 120 frames, each requiring the full diffusion denoising pass. Generation times of 1–10 minutes per clip are common even on high-end hardware. This makes real-time video generation impractical with current architectures — video generation is an asynchronous, batch workflow.
- Image-to-video models (Runway Gen-3, Stable Video Diffusion) take a still image as the starting frame and animate it. The motion is conditioned by the text prompt describing the desired movement. This is often more controllable than text-to-video because the initial appearance is precisely defined by the input image.
- API integration for video generation is async: you POST a generation request and receive a job ID, poll the API (or receive a webhook) until generation completes, then download the video file from a temporary URL. Generation can take minutes — design the UX around an asynchronous workflow with a progress indicator rather than a synchronous wait.
- Quality limitations dominate current production use cases. Current models excel at: short clips (< 10 seconds), simple motion (camera pans, object rotation, atmospheric effects), and abstract or artistic styles. They struggle with: multi-character scenes, fine-grained motion control, long temporal coherence, and text legibility in video.

**Common Misconceptions:**
- Video generation quality has reached broadcast-production level — current models produce 5–10 second clips with significant artefacts; they are best suited for concept visualisation and rough prototypes, not finished content.
- Video generation APIs are stable and production-ready — most video generation providers are in early access or beta; plan for reliability limitations and API changes.

**Interview Answer Skeleton:**
- **What it is:** Video diffusion models that extend image generation to temporal sequences, generating spatially and temporally coherent short clips from text or image prompts — integrated via async APIs due to multi-minute generation times.
- **Why it matters / trade-offs:** Video generation is a rapidly emerging capability with clear enterprise applications (product demos, training data synthesis, content automation) but current quality and reliability limitations restrict production use to controlled scenarios. Treat it as a complement to human production, not a replacement.
- **Example or context:** A current genuine use case: generating concept visualisation clips for early-stage product pitches, where 5-second clips showing a product interaction are useful for stakeholder communication and the "rough prototype" quality is acceptable. Inappropriate use case: replacing professional video production for customer-facing marketing — quality, consistency, and reliability are not yet there.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Video generation model architectures and temporal diffusion mechanics
- [Papers With Code](https://paperswithcode.com) — Video generation benchmarks, model comparisons, and open implementation releases for the latest video models

---

## Document AI

**Status:** ⬜ Not Started

**Definition:** Document AI combines OCR, layout analysis, and LLM understanding to extract structured data from complex documents — invoices, contracts, forms, tables, and PDFs. Modern vision-capable LLMs have significantly simplified this pipeline by handling layout and content extraction in a single call.

**Key Mental Model:** Document AI is a reading comprehension system for business documents — it not only reads the text but understands the document's structure (tables, fields, signatures) and extracts the specific information you need.

**How It Works:**
- Traditional Document AI uses a two-stage pipeline: a specialised OCR/layout model (AWS Textract, Azure Document Intelligence, Google Document AI) first detects text regions and their bounding boxes, identifies document structural elements (tables, key-value pairs, checkboxes), and returns structured output including spatial coordinates. The LLM then receives this structured output (rather than an image) and extracts the required fields.
- Vision LLM direct extraction bypasses the OCR layer: pass the document page as an image, describe the fields to extract in the prompt, and return a structured JSON output. This is simpler and often accurate enough for standard document types. The trade-off is that it lacks the precise spatial coordinates, confidence scores, and multi-page handling that dedicated Document AI tools provide.
- PDF handling requires document-specific preprocessing. A text-layer PDF (not scanned) can have text extracted directly without OCR — much faster and more accurate than running vision on rendered pages. Scanned PDFs require either vision processing or a separate OCR step first. Detecting which type of PDF you have (checking for text layer presence) allows routing to the appropriate pipeline.
- Table extraction is a challenge for both approaches. A table rendered in a PDF may span multiple pages, have merged cells, or have complex header hierarchies. Vision models sometimes misalign rows and columns in complex tables. Dedicated tools (Textract's table analysis, Camelot Python library for PDF table extraction) handle edge cases that general vision models miss.
- Confidence scoring and human-in-the-loop review are standard in production Document AI pipelines. Low-confidence extractions (flagged by the model's own uncertainty or by comparing multiple extraction methods) are routed to a human review queue rather than inserted into the database automatically. This is especially important for financial and legal documents where errors have significant consequences.

**Common Misconceptions:**
- PDF text extraction is equivalent to Document AI — extracting raw text from PDFs loses layout information crucial for tables, forms, and multi-column documents; Document AI preserves spatial relationships.
- Vision models eliminate the need for dedicated Document AI tools — specialised document models (AWS Textract, Azure Document Intelligence) still outperform general vision LLMs on structured form extraction at scale.

**Interview Answer Skeleton:**
- **What it is:** Pipelines combining OCR/layout detection (or vision LLMs directly) with structured extraction prompts to pull defined fields from complex business documents — invoices, contracts, forms — into structured database records.
- **Why it matters / trade-offs:** Document-heavy processes (procurement, insurance, legal, finance) are major automation opportunities. The trade-off between specialised Document AI tools (higher accuracy on structured forms, spatial coordinates, higher cost) and vision LLMs (simpler pipeline, adequate accuracy for standard docs, lower cost) depends on document complexity and accuracy requirements.
- **Example or context:** Insurance claim form processing pipeline: detect PDF type (text vs scanned). For scanned: call AWS Textract for OCR and table extraction, pass structured output to Claude for semantic field normalisation. For text PDFs: extract text layer, pass to Claude directly with a schema prompt. Route extractions with confidence < 0.9 to a human review queue in the case management system. Track extraction accuracy per document type to identify which categories need human review rates improved.

**Free Resources:**
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Document understanding models, layout-aware transformers (LayoutLM), and document processing pipelines
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Document AI examples including PDF processing, structured extraction with vision, and multi-page document handling
