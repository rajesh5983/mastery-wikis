# Databricks — Data Engineering

---

## Delta Live Tables (DLT)

**Status:** ⬜ Not Started

**Definition:** Delta Live Tables is a declarative pipeline framework in Databricks where you define transformations using Python or SQL, and DLT automatically manages execution order, retries, data quality enforcement, and incremental processing. You declare *what* you want; DLT figures out *how* to execute it. DLT pipelines run on dedicated compute managed by Databricks, separate from general-purpose clusters.

**Key Mental Model:** DLT is like a recipe system that manages its own kitchen — you write the recipes (transformations), and the system decides the cooking order, restarts failed steps, and tracks quality automatically.

**How It Works:**
- When a DLT pipeline is triggered, the DLT engine performs **graph analysis**: it parses all dataset definitions, identifies dependencies between `LIVE` tables and views, and constructs a directed acyclic graph (DAG) that determines execution order.
- Each node in the DAG is either a **Streaming Table** (maintains a checkpoint and processes only new data incrementally) or a **Materialized View** (recomputed on refresh based on upstream changes). DLT tracks which source records have been processed via a persistent checkpoint stored in cloud storage.
- **Expectations** (`CONSTRAINT` clauses) are evaluated at row level during the transformation — failing rows can be dropped, quarantined to a separate delta table, or cause the pipeline to fail, giving teams declarative data quality SLAs without custom exception handling code.
- DLT pipelines run in either **Triggered** mode (run once and stop) or **Continuous** mode (keep a streaming query running indefinitely); the former uses Delta-based micro-batch Structured Streaming, the latter maintains a live Kafka-like polling loop.
- The DLT event log captures all pipeline runs, data quality metrics, and row counts as a queryable Delta table, enabling operational dashboards without external monitoring infrastructure. See [[Cloud-Platforms/Databricks/05-Administration]] for pipeline monitoring patterns.

**Common Misconceptions:**
- DLT is not just a scheduler — it manages incremental state, checkpoints, and schema evolution automatically; using plain Workflows to run notebook transformations does not give you these guarantees.
- "Continuous mode is always better than triggered mode" is false — continuous mode holds a Spark cluster running 24/7, which is expensive; triggered mode is far more cost-efficient for hourly or daily batch pipelines.

**Interview Answer Skeleton:**
- **What it is:** A declarative, managed pipeline framework that builds a dependency DAG from dataset definitions, manages incremental checkpoints, enforces data quality constraints, and provisions dedicated compute for each pipeline run.
- **Why it matters / trade-offs:** Reduces pipeline boilerplate significantly and catches data quality issues at the framework level; the trade-off is that DLT's managed compute means less control over cluster configuration and higher per-DBU cost than hand-tuned Spark jobs.
- **Example or context:** A team replaces a fragile notebook-chained ETL with a DLT pipeline — DLT automatically skips re-processing files already ingested, applies `NOT NULL` constraints on critical keys, and restarts failed tasks from the last checkpoint rather than the beginning of the run.

