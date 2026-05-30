# Layer 1 — Foundation

> **Framework:** Core programming and tooling fluency that underpins all data engineering work.

---

## SQL Basics

**Status:** ⬜ Not Started

**Definition:** SQL (Structured Query Language) is the standard declarative language for querying and manipulating data in relational databases. It covers SELECT, filtering, joins, grouping, and aggregation — the building blocks every data engineer uses daily regardless of whether they're working in dbt, Spark SQL, BigQuery, Snowflake, or Redshift.

**Key Mental Model:** A database is a spreadsheet factory. SQL is the instruction set you hand to a query planner, which figures out the optimal physical execution path — it decides which indexes to hit, how to join tables, and what to filter early — before a single row is returned.

**How It Works:**
- SQL is declarative: you specify *what* you want, not *how* to retrieve it. The query planner translates your SQL into a physical execution plan, choosing between index scans, full table scans, hash joins, or merge joins depending on table statistics and available indexes.
- Execution follows a logical order different from write order: FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT. The optimizer may reorder physical operations, but the logical precedence is fixed — this is why you cannot reference a SELECT alias in a WHERE clause.
- The query engine builds intermediate result sets at each stage. WHERE filters rows before aggregation (reducing data early), while HAVING filters after GROUP BY on aggregated values — confusing the two causes either wrong results or wasted compute.
- JOINs are implemented as hash joins (build a hash table from the smaller side, probe with the larger), merge joins (both sides sorted on the join key), or nested loop joins (O(n²), used for small inputs or indexed lookups). The planner picks the cheapest based on estimated cardinality from table statistics.
- Window functions (ROW_NUMBER, RANK, LAG) execute after WHERE and GROUP BY but before the final SELECT projection, operating over a partition of rows without collapsing them — this is mechanically different from GROUP BY aggregation.

**Common Misconceptions:**
- SQL is "just for analysts" — data engineers write complex SQL constantly for pipeline transformations, incremental load logic, data quality assertions, and schema migrations in tools like [[DE-Engineer/02-SQL]] and dbt.
- JOIN order doesn't matter — while logically equivalent, join order dramatically affects the query plan. A poorly ordered join can cause a huge intermediate result set that spills to disk, turning a 2-second query into a 2-minute one.

**Interview Answer Skeleton:**
- **What it is:** A declarative language where you express the desired result and the query planner determines the optimal physical execution plan using table statistics, indexes, and join strategies.
- **Why it matters / trade-offs:** It's the universal interface across every modern data platform. The trade-off is that declarative abstraction can hide performance problems — you need to understand the execution plan to optimise at scale, not just write correct SQL.
- **Example or context:** In a query joining orders and customers grouped by region with a HAVING filter, the engine evaluates FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT. Understanding this order explains why pushing filters into CTEs or subqueries reduces the rows processed before an expensive join.

