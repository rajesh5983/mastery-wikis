# Layer 1 — Foundation

> **Framework:** Core programming and tooling fluency that underpins all data engineering work.

---

## SQL Basics

**Status:** ⬜ Not Started

**Definition:** SQL (Structured Query Language) is the standard language for reading and manipulating data in relational databases. It covers SELECT, filtering, joins, grouping, and aggregation — the building blocks every data engineer uses daily regardless of the platform they work on.

**Mental Model:** A database is a spreadsheet factory. SQL is the set of instructions you give it to pull exactly the rows and columns you need, filter them, combine tables, and summarise results.

**Common Misconceptions:**
- SQL is "just for analysts" — data engineers write complex SQL constantly for pipelines, data quality checks, and transformations.
- JOINs always have one correct order — logically equivalent in most cases, but join order affects query plan performance significantly.

**Interview Skeleton:**
- What it is: a declarative language for querying relational data, executed by a query planner
- Why it matters: universal interface across dbt, Spark SQL, BigQuery, Snowflake, Redshift
- Example: write a query joining orders and customers, group by region with a HAVING filter, explain the execution order

**Free Resources:** https://mode.com/sql-tutorial/ — Interactive SQL tutorials from basics through advanced queries

---

## Python/Scala for Data Processing

**Status:** ⬜ Not Started

**Definition:** Python and Scala are the two dominant languages for writing data pipelines. Python is favoured for its ecosystem (pandas, PySpark, dbt macros), while Scala is the native language of Apache Spark and offers stronger type safety and performance advantages at scale.

**Mental Model:** Python is the Swiss Army knife — quick, readable, enormous library support. Scala is the surgical instrument — stricter, faster at scale, but with a steeper learning curve.

**Common Misconceptions:**
- You must know Scala to use Spark — PySpark exposes full Spark capability in Python, though native Scala UDFs can be faster for complex logic.
- pandas is suitable for big data — pandas loads everything into memory; it breaks on datasets larger than available RAM.

**Interview Skeleton:**
- What it is: general-purpose languages adapted for data transformation and pipeline logic
- Why it matters: production pipelines need real control flow, testing, and error handling beyond what SQL provides
- Example: describe a PySpark job that reads Parquet, applies window functions, and writes incrementally back to the lake

**Free Resources:** https://realpython.com/python-data-engineer/ — Real Python guide to Python patterns used in data engineering

---

## Data Structures and Complexity

**Status:** ⬜ Not Started

**Definition:** Data structures are ways of organising data in memory (arrays, hash maps, trees, queues). Complexity (Big-O notation) describes how performance scales with input size — O(1) constant, O(n) linear, O(n log n) common for sorting algorithms.

**Mental Model:** Imagine shelving books. An array is books in order by number — fast by position, slow to search by title. A hash map is a catalogue by title — instant lookup, more memory cost.

**Common Misconceptions:**
- Big-O is only for software engineers — understanding hash joins vs nested loop joins in a query planner requires the same complexity intuition.
- The most complex data structure is always best — simpler structures with better cache locality often outperform theoretically faster ones at real data sizes.

**Interview Skeleton:**
- What it is: how data is organised in memory and how that affects speed and space trade-offs
- Why it matters: choosing the wrong join strategy or data layout causes bottlenecks that only appear at scale
- Example: explain why a hash join is O(n) vs a nested loop join at O(n²) for two large tables

**Free Resources:** https://www.bigocheatsheet.com/ — Visual cheat sheet of common data structure and algorithm complexities

---

## Files and Formats (CSV / JSON / Parquet)

**Status:** ⬜ Not Started

**Definition:** Data can be stored in text formats (CSV, JSON) that are human-readable but slow to query, or columnar binary formats (Parquet, ORC) that compress well and enable fast analytical reads. Choosing the right format affects storage cost and query speed dramatically.

**Mental Model:** CSV is a notepad — universal, readable by anything, but unstructured. Parquet is a filing cabinet with labelled drawers — each column stored together, compressed, readable without opening the other drawers.

**Common Misconceptions:**
- JSON is fine for large datasets — JSON has no schema enforcement, high storage overhead, and slow parse times at scale.
- Parquet is always the right choice — for streaming, small files, or row-level updates, row-oriented or open table formats (Delta, Iceberg) are better fits.

**Interview Skeleton:**
- What it is: serialisation formats that determine how data is stored on disk and read back into memory
- Why it matters: columnar formats reduce I/O by 10–100x for analytical queries that scan few columns
- Example: compare reading 10 columns from a 1TB CSV vs Parquet — explain predicate pushdown and column pruning

**Free Resources:** https://parquet.apache.org/docs/ — Apache Parquet official documentation explaining the columnar storage format

---

## Databases, APIs, and Linux Basics

**Status:** ⬜ Not Started

**Definition:** Data engineers must navigate relational databases (Postgres, MySQL), REST APIs for ingestion, and Linux command-line tools for log inspection, file manipulation, and process management. These form the connective tissue of every data stack.

**Mental Model:** Linux is the workshop floor, databases are organised storage rooms, and APIs are delivery windows to external suppliers. A data engineer moves material between all three constantly.

**Common Misconceptions:**
- APIs are just for application developers — data engineers consume APIs constantly for ingestion and expose them for downstream consumers.
- Linux knowledge is optional in cloud-first environments — container debugging, SSH access, cron jobs, and log tailing all require Linux literacy.

**Interview Skeleton:**
- What it is: foundational tooling for accessing, moving, and inspecting data at the infrastructure level
- Why it matters: pipelines fail at 2am; you need to SSH in, tail logs, query databases, and restart processes
- Example: describe how you'd ingest a paginated REST API into a warehouse, handling rate limits and resumable state

**Free Resources:** https://missing.csail.mit.edu/ — MIT "Missing Semester" covering Linux, shell, and the tools every engineer needs

---

## Clear Communication While Coding

**Status:** ⬜ Not Started

**Definition:** In technical interviews, talking through your approach as you code demonstrates thought process, catches errors early, and shows collaboration skills. This means narrating assumptions, trade-offs, and edge cases aloud rather than coding in silence.

**Mental Model:** Treat the interviewer as a pair-programming partner. You're not performing a solo magic trick — you're solving a problem together with running commentary on your reasoning.

**Common Misconceptions:**
- Silence signals deep focus — interviewers interpret silence as inability to communicate, even when the candidate is thinking hard.
- You must have the full solution before speaking — talking through a partial or imperfect approach is more valuable than silent perfection.

**Interview Skeleton:**
- What it is: the practice of narrating reasoning, assumptions, and trade-offs while writing code
- Why it matters: interviewers evaluate communication and collaboration as heavily as technical correctness
- Example: "I'm assuming the dataset fits in memory — if it doesn't I'd use a streaming approach; let me start with the simple case and we can scale it up"

**Free Resources:** https://www.techinterviewhandbook.org/coding-interview-cheatsheet/ — Practical checklist for communicating clearly in technical coding interviews
