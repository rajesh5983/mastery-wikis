# Layer 4 — Pipeline

> **Framework:** Building reliable data movement systems with robust error handling and recovery.

---

## ETL vs ELT Patterns

**Status:** ⬜ Not Started

**Definition:** ETL (Extract-Transform-Load) transforms data before loading it into the target system — common in legacy architectures. ELT (Extract-Load-Transform) loads raw data first, then transforms it inside the warehouse — the modern default because cloud warehouses are powerful enough to handle transformation at scale.

**Mental Model:** ETL is cooking before serving — clean, portion, and season before plating. ELT is a buffet — bring everything raw to the table, then each person prepares their own dish in place.

**Common Misconceptions:**
- ETL is obsolete and ELT is always better — ETL is still appropriate when data must be cleaned or masked before loading for compliance or size reduction reasons.
- ELT means no data quality — ELT defers transformation but not quality; data quality checks are still mandatory, just applied after loading.

**Interview Skeleton:**
- What it is: two architectural patterns for where data transformation happens relative to loading into the target system
- Why it matters: the choice affects tooling selection, cost, latency, and how easily transformation logic can be iterated on
- Example: explain why a dbt + Snowflake stack is ELT, and contrast it with a legacy Informatica pipeline

**Free Resources:** https://docs.getdbt.com/terms/elt — dbt explanation of ELT and where dbt fits in the modern data stack

---

## Batch Workflows, Scheduling, and Retries

**Status:** ⬜ Not Started

**Definition:** Batch workflows process data in discrete chunks at scheduled intervals (hourly, daily). Scheduling tools (Airflow, Prefect, dbt Cloud) manage task dependencies and trigger runs on time or event. Retry logic automatically re-runs failed tasks to handle transient errors without manual intervention.

**Mental Model:** A batch workflow is a delivery route — tasks are the stops, dependencies are the order you must visit them, and retries are what happens when a door is locked: wait and try again.

**Common Misconceptions:**
- If a task fails but retries succeed, the problem is solved — silent transient failures should still be investigated; they signal instability that will eventually cause data loss at a worse time.
- Fixed-interval retries are fine — exponential backoff prevents thundering-herd problems when a dependency is under heavy load.

**Interview Skeleton:**
- What it is: patterns for running data pipelines on a schedule with dependency management and error recovery
- Why it matters: pipelines run unattended; without retries and monitoring, transient failures become data gaps that reach production dashboards
- Example: describe an Airflow DAG structure for a daily pipeline with upstream/downstream task dependencies and retry policy

**Free Resources:** https://airflow.apache.org/docs/apache-airflow/stable/tutorial/fundamentals.html — Apache Airflow tutorial on DAGs, tasks, and scheduling

---

## Incremental Loads and CDC

**Status:** ⬜ Not Started

**Definition:** Incremental loading processes only new or changed data since the last run, rather than reloading the entire dataset. Change Data Capture (CDC) detects and captures row-level changes (inserts, updates, deletes) from a source system, typically by reading database transaction logs directly.

**Mental Model:** A full load is photocopying an entire book every day. Incremental load is copying only the new pages added since yesterday. CDC is reading the author's tracked-changes history directly from the document.

**Common Misconceptions:**
- Incremental is always better than full load — full loads are simpler and safer for small tables; incremental adds complexity and the risk of missing late-arriving changes.
- CDC requires access to database logs — CDC tools like Debezium read transaction logs (binlog, WAL), but some implementations use triggers or high-watermark polling as alternatives.

**Interview Skeleton:**
- What it is: strategies for efficiently processing only changed data to reduce pipeline cost and latency
- Why it matters: full loads don't scale; incremental patterns are essential for large tables and near-real-time requirements
- Example: describe how to implement incremental loading in dbt using `updated_at` watermarks, and the edge cases to handle

**Free Resources:** https://docs.getdbt.com/docs/build/incremental-models — dbt documentation on incremental models and available strategies

---

## Idempotency, Backfills, and Reprocessing

**Status:** ⬜ Not Started

