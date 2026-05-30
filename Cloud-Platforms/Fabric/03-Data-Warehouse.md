# Microsoft Fabric — Data Warehouse

---

## Synapse Data Warehouse

**Status:** ⬜ Not Started

**Definition:** The Fabric Data Warehouse is a fully managed T-SQL data warehouse built on top of OneLake, offering distributed query processing for large-scale analytics. Unlike traditional warehouses, it stores all data as Delta Parquet on OneLake, enabling cross-experience access by Spark and Power BI without data copies or ETL pipelines.

**Key Mental Model:** The Fabric Warehouse is a SQL-native window onto OneLake — it speaks T-SQL to analysts and BI tools, but underneath it's Delta files that other Fabric experiences can also read directly.

**How It Works:**
- The Fabric Warehouse engine is a **distributed T-SQL query processor** (derived from Synapse Analytics architecture) that compiles T-SQL statements into distributed query plans, partitions work across multiple compute nodes, and reads Delta Parquet files in parallel from OneLake using the ADLS Gen2 API.
- Each Warehouse query is compiled by the **distributed query optimizer** into a physical plan consisting of scan, shuffle, join, and aggregation operations distributed across compute nodes; the optimizer uses table statistics stored in the Warehouse metadata service to choose join strategies and partition pruning.
- All Warehouse tables are stored as Delta-Parquet files in OneLake at a `<workspace>/<warehouse>.Warehouse/Tables/<schema>/<table>/` path — this is the same Delta format used by Lakehouse tables, enabling zero-copy cross-experience reads by Spark notebooks and Power BI Direct Lake.
- When a query references a Lakehouse table via a cross-database three-part name, the Warehouse engine resolves the Lakehouse's OneLake path, reads the Delta transaction log to identify the current set of Parquet files, and includes those files in the distributed scan plan — the data is read directly from OneLake, not copied into the Warehouse storage.
- The Warehouse includes an automatically provisioned **SQL Analytics Endpoint** identical to the Lakehouse's, allowing Power BI and BI tools to connect via JDBC/ODBC using the same TDS protocol as SQL Server. See [[Cloud-Platforms/Fabric/03-Data-Warehouse#Cross-Database Queries]] for zero-copy cross-database mechanics.

**Common Misconceptions:**
- The Fabric Warehouse is not Azure Synapse Analytics rebranded — while it shares architectural lineage, Fabric Warehouse is a SaaS reimplementation with a different billing model (capacity CUs vs dedicated SQL pool DWUs), different scaling behaviour, and deeper OneLake integration; migration from Synapse requires explicit re-testing.
- "Stored as Delta means all tools can write to Warehouse tables" is false — only the Warehouse T-SQL engine and Fabric pipelines can write to Warehouse tables; external Spark jobs cannot directly modify Warehouse Delta files because Warehouse manages table metadata separately from the raw Delta files.

**Interview Answer Skeleton:**
- **What it is:** A distributed T-SQL warehouse on Fabric that compiles queries into parallel scan-and-shuffle plans over Delta Parquet files in OneLake, with zero-copy cross-experience reads and SaaS-managed compute scaling within the capacity CU pool.
- **Why it matters / trade-offs:** Enables SQL-native warehouse workloads without the operational overhead of Azure Synapse dedicated SQL pools; the trade-off is shared CU contention with other Fabric experiences and fewer tuning knobs (no distribution keys, no manual statistics management) compared to Synapse Dedicated SQL Pool.
- **Example or context:** An analytics team migrates from Azure Synapse Analytics to Fabric Warehouse — they reuse 90% of their T-SQL stored procedures unchanged, but now the underlying data is in OneLake Delta files that their Spark pipelines and Power BI Direct Lake reports also read simultaneously without any ETL between systems.

