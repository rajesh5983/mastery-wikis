# Snowflake — Architecture

---

## Virtual Warehouses

**Status:** ⬜ Not Started

**Definition:** A Virtual Warehouse (VW) is an independent, isolated compute cluster in Snowflake that executes SQL queries and DML operations. VWs are sized in T-shirt sizes (XS through 6XL), each size doubling the compute nodes and credit consumption rate of the previous. They can be started and suspended in seconds, and multiple VWs share the same cloud storage layer without interfering with each other.

**Key Mental Model:** Virtual Warehouses are independent calculator engines — each one performs its own computation against the same shared data. Starting a larger one or adding more warehouses adds compute power without touching the data; one VW's heavy query load never slows another VW's queries.

**How It Works:**
- Each virtual warehouse is a cluster of cloud VMs (EC2 on AWS, Azure VMs, GCE on GCP) provisioned and managed by Snowflake. An XS warehouse is a single node; an S is 2 nodes; each size doubles the node count. VW size determines the number of parallel workers for partition scanning and the total available memory for hash joins and aggregations.
- Warehouses are **suspended** (no VMs running, no credit consumption) when idle for the auto-suspend duration (configurable, default 10 minutes). **Resuming** a suspended warehouse provisions fresh VMs — on most cloud regions, this takes 1–5 seconds. The local disk cache on resumed VMs is cold (empty) until queries warm it.
- Query execution uses Snowflake's massively parallel processing (MPP) model: the cloud services layer generates an execution plan, partitions the work into tasks, and distributes tasks to warehouse worker nodes. Each node processes a subset of micro-partitions in parallel and returns results to a coordinator node for final aggregation.
- Warehouses have a **local disk cache** (SSD on each node) that caches micro-partition data read from cloud storage during the session. Subsequent queries that read the same micro-partitions hit this cache at near-memory speed. Cache is lost on suspend/resume — scheduling related queries on the same warehouse during the same session maximises cache hit rate.
- **Credit billing** is per-second (minimum 60 seconds) at the rate of the warehouse size. An XS warehouse costs 1 credit/hour; a 6XL costs 512 credits/hour. Cost optimisation means choosing the smallest warehouse that meets query latency SLAs, enabling auto-suspend aggressively, and using result caching to avoid repeated full-table scans. See [[Cloud-Platforms/Snowflake/01-Architecture#Multi-Cluster Architecture]] for concurrency scaling.

**Common Misconceptions:**
- Bigger warehouses always run queries faster — for I/O-bound queries (scanning large tables with few filters), the bottleneck is cloud storage throughput, not compute; doubling warehouse size yields diminishing returns. Bigger warehouses help for CPU-bound operations (complex joins, large aggregations) more than for simple scans.
- Suspending a warehouse loses data — warehouses are stateless compute; all data lives in the storage layer. Suspending loses only the local disk cache (warm data), not any stored data. This is Snowflake's storage-compute separation in practice.

**Interview Answer Skeleton:**
- **What it is:** An independent MPP compute cluster (cloud VMs managed by Snowflake) that executes queries against the shared storage layer, billed per-second when running, and completely isolated from other warehouses sharing the same data.
- **Why it matters / trade-offs:** VW isolation enables workload separation — BI users, pipelines, and ad-hoc analysts can run on separate warehouses without resource contention. The trade-off is that each warehouse has a cold local cache on start, and choosing the right size requires understanding whether the query bottleneck is I/O, CPU, or memory.
- **Example or context:** A Snowflake environment has three warehouses: `PIPELINE_WH` (Large, runs dbt jobs), `BI_WH` (Small, multi-cluster, serves Tableau), and `ADHOC_WH` (XS, for analyst exploration). Each is sized for its workload, auto-suspends when idle, and never contends with the others — a runaway dbt query on `PIPELINE_WH` has zero impact on BI dashboard queries.

**Free Resources:**
- [Snowflake Virtual Warehouse Overview](https://docs.snowflake.com/en/user-guide/warehouses-overview) — sizing, credit consumption, auto-suspend configuration, and caching behaviour
- [Snowflake Quickstarts](https://quickstarts.snowflake.com) — hands-on labs covering warehouse sizing experiments and query performance analysis

---

## Storage Layer

**Status:** ⬜ Not Started

**Definition:** Snowflake stores all table data in a proprietary columnar micro-partition format on cloud object storage (S3 for AWS, Azure Blob for Azure, GCS for GCP). Micro-partitions are 50–500MB compressed columnar files, each annotated with per-column metadata (min/max values, null counts, distinct value counts) that enable automatic partition pruning and efficient predicate pushdown without user-defined partition schemes.

**Key Mental Model:** Snowflake storage is a self-organising filing system — data is automatically divided into appropriately sized chunks and each chunk is labelled with a metadata summary of its contents. A query filtering `WHERE order_date > '2024-01-01'` skips all micro-partitions where the max `order_date` value is below the threshold — reading only the relevant chunks without scanning the full table.

**How It Works:**
- When data is loaded into a Snowflake table (via COPY INTO, Snowpipe, INSERT), Snowflake's storage layer partitions it into micro-partitions of 50–500MB (uncompressed). Within each micro-partition, data is stored column-by-column (columnar format), then compressed using algorithms optimised per column data type (LZO, Zstandard).
- **Micro-partition metadata** is written at creation time and maintained in the Cloud Services Layer's metadata store: each micro-partition has a metadata record containing column-level min/max values, null count, and distinct count. This metadata enables the query planner to prune micro-partitions without reading any data from object storage.
- **Automatic clustering** maintains the physical order of micro-partitions relative to a table's natural insert order. Over time, as DML operations create new micro-partitions in arbitrary order, a table becomes "fragmented" — its range of values per micro-partition overlaps, reducing pruning effectiveness. The `CLUSTER BY` clause defines clustering keys; Snowflake's automatic clustering service periodically re-sorts and re-partitions the table to improve clustering depth.
- Immutability is a key storage property: Snowflake never modifies existing micro-partitions in place. UPDATE and DELETE operations write new micro-partitions with the changes applied, mark the old micro-partitions as deleted, and update the table's file metadata pointer. The old micro-partitions are retained for the Time Travel retention period before being physically deleted by the background compaction process.
- Storage billing is separate from compute billing: charged per-TB-per-month of average compressed storage used, including active data, Time Travel data (historical versions), and Fail-safe data (7-day disaster recovery window, not user-accessible). See [[Cloud-Platforms/Snowflake/01-Architecture#Result Caching]] and [[Cloud-Platforms/Snowflake/02-Data-Engineering#Time Travel]] for features enabled by the immutable storage model.

**Common Misconceptions:**
- Snowflake's columnar storage works like Parquet on a data lake — Snowflake's proprietary micro-partition format is not Parquet and cannot be read by external engines without Iceberg Tables configured; the storage is opaque to everything except Snowflake's own query engine.
- `CLUSTER BY` is equivalent to traditional partitioning by date — traditional partitioning assigns each row to exactly one partition; Snowflake clustering influences micro-partition ordering and can span multiple clustering key columns but is maintained probabilistically by the background service, not enforced as a strict partition scheme.

**Interview Answer Skeleton:**
- **What it is:** Immutable columnar micro-partitions (50–500MB, compressed) stored on cloud object storage with per-column metadata enabling automatic partition pruning — all managed transparently without user-defined partition keys.
- **Why it matters / trade-offs:** Automatic micro-partition pruning eliminates the partition management overhead required by traditional data warehouses (Hive partition naming, Redshift distribution keys). The trade-off is that clustering degrades over time for high-DML tables and requires automatic clustering (at additional credit cost) to maintain optimal pruning performance.
- **Example or context:** A 10TB orders table scanned daily by a query filtering `WHERE region = 'EMEA'`. Without clustering, the query reads all 10TB (micro-partitions have overlapping region values). With `CLUSTER BY (region)`, Snowflake's automatic clustering service groups EMEA rows into contiguous micro-partitions — the same query prunes to ~2TB, reducing compute time and warehouse credit cost by ~80%.

**Free Resources:**
- [Snowflake Storage Architecture](https://docs.snowflake.com/en/user-guide/tables-storage-considerations) — micro-partition format, metadata, clustering, and storage billing documentation
- [Snowflake Quickstarts](https://quickstarts.snowflake.com) — hands-on labs on clustering keys, query pruning analysis, and storage optimisation

---

## Cloud Services Layer

**Status:** ⬜ Not Started

**Definition:** The Cloud Services Layer (CSL) is the always-on, multi-tenant control plane of Snowflake that handles authentication, query parsing, query optimisation, metadata management, transaction coordination, and result set caching. It runs on Snowflake-managed infrastructure across all cloud providers, not on the customer's cloud account, and is billed as a fraction of compute usage (first 10% of daily compute is free; excess is billed).

**Key Mental Model:** The Cloud Services Layer is the control centre of Snowflake — the virtual warehouses (compute) do the physical work of reading data and executing operators, but the control centre decides how to route each query, which micro-partitions to include, which results to serve from cache, and how to enforce transactions — before a single byte is read from object storage.

**How It Works:**
- Query lifecycle through the CSL: the JDBC/ODBC client submits a SQL statement → the CSL parser validates syntax and resolves object references (table names, column types) against the metadata store → the query optimiser generates and costs multiple execution plans using micro-partition statistics → the final plan is sent to the virtual warehouse for physical execution.
- The **query optimiser** is cost-based: it uses micro-partition metadata (row counts, distinct values, min/max per column) to estimate selectivity of predicates, choose join order (smaller-to-larger tables), and decide between broadcast joins (replicate small table to all nodes) and hash-join shuffle (repartition both tables by join key). Plan quality is directly dependent on statistics freshness.
- **Metadata management** in the CSL includes the table catalog (schema definitions, column types, constraints), the file metadata store (which micro-partitions belong to which table version, their cloud storage paths), transaction log entries, and user/role privilege grants. All DDL operations (CREATE, ALTER, DROP) are metadata-only operations against the CSL — no data files are moved.
- **Transaction coordination** enforces Snowflake's ACID properties at the CSL level using optimistic concurrency control (OCC). Concurrent DML statements read a snapshot of the table at transaction start; conflicts are detected at commit time. Read queries always see a consistent snapshot without blocking writes — this is Snowflake's multi-version concurrency control (MVCC) implementation.
- **Session management** in the CSL maintains active connections, session parameters (warehouse assignment, query timeout, date format), and role context. The CSL enforces RBAC access control checks before any query reaches the warehouse — a query referencing a table the user's active role doesn't have SELECT on is rejected by the CSL before any compute is consumed. See [[Cloud-Platforms/Snowflake/01-Architecture#Result Caching]] for how the CSL serves cached results.

**Common Misconceptions:**
- The Cloud Services Layer runs in the customer's cloud account — the CSL is Snowflake-managed multi-tenant infrastructure; customers have no access to CSL VMs. Only virtual warehouses (compute) run within the customer's cloud account in some deployment models.
- CSL overhead means Snowflake is slower for simple metadata queries — DDL operations (SHOW TABLES, DESCRIBE TABLE, COUNT(*) on a small table) execute entirely within the CSL without involving a virtual warehouse at all, making them extremely fast (sub-second) and free from compute billing.

**Interview Answer Skeleton:**
- **What it is:** Snowflake's always-on multi-tenant control plane handling query optimisation, metadata management, transaction coordination, and RBAC enforcement — running on Snowflake infrastructure, billed as a fraction of compute, and operating independently of virtual warehouses.
- **Why it matters / trade-offs:** The CSL enables Snowflake's zero-management model — no manual statistics maintenance, partition management, or vacuum operations because the CSL handles all of these automatically. The trade-off is that the CSL is a shared Snowflake-managed layer; customers have no control over its configuration or performance, and its multi-tenant nature means rare CSL incidents affect multiple customers simultaneously.
- **Example or context:** A Snowflake virtual warehouse is suspended — zero credits running. A user runs `SHOW TABLES IN DATABASE PROD` and `SELECT COUNT(*) FROM orders` where orders has 10M rows. Both return instantly because the CSL serves SHOW TABLES from the metadata store and COUNT(*) from micro-partition metadata (without launching the warehouse) — demonstrating that the CSL provides value independent of compute availability.

**Free Resources:**
- [Snowflake Key Concepts](https://docs.snowflake.com/en/user-guide/intro-key-concepts) — three-layer architecture documentation covering CSL, storage, and compute interactions
- [Snowflake Quickstarts](https://quickstarts.snowflake.com) — architecture walkthrough labs including query profiling and cloud services layer behaviour

---

## Multi-Cluster Architecture

**Status:** ⬜ Not Started

**Definition:** Multi-cluster virtual warehouses automatically provision additional compute clusters (each identical to the base warehouse) during periods of high query concurrency, then decommission them as load drops. This solves query queuing caused by many simultaneous users without requiring a permanent warehouse resize, and is the primary Snowflake feature for BI workloads with unpredictable concurrency spikes.

**Key Mental Model:** Multi-cluster is a bank that automatically opens more teller windows when the queue exceeds a threshold and closes the extra windows when the rush is over — customers experience consistent service time regardless of whether one person or fifty arrive simultaneously.

**How It Works:**
- Multi-cluster warehouses have two key configuration parameters: **minimum clusters** (always running, even at zero load — set to 1 to allow full auto-suspend) and **maximum clusters** (the upper bound on how many clusters can provision during peak load). Credit consumption scales linearly with running cluster count.
- **Scaling policies** control when additional clusters are added: "Standard" policy adds a cluster when a query has been queued for at least 1 second; "Economy" policy adds a cluster only when queuing is expected to last at least 6 minutes (conserving credits at the cost of initial queue wait). Standard is appropriate for user-facing BI; Economy for batch workloads that can tolerate latency.
- Query routing works as follows: the CSL load balancer assigns incoming queries to clusters with available slots. When all running clusters are at capacity (all warehouse slots filled with executing queries), the CSL queues new queries and begins provisioning an additional cluster. Provisioning takes 1–5 seconds — first-wave queries in the queue experience this startup delay.
- Each cluster in a multi-cluster warehouse is fully independent — it has its own set of VMs, its own local disk cache, and runs its own subset of queries. The CSL coordinates which queries go to which cluster but each cluster executes independently without cross-cluster coordination overhead.
- Cluster count is monitored via the `QUERY_HISTORY` view in `INFORMATION_SCHEMA` and the **Warehouse Load Monitoring** chart in Snowsight, which shows concurrent query count and cluster count over time — enabling data-driven decisions about minimum/maximum cluster configuration. See [[Cloud-Platforms/Snowflake/05-Administration#Resource Monitors]] for credit governance on multi-cluster warehouses.

**Common Misconceptions:**
- Multi-cluster warehouses run faster per-query because of more compute — each cluster in a multi-cluster warehouse handles a subset of queries independently; adding clusters improves throughput (more queries handled simultaneously) not individual query latency (each query uses only one cluster).
- Multi-cluster warehouses can be set to unlimited clusters safely — each additional cluster doubles the maximum credit consumption rate; setting maximum clusters without a Resource Monitor is a cost governance risk. Always pair multi-cluster warehouses with a Resource Monitor threshold.

**Interview Answer Skeleton:**
- **What it is:** A virtual warehouse configuration that automatically provisions additional identical compute clusters (each billed separately) when concurrent user count exceeds the single-cluster capacity, scaling out to the configured maximum and scaling back in as load drops.
- **Why it matters / trade-offs:** Eliminates query queuing for BI dashboards with unpredictable concurrency spikes without permanently sizing up the warehouse. The trade-off is that each additional cluster costs the same as the base warehouse — a 5-cluster peak on an L warehouse costs 5× an L warehouse's credit rate; monitor carefully with Resource Monitors.
- **Example or context:** A Tableau dashboard serves 80 concurrent analysts at month-end reporting time. A single Medium warehouse queues queries for 45 seconds during the 9–10am peak. Setting minimum=1, maximum=4 clusters means the warehouse automatically scales to 4 clusters during the peak (handling all 80 analysts simultaneously) and returns to 1 cluster by 10:30am — eliminating queuing without permanently paying for 4 clusters.

**Free Resources:**
- [Snowflake Multi-Cluster Warehouses](https://docs.snowflake.com/en/user-guide/warehouses-multicluster) — scaling policies, min/max cluster configuration, and credit implications
- [Snowflake Quickstarts](https://quickstarts.snowflake.com) — concurrency scaling labs and warehouse monitoring exercises

---

## Query Profile

**Status:** ⬜ Not Started

**Definition:** The Snowflake Query Profile is a visual, interactive execution plan for any completed or running query, accessible from Snowsight (History tab → select a query → Profile tab). It displays every operator in the execution plan as a node, with statistics for each node: rows produced, bytes scanned, execution time, spills to local or remote disk, and partitions scanned vs pruned.

**Key Mental Model:** The Query Profile is an X-ray of a query's execution — it reveals every step the warehouse took, how long each step took, how much data passed through each step, and exactly where the performance bottleneck is hiding. Optimisation without the Query Profile is guessing; with it, you target the most expensive node first.

**How It Works:**
- The Query Profile represents the execution plan as a **directed acyclic graph (DAG)** where nodes are relational operators (TableScan, Join, Aggregate, Sort, Filter, Exchange) and edges represent data flow. Data flows bottom-up: leaf nodes are scans, and the root node produces the final result set.
- Each node displays a **percentage of total execution time** (the most important metric for optimisation). The node consuming the highest percentage of time is the primary bottleneck. Common expensive nodes: `JoinFilter` (large hash join), `Aggregate` (large grouping), `Sort` (triggered by ORDER BY or sort-merge join on unsorted data), and `Exchange` (data redistribution across nodes — equivalent to a Spark shuffle).
- **Partition statistics** on the `TableScan` node show: partitions scanned vs partitions total. High ratio (e.g., 800K/800K partitions scanned) means no pruning occurred — the predicate does not align with clustering. Low ratio (e.g., 40K/800K) means effective pruning — the query is touching only 5% of the table data.
- **Spill to disk** appears as "Bytes spilled to local storage" or "Bytes spilled to remote storage" in node statistics. Local spill (SSD on warehouse nodes) is serious; remote spill (cloud object storage) is catastrophic for performance. Spill indicates the query requires more memory than the warehouse provides — the fix is either a larger warehouse size or query restructuring to reduce the intermediate result set size.
- **Statistics nodes** (small information icons in Snowsight) reveal additional diagnostics: whether the join build side fit in memory, the actual vs estimated row count per operator (large differences indicate stale statistics or complex query patterns that confuse the optimiser). See [[Cloud-Platforms/Snowflake/05-Administration#Cost Management]] for using Query Profile data to identify expensive queries systematically.

**Common Misconceptions:**
- A green checkmark on the query means it was optimal — the Query Profile shows green checkmarks for completed queries regardless of performance; a query that took 10 minutes shows the same checkmark as one that took 10ms. The percentage distribution of time across nodes, not the colour, indicates optimisation opportunities.
- Spills to local storage are acceptable for large queries — any spill-to-disk is a signal that the warehouse is undersized for the operation or that the query has an inefficient intermediate result. Local spill degrades performance 10–50x vs in-memory; remote spill can be 100x+ slower.

**Interview Answer Skeleton:**
- **What it is:** An interactive DAG visualisation of a query's execution plan with per-node statistics (execution time %, rows, bytes, partitions, spill) that identifies the most expensive operator in the plan and provides the data needed to target optimisation effort precisely.
- **Why it matters / trade-offs:** The Query Profile converts performance optimisation from guessing to measurement — the most expensive node tells you whether to add a clustering key, increase warehouse size, rewrite a join, or fix a data skew issue. The trade-off is that the Query Profile is only available for queries that complete (or are actively running); failed queries may not have full profile data.
- **Example or context:** A Snowflake query runs in 8 minutes instead of the expected 45 seconds. Query Profile shows the `JoinFilter` node consuming 85% of execution time, with 200GB spilled to remote storage. Diagnosis: a large hash join is exceeding warehouse memory. Fix options: increase warehouse from M to L (more memory per node), rewrite to use a window function avoiding the join, or pre-aggregate one side of the join before the join step.

**Free Resources:**
- [Snowflake Query Profile Documentation](https://docs.snowflake.com/en/user-guide/ui-query-profile) — node types, statistics interpretation, spill detection, and performance troubleshooting walkthrough
- [Snowflake Quickstarts](https://quickstarts.snowflake.com) — query performance optimisation labs using the Query Profile

---

## Result Caching

**Status:** ⬜ Not Started

**Definition:** Snowflake's result cache persists the complete result set of every successfully executed query for 24 hours in the Cloud Services Layer. An identical subsequent query — same SQL text, same role, same warehouse parameters, same underlying data — is served from cache instantly at zero compute cost. Results are invalidated when any micro-partition in the queried tables is modified by DML.

**Key Mental Model:** Result caching is Snowflake remembering the exact answer to every question it was asked in the last 24 hours. Ask the same question again within that window, with nothing changed, and the answer is returned instantly without touching the warehouse or the data.

**How It Works:**
- The result cache is managed entirely by the Cloud Services Layer. When a query executes, the CSL generates a cache key from the query's SQL text (normalised), the active role, the warehouse name, and a hash of the table's current micro-partition state. The result set (up to a configured size limit) is stored in the CSL's distributed result cache alongside this key.
- Cache lookup occurs before the query is sent to the virtual warehouse: the CSL computes the cache key for the incoming query, checks the result cache, and returns the cached result immediately if a match exists. **No warehouse credits are consumed** for a cache hit — the warehouse is not even activated for the query.
- **Invalidation** occurs automatically when any of the queried tables receive a new DML commit (INSERT, UPDATE, DELETE, MERGE). The CSL detects micro-partition state changes and marks the associated result cache entries as stale. The next query after an invalidation executes freshly against the warehouse; all subsequent identical queries hit the new cache entry.
- Cache hits are visible in `QUERY_HISTORY` via the `IS_RESULT_REUSED` boolean column. Monitoring `IS_RESULT_REUSED` across high-frequency BI queries reveals cache hit rate — a low hit rate on queries with stable underlying data suggests SQL text inconsistencies (e.g., timestamp literals that change on each execution) that defeat caching.
- Result cache is **user-agnostic but role-sensitive**: a result cached by User A is reusable by User B if they have the same active role and issue the same query. This enables shared cache benefits across large BI user populations. However, a user running the query with `SYSADMIN` role will not hit the cache entry from the same query run with a `REPORTER` role. See [[Cloud-Platforms/Snowflake/01-Architecture#Cloud Services Layer]] for the CSL infrastructure that hosts the result cache.

**Common Misconceptions:**
- Result caching requires explicit configuration — result caching is on by default for all Snowflake accounts and applies to all queries; there is no opt-in required, though it can be disabled per session with `ALTER SESSION SET USE_CACHED_RESULT = FALSE` for testing.
- Result caching works for queries with dynamic functions — queries containing `CURRENT_TIMESTAMP()`, `CURRENT_USER()`, `RANDOM()`, `SEQ()`, or `UUID_STRING()` are never cached because these functions produce different values on every execution by design; substituting literal values where possible enables caching.

**Interview Answer Skeleton:**
- **What it is:** A 24-hour Cloud Services Layer cache of complete query result sets, served on exact query match (SQL, role, warehouse params, data state) at zero compute cost, automatically invalidated on any DML commit to the queried tables.
- **Why it matters / trade-offs:** Result caching makes repetitive BI dashboard queries essentially free — a morning refresh of 200 Power BI reports querying the same stable summary tables hits the cache after the first execution, saving potentially thousands of credits per day. The trade-off is cache invalidation sensitivity: even a single row insert to a queried table invalidates all result cache entries for queries touching that table.
- **Example or context:** A Tableau dashboard with 15 charts all querying a `DAILY_SALES_SUMMARY` table (refreshed once daily at 6am) is viewed by 150 analysts from 8am–6pm. The first analyst's queries warm the result cache; all 149 subsequent analysts' identical queries return cached results instantly with zero warehouse credits — the 150-analyst load costs the same compute as a single-analyst load.

**Free Resources:**
- [Snowflake Result Cache Documentation](https://docs.snowflake.com/en/user-guide/querying-persisted-results) — cache eligibility criteria, invalidation behaviour, monitoring via QUERY_HISTORY, and session-level controls
- [Snowflake Quickstarts](https://quickstarts.snowflake.com) — performance optimisation labs demonstrating cache hit analysis and cost monitoring
