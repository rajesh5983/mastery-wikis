# Module 3 — RAG Systems

---

## Embeddings

**Status:** ⬜ Not Started

**Definition:** Embeddings are dense vector representations of text (or images, code) where semantically similar content is positioned close together in high-dimensional space. Embedding models (e.g., text-embedding-3-small, Cohere Embed) convert text chunks into vectors that can be stored in a vector database and searched by cosine or dot-product similarity.

**Mental Model:** Embeddings are coordinates in a meaning map. "King" and "Queen" are nearby. "Paris" and "London" are nearby. A search query's coordinates find nearby document chunks — semantically similar, even if no exact keywords match.

**Common Misconceptions:**
- Semantic similarity always means topical relevance — embeddings capture general semantic similarity but can match on surface-level style or tone rather than topical overlap.
- Any embedding model works for any domain — embedding models trained on general web text may perform poorly on domain-specific content (legal, medical, code); domain-specific models matter.

**Interview Skeleton:**
- What it is: dense vector representations enabling semantic similarity search over large document corpora
- Why it matters: the quality of the embedding model directly determines retrieval recall in RAG systems
- Example: explain the difference between keyword search and semantic search, and when you'd combine both (hybrid search)

**Free Resources:** https://python.langchain.com/docs/tutorials/rag — LangChain RAG tutorial covering embeddings, vector stores, and retrieval chains

---

## Chunking Strategies

**Status:** ⬜ Not Started

**Definition:** Chunking splits source documents into pieces that fit within the context window of an embedding model and are semantically coherent enough to be useful when retrieved. Common strategies include fixed-size chunking, sentence-level chunking, recursive character splitting, and semantic chunking based on topic shifts.

**Mental Model:** Chunking is dividing a textbook into index cards for revision. Too large and each card is overwhelming and unfocused. Too small and each card lacks enough context to stand alone. The goal is the smallest self-contained unit of meaning.

**Common Misconceptions:**
- Smaller chunks are always better for retrieval precision — very small chunks lose context, causing the retrieved text to be ambiguous without surrounding sentences.
- One chunking strategy fits all document types — structured tables need different chunking than narrative prose, which needs different chunking than code; match the strategy to the content type.

**Interview Skeleton:**
- What it is: the strategy for dividing source documents into segments that are both embeddable and coherent when retrieved
- Why it matters: chunking is one of the highest-leverage variables in RAG system quality; wrong chunking degrades retrieval regardless of model quality
- Example: describe how you'd chunk a 100-page PDF technical report differently from a customer support knowledge base

**Free Resources:** https://python.langchain.com/docs/tutorials/rag — LangChain RAG tutorial covering chunking, splitters, and their trade-offs

---

## Reranking

**Status:** ⬜ Not Started

**Definition:** Reranking is a two-stage retrieval approach where an initial fast retrieval (embedding similarity or keyword search) returns a candidate set, and a more powerful cross-encoder model then re-scores each candidate against the query to produce a more accurate final ranking before sending to the LLM.

**Mental Model:** Initial retrieval is a quick shortlist — fast but imprecise. Reranking is the second interview round — slower, more thorough, only applied to the shortlisted candidates.

**Common Misconceptions:**
- Reranking is redundant if you have a good embedding model — embedding models use bi-encoders (fast, independent query and document encoding); rerankers use cross-encoders (slow, joint encoding) that capture richer query-document interaction.
- Reranking is only needed for large corpora — even small corpora benefit from reranking when top-k precision is critical to avoid sending irrelevant context to the LLM.

**Interview Skeleton:**
- What it is: a two-stage retrieval pipeline that improves final precision using a computationally expensive cross-encoder on a smaller candidate set
- Why it matters: significantly improves the quality of the context sent to the LLM, reducing hallucinations and irrelevant answers
- Example: design a retrieval pipeline with initial vector search returning top-50 candidates, then reranked to top-5 for the LLM

**Free Resources:** https://python.langchain.com/docs/tutorials/rag — LangChain RAG tutorial covering retrieval, reranking, and advanced patterns

---

## Classic, Graph, and Agentic RAG

**Status:** ⬜ Not Started

