# Module 9 — Software Engineering Essentials

---

## Python Async

**Status:** ⬜ Not Started

**Definition:** Python's async/await syntax enables concurrent I/O-bound operations without threads. An async function returns a coroutine; `await` suspends it while waiting for I/O (network call, file read), allowing other coroutines to run on the same thread. For LLM applications, async is essential for handling multiple concurrent API calls efficiently.

**Key Mental Model:** Async is like a waiter who takes one table's order, puts it in the kitchen, takes another table's order while the first cooks, checks back on the first table when the food is ready — all on the same person without standing around waiting.

**How It Works:**
- Python's asyncio event loop is a single-threaded scheduler. It maintains a queue of ready coroutines and a set of pending I/O operations. When a coroutine hits an `await` on an I/O operation, it suspends and returns control to the event loop, which runs other ready coroutines until the I/O completes. No thread context switches occur — the coroutine is simply paused and resumed.
- An `async def` function, when called, does not execute immediately. It returns a coroutine object — a suspended function. The coroutine only starts executing when it is either `await`ed or scheduled with `asyncio.create_task()` or `asyncio.gather()`.
- `asyncio.gather(*coroutines)` schedules all coroutines concurrently and awaits all of them. If you have 10 LLM API calls to make, `gather` starts all 10 simultaneously and waits for all 10 to complete — total elapsed time is max(individual latencies) rather than sum. This is how parallel tool calls are implemented at the application layer.
- `asyncio.Semaphore` provides concurrency limiting: `async with asyncio.Semaphore(5)` ensures at most 5 concurrent operations run simultaneously. This is the standard pattern for rate-limit-aware parallel processing — process a batch of 100 items but never have more than 5 LLM API calls in-flight at once.
- CPU-bound work inside an async context blocks the event loop. ML inference (running a local embedding model), heavy data processing, or anything that doesn't yield to I/O must be offloaded to a thread pool with `asyncio.run_in_executor(None, blocking_function)` or `loop.run_in_executor()`. This runs the CPU work in a thread, allowing the event loop to continue serving other coroutines.

**Common Misconceptions:**
- Async makes code faster — async improves concurrency for I/O-bound tasks but adds no benefit (and some overhead) for CPU-bound tasks; use threading or multiprocessing for CPU-heavy work.
- Async functions are automatically parallel — async functions run concurrently (interleaved on one thread), not in parallel (simultaneously on multiple threads); for true parallelism, use asyncio.gather or thread/process pools.

**Interview Answer Skeleton:**
- **What it is:** Python's cooperative multitasking model where coroutines yield control to an event loop at `await` points, enabling high-concurrency I/O-bound operations (LLM API calls, database queries, HTTP requests) on a single thread without blocking.
- **Why it matters / trade-offs:** LLM applications are almost entirely I/O-bound — the bottleneck is waiting for API responses. Async allows a single Python process to handle 50+ concurrent API calls without spawning 50 threads. The trade-off is that CPU-bound work must be explicitly offloaded and debugging async code (especially stack traces) is harder than synchronous code.
- **Example or context:** Embedding a batch of 100 documents: `await asyncio.gather(*[embed(doc) for doc in docs])` dispatches all 100 embedding API calls concurrently and collects results. Wrap with a semaphore of 10 to respect the provider's concurrent request limit. Total time: latency of the slowest embedding call, not 100 × latency.

**Free Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com) — Async Python patterns, event loop integration, and concurrent request handling in async web applications
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Async LLM API call patterns including parallel requests and rate-limit-aware concurrency control

---

## FastAPI

**Status:** ⬜ Not Started

**Definition:** FastAPI is a modern Python web framework for building APIs with automatic OpenAPI documentation, type validation via Pydantic, and first-class async support. It is the standard choice for serving LLM-powered APIs due to its performance, schema validation, and developer ergonomics.

**Key Mental Model:** FastAPI is a self-documenting API framework — you define your endpoint, declare the input/output types, and FastAPI automatically validates requests, serialises responses, and generates interactive API docs.

