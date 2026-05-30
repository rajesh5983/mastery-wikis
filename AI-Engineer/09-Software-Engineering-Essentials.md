# Module 9 — Software Engineering Essentials

---

## Python Async

**Status:** ⬜ Not Started

**Definition:** Python's async/await syntax enables concurrent I/O-bound operations without threads. An async function returns a coroutine; `await` suspends it while waiting for I/O (network call, file read), allowing other coroutines to run on the same thread. For LLM applications, async is essential for handling multiple concurrent API calls efficiently.

**Mental Model:** Async is like a waiter who takes one table's order, puts it in the kitchen, takes another table's order while the first cooks, checks back on the first table when the food is ready — all on the same person without standing around waiting.

**Common Misconceptions:**
- Async makes code faster — async improves concurrency for I/O-bound tasks but adds no benefit (and some overhead) for CPU-bound tasks; use threading or multiprocessing for CPU-heavy work.
- Async functions are automatically parallel — async functions run concurrently (interleaved on one thread), not in parallel (simultaneously on multiple threads); for true parallelism, use asyncio.gather or thread/process pools.

**Interview Skeleton:**
- What it is: Python's concurrency model for I/O-bound operations using cooperative multitasking
- Why it matters: LLM API calls are I/O-bound; async allows one service to handle dozens of concurrent API calls without spawning dozens of threads
- Example: implement an async function that calls the LLM API for a list of inputs concurrently using asyncio.gather with proper rate-limit handling

**Free Resources:** https://fastapi.tiangolo.com — FastAPI documentation covering async Python patterns, async routes, and async dependency injection

---

## FastAPI

**Status:** ⬜ Not Started

**Definition:** FastAPI is a modern Python web framework for building APIs with automatic OpenAPI documentation, type validation via Pydantic, and first-class async support. It is the standard choice for serving LLM-powered APIs due to its performance, schema validation, and developer ergonomics.

**Mental Model:** FastAPI is a self-documenting API framework — you define your endpoint, declare the input/output types, and FastAPI automatically validates requests, serialises responses, and generates interactive API docs.

**Common Misconceptions:**
- FastAPI is only for simple APIs — FastAPI supports complex dependency injection, background tasks, WebSockets, streaming responses, and middleware; it scales to production complexity.
- Any Python web framework works equally well for LLM APIs — FastAPI's native async support is essential for LLM applications where each request involves multiple async I/O calls.

**Interview Skeleton:**
- What it is: a Python API framework optimised for async operations, type validation, and auto-documentation
- Why it matters: the standard framework for exposing LLM-powered features as production API endpoints
- Example: build a FastAPI endpoint that accepts a user query, calls Claude with streaming, and returns a Server-Sent Events stream

**Free Resources:** https://fastapi.tiangolo.com — FastAPI official documentation with tutorials covering routing, validation, async, and streaming

---

## Git

**Status:** ⬜ Not Started

**Definition:** Git is the standard distributed version control system for tracking code changes, enabling collaboration, and maintaining history. For AI engineering, good Git hygiene includes versioning prompts alongside code, tagging model versions in commits, and using branches for prompt experiments.

**Mental Model:** Git is the time machine for your code — every commit is a snapshot you can return to. Branches are parallel timelines where you can experiment without breaking the main timeline.

**Common Misconceptions:**
- Git is just for large teams — solo developers benefit from version control; the ability to revert a prompt change that degraded quality is invaluable in AI engineering.
- Prompts don't need version control — prompts are configuration that affects model behaviour; versioning them is as important as versioning application code.

**Interview Skeleton:**
- What it is: the version control system that tracks every change to code, prompts, configuration, and infrastructure
- Why it matters: enables safe iteration, collaboration, rollback, and audit trail for all application changes
- Example: describe your Git workflow for iterating on prompts in production: branching strategy, commit hygiene, and rollback procedure

**Free Resources:** https://fastapi.tiangolo.com — FastAPI documentation references modern development workflows including Git best practices for API projects

---

## Docker

**Status:** ⬜ Not Started

**Definition:** Docker packages an application and all its dependencies into a portable container image that runs consistently across development, staging, and production environments. For LLM applications, containers ensure reproducible inference environments and enable deployment to any cloud platform.

**Mental Model:** Docker is like a shipping container — it packages everything the application needs (code, runtime, libraries, config) into a standard box that runs identically on any ship (server) that can handle standard containers.

**Common Misconceptions:**
- Docker is only needed for production deployment — Docker eliminates "works on my machine" problems during development and makes local environment setup reproducible for new team members.
- Larger Docker images are fine — large images slow down CI/CD pipelines and increase cold start times; use multi-stage builds and minimal base images for LLM application containers.

**Interview Skeleton:**
- What it is: containerisation technology that packages application code and dependencies into portable, reproducible environments
- Why it matters: ensures consistent LLM application behaviour across all environments and simplifies cloud deployment
- Example: write a Dockerfile for a FastAPI LLM application and explain your layer ordering decisions for cache efficiency

**Free Resources:** https://fastapi.tiangolo.com — FastAPI documentation includes Docker deployment guides and container best practices

---

## Postgres and pgvector

**Status:** ⬜ Not Started

**Definition:** PostgreSQL is the standard open-source relational database. pgvector is a Postgres extension that adds a vector data type and approximate nearest-neighbour search, enabling semantic similarity search within an existing Postgres database without a separate vector store.

**Mental Model:** pgvector turns your Postgres database into a vector store — the same database that stores your application's relational data can also store and search embeddings, eliminating a separate infrastructure component for many use cases.

**Common Misconceptions:**
- pgvector is only suitable for small-scale vector search — pgvector with HNSW indexing handles millions of vectors efficiently; it is production-grade for most applications that don't require dedicated vector database scale.
- Using pgvector means you don't need a proper vector database strategy — as vector counts grow into hundreds of millions, dedicated vector databases (Pinecone, Weaviate, Qdrant) may still be necessary.

**Interview Skeleton:**
- What it is: Postgres extended with a vector type and similarity search, enabling RAG applications without a separate vector store
- Why it matters: simplifies architecture for applications that already use Postgres; reduces operational overhead
- Example: design a pgvector schema for a RAG system, including the vector column, HNSW index, and the metadata filter columns

**Free Resources:** https://fastapi.tiangolo.com — FastAPI documentation references database integration patterns including Postgres with SQLAlchemy async

---

## Redis

**Status:** ⬜ Not Started

**Definition:** Redis is an in-memory key-value data store used for caching, session storage, rate limiting, pub/sub messaging, and job queues. In LLM applications, Redis is commonly used for semantic cache storage, conversation session state, and rate-limit counters.

**Mental Model:** Redis is the scratch pad on your desk — everything important is kept there for instant access without going to the filing cabinet (database). It's temporary, fast, and limited in size.

**Common Misconceptions:**
- Redis is only a cache — Redis supports rich data structures (lists, sets, sorted sets, streams) and persistence options; it is used as a primary store for session state, leaderboards, and message queues.
- Redis data is always lost on restart — Redis supports persistence (RDB snapshots and AOF logs) and replication; data durability depends on how it is configured.

**Interview Skeleton:**
- What it is: an in-memory data store used for caching, session management, rate limiting, and queuing in LLM applications
- Why it matters: enables semantic caching, conversation state persistence, and request rate limiting without expensive database queries
- Example: design a Redis-based conversation session store for a multi-turn chatbot and explain TTL and eviction strategy choices

**Free Resources:** https://fastapi.tiangolo.com — FastAPI documentation covering Redis integration for caching and session management in API applications