**Free Resources:**
- [Mode SQL Tutorial](https://mode.com/sql-tutorial) — SQL tutorials from Mode Analytics covering basics through window functions and query optimisation
- [SQLZoo Interactive Exercises](https://sqlzoo.net) — hands-on SQL exercises that reinforce syntax and join patterns through real query writing

---

## Python/Scala for Data Processing

**Status:** ⬜ Not Started

**Definition:** Python and Scala are the two dominant languages for writing production data pipelines. Python dominates for its ecosystem (PySpark, pandas, dbt macros, Airflow DAGs), while Scala is the native language of Apache Spark and offers stronger compile-time type safety and better performance for complex UDF-heavy workloads.

**Key Mental Model:** Python is the Swiss Army knife — quick to write, readable, enormous library support, and the default for most DE teams. Scala is the precision instrument — stricter, faster for native Spark execution, but with a significantly steeper learning curve and slower iteration cycle.

**How It Works:**
- PySpark bridges Python and the JVM-based Spark runtime via Py4J. When you write PySpark DataFrame operations, they are serialised as logical plans and sent to the JVM Spark driver, which compiles them to Catalyst logical plans and then Tungsten physical plans — the Python code itself never runs on the Spark workers for DataFrame operations.
- Python UDFs (user-defined functions) break this model: each row is serialised from JVM memory to a Python process via pickle, processed in Python, then serialised back. This serialisation cost makes Python UDFs 10–100x slower than equivalent Spark-native operations; Pandas UDFs (Arrow-based) close much of this gap by passing batches.
- Scala UDFs run directly in the JVM alongside Spark internals — no serialisation boundary — which is why complex custom logic benefits from Scala even when the rest of the pipeline is PySpark.
- Python's GIL (Global Interpreter Lock) limits true multi-threading for CPU-bound tasks, making it unsuitable for parallelism at the process level. For data engineering this is largely irrelevant since Spark and async I/O handle concurrency outside the GIL.
- For non-Spark pipelines (ingestion scripts, API polling, dbt macros), Python is the clear choice: rich HTTP/DB driver ecosystem, fast iteration, and strong typing support via type hints and tools like mypy.

**Common Misconceptions:**
- You must know Scala to use Spark effectively — PySpark exposes nearly the full Spark API in Python, and for standard DataFrame operations there is no meaningful performance difference. Scala only wins for UDF-heavy or very low-latency streaming workloads.
- pandas is suitable for large datasets — pandas loads the entire DataFrame into a single machine's memory. On datasets beyond a few GB it will OOM or be unacceptably slow; use PySpark, Dask, or Polars instead. See [[DE-Engineer/05-Scale]] for distributed alternatives.

**Interview Answer Skeleton:**
- **What it is:** General-purpose programming languages used to write pipeline orchestration, transformations, and custom logic that SQL alone cannot express — including control flow, error handling, API calls, and reusable library code.
- **Why it matters / trade-offs:** Production pipelines need retry logic, parameterisation, unit tests, and dependency management beyond what declarative SQL provides. The trade-off is that Python code is harder to optimise at scale than native Spark operations, and UDF misuse is a common performance killer.
- **Example or context:** A PySpark job reads Parquet from S3, applies a window function to calculate 7-day rolling revenue per customer, then writes incrementally to a Delta table. The window function runs as a native Spark operation — no Python UDF, no serialisation penalty.

**Free Resources:**
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — Zach Wilson's open-source handbook covering Python pipeline patterns, Spark, and data engineering fundamentals
- [Databricks Academy](https://academy.databricks.com) — free Databricks courses covering PySpark, Delta Lake, and Spark architecture with hands-on labs

---

## Data Structures and Complexity

**Status:** ⬜ Not Started

**Definition:** Data structures are named patterns for organising data in memory — arrays, hash maps, trees, heaps, and queues each offer different trade-offs in lookup speed, insertion cost, and memory overhead. Big-O notation describes how these costs scale with input size, allowing engineers to predict performance degradation before it becomes a production incident.

**Key Mental Model:** Imagine shelving books. An array is books in order by number — instant access by position, slow to find by title. A hash map is a catalogue indexed by title — O(1) lookup regardless of collection size, but requires extra memory for the index and handles collision overhead under the hood.

**How It Works:**
- Hash maps achieve O(1) average lookup by applying a hash function to the key, mapping it to a bucket index. Collisions (two keys mapping to the same bucket) are resolved via chaining or open addressing, degrading worst-case performance to O(n) — but good hash functions make this rare in practice.
- Sorting algorithms like merge sort achieve O(n log n) because they recursively split the input in half (log n levels) and merge at each level (n work per level). This bound is also the theoretical minimum for comparison-based sorting, which is why database sort operations are bounded at O(n log n).
- Database query planners directly apply complexity reasoning: a hash join builds a hash table from the smaller relation (O(n)) then probes it with the larger relation (O(m)), giving O(n+m). A nested loop join is O(n×m) — catastrophic for large tables but optimal when the inner table has a covering index and outer cardinality is tiny.
- Columnar storage formats like Parquet exploit data locality: values in the same column are stored contiguously in memory, enabling CPU cache-friendly sequential scans and SIMD vectorised operations that dramatically outperform row-oriented scans for analytical aggregations.
- B-tree indexes (used in Postgres, MySQL) maintain sorted order for efficient range queries at O(log n) lookup, but each insert must rebalance the tree. Hash indexes give O(1) point lookups but cannot serve range queries at all — the right index type depends on the query pattern.

**Common Misconceptions:**
- Big-O is only relevant for software engineers, not data engineers — every join strategy, shuffle operation, and index choice in a data pipeline is a direct application of complexity analysis. Understanding why a Spark shuffle is expensive requires understanding O(n log n) sort-merge.
- A theoretically faster data structure is always the right choice — memory cache locality often makes a simple sorted array outperform a hash map for small inputs because sequential memory access is far faster than pointer-chased hash lookups on modern CPUs.

**Interview Answer Skeleton:**
- **What it is:** The study of how data is organised in memory and how that organisation determines the time and space cost of operations, expressed using Big-O notation to describe scaling behaviour independent of hardware.
- **Why it matters / trade-offs:** Choosing the wrong join strategy, index type, or data structure causes bottlenecks that only surface at production scale. A nested loop join that works fine on dev data can take hours on a 10B-row table that would run in minutes with a hash join.
- **Example or context:** A hash join between an orders table (100M rows) and a customers table (1M rows) builds a hash table on customers (smaller side) then probes it for each order row — O(n+m). A naive nested loop join on the same tables would be O(100M × 1M) = O(10^14) operations, completely infeasible.

**Free Resources:**
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference covering data engineering fundamentals including storage systems and algorithm trade-offs
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — open-source DE handbook with references to systems thinking and performance fundamentals

---

## Files and Formats (CSV / JSON / Parquet)

**Status:** ⬜ Not Started

**Definition:** Data is serialised to disk in either row-oriented text formats (CSV, JSON) that are human-readable but query-inefficient, or columnar binary formats (Parquet, ORC) that compress aggressively and enable fast analytical reads. Format choice directly determines storage cost, query latency, and interoperability with downstream tools.

**Key Mental Model:** CSV is a notepad — universally readable, zero encoding overhead, but you must scan the entire page to find one column. Parquet is a filing cabinet with labelled drawers: each column stored together, compressed per column type, and readable without touching the other drawers.

**How It Works:**
- Parquet uses a nested columnar layout: data is split into row groups (typically 128MB), and within each row group, values for each column are stored in a contiguous column chunk. This means a query reading 5 columns from a 200-column table physically reads roughly 2.5% of the file's bytes.
- Predicate pushdown works by reading column statistics (min/max values stored in Parquet row group footers) before reading actual data. If the filter value falls outside a row group's min/max range, the entire row group is skipped without reading it — this can eliminate 99% of I/O for selective filters on sorted or clustered data.
- Parquet uses encoding strategies per column type: dictionary encoding for low-cardinality columns (replaces repeated values with integer codes), run-length encoding for repetitive sequences, and delta encoding for monotonically increasing integers like timestamps. Combined with Snappy or ZSTD compression, this typically achieves 5–10x compression over CSV.
- JSON parsing is inherently row-oriented and schema-on-read: every record must be fully parsed to extract a single field, there is no column skip optimisation, and type inference is done at parse time rather than enforced at write time. At scale this creates significant CPU and I/O overhead versus Parquet.
- Open table formats like Delta Lake and Apache Iceberg add a transaction log on top of Parquet files, enabling ACID writes, schema evolution, time travel, and row-level deletes/updates without rewriting entire files — addressing Parquet's core weakness of immutability. See [[DE-Engineer/06-Platform]] for details.

**Common Misconceptions:**
- JSON is acceptable for large analytical datasets — without schema enforcement, column skipping, or binary encoding, JSON throughput is orders of magnitude slower than Parquet at analytics scale. It belongs at the ingestion boundary, not in the transformation layer.
- Parquet is always the best format — for streaming event pipelines where individual records arrive continuously, writing tiny Parquet files defeats the column compression benefit and creates a small-files problem. Row-oriented formats or open table format merge operations are better fits for high-frequency appends.

**Interview Answer Skeleton:**
- **What it is:** Serialisation formats that determine physical data layout on disk, affecting how much data must be read, how well it compresses, and what query engine optimisations (predicate pushdown, column pruning) can be applied.
- **Why it matters / trade-offs:** Switching from CSV to Parquet on a 1TB analytical dataset typically reduces storage by 80–90% and query time by 10–100x due to column pruning and predicate pushdown. The trade-off is that Parquet is immutable and binary, making row-level updates and direct human inspection awkward without tooling.
- **Example or context:** A query selecting 3 columns from a 200-column, 1TB Parquet dataset reads roughly 15GB of data. The same query on CSV must read all 1TB. Add a date filter and row group statistics may eliminate 95% of those 15GB, bringing actual I/O to under 1GB.

**Free Resources:**
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference covering file formats, storage layers, and columnar storage mechanics
- [Databricks Academy](https://academy.databricks.com) — free courses covering Parquet internals, Delta Lake, and storage optimisation on Databricks

---

## Databases, APIs, and Linux Basics

**Status:** ⬜ Not Started

**Definition:** Data engineers must navigate relational databases (Postgres, MySQL), REST APIs for data ingestion, and Linux command-line tooling for log inspection, file manipulation, process management, and cron scheduling. These form the connective tissue between every component of a production data stack.

**Key Mental Model:** Linux is the workshop floor, databases are organised storage rooms, and APIs are delivery windows to external suppliers. A data engineer moves material between all three constantly — and when something breaks at 2am, Linux CLI skills are what get you to the problem fast.

**How It Works:**
- REST APIs communicate over HTTP using standard verbs (GET, POST, PUT, DELETE). For data ingestion, engineers must handle pagination (cursor-based or offset-based), rate limiting (back-off with retry-after headers), authentication (OAuth2, API keys), and idempotency — ensuring re-running a failed ingestion doesn't duplicate records.
- Relational databases store data in B-tree indexed tables with ACID transaction semantics. Connection pooling (PgBouncer for Postgres) is critical for pipeline workloads: each new connection costs ~5ms and several MB of memory, so hundred-concurrent-pipeline jobs without pooling can exhaust database connection limits.
- Linux process management tools — ps, top, kill, systemctl — are essential for diagnosing stuck pipeline workers. Log inspection with journalctl, tail -f, and grep with context flags lets you trace pipeline failures without a GUI. Understanding exit codes (0 = success, non-zero = failure) is fundamental for pipeline error handling.
- cron and systemd timers are the simplest pipeline schedulers on Linux. Understanding cron expression syntax (minute/hour/day/month/weekday) and how to redirect stdout/stderr to log files is prerequisite knowledge before moving to Airflow or similar orchestrators. See [[DE-Engineer/04-Pipeline]] for orchestration.
- SSH tunnelling allows secure access to databases in private VPCs without exposing ports publicly. Port forwarding (ssh -L local_port:db_host:db_port) is a routine operational skill for debugging production database issues from a local machine.

**Common Misconceptions:**
- APIs are just for application developers — data engineers consume APIs constantly for SaaS ingestion (Salesforce, Stripe, HubSpot) and build internal APIs to expose processed datasets to downstream consumers.
- Linux proficiency is optional in cloud-first environments — even in Kubernetes-based data platforms, engineers regularly need to exec into containers, inspect logs, manage file permissions, and debug network connectivity. Cloud abstraction leaks at the edges.

**Interview Answer Skeleton:**
- **What it is:** The foundational operational layer — relational database access, HTTP API consumption, and Linux CLI tooling — that enables data engineers to ingest, inspect, debug, and manage data systems at the infrastructure level.
- **Why it matters / trade-offs:** Pipelines fail in production; Linux and database skills determine how quickly you can diagnose and recover. The trade-off is that raw Linux/API skills require more upfront investment than managed-service abstractions, but they remain essential when abstractions fail.
- **Example or context:** Ingesting a paginated REST API with rate limits requires: cursor-based pagination to avoid missed records on re-runs, exponential back-off on 429 responses, checkpoint state (last cursor position) stored in a database, and idempotent upsert logic so retries don't create duplicates.

**Free Resources:**
- [Linux Command Line Fundamentals](https://linuxcommand.org) — structured introduction to Linux shell, file operations, process management, and scripting for engineers
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — open-source DE handbook covering ingestion patterns, API handling, and foundational data engineering skills

---

## Clear Communication While Coding

**Status:** ⬜ Not Started

**Definition:** In technical interviews, narrating your approach as you code demonstrates thought process visibility, catches assumptions early, and signals collaboration fitness. This means voicing your reasoning about scope, trade-offs, and edge cases continuously rather than coding in silence and presenting a finished solution.

**Key Mental Model:** Treat the interviewer as a pair-programming partner. You are not performing a solo magic trick — you are solving a problem together, and your running commentary on reasoning is as important as the code itself. Silence forces the interviewer to guess what you're thinking.

**How It Works:**
- Begin by restating the problem in your own words and explicitly stating your assumptions — this surfaces misunderstandings before you spend time solving the wrong problem. Asking one clarifying question about scale or constraints signals systems thinking.
- Before writing code, briefly narrate your approach: "I'll start with a brute-force O(n²) solution to establish correctness, then optimise" — this gives the interviewer an opportunity to redirect you if your direction is wrong, saving both parties time.
- While coding, explain each non-obvious decision in real time: "I'm using a hash map here for O(1) lookup instead of scanning the list each time." This makes your complexity reasoning visible rather than implicit.
- When you encounter a bug or unexpected result, narrate the debugging process: "This isn't returning what I expected — let me trace through the loop manually with a small example." This demonstrates methodical debugging, not just lucky guessing.
- Conclude by narrating edge cases you know are unhandled: "This breaks on empty input and negative values — in production I'd add guard clauses here." Proactively identifying limitations scores higher than ignoring them.

**Common Misconceptions:**
- Silence signals deep concentration — interviewers interpret sustained silence as an inability to communicate or explain reasoning, regardless of code quality. Talking through a partial, imperfect approach is almost always evaluated more positively than silent perfection.
- You must have the complete solution before speaking — articulating a wrong or incomplete approach and then correcting it aloud demonstrates exactly the iterative problem-solving mindset senior engineers use in practice.

**Interview Answer Skeleton:**
- **What it is:** The practice of continuously externalising reasoning, assumptions, and trade-offs while writing code — turning the interview from a silent performance into a collaborative technical conversation.
- **Why it matters / trade-offs:** Interviewers evaluate communication and collaboration fitness as heavily as technical correctness, especially at senior levels where ambiguous problems are the norm. The only downside is that talking can slow initial typing speed, which is vastly outweighed by the signal it provides.
- **Example or context:** "I'm assuming the dataset fits in a single machine's memory — if it doesn't I'd move to a Spark-based approach. Let me start with the pandas solution for simplicity and we can scale it up if needed. I'll also need to handle nulls here because groupby in pandas silently drops null keys by default."

**Free Resources:**
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — open-source handbook covering interview preparation, system design, and communication frameworks for data engineering roles
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference with interview guides and behavioural frameworks for data engineering interviews

---