**How It Works:**
- FastAPI is built on top of Starlette (the ASGI web framework) and Pydantic (the validation library). An `async def` route function is wrapped by Starlette's request handling, which runs it as a coroutine on the event loop. This means each request is handled concurrently without blocking — essential for LLM API applications where each request involves multiple I/O waits.
- Request body types are declared as Pydantic `BaseModel` subclasses. FastAPI passes the raw JSON request body through Pydantic validation before the route function is called. If validation fails (wrong types, missing required fields), FastAPI automatically returns a 422 with a structured error message listing which fields failed. This eliminates manual request parsing and validation code.
- Dependency injection via `Depends()` allows shared resources (database connections, authenticated user objects, rate-limit state) to be declared as function parameters and resolved per-request. This is how you inject an async database connection pool or a shared LLM client without global state.
- Streaming responses for LLM output use `StreamingResponse` with an async generator. The generator yields chunks as they arrive from the LLM streaming API; FastAPI writes each chunk to the HTTP response immediately using chunked transfer encoding. The client receives tokens as they are generated. See [[AI-Engineer/08-Production-AI-Engineering]] for streaming implementation details.
- Background tasks (`BackgroundTasks`) allow post-request work (logging, analytics writes, async notification calls) to run after the response has been sent to the client. This avoids adding latency to the response for work that doesn't affect the response content — for example, writing a trace to Langfuse after the LLM response is streamed.

**Common Misconceptions:**
- FastAPI is only for simple APIs — FastAPI supports complex dependency injection, background tasks, WebSockets, streaming responses, and middleware; it scales to production complexity.
- Any Python web framework works equally well for LLM APIs — FastAPI's native async support is essential for LLM applications where each request involves multiple async I/O calls.

**Interview Answer Skeleton:**
- **What it is:** An ASGI-based Python web framework combining Starlette's async request handling with Pydantic's type validation, providing automatic request parsing, response serialisation, OpenAPI documentation generation, and native support for streaming responses.
- **Why it matters / trade-offs:** FastAPI's async-first architecture is a natural fit for LLM applications — every route can make concurrent async API calls without blocking. Pydantic validation eliminates boilerplate. The trade-off versus Flask/Django is a steeper learning curve for async patterns and less mature ecosystem for non-API use cases.
- **Example or context:** A FastAPI endpoint that accepts a user query, calls Claude with streaming, and returns an SSE stream: define a `StreamingResponse` endpoint with an async generator that iterates over the Anthropic streaming client, yields each text delta as an SSE event, and closes the stream when the `message_stop` event is received.

**Free Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com) — Full framework documentation including async routes, Pydantic validation, dependency injection, and streaming responses
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — FastAPI + Claude integration examples including streaming endpoints and production deployment patterns

---

## Git

**Status:** ⬜ Not Started

**Definition:** Git is the standard distributed version control system for tracking code changes, enabling collaboration, and maintaining history. For AI engineering, good Git hygiene includes versioning prompts alongside code, tagging model versions in commits, and using branches for prompt experiments.

**Key Mental Model:** Git is the time machine for your code — every commit is a snapshot you can return to. Branches are parallel timelines where you can experiment without breaking the main timeline.

**How It Works:**
- Git stores the repository history as a directed acyclic graph (DAG) of commit objects. Each commit contains a pointer to a tree object (the directory snapshot), metadata (author, timestamp, message), and a pointer to the parent commit(s). The HEAD pointer tracks the current position in the graph.
- Branching creates a lightweight pointer to a commit — no file copying occurs. A new branch is just a named reference in `.git/refs/heads/`. Switching branches updates HEAD and checks out the files for that commit's tree. This makes branching essentially free, which is why feature branches are the standard workflow.
- The staging area (index) is an intermediate layer between the working directory and the commit history. `git add` copies file snapshots into the index; `git commit` creates a commit from whatever is in the index, not from the working directory. This allows partial commits — staging specific files while leaving others unstaged.
- For AI engineering, prompt files should live in the repository alongside the code that uses them. A naming convention like `prompts/system_prompt_v3.txt` with a corresponding git tag (e.g., `prompt-v3`) allows exact prompt versions to be referenced in incident reports and rollbacks. When a prompt change degrades quality, `git revert` restores the previous version instantly.
- `.gitignore` must explicitly exclude `.env` files containing API keys and any files with model outputs or user data that should not be committed. AI engineering projects also commonly exclude large model weight files (use Git LFS for these) and generated eval output files that are too large for the repository.

