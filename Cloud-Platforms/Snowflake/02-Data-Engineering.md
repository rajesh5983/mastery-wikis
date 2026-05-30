# Snowflake — Data Engineering

---

## Snowpipe

**Status:** ⬜ Not Started

**Definition:** Snowpipe is Snowflake's continuous micro-batch ingestion service that automatically loads new files from cloud storage (S3, Azure Blob, GCS) as they arrive, using event notifications to trigger loads within seconds rather than waiting for a scheduled COPY INTO batch.

**Key Mental Model:** Snowpipe is an always-on loading dock — the moment a new file lands in the cloud storage bucket, Snowpipe is notified and loads it into the target table automatically, without a scheduled pipeline needing to poll.

**How It Works:**
- Snowpipe is triggered by cloud storage event notifications — AWS S3 sends an SQS event, Azure Blob sends an Event Grid notification, or you call Snowpipe's REST API directly. Each event carries the file path of the newly arrived file, which Snowpipe queues for loading.
- Internally, Snowpipe uses Snowflake-managed serverless compute (not your virtual warehouses) to execute COPY INTO operations. You are billed per-second for the compute Snowpipe uses, separate from your warehouse credits.
- Each Snowpipe is associated with a pipe object that specifies the target table, the source stage, and an optional file format. The pipe maintains a load history (queryable via `SYSTEM$PIPE_STATUS` and `COPY_HISTORY`) to avoid reloading the same file twice — files are deduplicated by path and modification time.
- For REST API-triggered Snowpipe, you call `insertFiles()` with a list of file paths; this is useful when you control the upload process and want to trigger loading without configuring cloud event infrastructure.
- Monitoring Snowpipe requires querying `INFORMATION_SCHEMA.LOAD_HISTORY` or the `SNOWPIPE_STREAMING_CLIENT_HISTORY` views. Snowpipe does not retry failed individual rows — files that partially fail leave their failed rows accessible via error rejection tables if configured.

**Common Misconceptions:**
- Snowpipe provides real-time row-level streaming — Snowpipe is a micro-batch file loader that operates in seconds to minutes per file; for true row-level streaming, Snowflake Streaming (the Snowpipe Streaming API) is the correct tool, but it requires the Kafka connector or SDK, not the standard Snowpipe object.
- Snowpipe uses your virtual warehouse credits — Snowpipe runs on Snowflake-managed serverless compute billed separately at a per-second rate; it does not consume credits from your named warehouses.

**Interview Answer Skeleton:**
- **What it is:** A serverless, event-driven file loading service that listens for cloud storage notifications (S3/Blob/GCS), queues new files automatically, and executes COPY INTO using Snowflake-managed compute — eliminating the need for scheduled batch pipelines for file-based ingestion.
- **Why it matters / trade-offs:** Snowpipe reduces data latency from hours (scheduled batch) to seconds without requiring pipeline orchestration infrastructure. The trade-off is that it is file-granular — small, frequent files increase per-file overhead and cost more than fewer large files; optimising file size is essential for efficient Snowpipe usage.
- **Example or context:** An e-commerce platform receives order event files from Kinesis Firehose into S3 every 60 seconds. Snowpipe auto-ingests each file as it lands, keeping the orders table within 90 seconds of real time — without a single Airflow DAG or Lambda function managing the load schedule.

