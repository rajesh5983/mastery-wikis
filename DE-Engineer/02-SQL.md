# Layer 2 — SQL

> **Framework:** Advanced query patterns and optimisation for production data engineering.

---

## Window Functions, CTEs, and Subqueries

**Status:** ⬜ Not Started

**Definition:** Window functions compute aggregates or rankings across a defined set of rows related to the current row without collapsing the result set — each input row produces exactly one output row. CTEs (Common Table Expressions) are named subqueries declared with `WITH` that decompose complex queries into readable, reusable named blocks. Subqueries are inline queries nested inside another query's FROM, WHERE, or SELECT clause.

**Key Mental Model:** A window function is like a moving average chart — each data point gets its own calculation based on its neighbours, but every point remains visible in the output. A CTE is a named whiteboard where you prepare an intermediate result before referencing it in the main query.

**How It Works:**
- Window functions execute in a specific phase of query processing: after WHERE, GROUP BY, and HAVING, but before the final SELECT projection. This means you can window over the post-GROUP BY aggregated rows, but you cannot use a window function result in a WHERE clause without wrapping it in a subquery or CTE.
- The OVER() clause defines the window with two components: PARTITION BY (which divides rows into independent groups, like GROUP BY but without collapsing) and ORDER BY (which determines row order within each partition for functions like ROW_NUMBER, RANK, LAG, and LEAD).
- RANK() and DENSE_RANK() differ in how they handle ties: RANK() leaves gaps (1, 1, 3) while DENSE_RANK() does not (1, 1, 2). ROW_NUMBER() assigns unique sequential integers regardless of ties, making it the tool for deduplication where you need exactly one winner.
- Frame clauses (ROWS BETWEEN / RANGE BETWEEN) define which surrounding rows are included in the calculation. RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW is the default for most aggregate window functions and produces a cumulative sum. ROWS BETWEEN 6 PRECEDING AND CURRENT ROW produces a true 7-row rolling window.
- In most major databases (Postgres, BigQuery, Snowflake, Spark SQL), CTEs are inlined by the query planner and not physically materialised as temp tables unless explicitly forced or the planner chooses to materialise for cost reasons. In SQL Server, CTEs are always inlined; Postgres 12+ changed from always-materialised to inline-by-default.

**Common Misconceptions:**
- Window functions replace GROUP BY — they serve different purposes. GROUP BY collapses many rows into one summary row per group; window functions compute aggregates while preserving every individual row, enabling patterns like "show each sale alongside the total for its region."
- CTEs are always materialised temp tables and therefore faster for reuse — in most engines, a CTE referenced multiple times in the same query is re-evaluated each time. For genuine caching and reuse, a temporary table or persisted intermediate model is needed.

**Interview Answer Skeleton:**
- **What it is:** Window functions extend SQL aggregation to operate over ordered partitions of rows without collapsing them, enabling running totals, rankings, and lead/lag comparisons. CTEs are named result blocks that make complex multi-step logic readable and maintainable.
- **Why it matters / trade-offs:** Session analysis, deduplication, top-N-per-group, and cumulative metrics all require window functions; they replace complex self-joins or application-side logic. The trade-off is that window functions can be expensive on large datasets without proper partitioning — a window over all rows with no PARTITION BY forces a full sort.
- **Example or context:** Deduplicating a table by keeping the most recently updated record per customer: wrap ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) in a CTE, then filter WHERE rn = 1 in the outer query. This pattern is idiomatic in dbt incremental models. See [[DE-Engineer/03-Data-Modeling]] for how deduplication fits into model design.