**Free Resources:**
- [Delta Live Tables Documentation](https://docs.databricks.com/en/delta-live-tables/index.html) — pipeline definition, quality constraints, monitoring, and incremental processing reference
- [Databricks Academy](https://academy.databricks.com) — free courses covering DLT pipeline design patterns and production deployment

---

## Auto Loader

**Status:** ⬜ Not Started

**Definition:** Auto Loader is a Databricks feature that incrementally and efficiently ingests new files from cloud storage as they arrive, using file notification or directory listing. It automatically handles schema inference, evolution, and checkpoint management for scalable file-based ingestion from S3, ADLS Gen2, or GCS.

**Key Mental Model:** Auto Loader is a mail sorter that processes new mail automatically as it arrives — no manual intervention to pick up, open, and sort; it handles new files continuously without rescanning everything already processed.

**How It Works:**
- Auto Loader uses two discovery modes: **File Notification mode** sets up cloud-native event subscriptions (SQS + S3 Event Notifications on AWS, Event Grid + Queue Storage on Azure) so new file arrivals trigger an event rather than requiring directory scans — this scales to millions of files per hour.
- **Directory Listing mode** is the fallback that periodically lists the source directory and compares against a checkpoint to find new files; simpler to set up but less scalable and adds cloud API cost at high file counts.
- A persistent **checkpoint directory** in cloud storage records the state of which files have been processed; if a streaming query restarts, Auto Loader reads the checkpoint and resumes from exactly where it stopped, guaranteeing exactly-once file ingestion.
- **Schema inference** samples a subset of new files to infer schema on first run; **schema evolution** (`cloudFiles.schemaEvolutionMode`) can be set to `rescue` (add new columns to a side table), `addNewColumns` (evolve the target schema), or `failOnNewColumns` — giving teams fine-grained control over how schema drift is handled.
- Auto Loader integrates natively with DLT Streaming Tables (`STREAM(cloudFiles(...))`) so incremental file ingestion feeds directly into the DLT pipeline DAG without manual checkpoint management. See [[Cloud-Platforms/Databricks/02-Data-Engineering#Delta Live Tables (DLT)]] for pipeline integration.

**Common Misconceptions:**
- Auto Loader is not equivalent to a full CDC or Kafka consumer — it detects new files arriving in object storage, not row-level changes to existing files; for table-level CDC, use [[Cloud-Platforms/Databricks/02-Data-Engineering#Change Data Feed (CDF)]].
- Schema inference does not re-run on every batch — Auto Loader infers schema once (or on explicit schema location update) and caches it; without configuring schema evolution, unexpected columns in new files will silently be dropped rather than causing an error.

**Interview Answer Skeleton:**
- **What it is:** A Structured Streaming source that uses cloud-native file event notifications to incrementally ingest new files from object storage with persistent checkpointing, schema inference, and schema evolution built in.
- **Why it matters / trade-offs:** Eliminates the need to build custom file tracking logic and handles schema drift gracefully; the trade-off is that File Notification mode requires cloud-side infrastructure (SQS queues, event subscriptions) to be set up, adding operational complexity.
- **Example or context:** A payments team ingests thousands of JSON files per hour from an S3 landing zone — Auto Loader with File Notification mode processes each new file within seconds of arrival, with schema rescue capturing unexpected fields without failing the pipeline.

**Free Resources:**
- [Auto Loader Documentation](https://docs.databricks.com/en/ingestion/auto-loader/index.html) — file notification setup, schema inference, evolution modes, and checkpoint configuration
- [Databricks Academy](https://academy.databricks.com) — free ingestion-focused learning paths covering Auto Loader best practices

---

## Workflows (Databricks Jobs)

**Status:** ⬜ Not Started

**Definition:** Databricks Workflows is the native orchestration service for scheduling and running multi-task pipelines — notebook runs, DLT pipelines, dbt tasks, Spark JAR jobs, and more. It manages task dependencies, retries, alerting, and compute provisioning within the Databricks ecosystem.

**Key Mental Model:** Databricks Workflows is the built-in project manager — it sequences your tasks, starts the right compute for each, retries failures, and tells you when something breaks without requiring a separate orchestration tool.

**How It Works:**
- A Workflow job is defined as a DAG of tasks; each task specifies its type (notebook, DLT pipeline, dbt, JAR, Python script, SQL query), its compute (Job Cluster, existing All-Purpose Cluster, or Serverless), and its dependencies on upstream tasks using `depends_on` edges.
- At run time, the Workflows engine resolves the dependency graph and dispatches tasks to compute in topological order — independent branches execute in parallel on separate clusters concurrently, maximising throughput.
- **Job Clusters** are automatically provisioned when the task starts and torn down when it completes; the job definition specifies the full cluster config (instance type, DBR version, libraries), ensuring clean, reproducible execution environments with no shared state between runs.
- **Retry policies** allow per-task configuration of max retries and retry interval; tasks can also be conditionally skipped or branched using `if/else` task constructs, enabling dynamic pipeline logic such as skipping downstream tasks when upstream data has no new records.
- Workflow run history, task logs, and Spark UI links are stored in the Jobs service and accessible via the Workflows UI and REST API, enabling monitoring without external log aggregation. See [[Cloud-Platforms/Databricks/05-Administration]] for cost management with Job Clusters.

**Common Misconceptions:**
- Databricks Workflows does not replace full-featured orchestration tools like Apache Airflow for complex cross-system pipelines — it excels at Databricks-native task chains but lacks Airflow's ecosystem of operators for external systems (databases, APIs, SaaS tools).
- Each task in a multi-task Workflow can use a different cluster type and size — engineers should not assume a single shared cluster runs all tasks; this flexibility enables right-sizing compute per task type.

**Interview Answer Skeleton:**
- **What it is:** A native Databricks DAG orchestration engine that manages task sequencing, compute provisioning, retries, and monitoring for multi-step pipelines combining notebooks, DLT, dbt, and arbitrary Spark workloads.
- **Why it matters / trade-offs:** Eliminates the need for a separate orchestration tool for Databricks-internal workflows and handles compute lifecycle automatically; the trade-off is limited support for cross-system orchestration compared to Airflow or Prefect.
- **Example or context:** A data platform team replaces an Airflow DAG that ran Databricks notebooks via the REST API with a native Databricks Workflow — task dependencies are visible in the UI, each task gets its own auto-terminating Job Cluster, and failures trigger Slack alerts without custom plugins.

**Free Resources:**
- [Databricks Workflows Documentation](https://docs.databricks.com/en/jobs/index.html) — task types, scheduling, dependency management, and retry configuration
- [Databricks Academy](https://academy.databricks.com) — free orchestration courses covering Workflow design patterns and compute cost optimisation

---

## Structured Streaming

**Status:** ⬜ Not Started

**Definition:** Structured Streaming is the Spark-based streaming engine in Databricks that treats live data streams as continuously appended tables. Queries written as batch DataFrame operations automatically run incrementally on streaming data, simplifying the transition from batch to streaming. It supports micro-batch and continuous processing modes with end-to-end exactly-once semantics.

**Key Mental Model:** Structured Streaming treats the data stream as an infinite table that's always being appended to — your query runs continuously against new data, processing it in micro-batches or continuously, using the same SQL/DataFrame API as batch.

**How It Works:**
- The streaming engine maintains a **WAL (Write-Ahead Log) checkpoint** in cloud storage that records the offset range processed in each micro-batch trigger and the offsets committed to the sink — this is what enables exactly-once delivery even if the Spark driver crashes mid-batch.
- In **micro-batch mode** (default), the engine wakes on a configurable trigger interval, reads the next range of offsets from the source (Kafka topic, Delta table, Auto Loader), runs the query as a standard Spark job, and writes results to the sink atomically before advancing the checkpoint.
- **Stateful operations** (aggregations, joins, deduplication with `dropDuplicates`) require the engine to maintain state across micro-batches using a **StateStore** — a versioned key-value store backed by RocksDB or HDFS that persists state to cloud storage for recovery after failures.
- **Watermarks** define how late data is tolerated: setting `withWatermark("event_time", "10 minutes")` tells the engine to keep state for a window until the maximum event time seen exceeds the window boundary by 10 minutes, after which late records for that window are dropped and state is evicted.
- Structured Streaming integrates with Delta Lake as both a source (via `readStream` on a Delta table) and a sink (via `writeStream`), enabling Delta-to-Delta pipelines that inherit Delta's ACID commit guarantees per micro-batch. See [[Cloud-Platforms/Databricks/02-Data-Engineering#Delta Live Tables (DLT)]] for DLT's use of Structured Streaming under the hood.

**Common Misconceptions:**
- "Structured Streaming has lower latency than batch" is true only in context — micro-batch mode has trigger latency of seconds to minutes; for sub-second latency, Continuous Processing mode or a dedicated stream processor (Flink) is needed.
- Exactly-once semantics require both idempotent sinks AND checkpoint integrity; if you delete the checkpoint directory, the engine loses its offset tracking and may reprocess or skip data even with an idempotent sink.

**Interview Answer Skeleton:**
- **What it is:** An incremental query execution engine built on Spark that processes streaming data sources in micro-batches using the same DataFrame/SQL API as batch, with checkpointed state management for exactly-once delivery and stateful operations.
- **Why it matters / trade-offs:** Dramatically reduces the complexity of streaming pipelines by reusing batch query logic; the trade-off is that the JVM-based stateful engine has higher memory overhead than purpose-built stream processors like Flink for complex multi-key join patterns.
- **Example or context:** A logistics platform streams shipment events from Kafka — a Structured Streaming job joins each event with a static lookup Delta table, aggregates counts per warehouse per 5-minute window with a 2-minute watermark, and writes results to a Delta table powering a live operations dashboard.

**Free Resources:**
- [Structured Streaming Documentation](https://docs.databricks.com/en/structured-streaming/index.html) — Databricks-specific guide covering triggers, checkpoints, Delta integration, and stateful operations
- [Databricks Academy](https://academy.databricks.com) — free streaming courses covering watermarks, state management, and production streaming patterns

---

## Change Data Feed (CDF)

**Status:** ⬜ Not Started

**Definition:** Delta Lake's Change Data Feed captures row-level changes (insert, update, delete) to Delta tables, making them available for downstream consumers as a CDC stream. This enables efficient incremental processing of Delta table changes without scanning the full table, and is particularly useful for propagating changes to downstream Silver or Gold tables.

**Key Mental Model:** CDF is a changelog for your Delta table — instead of checking every row for changes, downstream pipelines subscribe to the feed and process only the rows that changed since their last read.

**How It Works:**
- When CDF is enabled on a Delta table (`delta.enableChangeDataFeed = true`), every write operation (INSERT, UPDATE, DELETE, MERGE) causes the Delta writer to record change rows in a dedicated `_change_data/` subdirectory alongside the table's Parquet data files.
- Change records include a `_change_type` column (`insert`, `update_preimage`, `update_postimage`, `delete`) along with `_commit_version` and `_commit_timestamp`, giving downstream consumers full before/after row values for updates and precise temporal ordering.
- Downstream consumers read the CDF using `spark.readStream.format("delta").option("readChangeFeed", "true").option("startingVersion", N)` — the `startingVersion` acts as an offset; the consumer tracks the last processed version in its own checkpoint or metadata store.
- CDF data is stored within the Delta table's transaction log scope and is subject to the same `VACUUM` retention policy — if a consumer falls behind by more than the retention period (default 7 days), its starting version may be vacuumed away, requiring a full re-sync.
- CDF integrates naturally with DLT Streaming Tables: a downstream DLT node can `STREAM(table)` and receive only changed rows since the last pipeline run, without the consumer needing to implement its own change detection logic. See [[Cloud-Platforms/Databricks/01-Architecture#Delta Lake]] for transaction log mechanics.

**Common Misconceptions:**
- CDF is not a Kafka replacement — it does not provide persistent log retention beyond Delta's VACUUM window, lacks consumer group semantics, and is not designed for high-throughput pub/sub fan-out to many independent consumers.
- Enabling CDF increases write amplification: every write to the table now also writes change records to `_change_data/`, increasing storage costs and slightly increasing write latency; this trade-off should be evaluated per table.

**Interview Answer Skeleton:**
- **What it is:** A Delta Lake feature that writes row-level change records (inserts, update pre/post images, deletes) to a sidecar directory within the table, queryable as a bounded or streaming change stream keyed by commit version.
- **Why it matters / trade-offs:** Enables efficient incremental fan-out from a single Delta table to multiple downstream consumers without full table scans; the trade-off is added write overhead and the VACUUM retention constraint that requires consumers to stay current.
- **Example or context:** A data team maintains a `customers` Silver table updated via daily MERGE operations — downstream pipelines for fraud scoring and personalisation each read only the changed rows via CDF since their last run, reducing processing time from 2 hours (full table scan) to 3 minutes.

**Free Resources:**
- [Delta Change Data Feed Documentation](https://docs.databricks.com/en/delta/delta-change-data-feed.html) — enablement, change record schema, and consumption patterns
- [Delta Lake Open Source Documentation](https://docs.delta.io/latest/index.html) — CDF specification and compatibility notes across Delta Lake versions

---

## dbt on Databricks

**Status:** ⬜ Not Started

**Definition:** dbt (data build tool) runs natively on Databricks using the dbt-databricks adapter, executing SQL transformations against Delta Lake tables on Databricks SQL Warehouses or All-Purpose Clusters. This combines dbt's transformation testing and documentation framework with Databricks' compute and governance capabilities.

**Key Mental Model:** dbt on Databricks is the SQL transformation layer on top of the lakehouse — dbt writes and tests the transformation logic, Databricks Photon executes it at scale, and Unity Catalog governs the output tables.

**How It Works:**
- The `dbt-databricks` adapter connects to a Databricks SQL Warehouse or All-Purpose Cluster via the Databricks SQL connector, submitting compiled SQL statements and receiving results using the same wire protocol as JDBC/ODBC BI tools.
- dbt compiles model `.sql` files to `CREATE TABLE AS SELECT` or `INSERT OVERWRITE` statements targeting Delta Lake tables in the Unity Catalog namespace (`catalog.schema.table`), fully respecting the three-level namespace introduced in DBR 12+.
- **Incremental models** use dbt's `{{ is_incremental() }}` macro to generate a `MERGE INTO` or `INSERT INTO` statement rather than a full table rebuild, leveraging Delta Lake's MERGE capability for efficient upserts on large tables.
- **dbt tests** (`not_null`, `unique`, `accepted_values`, custom SQL assertions) are executed as separate SQL queries against the materialized Delta tables and return pass/fail metadata stored in the dbt artifacts — these complement but do not replace DLT Expectations for runtime data quality.
- dbt models can be orchestrated as a task type within Databricks Workflows, enabling dbt transformations to fit into a broader pipeline DAG alongside DLT, notebooks, and Spark JAR jobs. See [[Cloud-Platforms/Databricks/02-Data-Engineering#Workflows (Databricks Jobs)]] for orchestration integration.

**Common Misconceptions:**
- dbt does not execute Spark DataFrame operations — it only submits SQL; Python models in dbt are executed as Python scripts on the cluster, not as optimised Spark DataFrames, and are significantly slower for large transformations.
- dbt's lineage graph and Unity Catalog lineage are separate systems — dbt generates its own DAG-based documentation, but Unity Catalog's automatic lineage captures the actual SQL execution lineage separately; both can coexist but neither automatically populates the other.

**Interview Answer Skeleton:**
- **What it is:** An open-source SQL transformation framework whose `dbt-databricks` adapter submits compiled SQL to Databricks SQL Warehouses, materialising models as Delta Lake tables in the Unity Catalog namespace with built-in testing and documentation.
- **Why it matters / trade-offs:** Brings software engineering practices (version control, testing, CI/CD) to SQL transformations and integrates with Databricks' governance layer; the trade-off is that dbt's SQL-only paradigm requires Spark-native workarounds for complex transformations that are natural in PySpark DataFrames.
- **Example or context:** A data team manages their entire Gold layer in dbt, running incremental MERGE models on Databricks SQL Warehouses via Photon — each model has `unique` and `not_null` tests that fail the CI pipeline before deployment if data quality regresses.

**Free Resources:**
- [dbt Databricks Setup Guide](https://docs.getdbt.com/docs/core/connect-data-platform/databricks-setup) — adapter configuration, incremental strategy options, and Unity Catalog namespace setup
- [Databricks dbt Integration Documentation](https://docs.databricks.com/en/partners/prep/dbt.html) — Databricks-side configuration, SQL Warehouse selection, and Workflows integration

---