**Definition:** An idempotent pipeline produces the same result no matter how many times it runs for the same input — running it twice doesn't create duplicate data. Backfilling re-runs a pipeline over historical periods after a bug fix or new logic is deployed. Reprocessing re-runs a specific failed or incorrect historical window.

**Mental Model:** Idempotency is like a save button that always saves — pressing it ten times leaves you with one saved file, not ten. A backfill is re-painting a fence after changing your mind about the colour.

**Common Misconceptions:**
- Appending data with timestamps is idempotent — if the same event is processed twice, you get two rows; you need a deduplication step or upsert logic to achieve true idempotency.
- Backfills are rare and simple — backfills are common after bug fixes, and at scale they can overwhelm source systems; always design pipelines to support controlled backfills from the start.

**Interview Skeleton:**
- What it is: correctness properties and operational patterns that make pipelines safe to re-run without data corruption
- Why it matters: pipelines fail and need re-running; idempotency prevents duplication and corruption on retry
- Example: explain how you'd design an idempotent Airflow DAG that writes to daily partitions without duplication on retry

**Free Resources:** https://maximebeauchemin.medium.com/functional-data-engineering-a-modern-paradigm-for-batch-data-processing-2327ec32c42a — Maxime Beauchemin's seminal article on functional and idempotent data engineering

---

## Airflow, dbt, and Spark Tools

**Status:** ⬜ Not Started

**Definition:** Apache Airflow is a workflow orchestration tool for scheduling and monitoring pipelines as directed acyclic graphs (DAGs). dbt transforms data inside the warehouse using SQL models with dependency management and testing built in. Apache Spark is a distributed compute engine for large-scale data processing in Python, Scala, or SQL.

**Mental Model:** Airflow is the project manager — schedules and monitors who does what when. dbt is the SQL developer — writes and tests transformations. Spark is the heavy-machinery operator — handles data volumes that don't fit anywhere else.

**Common Misconceptions:**
- Airflow performs the data transformation — Airflow orchestrates; it calls Spark jobs, dbt runs, or Python scripts but does not transform data itself.
- dbt only works for simple SQL transformations — dbt supports incremental models, snapshots, seeds, hooks, and complex Jinja macros; it is a full transformation framework.

**Interview Skeleton:**
- What it is: the three primary tools in the modern data engineering stack covering orchestration, transformation, and large-scale processing
- Why it matters: knowing which tool to use for which job — and when they overlap — is a core data engineering competency
- Example: describe a pipeline where Airflow triggers a Spark job to stage raw data, then calls dbt to transform it into analytics marts

**Free Resources:** https://docs.getdbt.com/docs/introduction — dbt introduction explaining its role and philosophy in the modern data stack

---

## Monitoring, Alerting, and Failure Recovery

**Status:** ⬜ Not Started

**Definition:** Monitoring tracks pipeline health through metrics: row counts, latency, error rates, and data freshness. Alerting notifies engineers when metrics cross thresholds. Failure recovery is the systematic process of diagnosing, fixing, and re-running failed or incorrect pipeline components.

**Mental Model:** Monitoring is the dashboard of your pipeline car — speed, fuel, warning lights. Alerting is the seatbelt warning buzzer. Failure recovery is what you do after the flat tyre: diagnose, fix, and get back on the road.

**Common Misconceptions:**
- If the pipeline completes successfully, the data is correct — completion does not guarantee correctness; row count checks, null rate monitoring, and business metric validation are all essential.
- Alert on every failure — alert fatigue causes engineers to ignore all alerts; focus on actionable, high-signal alerts that represent genuine SLA breaches.

**Interview Skeleton:**
- What it is: the systems and practices that detect and recover from pipeline failures before they affect downstream consumers
- Why it matters: unmonitored pipelines fail silently; by the time someone notices, days of bad data may have propagated to production
- Example: describe the monitoring stack you'd build around a daily batch pipeline feeding a business-critical morning dashboard

**Free Resources:** https://www.montecarlodata.com/blog-data-observability/ — Monte Carlo's introduction to data observability principles and practices
