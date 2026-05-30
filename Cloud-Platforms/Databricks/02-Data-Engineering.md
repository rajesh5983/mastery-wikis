# Databricks — Data Engineering

---

## Delta Live Tables (DLT)

**Status:** ⬜ Not Started

**Definition:** Delta Live Tables is a declarative pipeline framework in Databricks where you define transformations using Python or SQL, and DLT automatically manages execution order, retries, data quality enforcement, and incremental processing. You declare *what* you want; DLT figures out *how* to execute it.

**Mental Model:** DLT is like a recipe system that manages its own kitchen — you write the recipes (transformations), and the system decides the cooking order, restarts failed steps, and tracks quality automatically.

**Free Resources:** https://docs.databricks.com/en/delta-live-tables/index.html — Delta Live Tables documentation covering pipeline definition, quality constraints, and monitoring

---

## Auto Loader

**Status:** ⬜ Not Started

**Definition:** Auto Loader is a Databricks feature that incrementally and efficiently ingests new files from cloud storage as they arrive, using file notification or directory listing. It automatically handles schema inference, evolution, and checkpoint management for scalable file-based ingestion.

**Mental Model:** Auto Loader is a mail sorter that processes new mail automatically as it arrives — no manual intervention to pick up, open, and sort; it handles new files continuously without rescanning everything already processed.

**Free Resources:** https://docs.databricks.com/en/ingestion/auto-loader/index.html — Auto Loader documentation covering file notification, schema inference, and incremental processing

---

## Workflows (Databricks Jobs)

**Status:** ⬜ Not Started

**Definition:** Databricks Workflows is the native orchestration service for scheduling and running multi-task pipelines — notebook runs, DLT pipelines, dbt tasks, Spark JAR jobs, and more. It manages task dependencies, retries, alerting, and compute provisioning within the Databricks ecosystem.

**Mental Model:** Databricks Workflows is the built-in project manager — it sequences your tasks, starts the right compute for each, retries failures, and tells you when something breaks without requiring a separate orchestration tool.

**Free Resources:** https://docs.databricks.com/en/jobs/index.html — Databricks Workflows documentation covering task types, scheduling, and dependency management

---

## Structured Streaming

**Status:** ⬜ Not Started

**Definition:** Structured Streaming is the Spark-based streaming engine in Databricks that treats live data streams as continuously appended tables. Queries written as batch DataFrame operations automatically run incrementally on streaming data, simplifying the transition from batch to streaming.

**Mental Model:** Structured Streaming treats the data stream as an infinite table that's always being appended to — your query runs continuously against new data, processing it in micro-batches or continuously, using the same SQL/DataFrame API as batch.

**Free Resources:** https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html — Apache Spark Structured Streaming programming guide

---

## Change Data Feed (CDF)

**Status:** ⬜ Not Started

**Definition:** Delta Lake's Change Data Feed captures row-level changes (insert, update, delete) to Delta tables, making them available for downstream consumers as a CDC stream. This enables efficient incremental processing of Delta table changes without scanning the full table.

**Mental Model:** CDF is a changelog for your Delta table — instead of checking every row for changes, downstream pipelines subscribe to the feed and process only the rows that changed since their last read.

**Free Resources:** https://docs.databricks.com/en/delta/delta-change-data-feed.html — Delta Change Data Feed documentation covering enablement and consumption patterns

---

## dbt on Databricks

**Status:** ⬜ Not Started

**Definition:** dbt (data build tool) runs natively on Databricks using the dbt-databricks adapter, executing SQL transformations against Delta Lake tables on Databricks SQL Warehouses or All-Purpose Clusters. This combines dbt's transformation, testing, and documentation framework with Databricks' compute and governance.

**Mental Model:** dbt on Databricks is the SQL transformation layer on top of the lakehouse — dbt writes and tests the transformation logic, Databricks Photon executes it at scale, and Unity Catalog governs the output tables.

**Free Resources:** https://docs.getdbt.com/docs/core/connect-data-platform/databricks-setup — dbt documentation for configuring and using dbt with Databricks