**Common Misconceptions:**
- Git is just for large teams — solo developers benefit from version control; the ability to revert a prompt change that degraded quality is invaluable in AI engineering.
- Prompts don't need version control — prompts are configuration that affects model behaviour; versioning them is as important as versioning application code.

**Interview Answer Skeleton:**
- **What it is:** A distributed version control system that tracks code, prompts, and configuration as a DAG of immutable commits, enabling branching for experiments, history for rollbacks, and collaboration through merge workflows.
- **Why it matters / trade-offs:** In AI engineering, prompts and model configurations are first-class code artifacts. Version-controlling them enables safe iteration (branch for prompt experiments), rollback (revert a bad prompt change), and reproducibility (tag the exact prompt + code version used for a given evaluation run).
- **Example or context:** Prompt experiment workflow: create a branch `prompt-experiment/cot-v2`, update the prompt file, run the eval suite on the branch, compare scores against main. If scores improve, merge; if not, discard the branch. Tag the merge commit with the eval score for future reference. This is the same workflow as any code feature branch — prompts are just code.

**Free Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com) — Covers modern development workflows for API projects including version control practices
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Project structure examples showing prompt versioning and configuration management in AI application repositories

---

## Docker

**Status:** ⬜ Not Started

**Definition:** Docker packages an application and all its dependencies into a portable container image that runs consistently across development, staging, and production environments. For LLM applications, containers ensure reproducible inference environments and enable deployment to any cloud platform.

**Key Mental Model:** Docker is like a shipping container — it packages everything the application needs (code, runtime, libraries, config) into a standard box that runs identically on any ship (server) that can handle standard containers.

**How It Works:**
- A Docker image is built from a `Dockerfile` — a sequence of layered instructions. Each instruction (FROM, RUN, COPY, ENV) creates an immutable layer. Layers are content-addressed and cached by the Docker build daemon. If a layer's content has not changed, the cached layer is reused — builds only recompute from the first changed layer downward.
- Layer ordering is critical for build cache efficiency. Instructions that change rarely (installing OS packages, Python dependencies from `requirements.txt`) should appear before instructions that change frequently (copying application code). This way, dependency layers are cached and only the application code layer is rebuilt on each code change.
- Multi-stage builds reduce final image size: the first stage (`FROM python:3.12 AS builder`) installs build dependencies and compiles any native extensions; the final stage (`FROM python:3.12-slim`) copies only the compiled artifacts from the builder stage. Build tools (gcc, build-essential) are excluded from the production image.
- For LLM applications, Docker containers typically include: Python runtime, application dependencies (anthropic, fastapi, langchain, etc.), and the application code. API keys must never be baked into the image — they are injected at runtime as environment variables or mounted secrets.
- Container orchestration (Kubernetes, ECS, Cloud Run) runs multiple replicas of the container for availability and horizontal scaling. Each request is routed to a replica by a load balancer. LLM applications are typically stateless (session state is in Redis, not the container), making horizontal scaling straightforward.

**Common Misconceptions:**
- Docker is only needed for production deployment — Docker eliminates "works on my machine" problems during development and makes local environment setup reproducible for new team members.
- Larger Docker images are fine — large images slow down CI/CD pipelines and increase cold start times; use multi-stage builds and minimal base images for LLM application containers.

**Interview Answer Skeleton:**
- **What it is:** A containerisation platform that packages application code and all runtime dependencies into portable, immutable images that run consistently on any Docker-compatible host — enabling reproducible environments from local development through production.
- **Why it matters / trade-offs:** Docker eliminates environment inconsistencies, simplifies cloud deployment, and enables horizontal scaling. For LLM applications, it ensures the exact Python version, dependency versions, and OS libraries are consistent. Build cache efficiency (layer ordering) and image size (multi-stage builds) are the key engineering concerns.
- **Example or context:** A FastAPI LLM app Dockerfile: stage 1 builds and caches pip dependencies (changes rarely), stage 2 copies application code (changes on every deploy). Production image is ~300MB rather than ~1GB because build tools are excluded. API keys are passed as environment variables at `docker run` time, never baked into the image. The same image runs locally, in CI, and in production.