**Free Resources:**
- [Snowflake Snowpipe Documentation](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro) — Snowpipe documentation covering setup, event notification configuration, and monitoring
- [Snowflake Snowpipe Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart guides covering Snowpipe setup with S3 event notifications and REST API ingestion

---

## Streams and Tasks

**Status:** ⬜ Not Started

**Definition:** A Snowflake Stream is a CDC (change data capture) object that tracks row-level DML changes (INSERT, UPDATE, DELETE) on a table. A Task is a scheduled or event-triggered SQL or stored procedure execution. Together, Streams + Tasks enable lightweight ELT pipelines entirely within Snowflake.

**Key Mental Model:** A Stream is a change log that records what happened to a table; a Task is a scheduled worker that processes that change log. Together they are a built-in mini-Airflow for simple Snowflake-native pipelines.

**How It Works:**
- A Stream records the offset (position) in a table's change history at the time the stream is created. Subsequent DML to the source table is captured in the stream as rows with metadata columns: `METADATA$ACTION` (INSERT/DELETE), `METADATA$ISUPDATE` (boolean), and `METADATA$ROW_ID` (unique row identifier). Reading the stream advances the offset; unconsumed changes accumulate.
- Streams use table versioning under the covers — Snowflake's Time Travel mechanism tracks the before/after state of each changed row. Stream retention is tied to the source table's Time Travel retention period; a stream not consumed within the retention window goes stale and must be recreated.
- A Task schedules SQL or stored procedure execution on a cron schedule (using standard cron syntax) or triggers on a predecessor task completing (task DAG). Tasks can be configured to only run when a stream contains data (`WHEN SYSTEM$STREAM_HAS_DATA(stream_name) = TRUE`), avoiding empty runs.
- Task DAGs allow chaining: a root task triggers on schedule, child tasks execute when their parent succeeds. This enables multi-step ELT — e.g., Task 1 reads the raw stream and inserts to a staging table, Task 2 merges from staging to the final table.
- Serverless tasks run on Snowflake-managed compute (billed per-second); user-managed tasks run on a specified virtual warehouse. Serverless tasks are preferred for intermittent, short-duration workloads where warehouse startup time would dominate a user-managed approach.

**Common Misconceptions:**
- Streams capture schema changes — Streams only capture DML changes (INSERT, UPDATE, DELETE); DDL changes (ALTER TABLE, column additions) are not captured and require separate handling.
- Streams provide guaranteed delivery — if a stream goes stale (not consumed within the Time Travel retention window), the change history is lost and the stream must be recreated, re-reading the entire source; monitoring stream staleness is essential in production.

**Interview Answer Skeleton:**
- **What it is:** A Snowflake-native CDC mechanism (Streams) combined with a scheduling engine (Tasks) that enables incremental ELT pipelines within Snowflake — capturing row-level changes to source tables and processing them on a cron or event-driven schedule.
- **Why it matters / trade-offs:** Streams + Tasks eliminate the need for external orchestration tools for simple incremental loads within Snowflake, reducing pipeline complexity and latency. The trade-off is limited expressiveness compared to dedicated orchestrators — no dynamic parallelism, limited error handling, and stream staleness creates operational risk if monitoring is not in place.
- **Example or context:** A customer 360 pipeline uses a Stream on the raw_events table to capture new event rows, and a Task every 5 minutes that merges new events into the customer_summary table using `METADATA$ACTION` to handle deletes. When no new rows exist, the task skips execution via `SYSTEM$STREAM_HAS_DATA` — zero wasted credits.

**Free Resources:**
- [Snowflake Streams Documentation](https://docs.snowflake.com/en/user-guide/streams-intro) — Snowflake Streams documentation covering stream types, consumption patterns, and staleness management
- [Snowflake Tasks Documentation](https://docs.snowflake.com/en/user-guide/tasks-intro) — Snowflake Tasks documentation covering scheduling, DAG configuration, and serverless vs user-managed compute

---

## Dynamic Tables

**Status:** ⬜ Not Started

**Definition:** Dynamic Tables are Snowflake's declarative pipeline feature — you define the target transformation as a SELECT query, set a target lag (e.g., 1 minute, 1 hour), and Snowflake automatically keeps the table up to date by incrementally refreshing it. They replace complex Streams + Tasks logic for many use cases.

**Key Mental Model:** Dynamic Tables are materialised views with a freshness guarantee — you declare what the table should look like (the query) and how stale it can be (the lag), and Snowflake figures out the refresh schedule and incremental computation.

**How It Works:**
- You create a Dynamic Table with `CREATE DYNAMIC TABLE ... TARGET_LAG = '1 minute' WAREHOUSE = my_wh AS SELECT ...`. Snowflake schedules incremental refreshes to maintain the lag guarantee — if lag is 1 minute, Snowflake ensures the table is at most 1 minute behind the source at all times.
- Refreshes are incremental where possible: Snowflake tracks which source rows changed since the last refresh and recomputes only the affected output rows. For queries that support incremental computation (simple joins, aggregations, filters), this dramatically reduces the compute cost per refresh compared to a full recomputation.
- Dynamic Tables can be chained — a downstream Dynamic Table can reference an upstream Dynamic Table, creating a declarative transformation pipeline. Snowflake determines the end-to-end refresh order automatically, propagating freshness from upstream to downstream tables.
- The `TARGET_LAG` is a maximum staleness guarantee, not a refresh interval. Snowflake may refresh more frequently if source write volume is high. If the query cannot be refreshed within the lag (e.g., due to warehouse contention), Snowflake emits a warning but does not fail.
- Monitoring uses `DYNAMIC_TABLE_REFRESH_HISTORY` and `DYNAMIC_TABLE_GRAPH_HISTORY` views — queryable in `INFORMATION_SCHEMA` — to inspect refresh success, latency, bytes scanned, and credits consumed per refresh cycle.

**Common Misconceptions:**
- Dynamic Tables replace all Streams + Tasks use cases — Dynamic Tables are ideal for transformation pipelines with stable queries; Streams + Tasks remain necessary for procedural logic, multi-step MERGE operations with conditional branching, or non-SELECT transformations that Dynamic Tables cannot express as a single query.
- Dynamic Tables are always incremental — queries involving certain operations (cross joins, UDFs, non-deterministic functions) fall back to full refresh; the incremental optimisation is query-structure-dependent and not guaranteed.

**Interview Answer Skeleton:**
- **What it is:** A declarative pipeline primitive in Snowflake where you define a transformation query and a freshness lag, and Snowflake handles the incremental refresh scheduling automatically — replacing manual Streams + Tasks orchestration for query-expressible pipelines.
- **Why it matters / trade-offs:** Dynamic Tables dramatically reduce pipeline boilerplate for incremental refresh use cases — no task scheduling, no stream management, no staleness monitoring. The trade-off is expressiveness: only SQL-expressible transformations are supported, and full refresh fallback for complex queries can be expensive if not anticipated.
- **Example or context:** A finance team defines a Dynamic Table `daily_revenue_by_product` with a 5-minute lag over the raw transactions table. As transactions arrive via Snowpipe, the Dynamic Table refreshes incrementally within 5 minutes — without a single Streams + Tasks setup — replacing what would have been a 40-line SQL task pipeline with a 5-line Dynamic Table definition.

**Free Resources:**
- [Snowflake Dynamic Tables Documentation](https://docs.snowflake.com/en/user-guide/dynamic-tables-intro) — Snowflake Dynamic Tables documentation covering definition, lag, incremental refresh, and monitoring
- [Snowflake Dynamic Tables Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart guides for building Dynamic Table pipelines and chaining transformations

---

## Iceberg Tables

**Status:** ⬜ Not Started

**Definition:** Snowflake Iceberg Tables store data in open Apache Iceberg format on your own cloud storage rather than in Snowflake's proprietary storage. This allows Snowflake to query data that other engines (Spark, Flink, Trino) also access, enabling a multi-engine lakehouse architecture with Snowflake as one of the query engines.

**Key Mental Model:** Iceberg Tables are Snowflake's "bring your own storage" option — data lives in your bucket in open format, Snowflake reads it as a first-class citizen, but you're not locked in to Snowflake-only access.

**How It Works:**
- An Iceberg Table in Snowflake writes Parquet data files and Iceberg metadata files (manifest lists, manifests, snapshot JSON) to a customer-owned cloud storage location (S3, ADLS, GCS). Snowflake acts as the catalog — managing the Iceberg metadata — while the storage is external.
- Two catalog modes exist: Snowflake-managed catalog (Snowflake writes and tracks Iceberg metadata, other engines read via catalog integration) and external catalog (Glue, Polaris, or a REST catalog manages metadata; Snowflake registers and reads via an external catalog integration). The catalog mode determines who controls the write path.
- When Snowflake writes to an Iceberg Table, it creates optimised Parquet files with column statistics and partitioning metadata in the Iceberg manifest. Other engines (Spark, Trino, Flink) read the same files directly — no data conversion or copy needed.
- Performance characteristics differ from native Snowflake tables: Iceberg Tables lack Snowflake's proprietary micro-partition format and automatic clustering, so query pruning depends on Iceberg partition spec and file layout rather than Snowflake's native micro-partitioning. Manual partition definition is important for performance.
- Iceberg Tables support full DML (INSERT, UPDATE, DELETE, MERGE) in Snowflake for Snowflake-managed catalogs. External-catalog Iceberg Tables are read-only from Snowflake — writes must go through the owning engine.

**Common Misconceptions:**
- Iceberg Tables perform identically to native Snowflake tables — native Snowflake tables benefit from automatic micro-partitioning, clustering, and metadata that Iceberg tables don't have by default; query performance on Iceberg Tables often requires explicit partitioning design and manual optimisation.
- Using Iceberg Tables eliminates Snowflake storage costs — Iceberg Tables shift storage billing to your cloud provider (S3/ADLS/GCS), but Snowflake still charges compute for queries; the storage cost profile changes but doesn't disappear.

**Interview Answer Skeleton:**
- **What it is:** Snowflake tables backed by open Apache Iceberg format on customer-owned cloud storage — enabling multi-engine access to the same data from Spark, Flink, Trino, and Snowflake simultaneously, with Snowflake managing or reading the Iceberg catalog.
- **Why it matters / trade-offs:** Iceberg Tables eliminate vendor lock-in for storage and enable hybrid architectures where Snowflake coexists with open-source query engines on shared data. The trade-off is that performance optimisation requires explicit Iceberg partitioning design; the automatic micro-partition optimisation of native Snowflake tables is not available.
- **Example or context:** A data platform team stores clickstream data as Iceberg Tables in S3, written by Spark streaming jobs. Snowflake queries the same tables for ad-hoc analytics and data science; Flink reads them for real-time aggregations. No data is copied between systems — all engines share the same Parquet files via the Iceberg catalog, with Snowflake providing SQL governance and BI tooling access.

**Free Resources:**
- [Snowflake Iceberg Tables Documentation](https://docs.snowflake.com/en/user-guide/tables-iceberg) — Snowflake Iceberg Tables documentation covering setup, catalog modes, DML support, and integration with external engines
- [Snowflake Iceberg Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for creating Iceberg Tables in Snowflake with S3 storage and cross-engine access patterns

---

## Stages and File Formats

**Status:** ⬜ Not Started

**Definition:** Stages are named cloud storage locations in Snowflake — either internal (Snowflake-managed storage) or external (your S3/ADLS/GCS bucket). File Formats define how files should be parsed (CSV delimiter, JSON strip nulls, Parquet compression). COPY INTO uses stages and file formats to load data into tables.

**Key Mental Model:** A Stage is the loading bay — where files wait before being loaded. A File Format is the instruction sheet for how to read each file type. Together they make COPY INTO statements reusable and configurable.

**How It Works:**
- Internal stages (`@~` for user stage, `@%table_name` for table stage, or named stages) store files in Snowflake-managed cloud storage. Files are uploaded via `PUT` command from SnowSQL or the Snowflake connector, and are automatically encrypted and compressed. Internal stages are useful for one-off loads or when you don't have a cloud storage bucket.
- External stages reference customer-managed cloud storage buckets with a storage integration (IAM role for S3, service principal for ADLS, GCS service account). The stage object stores the connection credentials securely; COPY INTO references the stage by name without embedding credentials in the SQL.
- Named file formats (`CREATE FILE FORMAT my_csv TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1 NULL_IF = ('NULL', 'null')`) encapsulate parsing options that would otherwise be specified inline on every COPY INTO statement. File formats support CSV, JSON, AVRO, ORC, PARQUET, and XML.
- `COPY INTO` reads from stage using the file format and supports transformation at load time — column reordering, basic expressions, and SELECT sub-clauses allow light transformation without a separate ELT step. `COPY INTO ... FROM (SELECT $1::INT, $2::DATE FROM @stage)` applies casting during load.
- `LIST @stage_name` shows files currently in a stage; `COPY_HISTORY` view in `INFORMATION_SCHEMA` shows which files have been loaded, their load status, and error counts — essential for auditing and re-loading failed files.

**Common Misconceptions:**
- External stages store data in Snowflake — external stages are pointers to your cloud storage; Snowflake reads from (and optionally writes to) that location but does not copy data into its own storage unless you explicitly run COPY INTO a table.
- File formats must be specified for every COPY INTO — named file formats can be referenced by name, and stages can have a default file format attached; COPY INTO then inherits the format without specifying it inline.

**Interview Answer Skeleton:**
- **What it is:** Named cloud storage references (Stages) paired with parsing configuration objects (File Formats) that provide a reusable, parameterised interface for COPY INTO bulk loading — decoupling storage location and file parsing from the load SQL.
- **Why it matters / trade-offs:** Stages and file formats encapsulate the operational complexity of cloud storage access and file parsing into reusable Snowflake objects — credentials are managed by storage integrations, parsing rules are centralised, and COPY INTO statements remain portable across environments. The trade-off is the initial setup overhead for storage integrations, especially managing IAM trust relationships across cloud providers.
- **Example or context:** A data team creates an external stage pointing to their S3 landing bucket with an IAM role integration, a named CSV file format for their vendor files (pipe-delimited, UTF-8, first row header). Every downstream pipeline team uses `COPY INTO target_table FROM @vendor_stage FILE_FORMAT = (FORMAT_NAME = vendor_csv)` — credentials are never embedded, and file format changes are made once at the named format level.

**Free Resources:**
- [Snowflake Data Loading Overview](https://docs.snowflake.com/en/user-guide/data-load-overview) — Snowflake data loading documentation covering stages, file formats, and COPY INTO with transformation options
- [Snowflake Loading Data Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for staging files and running COPY INTO with various file formats and transformations

---

## Time Travel

**Status:** ⬜ Not Started

**Definition:** Snowflake Time Travel allows querying data as it existed at any point within the retention period (1 day by default, up to 90 days on Enterprise). You can query historical data with `AT (TIMESTAMP => ...)`, restore accidentally dropped or modified tables, and create clones from any historical point.

**Key Mental Model:** Time Travel is an undo button for your data — not just one level back, but any point within the retention window. Accidentally deleted a table? Travel back 30 minutes and restore it.

**How It Works:**
- Snowflake maintains historical versions of data through its micro-partition immutability model — when rows are updated or deleted, Snowflake writes new micro-partitions and marks the old ones as logically deleted but physically retained for the Time Travel retention period. No separate audit log is maintained; the versioning is intrinsic to the storage format.
- Time Travel queries use the `AT` or `BEFORE` clause: `SELECT * FROM orders AT (TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP)` or `AT (OFFSET => -3600)` (seconds before now) or `AT (STATEMENT => 'query_id')` (as of a specific query execution). The query reads the micro-partitions valid at that historical point.
- Dropped tables are recoverable with `UNDROP TABLE table_name` within the retention period — Snowflake retains the table definition and data micro-partitions even after DROP TABLE. After the retention period expires, the table is permanently purged.
- Table cloning leverages Time Travel — `CREATE TABLE new_table CLONE source_table AT (TIMESTAMP => ...)` creates a zero-copy clone of the table at any historical point. This is the fastest way to recover accidentally modified data: clone the pre-modification version, verify, then SWAP to replace the damaged table.
- Time Travel retention is configurable at table, schema, or database level via `DATA_RETENTION_TIME_IN_DAYS`. Standard edition supports 0-1 days; Enterprise edition supports 0-90 days. Storage costs accrue for retained historical micro-partitions — longer retention periods increase storage consumption proportionally.

**Common Misconceptions:**
- Time Travel queries historical audit logs — Time Travel reads the actual historical micro-partitions of data, not a separate audit log; it shows the complete row state at any historical point, not just which queries ran or which users made changes.
- Fail-safe and Time Travel are the same — Time Travel (0-90 days, user-accessible) is the operational recovery window; Fail-safe is an additional 7-day non-user-accessible period after Time Travel expires, managed only by Snowflake Support for disaster recovery scenarios.

**Interview Answer Skeleton:**
- **What it is:** A built-in data versioning capability in Snowflake's storage layer that retains historical micro-partitions for a configurable period (up to 90 days on Enterprise), enabling point-in-time queries, accidental data recovery via UNDROP, and zero-copy historical cloning.
- **Why it matters / trade-offs:** Time Travel provides data recovery and audit capabilities without any additional infrastructure — no separate backup jobs, no log tables, no manual versioning. The trade-off is storage cost: historical micro-partitions are retained and billed at standard Snowflake storage rates, making 90-day retention on large, high-write tables expensive.
- **Example or context:** A data engineer accidentally runs `DELETE FROM customer_dim WHERE region = 'EU'` without a WHERE clause filter, deleting all rows. Using Time Travel, they clone the table from 10 minutes before the DELETE, verify the recovered data, and SWAP the clone with the damaged table — total recovery time under 5 minutes, zero data loss.

**Free Resources:**
- [Snowflake Time Travel Documentation](https://docs.snowflake.com/en/user-guide/data-time-travel) — Snowflake Time Travel documentation covering syntax, use cases, retention configuration, and Fail-safe interaction
- [Snowflake Cloning and Time Travel Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for Time Travel queries, UNDROP, and zero-copy cloning from historical snapshots
