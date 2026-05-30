# Layer 2 — SQL

> **Framework:** Advanced query patterns and optimisation for production data engineering.

---

## Window Functions, CTEs, and Subqueries

**Status:** ⬜ Not Started

**Definition:** Window functions compute aggregates across a sliding "window" of rows related to the current row without collapsing the result set. CTEs (Common Table Expressions) are named subqueries defined with `WITH` that improve readability and allow reuse within a query. Subqueries are queries nested inside other queries.

**Mental Model:** A window function is like a moving average on a chart — each row gets its own calculation based on neighbours, but all rows remain visible. A CTE is a named scratchpad where you prepare intermediate results before the main query.

**Common Misconceptions:**
- Window functions replace GROUP BY — they don't; GROUP BY collapses rows into one per group, window functions preserve all rows.
- CTEs are always materialised as temp tables — in most databases, CTEs are syntax sugar and are not cached unless the database explicitly decides to materialise them.

**Interview Skeleton:**
- What it is: tools for row-level computation across partitions and a pattern for decomposing complex queries
- Why it matters: session analysis, running totals, and rank-based filtering all require window functions; CTEs make complex logic maintainable
- Example: write a query using ROW_NUMBER() to deduplicate, RANK() for top-N per group, wrapped in a CTE for readability

**Free Resources:** https://mode.com/sql-tutorial/sql-window-functions/ — Mode Analytics interactive window function tutorial with live examples

---

## Deduplication, Sessionization, and Top-N

**Status:** ⬜ Not Started

**Definition:** Deduplication removes duplicate rows by keeping one record per logical entity using ROW_NUMBER() or DISTINCT. Sessionization groups user events into sessions based on inactivity gaps. Top-N queries return the N highest-ranked rows per group using RANK() or ROW_NUMBER() in a subquery.

**Mental Model:** Deduplication is like removing duplicate entries from a guest list — keep the most recent or most complete record. Sessionization is grouping phone calls by conversation — a new session starts when there's been a long enough silence.

**Common Misconceptions:**
- DISTINCT is always the right deduplication tool — DISTINCT deduplicates entire rows; ROW_NUMBER() lets you choose which duplicate to keep based on a specific ordering.
- Sessionization requires complex procedural code — a LAG() window function combined with a conditional SUM() is the standard clean SQL pattern.

**Interview Skeleton:**
- What it is: patterns for cleaning and structuring event-level data into meaningful analytical units
- Why it matters: raw data almost always has duplicates; session analysis is a core product analytics requirement
- Example: given a click events table, write SQL to sessionize events with a 30-minute inactivity timeout using LAG and SUM

**Free Resources:** https://www.sisense.com/blog/sql-window-functions/ — Practical guide to sessionization and ranking patterns using SQL window functions

---

## Query Optimisation and Index Awareness

**Status:** ⬜ Not Started

**Definition:** Query optimisation is the process of rewriting or structuring SQL so the database engine executes it efficiently. Index awareness means understanding when a column index will be used (equality filters, range scans) vs. ignored (functions applied to indexed columns, low-cardinality scans).

**Mental Model:** An index is like a book's index — if you're looking for "partitioning" you jump to page 47 instead of reading every page. But searching for "words containing 'part'" cannot use the index.

**Common Misconceptions:**
- Adding more indexes is always better — indexes slow down writes and increase storage; only index columns that appear frequently in WHERE, JOIN, and ORDER BY clauses.
- EXPLAIN output is only for DBAs — data engineers should read EXPLAIN/EXPLAIN ANALYZE to understand scan types, row estimates, and join strategies for their pipeline queries.

**Interview Skeleton:**
- What it is: techniques to reduce the I/O, memory, and CPU a SQL query consumes
- Why it matters: a slow query in a pipeline causes SLA breaches and cascading failures downstream
- Example: show how applying a function to an indexed column defeats the index, and rewrite it to preserve the index

**Free Resources:** https://use-the-index-luke.com/ — Free book entirely dedicated to SQL indexing and query performance

