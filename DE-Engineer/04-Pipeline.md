# Layer 4 — Pipeline

> **Framework:** Building reliable data movement systems with robust error handling and recovery.

---

## ETL vs ELT Patterns

**Status:** ⬜ Not Started

**Definition:** ETL (Extract-Transform-Load) applies transformations before loading data into the target system, requiring a separate compute environment between source and destination. ELT (Extract-Load-Transform) loads raw data directly into the warehouse first, then leverages the warehouse's own compute power to transform it — the default architecture in cloud-native data stacks.

**Key Mental Model:** ETL is cooking before serving — clean, portion, and season every ingredient in the kitchen before it reaches the table. ELT is a buffet — bring everything raw to the table and let each station transform what it needs in place, using the industrial kitchen equipment already built into the venue.

**How It Works:**
- In ETL, a dedicated transformation engine (Spark, Informatica, custom Python) reads source data, applies business logic, and writes clean output to the warehouse. The warehouse never sees raw data. This architecture suits compliance requirements (mask PII before it enters the warehouse) or cases where raw data is too large or malformed to load directly.
- In ELT, a lightweight extraction tool (Fivetran, Airbyte, custom ingestion script) loads raw data as-is into a staging schema in the warehouse. Transformation tools like dbt then run SQL transformations entirely inside the warehouse, using the warehouse's MPP compute. The warehouse handles both storage and transformation compute.
- ELT's key advantage is that transformation logic (dbt SQL models) is version-controlled, testable, and independently re-runnable without data re-extraction. When a transformation bug is fixed, you rerun just the dbt model from the already-loaded raw data — no re-extraction needed.
- The cost model differs: ETL pays for transformation compute outside the warehouse (Spark cluster uptime), while ELT pays for transformation queries inside the warehouse (BigQuery slots, Snowflake credits). For most analytical workloads, ELT is cheaper because warehouse compute is optimised for SQL operations on columnar data.
- ELT still requires data quality enforcement — it defers transformation but not validation. dbt tests, source freshness checks, and data contracts applied to the raw landing zone catch quality issues before they propagate through the transformation layers. See [[DE-Engineer/03-Data-Modeling]] for data quality patterns.

**Common Misconceptions:**
- ETL is obsolete and ELT is always the correct architecture — ETL remains appropriate for GDPR/CCPA compliance (PII must never land in the warehouse unmasked), for scenarios where raw data volume is 100x the transformed output (pre-aggregation reduces warehouse storage cost), or for low-latency streaming where warehouse load adds unacceptable lag.
- ELT means no transformation logic and no data quality — ELT defers transformation to inside the warehouse, not eliminates it. A dbt-based ELT stack has just as much transformation logic as an ETL stack; it is expressed in SQL models rather than application code, making it more accessible and testable.

**Interview Answer Skeleton:**
- **What it is:** Two pipeline architectural patterns that differ in where transformation logic executes — outside the warehouse before loading (ETL) or inside the warehouse after loading (ELT) — each with distinct tooling, cost profiles, and operational trade-offs.
- **Why it matters / trade-offs:** The choice determines the full tech stack: ETL leads to Spark/Informatica tooling with separate compute infrastructure; ELT leads to dbt + modern cloud warehouse with unified compute. ELT wins on iteration speed and testability; ETL wins for compliance and pre-aggregation of massive datasets.
- **Example or context:** A dbt + Snowflake stack is ELT: Fivetran extracts and loads raw Salesforce data into a Snowflake raw schema unchanged; dbt models then transform that raw data into clean staging, intermediate, and mart models entirely within Snowflake SQL — no external compute cluster required.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — explains ELT philosophy, where dbt fits in the modern data stack, and how to structure transformation layers
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — covers ETL vs ELT architectural trade-offs with real-world examples and tooling comparisons

---

## Batch Workflows, Scheduling, and Retries

**Status:** ⬜ Not Started

**Definition:** Batch workflows process data in discrete chunks at defined time intervals or on event triggers. Scheduling tools manage task execution order, dependency resolution, and timing. Retry logic automatically re-attempts failed tasks with configurable backoff strategies to recover from transient errors without manual intervention.