**Free Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com) — Includes Docker deployment guides with Dockerfile examples for FastAPI LLM applications
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook) — Production deployment patterns showing containerised LLM application setup

---

## Postgres and pgvector

**Status:** ⬜ Not Started

**Definition:** PostgreSQL is the standard open-source relational database. pgvector is a Postgres extension that adds a vector data type and approximate nearest-neighbour search, enabling semantic similarity search within an existing Postgres database without a separate vector store.

**Key Mental Model:** pgvector turns your Postgres database into a vector store — the same database that stores your application's relational data can also store and search embeddings, eliminating a separate infrastructure component for many use cases.

**How It Works:**
- The pgvector extension adds a `vector(N)` column type to Postgres, where N is the embedding dimension. Vectors are stored as packed float32 arrays in a Postgres heap table alongside regular relational columns (metadata, document text, timestamps, user IDs).
- Similarity search is executed as a SQL query using pgvector operators. `embedding <=> query_vector` computes cosine distance; `embedding <-> query_vector` computes L2 (Euclidean) distance; `embedding <#> query_vector` computes negative inner product (for dot-product similarity). `ORDER BY embedding <=> $1 LIMIT 5` returns the 5 most similar vectors.
- Without an index, pgvector performs exact sequential scan over all vectors — accurate but O(N) per query, which becomes slow at millions of vectors. The HNSW index (`CREATE INDEX ... USING hnsw (embedding vector_cosine_ops)`) builds a Hierarchical Navigable Small World graph that enables approximate nearest-neighbour search in O(log N) time at a small recall penalty (configurable with `ef_search` and `ef_construction` parameters).
- HNSW builds a layered graph where each vector is connected to its nearest neighbours at multiple granularity levels. At query time, the search starts at the top (sparsest) layer, greedily navigates toward the query vector, drops down to finer layers, and continues until it reaches the base layer where the approximate top-k results are returned. Higher `ef_search` increases recall but slows query time.
- pgvector's killer feature for AI engineering is SQL join pushdown: you can filter by metadata columns in the same query as the vector search, e.g., `WHERE user_id = $2 AND created_at > $3 ORDER BY embedding <=> $1 LIMIT 5`. This pre-filters the search space before the ANN search, combining structured filtering and semantic search without a separate vector database.

**Common Misconceptions:**
- pgvector is only suitable for small-scale vector search — pgvector with HNSW indexing handles millions of vectors efficiently; it is production-grade for most applications that don't require dedicated vector database scale.
- Using pgvector means you don't need a proper vector database strategy — as vector counts grow into hundreds of millions, dedicated vector databases (Pinecone, Weaviate, Qdrant) may still be necessary.

**Interview Answer Skeleton:**
- **What it is:** A Postgres extension that adds a native vector column type and HNSW-based approximate nearest-neighbour search to standard PostgreSQL, enabling semantic similarity search within the same database that stores application relational data.
- **Why it matters / trade-offs:** pgvector eliminates the operational overhead of a separate vector database for most applications, and enables metadata-filtered semantic search via SQL joins — a capability that requires extra complexity in dedicated vector databases. The trade-off is that pgvector's throughput and scale ceiling is lower than purpose-built vector databases at very large vector counts.
- **Example or context:** Schema for a RAG system: table `documents` with columns `id BIGSERIAL`, `content TEXT`, `embedding vector(1536)`, `source_url TEXT`, `created_at TIMESTAMPTZ`. HNSW index on `embedding`. Query: `SELECT content FROM documents WHERE source_url LIKE 'docs.company.com%' ORDER BY embedding <=> $1 LIMIT 5` — filters to company docs then returns top-5 by semantic similarity.