**Free Resources:**
- [Fabric Data Warehouse Overview](https://learn.microsoft.com/en-us/fabric/data-warehouse/data-warehousing) — architecture, query engine, Delta storage, and cross-experience access
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Warehouse vs Lakehouse guidance, T-SQL surface area, and performance tuning

---

## T-SQL in Fabric

**Status:** ⬜ Not Started

**Definition:** Fabric's Data Warehouse supports a broad subset of T-SQL (Transact-SQL) including DML (SELECT, INSERT, UPDATE, DELETE, MERGE), DDL (CREATE TABLE, ALTER, DROP), stored procedures, views, functions, and common T-SQL syntax. This enables SQL Server and Azure Synapse-experienced engineers to work productively without learning new query syntax.

**Key Mental Model:** T-SQL in Fabric is a familiar dialect — if you know SQL Server or Azure Synapse, the core SQL works as expected, with modern distributed warehouse semantics underneath.

**How It Works:**
- T-SQL submitted to the Fabric Warehouse is parsed by a **distributed query compiler** that generates a query plan in the form of a physical operator tree; the compiler applies cost-based optimisation using table statistics (row counts, column statistics) maintained in the Warehouse metadata service.
- The compiled plan is **distributed**: scan operations are parallelised across the compute nodes processing different Parquet file partitions, while shuffle operations (for hash joins and GROUP BY aggregations) redistribute rows across nodes by a hash key before executing the join or aggregation.
- **Stored procedures** compile to execution plans at creation time; subsequent executions use the cached plan, enabling stored procedures to serve as the primary transformation layer for SQL-first teams without the overhead of re-compilation per call.
- The Warehouse supports **ANSI SQL standard functions** as well as T-SQL-specific windowing functions (OVER, PARTITION BY, LEAD/LAG, ROW_NUMBER) — these are compiled to distributed window partitioning operators that partition rows by the window key across compute nodes before applying the window calculation.
- T-SQL **DML statements** (INSERT, UPDATE, DELETE, MERGE) write transactionally to the underlying Delta Parquet files — MERGE generates a new set of Parquet files representing the upserted/deleted state and commits a Delta transaction log entry, ensuring ACID semantics consistent with the Delta protocol. See [[Cloud-Platforms/Fabric/03-Data-Warehouse#Synapse Data Warehouse]] for the distributed execution model.

**Common Misconceptions:**
- Fabric Warehouse T-SQL is not 100% compatible with SQL Server or Azure Synapse Dedicated SQL Pool — some SQL Server features (CLR integration, full-text search, certain system stored procedures, linked server queries) are not supported; migration scripts must be reviewed for compatibility before deployment.
- "Stored procedures run faster than ad hoc SQL" is not universally true in distributed Fabric — stored procedure plan caching helps for repeated executions, but the distributed nature of large scans means query selectivity and OneLake partition layout have a greater impact on performance than cached plans.

**Interview Answer Skeleton:**
- **What it is:** A distributed T-SQL execution surface in Fabric Warehouse that compiles standard T-SQL (DML, DDL, stored procedures, window functions) into parallelised physical plans that scan OneLake Delta Parquet partitions across compute nodes with ACID-compliant writes via the Delta protocol.
- **Why it matters / trade-offs:** Allows SQL Server-skilled analysts and engineers to build production warehouse workloads on Fabric without learning new syntax; the trade-off is that not all T-SQL features are supported, and performance tuning requires understanding distributed plan mechanics rather than single-node SQL Server optimisation patterns.
- **Example or context:** A data engineering team ports their Azure Synapse Analytics stored procedures to Fabric Warehouse — 95% run unchanged; they discover a few unsupported system functions and rewrite those steps as native T-SQL equivalents, gaining the benefit of OneLake Delta storage shared with their Lakehouse Spark jobs.

**Free Resources:**
- [Fabric T-SQL Surface Area](https://learn.microsoft.com/en-us/fabric/data-warehouse/tsql-surface-area) — supported statements, functions, and unsupported features reference
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — T-SQL query optimisation, statistics management, and stored procedure best practices

---

## Cross-Database Queries

**Status:** ⬜ Not Started

**Definition:** Fabric's zero-copy cross-database queries allow a Warehouse to query tables from other Warehouses and Lakehouses in the same workspace using three-part name syntax (`[workspace].[item].[schema].[table]`), reading from shared OneLake storage without data movement or replication between items.

**Key Mental Model:** Cross-database queries are reading from any shelf in the shared library — you can query the sales Lakehouse from within the finance Warehouse in one SQL statement, without copying data between them first.

**How It Works:**
- Cross-database references resolve to **OneLake shortcuts** under the hood — when a Warehouse query references a three-part name pointing to a Lakehouse table, the Warehouse query engine resolves the Lakehouse's OneLake storage path and reads the Delta Parquet files at that path directly from OneLake storage, without moving or duplicating the data.
- The Warehouse metadata service maintains a **federated schema registry** that caches schema information (column names, data types, statistics) from referenced Lakehouses and other Warehouses — this allows the distributed query optimizer to plan cross-database joins using column statistics from both sources without querying the remote item's metadata service at plan time.
- Cross-database joins between a Warehouse table and a Lakehouse table are executed as **distributed hash joins** or **broadcast joins** depending on the relative sizes of the tables — the optimizer reads Parquet files from both OneLake paths in parallel across compute nodes and performs the join in memory, making cross-database joins perform similarly to same-database joins.
- T-SQL **views** can encapsulate cross-database references — a view in the Warehouse can reference tables from multiple Lakehouses, giving BI tools and analysts a single-Warehouse query surface that transparently reads from multiple underlying items without requiring users to know the cross-database syntax.
- Cross-database queries are subject to **workspace-level permissions** — a user needs at least Read access to the referenced Lakehouse or Warehouse item (governed by Fabric workspace roles) to execute a cross-database query; the query fails with a permission error if the executing user lacks access to any referenced item. See [[Cloud-Platforms/Fabric/05-Administration]] for workspace permission management.

**Common Misconceptions:**
- Cross-database queries are not instantaneous for large tables — while they avoid data movement, they still require reading the full Delta Parquet files from OneLake across a network; performance depends on the size of the tables being joined and the effectiveness of predicate push-down for partition pruning.
- Three-part names in Fabric Warehouse do not reference Azure SQL Database or external systems — cross-database syntax only works between Fabric items (Warehouses and Lakehouses) within the same workspace; querying external databases requires a Lakehouse shortcut or Fabric Mirroring, not native three-part names.

**Interview Answer Skeleton:**
- **What it is:** A zero-copy T-SQL query mechanism that resolves three-part cross-item references to OneLake Delta file paths, enabling distributed joins between Warehouse and Lakehouse tables through the Warehouse's federated schema registry and distributed query planner.
- **Why it matters / trade-offs:** Enables a "query anywhere in the workspace" experience that eliminates inter-experience ETL pipelines for read-heavy analytical workloads; the trade-off is that cross-database queries still incur full table scan cost when predicates cannot be pushed down to filter the source Delta Parquet files.
- **Example or context:** A finance analyst joins a `sales.Lakehouse.dbo.transactions` table with a `reporting.Warehouse.dbo.cost_centres` table in a single T-SQL query — the Warehouse planner reads Delta files from both OneLake paths in parallel, performs a distributed hash join, and returns the result in seconds with no prior ETL required.

**Free Resources:**
- [Fabric Cross-Database Query Documentation](https://learn.microsoft.com/en-us/fabric/data-warehouse/query-warehouse) — three-part name syntax, federated queries, and cross-item access permissions
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — OneLake shortcuts, cross-workspace queries, and warehouse federation patterns

---

## Warehouse vs Lakehouse

**Status:** ⬜ Not Started

**Definition:** In Fabric, choose a Warehouse when you need full T-SQL DML support (UPDATE, DELETE, MERGE), traditional warehouse semantics, stored procedures, and governed schema management with transactions. Choose a Lakehouse when you need Spark-based processing, file operations, schema-on-read flexibility, or mixed Python and SQL workflows.

**Key Mental Model:** Warehouse is for the SQL-first analyst or BI engineer — familiar T-SQL, full DML, governed. Lakehouse is for the data engineer or data scientist — code-first, Spark-native, more flexible with raw and semi-processed data.

**How It Works:**
- The Warehouse provides a **schema-on-write, full DML** experience: tables must be created with explicit DDL before data is written, and all DML operations (INSERT, UPDATE, DELETE, MERGE) commit transactionally to Delta files via the Warehouse engine — this mirrors traditional warehouse discipline and enforces structural consistency.
- The Lakehouse provides a **schema-on-read** experience for unmanaged files (`/Files/`) and a schema-on-write experience for managed Delta tables (`/Tables/`) — Spark notebooks can read and write arbitrary file formats and infer schemas at query time, enabling exploration of raw data before defining structured tables.
- The Lakehouse's **SQL Analytics Endpoint** provides read-only T-SQL access to managed Delta tables, but does not support DML or stored procedures — it is a query surface, not a warehouse engine; teams that start with a Lakehouse and need DML capabilities should migrate those workloads to a Warehouse or use Spark for writes.
- Both experiences store data as Delta Parquet in OneLake and share the same workspace capacity CU pool — the choice affects the authoring experience and write semantics, not the physical storage format; a table can start in a Lakehouse and be referenced from a Warehouse via cross-database query without moving the data.
- **Medallion architecture** in Fabric typically uses a Lakehouse for Bronze and Silver layers (raw ingest and Spark-based cleaning) and a Warehouse for the Gold layer (governed, SQL-first, DML-enabled final tables for BI) — the two experiences complement each other within a single Fabric workspace. See [[Cloud-Platforms/Fabric/02-Data-Engineering#Lakehouse]] for Lakehouse SQL Endpoint limitations.

**Common Misconceptions:**
- You are not forced to choose one — many Fabric architectures use both in the same workspace, with Lakehouses for data engineering stages and a Warehouse for the governed analytical layer; cross-database queries connect them without ETL.
- Lakehouse SQL Endpoint does not become a Warehouse by adding it to a Warehouse shortcut — the SQL Endpoint remains read-only and lacks stored procedures, transactions, and DML regardless of how it is referenced; to get full SQL write capabilities, the data must be moved to or created in a Warehouse item.

**Interview Answer Skeleton:**
- **What it is:** A choice between two Fabric compute experiences with different write semantics — Warehouse offers full DML, stored procedures, and schema-on-write T-SQL; Lakehouse offers Spark-native processing, schema-on-read file access, and a read-only SQL endpoint — both storing Delta Parquet in OneLake.
- **Why it matters / trade-offs:** Choosing incorrectly causes pain: using a Lakehouse for workloads requiring complex T-SQL DML requires Spark workarounds; using a Warehouse for Spark-heavy ETL pipelines adds unnecessary T-SQL overhead; the two experiences are complementary, not competing.
- **Example or context:** A data platform team uses a Lakehouse for raw JSON ingestion and Spark-based data cleaning (Bronze/Silver), then writes cleaned data to a Fabric Warehouse for the Gold layer where the finance team runs T-SQL stored procedures with MERGE operations for SCD Type 2 dimensions — both experiences share OneLake storage with no ETL between them.

**Free Resources:**
- [Lakehouse vs Warehouse Guide](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-vs-data-warehouse) — decision criteria, feature comparison, and recommended use cases
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — architecture guidance for medallion patterns using Lakehouse and Warehouse together

---

## COPY INTO

**Status:** ⬜ Not Started

**Definition:** COPY INTO is a T-SQL command in Fabric Warehouse for efficiently loading data from OneLake files (CSV, Parquet, JSON) directly into Warehouse tables. It handles schema mapping, error tolerance, and parallel file loading, and is the primary bulk data ingestion pattern for loading staged OneLake files into governed Warehouse tables.

**Key Mental Model:** COPY INTO is the loading dock for the Warehouse — it moves files sitting in OneLake into organised Warehouse tables efficiently, handling format conversion and error tolerance automatically.

**How It Works:**
- `COPY INTO` reads source files from an OneLake path (or external OneLake shortcut pointing to ADLS Gen2 or S3) and loads them into the target Warehouse table in a single parallelised operation — the Warehouse query engine spawns multiple reader threads that read different source files concurrently, then write partitioned Parquet files to the Warehouse's Delta storage in OneLake.
- **Schema mapping** is specified in the `COPY INTO` statement via a column mapping clause — source columns can be mapped to target columns by name or position, enabling loading files with different column ordering or extra columns that should be discarded.
- **Error handling** is controlled by `MAXERRORS` and `ERRORFILE` options — `MAXERRORS` specifies how many malformed rows are tolerated before the statement fails; rejected rows are written to an error file in a specified OneLake path for inspection and reprocessing, without failing the entire load.
- `COPY INTO` uses a **transactional commit model** consistent with Delta — if the statement succeeds, all rows are committed atomically to the Delta transaction log as a single commit; if it fails midway (exceeding `MAXERRORS` or a system error), the entire load is rolled back and no partial rows are committed to the table.
- For incremental loads, `COPY INTO` can be combined with a **file metadata column** (`$path`, `$modifieddate`) to filter which files from an OneLake stage path are loaded in each batch, enabling date-partitioned or checkpoint-based incremental loading patterns without a separate file tracking table. See [[Cloud-Platforms/Fabric/02-Data-Engineering#Data Factory Pipelines]] for orchestrating COPY INTO via pipeline stored procedure activities.

**Common Misconceptions:**
- COPY INTO does not support all file formats equally — Parquet loading is fastest because column formats map directly to the Delta Parquet target; CSV loading requires row parsing and type conversion overhead; JSON semi-structured loading has the highest parsing cost per row.
- COPY INTO is not an upsert operation — it appends or overwrites (with `OVERWRITE = TRUE`); for upsert semantics (matching existing rows and updating them), MERGE must be used after COPY INTO to load staging data and then merge into the target table.

**Interview Answer Skeleton:**
- **What it is:** A T-SQL bulk load statement in Fabric Warehouse that reads source files from OneLake in parallel, maps columns, tolerates a configurable number of errors, and commits results atomically to a Delta Parquet table via a single Delta transaction.
- **Why it matters / trade-offs:** The most efficient bulk ingestion path from staged OneLake files into governed Warehouse tables; the trade-off is that COPY INTO is append/overwrite only and does not replace the need for MERGE when loading mutable dimension or fact tables with SCD logic.
- **Example or context:** A pipeline loads daily sales CSV files from an S3 shortcut in OneLake using `COPY INTO sales_staging` — the Warehouse reads 500 CSV files in parallel, maps columns, rejects 3 malformed rows to an error file, and commits the clean rows atomically; the downstream MERGE step then upserts staging rows into the production `sales` fact table.

**Free Resources:**
- [Fabric COPY INTO Documentation](https://learn.microsoft.com/en-us/fabric/data-warehouse/ingest-data-copy) — syntax, format options, error handling, and incremental patterns
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — bulk load patterns, staging strategies, and Warehouse ingestion best practices

---

## Auto-Scale

**Status:** ⬜ Not Started

**Definition:** Fabric Warehouse compute automatically scales to handle concurrent query bursts using the shared capacity CU pool, and releases resources when demand drops. Unlike explicit virtual warehouse sizing models (Snowflake, Redshift), Fabric scales dynamically within the purchased capacity without manual size selection or cluster resizing.

**Key Mental Model:** Fabric auto-scale is an elastic staffing model — when queries pile up, more resources are pulled from the shared capacity pool automatically, and released when the rush is over.

**How It Works:**
- Fabric capacity's **smoothing engine** manages CU allocation: when a burst of concurrent Warehouse queries arrives, the Fabric orchestration layer allocates additional CUs from the capacity pool to the Warehouse workload beyond the workspace's baseline allocation, processing the burst at higher parallelism.
- The smoothing engine tracks a **rolling 24-hour consumption window** — if burst usage exceeds the capacity's allocation within the window, **throttling** is applied by queuing new queries rather than rejecting them; the queue clears as older consumption ages out of the 24-hour window, preventing permanent degradation.
- **Burstable capacity** allows short-duration spikes (up to 10x the base allocation for brief periods) without incurring throttling — this handles morning dashboard refresh storms where many Power BI reports trigger Warehouse queries simultaneously, without requiring the capacity to be sized for peak load permanently.
- Fabric's auto-scale model means there is **no manual warehouse size selection** as in Snowflake (XS/S/M/L/XL) or Redshift node resizing — the system expands and contracts transparently within the CU pool; the primary tuning lever is the capacity SKU size (F2 vs F64 vs F128) which sets the pool ceiling.
- The **Capacity Metrics app** shows Warehouse CU consumption over time alongside other Fabric workloads, enabling admins to identify whether throttling events are caused by Warehouse queries or competing Spark/Pipeline workloads and adjust capacity SKU accordingly. See [[Cloud-Platforms/Fabric/01-Architecture#Fabric Capacities]] for the overall capacity model.

**Common Misconceptions:**
- Fabric auto-scale does not mean unlimited performance — the capacity SKU is an absolute ceiling; if the capacity is genuinely undersized for the workload volume, throttling is permanent rather than transient, and upgrading the capacity SKU is the only remedy.
- Fabric Warehouse auto-scale does not isolate workloads — Spark jobs, Pipeline activities, and Power BI refreshes all consume CUs from the same pool; a large Spark ETL job running concurrently with a morning dashboard refresh can cause Warehouse queries to be throttled despite auto-scale, because the pool itself is constrained.

**Interview Answer Skeleton:**
- **What it is:** A Fabric capacity smoothing mechanism that allocates additional CUs from the shared pool to Warehouse queries during demand spikes, with burst tolerance up to the capacity SKU ceiling and a 24-hour rolling window throttle that queues queries when sustained consumption exceeds the capacity allocation.
- **Why it matters / trade-offs:** Eliminates manual warehouse resizing operations and absorbs burst demand automatically; the trade-off is that CU sharing with other Fabric workloads means Warehouse performance is not isolated — concurrent Spark or Pipeline workloads can reduce available CUs for Warehouse queries.
- **Example or context:** A Fabric Warehouse on an F64 capacity handles 100 simultaneous Power BI Direct Query requests each morning — the smoothing engine absorbs the burst by temporarily using additional CUs; by mid-morning when the rush subsides, CU consumption normalises without any administrator intervention or warehouse resizing.

**Free Resources:**
- [Fabric Burstable Capacity Documentation](https://learn.microsoft.com/en-us/fabric/data-warehouse/burstable-capacity) — burst limits, throttling behaviour, and 24-hour smoothing window mechanics
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — capacity metrics, workload management, and right-sizing guidance

---