**Definition:** Classic RAG retrieves flat text chunks and injects them into the prompt. Graph RAG models documents as a knowledge graph, enabling retrieval of entities and their relationships. Agentic RAG uses an agent that can iteratively query, refine, and decide when enough context has been gathered, rather than doing a single fixed retrieval.

**Mental Model:** Classic RAG is grabbing the three most relevant pages from a filing cabinet. Graph RAG is following the threads of a corkboard with connected pins. Agentic RAG is a research assistant who keeps pulling more files until they've answered the question fully.

**Common Misconceptions:**
- Classic RAG is outdated and should always be replaced with agentic RAG — classic RAG is simpler, faster, and more predictable; agentic and graph RAG add complexity that is only justified for specific use cases.
- Graph RAG always outperforms flat RAG — graph RAG excels for relationship and entity queries; for straightforward text retrieval it adds overhead without improvement.

**Interview Skeleton:**
- What it is: a spectrum of RAG architectures with increasing sophistication, cost, and capability for complex retrieval tasks
- Why it matters: choosing the right RAG variant for the use case determines system quality, cost, and latency
- Example: when would you choose agentic RAG over classic RAG, and what guardrails would you add to control cost?

**Free Resources:** https://python.langchain.com/docs/tutorials/rag — LangChain RAG tutorials covering classic through advanced agentic patterns

---

## HyDE (Hypothetical Document Embeddings)

**Status:** ⬜ Not Started

**Definition:** HyDE is a retrieval technique where, instead of embedding the user query directly, the LLM first generates a hypothetical answer to the query, then embeds that hypothetical answer for similarity search. The hypothesis is more likely to semantically match real document language than a short, sparse query.

**Mental Model:** Instead of searching for what you asked, HyDE searches for what a good answer would look like. A question like "how does photosynthesis work?" gets expanded into a hypothetical paragraph about chlorophyll and light — which is far more likely to match relevant document text.

**Common Misconceptions:**
- HyDE always improves retrieval — for short, clear queries that already match document language, HyDE adds an unnecessary LLM call; it is most valuable for vague or short queries.
- The hypothetical document must be factually correct — HyDE only needs the generated text to be semantically plausible in the vector space; factual correctness of the hypothesis is not the goal.

**Interview Skeleton:**
- What it is: using LLM-generated hypothetical answers as the retrieval query instead of the raw user question
- Why it matters: improves retrieval for sparse or ambiguous queries by bridging the vocabulary gap between query and document language
- Example: implement HyDE for a technical documentation search system and describe when you'd A/B test it against direct query embedding

**Free Resources:** https://python.langchain.com/docs/tutorials/rag — LangChain RAG documentation covering advanced retrieval techniques including HyDE

---

## Ragas Evaluation

**Status:** ⬜ Not Started

**Definition:** Ragas is a framework for evaluating RAG systems using LLM-as-judge metrics. Key metrics include faithfulness (does the answer only use the retrieved context?), answer relevancy (does the answer address the question?), context precision (is the retrieved context relevant?), and context recall (was the necessary context retrieved?).

**Mental Model:** Ragas is a report card for your RAG system — not one score but four separate grades covering retrieval quality and generation quality independently, allowing you to isolate which part of the pipeline is failing.

**Common Misconceptions:**
- A high RAGAS score means the system is production-ready — RAGAS uses LLM judges which can be fooled by plausible-sounding but incorrect outputs; combine with human evaluation for critical applications.
- You only need to evaluate end-to-end answer quality — isolating retrieval metrics from generation metrics is essential for debugging; a wrong answer could be caused by poor retrieval or poor generation independently.

**Interview Skeleton:**
- What it is: a framework for systematically evaluating both retrieval and generation quality in RAG systems
- Why it matters: without evaluation, you cannot tell whether a RAG improvement is real or coincidental; metrics enable systematic iteration
- Example: describe a RAGAS evaluation pipeline for a RAG system and how you'd use the results to diagnose and improve the system

**Free Resources:** https://python.langchain.com/docs/tutorials/rag — LangChain RAG tutorial with evaluation patterns and references to the Ragas framework