**Free Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com) — Database integration patterns with async SQLAlchemy and Postgres for AI applications
- [LangChain RAG Tutorials](https://python.langchain.com/docs/tutorials/rag) — pgvector integration examples showing schema setup, HNSW indexing, and similarity search in RAG pipelines

---

## Redis

**Status:** ⬜ Not Started

**Definition:** Redis is an in-memory key-value data store used for caching, session storage, rate limiting, pub/sub messaging, and job queues. In LLM applications, Redis is commonly used for semantic cache storage, conversation session state, and rate-limit counters.

**Key Mental Model:** Redis is the scratch pad on your desk — everything important is kept there for instant access without going to the filing cabinet (database). It's temporary, fast, and limited in size.

**How It Works:**
- Redis stores all data in RAM, making reads and writes sub-millisecond (typically <1ms). Data is persisted optionally via RDB (periodic snapshots) or AOF (append-only file logging every write). AOF with `fsync everysec` provides at-most-1-second durability; RDB provides periodic snapshots with potential data loss between snapshots.
- Redis data structures go beyond key-value: Strings (cache values), Lists (queues), Sets (unique membership checks, deduplication), Sorted Sets (leaderboards, priority queues), Hashes (structured objects like session state), and Streams (message queues with consumer groups). For LLM applications: Strings for cached LLM responses, Sorted Sets for rate-limit sliding windows, Hashes for conversation session state.
- TTL (Time-to-Live) expiry is set per key: `SET key value EX 3600` sets the key to expire after 3600 seconds. Expired keys are lazily deleted (on access) or eagerly deleted by a background expiry sweep. For semantic caches, TTL should match the freshness requirement of the underlying data.
- Eviction policies control what Redis does when memory is full. `allkeys-lru` evicts the least-recently-used key across all keys — appropriate for pure caches where losing any key is acceptable. `volatile-lru` only evicts keys with TTL set — protects keys without expiry from eviction, useful when Redis holds both cache data (with TTL) and persistent session data (without TTL) in the same instance.
- Rate limiting with Redis uses the atomic increment pattern: `INCR user:123:requests` increments a counter; `EXPIRE user:123:requests 60` sets a 60-second window. The pre-computed sliding window approach uses a Sorted Set keyed by timestamp: `ZADD user:123:requests timestamp timestamp`, `ZREMRANGEBYSCORE ... 0 (now-window)`, `ZCARD ... > limit` checks current rate. The atomic Sorted Set operations prevent race conditions in distributed rate limiting.

**Common Misconceptions:**
- Redis is only a cache — Redis supports rich data structures (lists, sets, sorted sets, streams) and persistence options; it is used as a primary store for session state, leaderboards, and message queues.
- Redis data is always lost on restart — Redis supports persistence (RDB snapshots and AOF logs) and replication; data durability depends on how it is configured.

**Interview Answer Skeleton:**
- **What it is:** An in-memory data store with rich data structures (strings, hashes, sorted sets, streams), sub-millisecond latency, TTL-based expiry, and configurable eviction policies — used in LLM applications for caching responses, persisting conversation sessions, and enforcing API rate limits.
- **Why it matters / trade-offs:** Redis provides the performance tier between application logic and the database. For LLM applications, it enables semantic cache lookups (< 5ms vs > 200ms for LLM calls), conversation state retrieval (< 1ms per turn), and distributed rate limiting (atomic operations across multiple app instances). The trade-off is memory cost — all data must fit in RAM, making Redis expensive for large datasets.
- **Example or context:** Conversation session state for a multi-turn chatbot: store each session as a Redis Hash keyed by `session:{session_id}`, with fields for user ID, message history (JSON-serialised), and session metadata. Set TTL to 30 minutes of inactivity: reset the TTL on every user message. Use `allkeys-lru` eviction — if memory pressure forces eviction, old inactive sessions are evicted first.

**Free Resources:**
- [FastAPI Documentation](https://fastapi.tiangolo.com) — Redis integration patterns for caching, session management, and rate limiting in async Python web applications
- [Langfuse Documentation](https://langfuse.com/docs) — Session tracking and conversation state management patterns relevant to Redis-backed session architectures
