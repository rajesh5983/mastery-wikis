# Module 3 — RAG Systems

---

## Embeddings

**Status:** ⬜ Not Started

**Definition:** Embeddings are dense vector representations of text (or images, code) where semantically similar content is positioned close together in high-dimensional space. Embedding models (e.g., text-embedding-3-small, Cohere Embed) convert text chunks into vectors that can be stored in a vector database and searched by cosine or dot-product similarity.

**Key Mental Model:** Embeddings are coordinates in a meaning map. "King" and "Queen" are nearby. "Paris" and "London" are nearby. A search query's coordinates find nearby document chunks — semantically similar, even if no exact keywords match.

**How It Works:**
- An embedding model (e.g., a bi-encoder transformer) passes a text chunk through all attention layers and then pools the final hidden states (mean pooling or CLS token) into a single fixed-dimension vector (commonly 768, 1536, or 3072 floats). This vector is the embedding.
- At index time, each document chunk is encoded independently and its vector is stored in a vector database (Pinecone, Weaviate, pgvector). The database also stores the original text and metadata alongside the vector for retrieval.
- At query time, the user query is encoded by the same embedding model into a query vector. The database computes cosine similarity (or dot product for normalised vectors) between the query vector and every stored document vector. The top-k most similar vectors are returned as candidate chunks.
- Approximate Nearest Neighbour (ANN) algorithms — HNSW (Hierarchical Navigable Small World graphs) and IVF (Inverted File Index) — allow sub-linear search at scale. HNSW builds a layered graph of vectors; search navigates the graph greedily, never examining the full corpus. This trades a small recall penalty for orders-of-magnitude speed improvements at millions of vectors.
- Embedding quality is domain-sensitive. General-purpose embedding models trained on web text may conflate terms that are semantically similar in general language but distinct in your domain (e.g., "Python" as a language vs as an animal). Domain-specific fine-tuning of the embedding model can significantly improve retrieval recall.

**Common Misconceptions:**
- Semantic similarity always means topical relevance — embeddings capture general semantic similarity but can match on surface-level style or tone rather than topical overlap.
- Any embedding model works for any domain — embedding models trained on general web text may perform poorly on domain-specific content (legal, medical, code); domain-specific models matter.

**Interview Answer Skeleton:**
- **What it is:** Dense fixed-dimension vector representations of text produced by a bi-encoder transformer, enabling semantic similarity search where distance in vector space approximates semantic relatedness.
- **Why it matters / trade-offs:** Embedding quality is the retrieval ceiling in a RAG system — a weak embedding model cannot be compensated by a better LLM. ANN indexes (HNSW) make similarity search practical at scale but introduce approximate rather than exact nearest-neighbour guarantees.
- **Example or context:** Explain the difference between keyword search (BM25, exact term matching) and embedding-based semantic search, and when hybrid search combining both is better — BM25 handles rare exact-match terms (product codes, proper nouns), while embeddings handle paraphrase and concept-level queries.