**Key Mental Model:** A batch workflow is a delivery route — each task is a stop, dependency edges define the mandatory visiting order, and retries are what happens when a door is locked: wait a configured interval and try again, before escalating to a human.

**How It Works:**
- Airflow represents pipelines as DAGs (Directed Acyclic Graphs) where nodes are tasks (PythonOperator, BashOperator, SparkSubmitOperator) and edges define dependencies. The Airflow scheduler inspects DAG files, determines which tasks are eligible to run based on upstream completion states, and pushes them to the task queue for workers to pick up.
- Airflow's execution_date (or data_interval_start in newer versions) represents the logical time period a DAG run processes, not the time the run actually executes. A daily DAG scheduled at midnight processes the previous day's data_interval. This distinction is critical for backfills and incremental logic — the pipeline code should always use execution_date to determine which partition to process, never datetime.now().
- Task states in Airflow: queued → running → success/failed/upstream_failed. The scheduler evaluates trigger rules (all_success, all_done, one_failed) to determine whether downstream tasks should run based on upstream outcomes. A critical task failure propagates upstream_failed to all dependents, preventing downstream execution on incomplete data.
- Retry configuration on tasks (retries=3, retry_delay=timedelta(minutes=5)) re-queues the failed task after the delay. Exponential backoff (retry_exponential_backoff=True) prevents thundering-herd problems where many retrying tasks simultaneously overwhelm a recovering dependency. Max retry delay caps the backoff to avoid indefinite waiting.
- SLA misses in Airflow are time-based alerts: if a task hasn't completed N minutes after its scheduled time, Airflow fires an SLA miss callback. This is distinct from failure alerting — SLA misses catch tasks that are running but slow, not tasks that have failed. See [[DE-Engineer/04-Pipeline]] monitoring section for broader observability.

**Common Misconceptions:**
- If a task fails but retries succeed, the problem is resolved — a task that regularly fails on first attempt and succeeds on retry signals instability (memory pressure, flaky network, race condition) that will eventually fail all retries at a worse time. Transient failures warrant investigation, not just retry configuration.
- Fixed-interval retries are an adequate retry strategy — fixed retries under load create thundering-herd effects: ten pipelines all retrying the same overwhelmed database simultaneously, all failing together, all retrying again simultaneously. Exponential backoff with jitter disperses retry load and gives the dependency time to recover.

**Interview Answer Skeleton:**
- **What it is:** The orchestration layer that schedules pipeline tasks according to time and dependency constraints, manages execution state across DAG runs, and automatically recovers from transient failures through configurable retry policies.
- **Why it matters / trade-offs:** Pipelines run unattended overnight; without dependency management and retries, a transient network error at 2am becomes a data gap that reaches the morning dashboard. The trade-off of complex DAGs is observability cost — deeply nested dependencies are harder to debug when they fail.
- **Example or context:** An Airflow daily pipeline: extract_from_api (retries=3, exponential backoff) → validate_raw_data (no retry, deterministic) → run_dbt_models (retries=1) → notify_downstream. The extract task uses execution_date to determine which day's data to fetch — ensuring idempotent re-runs on backfill without fetching today's data instead of the historical date.

