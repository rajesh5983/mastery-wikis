# Snowflake — Architecture

---

## Virtual Warehouses

**Status:** ⬜ Not Started

**Definition:** A Virtual Warehouse (VW) is an independent compute cluster in Snowflake that executes queries. VWs are sized in T-shirt sizes (XS through 6XL), can be started and suspended in seconds, and are completely isolated from each other — one VW's workload never impacts another. Storage is shared across all VWs.

**Mental Model:** Virtual Warehouses are independent calculator engines — each one does its own computation against the same shared data. Spinning up a bigger one or adding more doesn't affect the data; it just adds more compute power.

**Free Resources:** https://docs.snowflake.com/en/user-guide/warehouses-overview — Snowflake virtual warehouse overview and sizing documentation

---

## Storage Layer

**Status:** ⬜ Not Started

**Definition:** Snowflake stores all data in a proprietary columnar micro-partition format on cloud object storage (S3, Azure Blob, GCS). Micro-partitions are 50–500MB compressed columnar files with rich metadata (min/max values per column) that enable automatic partition pruning without user-defined partition keys.

**Mental Model:** Snowflake storage is a self-organising filing system — data is automatically divided into optimally sized chunks, each labelled with what values they contain so queries can skip irrelevant chunks without scanning them.

**Free Resources:** https://docs.snowflake.com/en/user-guide/tables-storage-considerations — Snowflake storage architecture and micro-partition documentation

---

## Cloud Services Layer

**Status:** ⬜ Not Started

**Definition:** The Cloud Services Layer is the "brain" of Snowflake — it handles authentication, query parsing and optimisation, metadata management, transaction coordination, and result caching. It runs continuously on Snowflake's infrastructure and is not billed separately (up to 10% of compute usage).

**Mental Model:** The Cloud Services Layer is the control centre of Snowflake — the warehouse (compute) does the physical work, but the control centre decides how to route queries, which micro-partitions to scan, and what results to cache.

**Free Resources:** https://docs.snowflake.com/en/user-guide/intro-key-concepts — Snowflake key concepts documentation covering all three architecture layers

---

## Multi-Cluster Architecture

**Status:** ⬜ Not Started

**Definition:** Multi-cluster warehouses automatically add additional compute clusters during high-concurrency periods and remove them when demand drops. This solves the queuing problem for BI workloads with many simultaneous users, without requiring a manual warehouse resize.

**Mental Model:** Multi-cluster is like a bank opening more teller windows when the queue gets long — automatic, seamless to customers, and the extra windows close when the rush is over.

**Free Resources:** https://docs.snowflake.com/en/user-guide/warehouses-multicluster — Snowflake multi-cluster warehouse documentation covering scaling policies

---

## Query Profile

**Status:** ⬜ Not Started

**Definition:** The Snowflake Query Profile is a visual execution plan for any completed query, showing every operation node, rows processed, bytes scanned, execution time per node, and spills to disk. It is the primary tool for diagnosing and optimising slow Snowflake queries.

**Mental Model:** The Query Profile is the X-ray of a query's execution — it shows every step, how long it took, how much data it touched, and exactly where the bottleneck is.

**Free Resources:** https://docs.snowflake.com/en/user-guide/ui-query-profile — Snowflake Query Profile documentation covering node types and performance interpretation

---

## Result Caching

**Status:** ⬜ Not Started

**Definition:** Snowflake automatically caches the results of every query for 24 hours. An identical subsequent query served from cache returns instantly at no compute cost. Results are invalidated only if the underlying data changes. This makes frequently-run BI queries effectively free after the first execution.

**Mental Model:** Result caching is Snowflake remembering the answer to a question it was asked before — if you ask the same question within 24 hours and nothing changed, it gives you the same answer instantly without doing any new computation.

**Free Resources:** https://docs.snowflake.com/en/user-guide/querying-persisted-results — Snowflake result cache documentation covering eligibility criteria and invalidation