**Free Resources:**
- [Mode SQL Tutorial](https://mode.com/sql-tutorial) — Mode Analytics SQL course covering window functions with interactive live query examples
- [SQLZoo Interactive Exercises](https://sqlzoo.net) — hands-on exercises for practising window function and CTE patterns on real datasets

---

## Deduplication, Sessionization, and Top-N

**Status:** ⬜ Not Started

**Definition:** Deduplication removes redundant copies of logical records, keeping one canonical row per entity based on a defined ordering rule. Sessionization groups a sequence of user events into discrete sessions separated by inactivity gaps. Top-N queries return exactly N rows per partition ranked by a specified metric, a fundamental pattern in product analytics.

**Key Mental Model:** Deduplication is pruning a guest list — keep the most recent or most complete entry, discard the others. Sessionization is grouping phone calls by conversation — a new session starts after a silence long enough to signal the previous conversation ended.

**How It Works:**
- The standard deduplication pattern uses ROW_NUMBER() OVER (PARTITION BY entity_key ORDER BY recency_column DESC) to assign rank 1 to the "best" duplicate per entity. Wrapping this in a CTE and filtering WHERE row_num = 1 in the outer query produces a clean deduplicated table. This pattern is deterministic: the ordering column determines which duplicate survives.
- DISTINCT operates on full row equality — it cannot selectively keep one duplicate over another based on a column value. It is only suitable when all copies of a duplicate are truly identical and you don't care which one survives.
- Sessionization uses LAG() to compare each event's timestamp with the previous event for the same user: if the gap exceeds the session timeout (e.g. 30 minutes), the event starts a new session. A SUM() window function over the boolean gap indicator then assigns a cumulative session ID to each event — this is the idiomatic pure-SQL sessionization pattern without procedural loops.
- Top-N per group (e.g. top 3 products by revenue per country) is typically implemented with RANK() or DENSE_RANK() OVER (PARTITION BY country ORDER BY revenue DESC), wrapped in a subquery, then filtered WHERE rank <= 3. Using LIMIT without a window function only gives global top-N, not per-group.
- For sessionization in Spark, the same LAG + cumulative SUM approach applies in Spark SQL, but very large event streams may benefit from using the Window API in PySpark with explicit partitioning and sorting to control shuffle cost. See [[DE-Engineer/05-Scale]] for Spark window optimisation.

**Common Misconceptions:**
- DISTINCT is the standard deduplication tool — DISTINCT removes rows only when every column matches. In practice, duplicate records differ in at least one column (timestamps, load metadata), so DISTINCT fails silently and ROW_NUMBER() is almost always the correct approach.
- Sessionization requires custom application code or procedural SQL — a LAG() + conditional SUM() pattern handles standard session boundaries cleanly in pure SQL across Postgres, BigQuery, Snowflake, and Spark SQL without procedural logic.

**Interview Answer Skeleton:**
- **What it is:** SQL patterns that transform raw event data into clean analytical units — deduplication removes redundant records, sessionization groups events into meaningful user interactions, and Top-N extracts ranked subsets per dimension.
- **Why it matters / trade-offs:** Raw event data from production systems is almost always dirty with duplicates and needs to be sessionized before product metrics are meaningful. These patterns appear in nearly every DE pipeline. The trade-off is that window-function-heavy queries on large, unpartitioned event tables can be extremely expensive — partitioning by user_id or date is essential.
- **Example or context:** Given a click events table, sessionize events with a 30-minute timeout: compute time_since_last_event using LAG(), flag rows where the gap exceeds 1800 seconds as new_session = 1, then use SUM(new_session) OVER (PARTITION BY user_id ORDER BY event_time) as a cumulative session counter. Each user ends up with session IDs 1, 1, 1, 2, 2, 3 etc.

**Free Resources:**
- [Mode SQL Tutorial](https://mode.com/sql-tutorial) — covers advanced SQL patterns including sessionization and ranking with worked examples
- [SQLZoo Interactive Exercises](https://sqlzoo.net) — practice problems for deduplication and ranking patterns on real tabular data

---

## Query Optimisation and Index Awareness

**Status:** ⬜ Not Started

**Definition:** Query optimisation is the practice of writing and structuring SQL so the database engine executes it with minimal I/O, CPU, and memory. Index awareness means understanding when the query planner will use an index — and critically, when transformations or filter patterns silently prevent index use.

**Key Mental Model:** An index is like a book's back-of-index — if you want "partitioning" you jump directly to page 47. But looking for "words containing 'part'" cannot use the alphabetical index; you must read every page. Applying a function to an indexed column has exactly the same effect.

**How It Works:**
- The query planner uses table statistics (row counts, column cardinality, value histograms) to estimate the cost of alternative execution plans. ANALYZE (Postgres) or equivalent statistics refresh commands keep these estimates accurate — stale statistics cause the planner to choose wrong join strategies or miss useful indexes.
- Index scans are used when the filter is selective enough that reading the index and fetching matching rows is cheaper than a full table scan. For very low selectivity (e.g., filtering on a boolean column), the planner may prefer a full sequential scan because random I/O to fetch individual rows costs more than sequential scan of the whole table.
- Applying a function to an indexed column defeats the index: `WHERE YEAR(created_at) = 2024` cannot use an index on created_at. The rewrite `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'` preserves the index seek. The same principle applies to `LOWER(email) = 'user@example.com'` vs a functional index on `LOWER(email)`.
- Implicit type coercion also defeats indexes: comparing a VARCHAR column to an integer literal forces a cast on every row, making the index unusable. Always match the literal type to the column type in WHERE clauses.
- EXPLAIN (Postgres) / EXPLAIN ANALYZE shows the physical execution plan with estimated vs actual row counts, scan types, join strategies, and cost estimates. Actual row counts diverging significantly from estimates signal stale statistics. Node types to watch: Seq Scan on large tables, Nested Loop with large outer cardinality, and Sort with high memory usage indicate optimisation opportunities.

**Common Misconceptions:**
- Adding more indexes improves query performance universally — indexes impose write amplification: every INSERT, UPDATE, and DELETE must update all relevant indexes. Over-indexing a frequently written table (e.g., an event log) can make writes dramatically slower and increase storage costs significantly.
- EXPLAIN output is only relevant for DBAs — data engineers who write complex transformation queries should routinely inspect EXPLAIN ANALYZE output for their pipeline queries. A query that runs fine on dev data can produce a completely different execution plan on production at 100x the volume.

**Interview Answer Skeleton:**
- **What it is:** The discipline of structuring SQL queries and database schemas so the query planner can choose efficient physical execution plans — primarily through filter sargability (index-compatible predicates), join ordering, and statistics accuracy.
- **Why it matters / trade-offs:** A slow transformation query in a daily pipeline creates SLA breaches and can cascade to block downstream models. The optimisation trade-off is always write cost vs read cost: indexes accelerate reads but slow writes, so the right index strategy depends on read/write ratio.
- **Example or context:** A pipeline query using `WHERE DATE(event_timestamp) = '2024-01-15'` is doing a full scan because DATE() is applied to an indexed column. Rewriting to `WHERE event_timestamp >= '2024-01-15' AND event_timestamp < '2024-01-16'` makes the predicate sargable, enabling an index range scan that reads a fraction of the table.

**Free Resources:**
- [Mode SQL Tutorial](https://mode.com/sql-tutorial) — includes sections on query performance, execution plans, and optimisation patterns
- [W3Schools SQL Reference](https://www.w3schools.com/sql) — SQL reference covering index types, query structure, and database-specific optimisation hints

---

## NULL Handling and Data Cleaning

**Status:** ⬜ Not Started

**Definition:** NULL in SQL is a three-valued logic marker representing an unknown or absent value — it is not zero, empty string, or false. NULL propagates through almost all SQL expressions: any arithmetic, comparison, or string operation involving NULL returns NULL, which silently filters or miscounts rows in ways that are hard to trace in production pipelines.

**Key Mental Model:** NULL is the "unknown" card in a deck — not a blank card, not a zero card, but an unreadable one. Asking "is this unknown card equal to the Ace of Spades?" correctly returns "unknown" (NULL), not false — which is why `WHERE col != 'X'` silently drops NULL rows rather than including them.

**How It Works:**
- SQL uses three-valued logic: TRUE, FALSE, and UNKNOWN (the result of any comparison with NULL). `NULL = NULL` evaluates to UNKNOWN, not TRUE — this is why the equality operator cannot detect NULL values and IS NULL / IS NOT NULL are required instead.
- NULL propagation in aggregations is counterintuitive: COUNT(*) counts all rows, COUNT(column) counts only non-NULL values, and SUM/AVG/MIN/MAX silently ignore NULLs rather than returning NULL for the group. This means AVG(column) returns the average of present values only, potentially misrepresenting sparsely populated columns.
- LEFT JOINs introduce NULLs for every column in the right-side table when no match is found. A common pipeline bug is `WHERE right_table.column = 'value'` after a LEFT JOIN — this filters out unmatched rows (NULL = 'value' is UNKNOWN), effectively converting the LEFT JOIN to an INNER JOIN. To keep unmatched rows, the filter must include `OR right_table.column IS NULL`.
- COALESCE(a, b, c) returns the first non-NULL value in its argument list — it is the standard ANSI SQL approach to NULL substitution. NULLIF(a, b) is the inverse: it returns NULL if a equals b, useful for suppressing sentinel values (e.g., NULLIF(revenue, 0) to treat zero as unknown).
- NULL in ORDER BY: Postgres defaults to NULLS LAST for ASC sorts; MySQL and SQL Server differ. Explicit NULLS FIRST / NULLS LAST is the portable approach for deterministic ordering when NULL presence is expected.

**Common Misconceptions:**
- `WHERE column != 'X'` excludes the value 'X' and keeps everything else — it also silently excludes NULL rows because `NULL != 'X'` evaluates to UNKNOWN, not TRUE. This is one of the most common sources of incorrect pipeline output: rows with missing dimension values are silently dropped from aggregations.
- COALESCE and ISNULL are interchangeable — COALESCE is ANSI standard, accepts any number of arguments, and works across all databases. ISNULL is SQL Server and MySQL specific and takes exactly two arguments. Using ISNULL in dbt models creates portability issues when the target database changes.

**Interview Answer Skeleton:**
- **What it is:** A special three-valued logic marker in SQL representing unknown or absent data, which propagates through expressions and requires explicit handling with IS NULL, COALESCE, and NULLIF rather than equality comparisons.
- **Why it matters / trade-offs:** NULL bugs are among the most common sources of silently incorrect pipeline output — wrong row counts, wrong averages, and wrong join results that pass basic validation checks. The trade-off is that defensive NULL handling adds verbosity; the pragmatic approach is to be explicit about NULL treatment at ingestion and document column-level nullable contracts.
- **Example or context:** COUNT(*) = 1000, COUNT(revenue) = 800 — this gap means 200 rows have NULL revenue. An AVG(revenue) on this table returns the average of the 800 present values, not the average over all 1000 records. Whether this is correct depends on business context, but it must be a deliberate choice, not an accident.

**Free Resources:**
- [SQLZoo Interactive Exercises](https://sqlzoo.net) — includes dedicated NULL handling exercises with immediate feedback on query results
- [W3Schools SQL Reference](https://www.w3schools.com/sql) — SQL reference covering NULL functions, IS NULL syntax, and COALESCE/NULLIF with examples

---

## Aggregations, Date Logic, and Ranking

**Status:** ⬜ Not Started

**Definition:** Aggregations collapse multiple rows into summary values using SUM, COUNT, AVG, MIN, MAX grouped by one or more dimensions. Date logic involves truncation, arithmetic, and timezone conversion on timestamp columns. Ranking assigns ordinal positions within groups, distinguishing between RANK (with gaps for ties), DENSE_RANK (without gaps), and ROW_NUMBER (unique sequential integers).

**Key Mental Model:** Aggregation compresses many rows into one summary metric per group. Date arithmetic is calendar mathematics — truncating to month, adding intervals, converting timezones. Ranking is building a leaderboard with explicit rules for how tied scores are handled.

**How It Works:**
- DATE_TRUNC(unit, timestamp) is the foundation of time-series aggregation: it floors a timestamp to the start of the specified period (hour, day, week, month, year), enabling GROUP BY DATE_TRUNC('day', event_at) to bucket events by calendar day. The result is always a timestamp, not a date, which matters for timezone-aware grouping.
- Timezone handling is a frequent source of pipeline bugs: storing all timestamps in UTC and converting only at the reporting layer is the standard convention. Converting in the GROUP BY (AT TIME ZONE 'America/New_York') changes which calendar day an event falls into — an event at 11pm UTC on the 1st becomes 6pm EST on the 1st or 1pm PST on the 1st, shifting daily counts.
- FILTER (WHERE condition) on aggregates (standard in Postgres, BigQuery, Snowflake) allows multiple conditional aggregations in a single pass: SUM(revenue) FILTER (WHERE channel = 'organic') alongside SUM(revenue) FILTER (WHERE channel = 'paid') without a self-join or CASE WHEN pivot.
- RANK() vs DENSE_RANK() vs ROW_NUMBER(): RANK leaves sequence gaps after ties (scores 100, 100, 80 → ranks 1, 1, 3); DENSE_RANK does not (ranks 1, 1, 2); ROW_NUMBER assigns arbitrary tie-breaking order (ranks 1, 2, 3). For deduplication, always use ROW_NUMBER to guarantee uniqueness. For top-N-per-group where ties should both appear, use RANK or DENSE_RANK.
- GROUPING SETS, ROLLUP, and CUBE are extensions that compute multiple GROUP BY combinations in a single query pass, avoiding repeated table scans. ROLLUP(country, city) generates totals at city level, country level, and grand total in one pass — essential for hierarchical reporting without UNION ALL chains.

**Common Misconceptions:**
- COUNT(1) is faster than COUNT(*) — all major databases optimise COUNT(1) and COUNT(*) identically. The meaningful distinction is COUNT(column) vs COUNT(*): COUNT(column) excludes NULL values, COUNT(*) counts every row, producing different results on nullable columns.
- DATE_TRUNC is only for display formatting and can be safely added later — date truncation decisions affect which rows fall into each bucket and therefore whether data is correct. Applying DATE_TRUNC in the wrong timezone or at the wrong granularity in a pipeline creates permanently wrong historical data.

**Interview Answer Skeleton:**
- **What it is:** Core SQL functions for compressing event data into business metrics by dimension and time period, combined with ranking functions that assign ordinal positions within groups — the building blocks of virtually every analytical pipeline output.
- **Why it matters / trade-offs:** Most business KPIs are time-series aggregations with dimensional breakdowns — daily active users by country, weekly revenue by product, rolling 7-day averages. Getting timezone handling and NULL aggregation semantics wrong produces silently incorrect dashboards. The trade-off is that GROUPING SETS reduce scan cost but increase query complexity.
- **Example or context:** Daily active users by country for the last 30 days with a 7-day rolling average: GROUP BY DATE_TRUNC('day', event_at), country for the base DAU count, then wrap in a window with AVG() OVER (PARTITION BY country ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) for the rolling average — all in UTC to avoid timezone boundary shifts.

**Free Resources:**
- [Mode SQL Tutorial](https://mode.com/sql-tutorial) — covers aggregate functions, date functions, and window-based ranking with interactive exercises
- [W3Schools SQL Reference](https://www.w3schools.com/sql) — reference for SQL date functions, GROUP BY behaviour, and aggregate function semantics across database variants

---

## Explain Trade-offs, Not Just Syntax

**Status:** ⬜ Not Started

**Definition:** Strong SQL practitioners articulate why one query pattern is preferable to another — in terms of performance, correctness, or maintainability — rather than simply producing working syntax. This means knowing when to use CTEs vs temp tables, when window functions outperform self-joins, and how the same SQL construct behaves differently across database engines.

**Key Mental Model:** Anyone can follow a recipe. A skilled engineer explains why the recipe calls for low heat, what goes wrong at high heat, and how the recipe changes at altitude. In SQL terms: not just "here is a working query" but "here is why I chose this join strategy over that one, and when the other approach is better."

**How It Works:**
- The decision between a CTE and a temp table comes down to materialisation and reuse: a CTE is inlined and re-evaluated each time it's referenced in most databases; a temp table is physically materialised once and reused. For expensive intermediate results referenced multiple times, a temp table reduces redundant computation at the cost of explicit materialisation overhead.
- Window functions vs self-joins: computing "revenue as a percentage of total revenue per region" can be done with a self-join (join the table to an aggregated subquery) or a window function (SUM() OVER PARTITION BY region). The window function approach reads the table once; the self-join reads it twice — a meaningful difference at scale.
- Correlated subqueries execute once per outer row — for large outer tables this creates O(n) subquery executions that become extremely slow. Rewriting as a JOIN or a window function converts this to a single pass. Correlated subqueries are a common SQL anti-pattern in production pipelines.
- Database portability trade-offs: QUALIFY (Snowflake, BigQuery) allows filtering on window function results without a CTE wrapper; this is unavailable in Postgres. Lateral joins (Postgres) enable correlated subqueries that reference outer table columns — not available in all databases. Writing portable SQL across engines requires understanding which features are standard vs engine-specific.
- Readability is a legitimate technical trade-off: a CTE chain that breaks a complex transformation into 8 named steps is easier to test, debug, and review than a single deeply nested subquery — even if the execution plan is identical. In dbt-based pipelines, CTE style is the community standard specifically for testability. See [[DE-Engineer/04-Pipeline]] for dbt patterns.

**Common Misconceptions:**
- There is one correct way to write any SQL query — logically equivalent queries can have radically different performance profiles depending on data volume, cardinality, available indexes, and database engine. The "correct" approach depends on context, not universal rules.
- Trade-off discussions are expected only from senior candidates — articulating trade-offs is one of the clearest signals that separates candidates who write SQL from those who engineer with SQL. Doing this at any level stands out immediately in interviews.

**Interview Answer Skeleton:**
- **What it is:** The habit of reasoning out loud about why a particular SQL approach is preferred in a given context — covering performance implications, correctness risks, maintainability costs, and engine-specific behaviour differences.
- **Why it matters / trade-offs:** Interviewers at senior DE levels evaluate the reasoning behind SQL choices as heavily as the syntax itself. Code that works but whose author cannot explain the trade-offs signals a gap in production readiness.
- **Example or context:** Asked to find duplicate records: ROW_NUMBER() + CTE is preferable when you need to choose which duplicate to keep (it's deterministic and flexible); GROUP BY + HAVING is better when you only need to identify which values have duplicates without caring about individual rows. Explaining this distinction demonstrates the difference between knowing SQL and engineering with it.

**Free Resources:**
- [Mode SQL Tutorial](https://mode.com/sql-tutorial) — teaches SQL from the perspective of analytical thinking and trade-off reasoning, not just syntax
- [W3Schools SQL Reference](https://www.w3schools.com/sql) — comprehensive SQL reference for verifying syntax and comparing behaviour across database engines

---