---

## NULL Handling and Data Cleaning

**Status:** ⬜ Not Started

**Definition:** NULL in SQL represents an unknown or missing value with special behaviour: NULL is not equal to NULL, arithmetic with NULL returns NULL, and NULL propagates through most expressions. Data cleaning handles NULLs explicitly using COALESCE, NULLIF, IS NULL, and CASE WHEN.

**Mental Model:** NULL is the "unknown" — not zero, not empty string, not false. It's the absence of information. Treating it like a value causes silent bugs that are extremely hard to trace in production.

**Common Misconceptions:**
- `WHERE column != 'X'` excludes NULLs — `NULL != 'X'` evaluates to NULL (unknown), so NULL rows are silently excluded; you need an explicit `OR column IS NULL` clause.
- COALESCE and ISNULL are equivalent — COALESCE is ANSI standard and takes multiple arguments; ISNULL is SQL Server-specific and takes exactly two arguments.

**Interview Skeleton:**
- What it is: special SQL semantics for missing or unknown values that don't behave like normal values in comparisons or arithmetic
- Why it matters: NULL bugs in pipelines cause incorrect aggregations and silently wrong dashboards
- Example: demonstrate COUNT(*) vs COUNT(column) difference, and explain how LEFT JOIN NULLs interact with WHERE filters

**Free Resources:** https://sqlzoo.net/ — SQL Zoo interactive exercises covering NULL handling, joins, and data cleaning

---

## Aggregations, Date Logic, and Ranking

**Status:** ⬜ Not Started

**Definition:** Aggregations collapse multiple rows into summary values using SUM, COUNT, AVG, MIN, MAX with GROUP BY. Date logic involves arithmetic on timestamps (truncation, intervals, time zones). Ranking assigns ordinal positions within groups using RANK(), DENSE_RANK(), and ROW_NUMBER().

**Mental Model:** Aggregation is compressing many rows into one summary. Date arithmetic is a calculator for calendars. Ranking is a leaderboard — who's first in each category, with rules for how to handle ties.

**Common Misconceptions:**
- COUNT(1) is faster than COUNT(*) — most modern databases optimise these identically; COUNT(column) is the meaningful difference as it excludes NULLs.
- Date truncation is only for display formatting — DATE_TRUNC is critical for bucketing events by hour, day, or month in time-series aggregations.

**Interview Skeleton:**
- What it is: SQL functions for summarising data, working with time, and ordering within partitioned groups
- Why it matters: most business metrics involve time-series aggregations and ranked comparisons by dimension
- Example: write a query for daily active users by country over the last 30 days with a 7-day rolling average

**Free Resources:** https://www.postgresqltutorial.com/postgresql-aggregate-functions/ — PostgreSQL aggregate and date function reference with examples

---

## Explain Trade-offs, Not Just Syntax

**Status:** ⬜ Not Started

**Definition:** Strong SQL practitioners don't just know how to write a query — they can articulate why one approach is better than another (performance, readability, correctness), when to choose CTEs vs temp tables vs subqueries, and how different databases handle the same pattern differently.

**Mental Model:** Anyone can follow a recipe. A good engineer explains why the recipe calls for low heat, what goes wrong at high heat, and how the recipe changes at altitude.

**Common Misconceptions:**
- There's one right way to write any SQL query — multiple patterns may be logically equivalent; the right choice depends on database engine, data volume, and maintainability.
- Trade-off discussions are only expected from senior engineers — entry-level candidates who articulate trade-offs stand out immediately in interviews.

**Interview Skeleton:**
- What it is: the habit of explaining the reasoning and trade-offs behind SQL choices, not just the mechanics
- Why it matters: interviewers hire for the thinking behind the code, not just the code itself
- Example: when asked to find duplicates, explain ROW_NUMBER() vs GROUP BY + HAVING — when you'd choose each and why

**Free Resources:** https://dataschool.com/how-to-teach-people-sql/ — Articles on SQL thinking patterns, trade-off reasoning, and common mistakes