**Free Resources:**
- [Apache Airflow Documentation](https://airflow.apache.org/docs) — official Airflow docs covering DAGs, operators, scheduling, retry configuration, and SLA management
- [Data Engineer Handbook](https://github.com/DataExpert-io/data-engineer-handbook) — covers batch pipeline patterns, scheduling strategies, and workflow design for production data engineering

---

## Incremental Loads and CDC

**Status:** ⬜ Not Started

**Definition:** Incremental loading processes only records that are new or modified since the pipeline's last successful run, identified via a high-watermark timestamp or row hash comparison. Change Data Capture (CDC) detects row-level database changes (inserts, updates, deletes) at the source by reading the database transaction log directly, capturing every state transition rather than only the current row snapshot.

**Key Mental Model:** A full load photocopies an entire book every day. An incremental load copies only the new pages added since yesterday using a bookmark. CDC reads the author's tracked-changes log directly — capturing every edit, deletion, and addition from the moment they were made, not just the final state.

**How It Works:**
- High-watermark incremental loads query the source with a filter like `WHERE updated_at > last_run_watermark`. The pipeline stores the maximum updated_at from the last successful run and uses it as the lower bound for the next run. This approach requires the source table to have a reliable updated_at column that is set on every record modification — if source records are updated without changing updated_at, those changes are silently missed.
- dbt incremental models implement this pattern in SQL: on the first run they build the full table; on subsequent runs they query only new/changed rows from the source and either append or merge them into the existing table. The `unique_key` config triggers a MERGE operation that handles both new rows and updates to existing rows, preventing duplicates.
- Log-based CDC (Debezium + Kafka) reads the database binary log (MySQL binlog, Postgres WAL) as a real-time stream of change events. Each event carries the before and after state of the row plus the operation type (I/U/D). This captures hard deletes (which high-watermark polling misses entirely) and provides sub-second latency without adding query load to the source database.
- Late-arriving data is the primary correctness challenge for incremental loads: a record with an updated_at of yesterday that arrives in the source today will be missed if yesterday's partition has already been processed. The solution is a configurable lookback window — reprocessing the last N days of partitions on each run — trading compute cost for completeness.
- Delta Live Tables and Databricks automate CDC application using APPLY CHANGES INTO syntax, which manages SCD Type 1 (overwrite) and SCD Type 2 (history preservation) semantics on Delta tables. The framework handles out-of-order events, sequence number tracking, and deduplication automatically. See [[DE-Engineer/06-Platform]] for Delta Lake details.

**Common Misconceptions:**
- Incremental is always superior to full load — for small tables (under a few million rows), full loads are simpler, more reliable, and easier to validate. Incremental logic adds complexity, watermark state management, and late-arriving data risk. Choose incremental only when full load latency or compute cost is genuinely prohibitive.
- CDC requires direct access to database transaction logs — while log-based CDC (Debezium) is the most capable approach, alternatives exist: trigger-based CDC captures changes via database triggers, timestamp-based polling uses updated_at high-watermarks, and some databases expose logical replication slots that middleware tools (Airbyte, Fivetran) use without direct log access.

**Interview Answer Skeleton:**
- **What it is:** Strategies for processing only data that has changed since the last pipeline run — high-watermark incremental loads for new/updated records, and CDC for complete change capture including deletes — reducing compute cost and enabling near-real-time latency compared to full table reloads.
- **Why it matters / trade-offs:** Full reloads of large tables (billions of rows) become prohibitively expensive in time and compute cost; incremental patterns are essential for production scale. The trade-off is correctness risk: missed late-arriving updates, unhandled deletes, and watermark drift require careful testing and operational monitoring.
- **Example or context:** A dbt incremental model on an orders source: `unique_key='order_id'`, `strategy='merge'`, filter `WHERE updated_at > max(updated_at) - interval '3 days'` (3-day lookback). The 3-day window catches late-arriving records; the merge handles both new orders and status updates to existing ones. A hard delete in the source is invisible to this approach — if deleted orders matter, CDC (Debezium/WAL) is required.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — covers incremental model strategies, unique_key merge behaviour, and incremental predicates in detail
- [Databricks Delta Live Tables Documentation](https://docs.databricks.com/delta-live-tables) — covers CDC with APPLY CHANGES INTO, SCD implementations, and event-driven pipeline patterns on Delta Lake

---

## Idempotency, Backfills, and Reprocessing

**Status:** ⬜ Not Started

**Definition:** An idempotent pipeline produces the same output regardless of how many times it is run for a given logical time period — running it twice on the same input does not create duplicate records or corrupt state. Backfilling re-runs a pipeline over historical time periods after a bug fix or new logic deployment to correct or populate historical data. Reprocessing re-runs a specific failed or erroneous window without affecting other periods.

**Key Mental Model:** Idempotency is a save button that always saves — pressing it ten times leaves exactly one saved file. A backfill is repainting a fence after changing your mind about the colour — you systematically repaint every panel, not just the ones painted today.

**How It Works:**
- Idempotency is typically achieved through partition overwrite semantics: a pipeline run for execution_date=2024-01-15 overwrites the 2024-01-15 partition entirely rather than appending to it. This means re-running the same date produces the same partition contents as the first run — no duplicate rows accumulate. In Spark, this is `df.write.mode("overwrite").insertInto("table")` with a date partition filter.
- INSERT OVERWRITE in partitioned tables replaces only the target partition, leaving other partitions untouched. In BigQuery, writing to a partition with WRITE_TRUNCATE disposition achieves the same semantics — the destination partition is replaced atomically. This is fundamentally different from APPEND mode, which accumulates rows on every run.
- The Airflow execution_date parameter makes idempotent backfills possible: a pipeline parameterised on execution_date that processes exactly that date's data can be run for any historical date and produce the correct output for that date, independent of when the run actually executes. Using datetime.now() instead of execution_date breaks backfill idempotency.
- Backfills in Airflow are triggered with the `airflow dags backfill` command specifying a date range. Airflow creates DAG runs for each execution_date in the range, respecting task dependencies and concurrency limits. Concurrency throttling (max_active_runs) prevents a large backfill from overwhelming source systems or downstream dependencies.
- Reprocessing a specific window after a bug fix: mark the affected DAG runs as failed or cleared in Airflow UI, allowing the scheduler to re-queue them. With idempotent partition overwrite semantics, the corrected pipeline code will overwrite the incorrect historical partitions cleanly without manual data deletion steps.

**Common Misconceptions:**
- Appending data with a load timestamp is idempotent — appending creates a new row for every run; running twice produces two copies of every record. Idempotency requires either deduplication at read time (expensive and error-prone) or overwrite semantics at write time (correct and efficient). APPEND mode is not idempotent.
- Backfills are a rare, one-time operational event — backfills are a routine consequence of bug fixes, schema changes, new metric logic, and onboarding new data sources into historical periods. Every pipeline should be designed to support backfills from the beginning; retrofitting idempotency onto a non-idempotent pipeline is painful and risky.

**Interview Answer Skeleton:**
- **What it is:** A correctness property (idempotency) ensuring re-runs produce identical outputs, combined with operational patterns (backfills/reprocessing) that use this property to correct historical data safely after pipeline failures or logic changes.
- **Why it matters / trade-offs:** Pipelines fail and require re-running; without idempotency, every retry risks data duplication or corruption. The trade-off of partition overwrite semantics is that partial failures can leave a partition in an inconsistent state — atomic swap operations (Delta Lake ACID writes) address this by making the overwrite itself transactional.
- **Example or context:** An idempotent Airflow daily pipeline writes to a partitioned BigQuery table using WRITE_TRUNCATE on the execution_date partition. When a bug is discovered in the revenue calculation logic, the fix is deployed and the affected 30-day date range is backfilled: Airflow re-runs each day sequentially, overwriting each partition with corrected values — no manual SQL DELETE statements required.

**Free Resources:**
- [Apache Airflow Documentation](https://airflow.apache.org/docs) — covers execution_date semantics, backfill commands, DAG run management, and idempotent pipeline design patterns
- [dbt Documentation](https://docs.getdbt.com) — covers idempotent incremental model design, full-refresh semantics, and safely reprocessing historical dbt model runs

---

## Airflow, dbt, and Spark Tools

**Status:** ⬜ Not Started

**Definition:** Apache Airflow is a workflow orchestration platform that schedules, monitors, and manages the execution of pipeline tasks as DAGs. dbt is a SQL-first transformation framework that builds, tests, and documents data models inside the warehouse with dependency management. Apache Spark is a distributed in-memory processing engine for large-scale transformations that exceed single-machine memory limits.

**Key Mental Model:** Airflow is the project manager — decides when tasks run, in what order, and what to do when they fail. dbt is the SQL engineer — writes, tests, and documents the transformation logic. Spark is the heavy-machinery operator — handles the data volumes and complexity that SQL alone cannot process efficiently.

**How It Works:**
- Airflow's architecture separates the scheduler (determines what to run when), the web server (UI), and the workers (execute tasks). The scheduler parses DAG files continuously, creates DAG runs based on schedule intervals, evaluates task dependencies, and places ready tasks on the message broker (Celery or Kubernetes). Workers pull tasks from the broker, execute them, and report state back.
- dbt compiles SQL models into raw SQL and executes them against the warehouse in dependency order determined by `ref()` and `source()` function calls. The `ref()` function creates a dependency edge between models — dbt's DAG is built by parsing these references across all model files. At runtime, dbt replaces `ref('orders')` with the fully qualified table name for the current target environment.
- dbt materialisation types determine how SQL models are written to the warehouse: view (just a stored query, no data), table (full materialized table rebuilt each run), incremental (append or merge new rows only), ephemeral (a CTE inlined into dependent models, never written to disk). The choice is a performance vs freshness trade-off. See [[DE-Engineer/04-Pipeline]] incremental section for detail.
- Spark executes transformations as a DAG of RDD or DataFrame operations. The driver program builds a logical plan; Catalyst optimiser converts it to an optimised physical plan; the plan is decomposed into stages separated by shuffle boundaries; each stage runs as parallel tasks on executor JVMs across the cluster. Shuffle operations (groupBy, join, repartition) require all-to-all data movement across the network — they are the primary performance bottleneck to design around.
- Airflow + dbt + Spark integration: Airflow orchestrates the sequence — trigger Spark ingestion jobs (via SparkSubmitOperator or KubernetesPodOperator), wait for completion, then trigger dbt runs (via BashOperator running `dbt run --select`). Airflow doesn't transform data; it coordinates the tools that do. This separation of concerns is the modern data platform architecture. See [[DE-Engineer/06-Platform]] for platform stack patterns.

**Common Misconceptions:**
- Airflow performs data transformation — Airflow is purely an orchestration layer. It executes operators that call external tools (Spark, dbt, Python scripts, SQL queries), but all actual data processing happens in those tools. An Airflow DAG for a batch pipeline might contain zero data transformation logic itself.
- dbt is limited to simple SQL models — dbt supports incremental models with configurable merge strategies, snapshots (SCD Type 2 automation), seeds (CSV to table loading), hooks (pre/post-run SQL execution), custom tests, Jinja macros for DRY SQL logic, and packages. It is a full transformation framework, not a simple query runner.

**Interview Answer Skeleton:**
- **What it is:** Three distinct but complementary tools covering the three main pipeline concerns — orchestration (Airflow: when and in what order), SQL transformation (dbt: what the warehouse computes), and distributed processing (Spark: transformations too large or complex for warehouse SQL).
- **Why it matters / trade-offs:** Knowing which tool serves which purpose prevents architecture mistakes — using Airflow for transformation logic (it shouldn't), or Spark for small SQL transforms (expensive and slow for what a warehouse handles natively). The trade-off of maintaining all three is operational complexity; simpler stacks use dbt + Airflow alone when data volumes stay within warehouse scale.
- **Example or context:** A daily pipeline: Airflow triggers a SparkSubmitOperator to ingest raw clickstream events from S3 into a Delta table (100B rows, requires distributed processing). On success, Airflow triggers a BashOperator running `dbt run --select staging.clickstream+` to build staging and downstream mart models inside Snowflake. Airflow manages the handoff; Spark handles the volume; dbt handles the transformation logic.

**Free Resources:**
- [Apache Airflow Documentation](https://airflow.apache.org/docs) — complete reference for DAG authoring, operators, scheduling, and Airflow architecture
- [dbt Documentation](https://docs.getdbt.com) — covers all dbt features including models, tests, snapshots, incremental patterns, and project structure best practices

---

## Monitoring, Alerting, and Failure Recovery

**Status:** ⬜ Not Started

**Definition:** Pipeline monitoring tracks the health of data pipelines through operational metrics — row counts, processing latency, error rates, and data freshness relative to expected SLAs. Alerting notifies engineers when metrics breach thresholds or tasks fail. Failure recovery is the systematic process of diagnosing root cause, fixing the underlying issue, and safely reprocessing affected data windows without corrupting downstream state.

**Key Mental Model:** Monitoring is the dashboard of your pipeline vehicle — shows speed, fuel level, and warning lights. Alerting is the warning buzzer when a critical indicator crosses a threshold. Failure recovery is the roadside response — diagnose the flat tyre, fix it, and get back on route without losing the cargo.

**How It Works:**
- Operational monitoring layers: task-level monitoring (Airflow task duration, retry count, failure rate), data volume monitoring (row counts in vs rows out per pipeline stage, compared to rolling historical average), data freshness monitoring (max(event_timestamp) in destination table vs current time vs expected freshness SLA), and business metric monitoring (are KPI values within expected bounds).
- Row count anomaly detection compares today's ingested record count against a rolling N-day average with configurable standard deviation bounds. A pipeline that loaded 0 rows (structural failure) or 10x expected rows (deduplication bug) both represent correctness failures that don't trigger task-level failure alerts but are caught by volume checks.
- dbt source freshness checks query source tables' maximum timestamp and compare it to a warn_after / error_after threshold. This detects upstream pipeline stalls before dbt transformation models even run, preventing dbt from building downstream models on stale source data and surfacing potentially incorrect results downstream.
- Failure recovery sequence: (1) identify the failed task and its error in Airflow logs or the task's execution environment logs; (2) determine the data impact — which partitions or windows are affected; (3) fix the root cause (code change, dependency recovery, infrastructure fix); (4) clear the failed tasks in Airflow to allow re-execution; (5) verify output row counts match expected values after recovery.
- Alert fatigue is a genuine operational risk: too many low-signal alerts cause engineers to mute or ignore all alerts. The solution is tiered alerting — PagerDuty/phone for SLA-breaching failures affecting production dashboards, Slack for warning-level anomalies requiring investigation, and aggregated daily health reports for informational metrics. Each alert must be actionable — if an engineer receives it and has nothing to do, the alert is wrong.

**Common Misconceptions:**
- A pipeline that completes without error contains correct data — task completion signals successful execution, not data correctness. A pipeline that silently drops half its rows, fails to deduplicate, or writes incorrect aggregations will complete successfully while producing wrong output. Data volume and business metric validation are orthogonal to task success state.
- Alert on every pipeline failure immediately — alerting on every transient failure that self-recovers via retry creates alert noise that trains engineers to ignore alerts. Alert when an SLA is at risk or breached, not when a first retry attempt fails.

**Interview Answer Skeleton:**
- **What it is:** The observability and operational response system that detects pipeline health degradation through metrics and alerts, enabling engineers to identify, diagnose, and remediate failures before downstream consumers notice incorrect or missing data.
- **Why it matters / trade-offs:** Unmonitored pipelines fail silently; by the time a stakeholder notices a wrong number on a dashboard, days of bad data may have propagated through the entire warehouse. The trade-off of comprehensive monitoring is implementation and maintenance cost — the pragmatic approach is to instrument the highest-risk pipeline stages first and expand coverage iteratively.
- **Example or context:** A daily pipeline feeding a morning executive dashboard needs: (1) Airflow SLA alert if DAG doesn't complete by 7am; (2) row count check comparing today's loaded rows to 7-day average ±30%; (3) dbt source freshness check asserting source data is less than 6 hours old before transformation runs; (4) a final data quality test on the mart table checking no NULL values in key metrics. Any failure pages the on-call engineer via PagerDuty.

**Free Resources:**
- [Apache Airflow Documentation](https://airflow.apache.org/docs) — covers SLA configuration, callback functions, task state monitoring, and alerting integration
- [dbt Documentation](https://docs.getdbt.com) — covers source freshness configuration, test failure alerting, and data observability patterns within dbt projects

---