**Free Resources:**
- [LangChain RAG Tutorials](https://python.langchain.com/docs/tutorials/rag) — Covers embedding models, vector store integration, and semantic search implementation
- [Hugging Face NLP Course](https://huggingface.co/learn/nlp-course) — Explains how encoder models produce embeddings and the mechanics of similarity search

---

## Chunking Strategies

**Status:** ⬜ Not Started

**Definition:** Chunking splits source documents into pieces that fit within the context window of an embedding model and are semantically coherent enough to be useful when retrieved. Common strategies include fixed-size chunking, sentence-level chunking, recursive character splitting, and semantic chunking based on topic shifts.

**Key Mental Model:** Chunking is dividing a textbook into index cards for revision. Too large and each card is overwhelming and unfocused. Too small and each card lacks enough context to stand alone. The goal is the smallest self-contained unit of meaning.

**How It Works:**
- Fixed-size chunking splits text every N characters or N tokens with an overlap (e.g., 512 tokens with 50-token overlap). The overlap ensures that sentences split at a boundary appear in both adjacent chunks, preserving continuity. This is the simplest approach and works adequately for prose documents.
- Recursive character splitting works through a hierarchy of separators: it first tries to split on paragraph breaks (`\n\n`), then on sentence breaks (`\n`), then on word breaks (` `). This respects document structure while still hitting a target chunk size — it only splits at finer boundaries when coarser ones don't fit the size budget.
- Semantic chunking embeds each sentence and groups consecutive sentences whose embedding cosine similarity drops below a threshold (indicating a topic shift). This produces chunks that are topically coherent rather than mechanically sized, which improves retrieval precision — each chunk covers one idea.
- Parent-child chunking (also called small-to-big retrieval) stores two versions of each chunk: a small child chunk (e.g., 128 tokens, for precise retrieval) and a larger parent chunk (e.g., 512 tokens, for broader context). Retrieval happens on child chunks; the LLM receives the parent chunk. This preserves retrieval precision without losing surrounding context.
- Structured documents (PDFs, HTML pages) require structure-aware chunking. A PDF table should be kept as a single chunk — row-splitting a table destroys its meaning. Markdown headers and section boundaries are natural chunk boundaries and should be preserved rather than split through.

**Common Misconceptions:**
- Smaller chunks are always better for retrieval precision — very small chunks lose context, causing the retrieved text to be ambiguous without surrounding sentences.
- One chunking strategy fits all document types — structured tables need different chunking than narrative prose, which needs different chunking than code; match the strategy to the content type.

**Interview Answer Skeleton:**
- **What it is:** The strategy for splitting source documents into segments that are semantically coherent, fit the embedding model's token limit, and contain enough context to be independently useful when retrieved.
- **Why it matters / trade-offs:** Chunking strategy is one of the highest-impact variables in RAG quality and is often the first thing to tune. Poor chunking degrades retrieval regardless of model or vector store quality. The optimal strategy is document-type-specific.
- **Example or context:** A 100-page technical PDF should use recursive splitting with section-header awareness (H1/H2 boundaries as natural chunk boundaries) and a parent-child setup. A customer support knowledge base with short discrete articles can use article-level chunking — each article is naturally self-contained and should not be split further.

**Free Resources:**
- [LangChain RAG Tutorials](https://python.langchain.com/docs/tutorials/rag) — Covers text splitters, chunking strategies, and their impact on retrieval quality
- [Eugene Yan's Blog](https://eugeneyan.com) — Applied ML posts covering RAG system design including chunking trade-offs and production lessons

---

## Reranking

**Status:** ⬜ Not Started

**Definition:** Reranking is a two-stage retrieval approach where an initial fast retrieval (embedding similarity or keyword search) returns a candidate set, and a more powerful cross-encoder model then re-scores each candidate against the query to produce a more accurate final ranking before sending to the LLM.

**Key Mental Model:** Initial retrieval is a quick shortlist — fast but imprecise. Reranking is the second interview round — slower, more thorough, only applied to the shortlisted candidates.

**How It Works:**
- Stage one uses a bi-encoder embedding search (ANN lookup) or BM25 keyword search to retrieve the top-50 to top-100 candidate chunks in low latency (typically 5–20ms for a well-indexed corpus). The candidates are ranked by approximate similarity score.
- Stage two passes each (query, candidate_chunk) pair through a cross-encoder model. The cross-encoder processes the query and the candidate together as a single concatenated input, enabling the model to score relevance based on fine-grained interaction between query and document terms — something bi-encoders cannot do because they encode independently.
- Cross-encoders (e.g., Cohere Rerank, BGE-reranker, ms-marco models) are computationally expensive because they must run a separate forward pass for each candidate pair. This is why they are only applied to the shortlist (50–100 candidates), not the full corpus.
- The top-k (typically 3–10) candidates after reranking are passed to the LLM as context. Reranking consistently improves MRR (Mean Reciprocal Rank) and NDCG (Normalised Discounted Cumulative Gain) by 10–30% over bi-encoder-only retrieval in practice.
- Reciprocal Rank Fusion (RRF) is a simpler alternative to learned reranking — it merges the ranked lists from multiple retrieval systems (embedding search + BM25) by summing reciprocal rank scores. This avoids running a second model but gives up the fine-grained query-document scoring that cross-encoders provide.

**Common Misconceptions:**
- Reranking is redundant if you have a good embedding model — embedding models use bi-encoders (fast, independent query and document encoding); rerankers use cross-encoders (slow, joint encoding) that capture richer query-document interaction.
- Reranking is only needed for large corpora — even small corpora benefit from reranking when top-k precision is critical to avoid sending irrelevant context to the LLM.

**Interview Answer Skeleton:**
- **What it is:** A two-stage retrieval pipeline where fast ANN/BM25 retrieval generates a candidate set, then a cross-encoder model re-scores each (query, chunk) pair jointly to produce a more accurate final ranking.
- **Why it matters / trade-offs:** Reranking consistently improves context precision, reducing the irrelevant content sent to the LLM and therefore reducing hallucination. The trade-off is added latency (one cross-encoder forward pass per candidate) and the cost of a second model call.
- **Example or context:** Design a pipeline: embed search retrieves top-50 candidates from pgvector in 15ms, Cohere Rerank re-scores all 50 pairs and returns the top-5 in 80ms, the top-5 chunks are assembled into the LLM context. Total retrieval latency is ~100ms, but context quality is significantly better than using top-5 from embedding search alone.

**Free Resources:**
- [LangChain RAG Tutorials](https://python.langchain.com/docs/tutorials/rag) — Covers reranking integration, cross-encoder models, and two-stage retrieval pipeline construction
- [Eugene Yan's Blog](https://eugeneyan.com) — Production RAG lessons including retrieval pipeline design and reranking evaluation

---

## Classic, Graph, and Agentic RAG

**Status:** ⬜ Not Started

**Definition:** Classic RAG retrieves flat text chunks and injects them into the prompt. Graph RAG models documents as a knowledge graph, enabling retrieval of entities and their relationships. Agentic RAG uses an agent that can iteratively query, refine, and decide when enough context has been gathered, rather than doing a single fixed retrieval.

**Key Mental Model:** Classic RAG is grabbing the three most relevant pages from a filing cabinet. Graph RAG is following the threads of a corkboard with connected pins. Agentic RAG is a research assistant who keeps pulling more files until they've answered the question fully.

**How It Works:**
- Classic RAG executes a single retrieval-then-generate pipeline: embed query → ANN search → inject top-k chunks → LLM generates answer. The retrieval step is stateless and runs once. This is the baseline and works well for most simple Q&A and document search use cases.
- Graph RAG (Microsoft GraphRAG) first extracts entities and relationships from documents using an LLM and stores them as a knowledge graph. At query time, the system traverses graph edges to find related entities rather than using embedding similarity. This excels at multi-hop questions like "what companies does person X have relationships with?" that cannot be answered from a single chunk.
- Agentic RAG implements a multi-step loop: the agent receives a query, performs retrieval, decides if the retrieved context is sufficient (using an LLM judge), and either generates an answer or refines the query and retrieves again. The loop terminates when sufficiency is met or a step limit is reached. See [[AI-Engineer/04-Agentic-Systems]] for loop mechanics.
- Query transformation techniques (query decomposition, step-back prompting, multi-query generation) improve retrieval by generating multiple reformulated versions of the user query. The system retrieves against each variant and merges the results before reranking. This handles queries that are underspecified or phrased in ways that don't match document language.
- Agentic RAG requires guardrails to prevent runaway retrieval loops. A maximum step count, a cost budget, and a timeout must all be enforced at the orchestration layer. Without these, a single complex query can trigger dozens of retrieval calls.

**Common Misconceptions:**
- Classic RAG is outdated and should always be replaced with agentic RAG — classic RAG is simpler, faster, and more predictable; agentic and graph RAG add complexity that is only justified for specific use cases.
- Graph RAG always outperforms flat RAG — graph RAG excels for relationship and entity queries; for straightforward text retrieval it adds overhead without improvement.

**Interview Answer Skeleton:**
- **What it is:** A spectrum of RAG architectures from single-shot flat retrieval (classic), through structured relationship-based retrieval (graph), to iterative agent-driven retrieval (agentic) — each adding capability at the cost of complexity and latency.
- **Why it matters / trade-offs:** Classic RAG is the right default. Graph RAG adds value when queries involve entity relationships across documents. Agentic RAG adds value when one retrieval pass is provably insufficient. The cost and complexity multipliers for graph and agentic RAG must be justified by measurable quality improvements.
- **Example or context:** A customer support FAQ is well-served by classic RAG. An enterprise knowledge management system where users ask "how does our refund policy interact with our international shipping policy?" benefits from agentic RAG that can retrieve both policies and reason over their interaction. A research assistant that must map relationships between papers benefits from graph RAG.

**Free Resources:**
- [LangChain RAG Tutorials](https://python.langchain.com/docs/tutorials/rag) — Covers classic RAG through advanced agentic and graph-based retrieval patterns
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph) — Framework for building agentic RAG systems with controllable retrieval loops

---

## HyDE (Hypothetical Document Embeddings)

**Status:** ⬜ Not Started

**Definition:** HyDE is a retrieval technique where, instead of embedding the user query directly, the LLM first generates a hypothetical answer to the query, then embeds that hypothetical answer for similarity search. The hypothesis is more likely to semantically match real document language than a short, sparse query.

**Key Mental Model:** Instead of searching for what you asked, HyDE searches for what a good answer would look like. A question like "how does photosynthesis work?" gets expanded into a hypothetical paragraph about chlorophyll and light — which is far more likely to match relevant document text.

**How It Works:**
- HyDE adds one LLM call before the retrieval step. The LLM receives the user query and a prompt like "write a short passage that directly answers this question." The output is a plausible but potentially fabricated paragraph-length answer.
- The generated hypothetical document is embedded using the same embedding model as the indexed corpus. Because it is written in the style and vocabulary of a real answer, its embedding vector occupies a region of vector space much closer to relevant corpus documents than the original short query would.
- The original query and the hypothetical document embeddings can be averaged or the hypothetical document can be used alone — experiments show using the hypothetical document embedding alone generally outperforms query-only for sparse or short queries.
- The LLM generating the hypothesis does not need to know the correct answer. HyDE relies purely on the semantic alignment between the hypothetical passage's language patterns and real document language, not on factual accuracy. A plausible but wrong hypothesis often still retrieves the correct documents.
- HyDE adds one LLM call worth of latency to the retrieval pipeline. This is worth the cost when the user query is known to be short, sparse, or phrased very differently from document language. For long, well-formed queries, direct embedding typically matches or exceeds HyDE performance.

**Common Misconceptions:**
- HyDE always improves retrieval — for short, clear queries that already match document language, HyDE adds an unnecessary LLM call; it is most valuable for vague or short queries.
- The hypothetical document must be factually correct — HyDE only needs the generated text to be semantically plausible in the vector space; factual correctness of the hypothesis is not the goal.

**Interview Answer Skeleton:**
- **What it is:** A retrieval technique that uses an LLM to generate a hypothetical answer, then embeds that answer for ANN search rather than embedding the original sparse query — bridging the vocabulary gap between query style and document style.
- **Why it matters / trade-offs:** HyDE improves retrieval recall for short or underspecified queries at the cost of one additional LLM call per query. A/B test it against direct query embedding on your specific corpus — improvements are query-type dependent.
- **Example or context:** In a technical documentation search system, the user query "why is my API returning 429?" is short and might not match chunks using the phrase "rate limiting." HyDE generates a hypothetical paragraph explaining rate limit headers, retry logic, and quota management — language that now closely matches the documentation chunks that should be retrieved.

**Free Resources:**
- [LangChain RAG Tutorials](https://python.langchain.com/docs/tutorials/rag) — Advanced retrieval documentation covering HyDE and query transformation techniques
- [Papers With Code](https://paperswithcode.com) — Original HyDE paper and subsequent retrieval augmentation research

---

## Ragas Evaluation

**Status:** ⬜ Not Started

**Definition:** Ragas is a framework for evaluating RAG systems using LLM-as-judge metrics. Key metrics include faithfulness (does the answer only use the retrieved context?), answer relevancy (does the answer address the question?), context precision (is the retrieved context relevant?), and context recall (was the necessary context retrieved?).

**Key Mental Model:** Ragas is a report card for your RAG system — not one score but four separate grades covering retrieval quality and generation quality independently, allowing you to isolate which part of the pipeline is failing.

**How It Works:**
- Faithfulness is measured by decomposing the generated answer into individual factual claims, then using an LLM judge to verify whether each claim is supported by the retrieved context. The score is the fraction of claims that can be attributed to the context — a faithfulness score of 1.0 means every claim is grounded in what was retrieved.
- Answer relevancy is measured by prompting the LLM judge to generate questions that the answer addresses, then comparing those generated questions to the original query via embedding similarity. Answers that go off-topic or include unnecessary information score lower.
- Context precision measures whether the retrieved chunks are relevant — specifically, what fraction of the retrieved context was actually useful for generating the answer. High recall with low precision means you are retrieving too many irrelevant chunks, polluting the LLM's context.
- Context recall (requires a ground-truth answer) checks whether all the information needed to generate the reference answer was present in the retrieved context. Low recall means the retrieval step is missing critical information, causing the model to either hallucinate or say it doesn't know.
- These four metrics together form a diagnostic matrix. Low faithfulness points to generation problems (model not grounding in context). Low context precision points to retrieval returning irrelevant results. Low context recall points to retrieval missing relevant content — likely a chunking or embedding issue. See [[AI-Engineer/07-Observability-Evals]] for integrating these into a monitoring pipeline.

**Common Misconceptions:**
- A high RAGAS score means the system is production-ready — RAGAS uses LLM judges which can be fooled by plausible-sounding but incorrect outputs; combine with human evaluation for critical applications.
- You only need to evaluate end-to-end answer quality — isolating retrieval metrics from generation metrics is essential for debugging; a wrong answer could be caused by poor retrieval or poor generation independently.

**Interview Answer Skeleton:**
- **What it is:** An evaluation framework that uses LLM judges to measure four orthogonal dimensions of RAG quality: faithfulness, answer relevancy, context precision, and context recall — enabling component-level diagnosis of pipeline failures.
- **Why it matters / trade-offs:** Without systematic evaluation, RAG improvements are anecdotal. Ragas metrics enable A/B testing of chunking strategies, embedding models, and retrieval parameters with a repeatable signal. LLM-as-judge scoring has its own noise — calibrate against human labels on a subset before trusting the automated scores.
- **Example or context:** Your RAG system produces answers that are occasionally wrong. Run a Ragas evaluation on 100 queries. If faithfulness is 0.95 but context recall is 0.6, the retrieval is missing relevant chunks — tune chunking strategy or embedding model. If faithfulness is 0.7 but context recall is 0.9, the LLM is hallucinating beyond what was retrieved — add explicit grounding instructions to the prompt.

**Free Resources:**
- [LangChain RAG Tutorials](https://python.langchain.com/docs/tutorials/rag) — Includes evaluation setup and references to the Ragas framework for systematic RAG measurement
- [Arize Phoenix Documentation](https://docs.arize.com/phoenix) — LLM evaluation and RAG-specific tracing with built-in Ragas metric support
