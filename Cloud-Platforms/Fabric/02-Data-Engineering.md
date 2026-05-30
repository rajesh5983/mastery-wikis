# Microsoft Fabric — Data Engineering

---

## Lakehouse

**Status:** ⬜ Not Started

**Definition:** The Fabric Lakehouse is a data engineering experience that combines Delta Lake table management with an automatically provisioned SQL analytics endpoint on top of OneLake storage. Every Lakehouse has two views: the Lakehouse explorer (for file and table management via Spark or file upload) and the SQL analytics endpoint (for T-SQL queries and Power BI connections).

**Key Mental Model:** A Fabric Lakehouse is a warehouse with an automatic front-counter service — the warehouse (OneLake + Delta tables) is where the data lives, and the service counter (SQL endpoint) serves SQL queries without any separate setup.

**How It Works:**
- When a Lakehouse item is created in a Fabric workspace, Fabric provisions two things simultaneously: a folder structure in OneLake (`/Tables/` for Delta tables and `/Files/` for unmanaged files) and a read-only **SQL Analytics Endpoint** that introspects the `/Tables/` directory to expose Delta tables as T-SQL queryable objects.
- Data written by Spark notebooks to the Lakehouse `/Tables/` path as Delta-Parquet files is automatically detected and registered by the SQL Analytics Endpoint — within seconds of a Spark write completing, the resulting Delta table is queryable via T-SQL through the endpoint without any explicit table registration step.
- The **SQL Analytics Endpoint** is a read-only T-SQL service backed by Fabric's distributed query engine; it cannot be used for DML (INSERT/UPDATE/DELETE) — writes must go through Spark, a Pipeline, or the Fabric Warehouse experience. This distinction matters for workload routing decisions.
- **Lakehouse shortcuts** allow the Lakehouse to reference external storage (ADLS Gen2, S3, other OneLake paths) as virtual folders — Spark notebooks and the SQL Endpoint can query shortcutted external data as if it were local OneLake data, without copying. This is the primary mechanism for accessing data from Azure services outside Fabric.
- All Lakehouse Delta tables are automatically included in the Lakehouse's **default semantic model** for Power BI, enabling report authors to build Power BI reports via Direct Lake mode against Lakehouse data without manually defining a semantic model. See [[Cloud-Platforms/Fabric/04-Power-BI]] for Direct Lake mode mechanics.

**Common Misconceptions:**
- The SQL Analytics Endpoint is not the same as a Fabric Warehouse — the endpoint is read-only and does not support concurrent write transactions, T-SQL DML, or stored procedures; it is a query-only interface over the Lakehouse Delta tables, whereas the Warehouse is a full read-write distributed T-SQL engine.
- Writing files to the Lakehouse `/Files/` folder does not automatically create queryable tables — only Delta-formatted files in `/Tables/` are exposed via the SQL endpoint; raw CSV or Parquet files in `/Files/` require explicit Spark code to convert them to Delta tables.

**Interview Answer Skeleton:**
- **What it is:** A Fabric workspace item that pairs OneLake storage (Delta tables in `/Tables/`, unmanaged files in `/Files/`) with an auto-provisioned read-only SQL Analytics Endpoint, enabling Spark-based writes and T-SQL/Power BI reads over the same Delta files without data movement.
- **Why it matters / trade-offs:** Provides a unified interface for Spark-based data engineering and T-SQL analytics on the same physical data; the trade-off is that the SQL Endpoint's read-only constraint means any write-heavy SQL workload (staging tables, transformations) must use the Warehouse experience instead.
- **Example or context:** A data engineer loads cleaned Delta tables into the Lakehouse using Spark notebooks — business analysts immediately query those tables via Power BI Direct Lake or T-SQL through the SQL Endpoint without waiting for any data movement or warehouse load process.

**Free Resources:**
- [Fabric Lakehouse Overview](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview) — Lakehouse architecture, SQL Endpoint, shortcuts, and Delta table management
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — cross-experience data access patterns between Lakehouse, Warehouse, and Power BI

---

## Notebooks

**Status:** ⬜ Not Started

**Definition:** Fabric Notebooks provide an interactive Python, Scala, R, and Spark SQL development environment with native integration with OneLake, Lakehouses, and the Fabric Spark runtime. Notebooks support collaborative editing, scheduled runs via Fabric Data Factory Pipelines, and source control integration through Azure DevOps or GitHub.

**Key Mental Model:** Fabric Notebooks are the data scientist's workbench — write code, run it against live data in OneLake, collaborate in real-time, and schedule the notebook as a pipeline job when it's ready for production.

**How It Works:**
- When a Fabric Notebook session starts, Fabric assigns the notebook a Spark context from the **Starter Pool** — a pool of pre-warmed Spark containers maintained by the Fabric platform that allows the first cell to execute within 10–30 seconds rather than the 3–5 minutes required for cold-start cluster provisioning on Azure PaaS Spark.
- Notebooks connect to Lakehouses via the **notebook attachment** mechanism — a Lakehouse is attached to the notebook as the default file system, so relative paths like `Files/` and `Tables/` resolve to the attached Lakehouse's OneLake folder without specifying full `abfss://` URIs.
- **Auto-discovery** means that Delta tables written by notebook code to the attached Lakehouse's `/Tables/` path are automatically registered in the Lakehouse's metadata and available through the SQL Analytics Endpoint — no `CREATE TABLE` DDL statement is required; the Fabric platform monitors the `/Tables/` path and registers new Delta table directories automatically.
- Fabric Notebooks support **real-time co-authoring** similar to Office 365 — multiple users can edit and run cells simultaneously, with each user's cursor position and cell outputs visible to collaborators in real-time.
- Notebooks executed as part of a **Data Factory Pipeline** run as a pipeline activity with pass-through parameters; pipeline parameters are injected into the notebook via the `mssparkutils.notebook.run()` API, enabling parameterised notebook patterns for date-partitioned processing. See [[Cloud-Platforms/Fabric/02-Data-Engineering#Data Factory Pipelines]] for orchestration integration.

**Common Misconceptions:**
- Fabric Notebooks do not automatically use the largest available Spark pool — the Starter Pool has predefined node sizes; for workloads requiring specific instance types or large memory, a custom Spark pool must be explicitly configured and attached to the notebook.
- "Notebooks are only for exploration" is a common anti-pattern — Fabric Notebooks can be operationalised as pipeline activities for production workloads; the same notebook code runs in both interactive and scheduled contexts without modification.

**Interview Answer Skeleton:**
- **What it is:** A Spark-backed interactive development environment within Fabric that uses pre-warmed Starter Pool containers for fast session starts, attaches to Lakehouses for seamless OneLake access, and supports both real-time collaboration and scheduled pipeline execution.
- **Why it matters / trade-offs:** Provides a development experience comparable to Databricks notebooks within the Fabric ecosystem, with instant OneLake integration eliminating storage URI configuration; the trade-off is less cluster configuration flexibility compared to Databricks All-Purpose clusters.
- **Example or context:** A data engineer prototypes a complex Spark transformation interactively in a Fabric Notebook attached to a Bronze Lakehouse — after validation, the same notebook is added as an activity in a Data Factory Pipeline, scheduled hourly, with the target date passed as a pipeline parameter.

**Free Resources:**
- [Fabric Notebooks Documentation](https://learn.microsoft.com/en-us/fabric/data-engineering/how-to-use-notebook) — notebook development, attachment, scheduling, and parameterisation
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Spark runtime configuration and Starter Pool documentation

---

## Data Factory Pipelines

**Status:** ⬜ Not Started

**Definition:** Fabric Data Factory provides low-code pipeline orchestration with 150+ connectors for data ingestion and movement. Pipelines sequence activities (Notebook runs, Dataflow executions, stored procedure calls, copy activities) with conditional branching, retry policies, and monitoring — and run on Fabric-managed compute using capacity CUs rather than separately provisioned integration runtimes.

**Key Mental Model:** Data Factory Pipelines are the assembly line manager in Fabric — they coordinate the sequence of activities, handle failures, and ensure data moves from source to destination on schedule.

**How It Works:**
- A Fabric Pipeline is a JSON-defined DAG of activities similar to Azure Data Factory; the pipeline definition specifies activity types (Copy Activity, Notebook Activity, Dataflow Gen2 Activity, Stored Procedure Activity), their execution order, dependencies, and retry settings.
- The **Copy Activity** is the primary bulk data ingestion mechanism — it connects to a source system (via a Linked Service and Dataset, using one of 150+ connectors) and copies data to OneLake in a single parallelised operation, with options to write as Delta, Parquet, or CSV. The copy engine runs on Fabric-managed compute, not customer-provisioned integration runtimes.
- **Control flow activities** (If Condition, ForEach, Until, Set Variable, Get Metadata) enable dynamic pipeline behaviour — a ForEach activity can iterate over a list of tables and run a Copy Activity for each, enabling metadata-driven pipeline patterns where the set of tables to process is read from a configuration table.
- Pipelines run on the **capacity CUs** of the workspace's attached Fabric capacity — there are no integration runtime billing tiers as in Azure Data Factory; execution costs are pooled with other workloads under the capacity model, which simplifies billing but introduces CU contention for large pipelines.
- Pipeline runs are monitored through the **Monitor Hub** in Fabric, which shows run history, activity-level execution times, and failure reasons; pipeline run data is also available in OneLake via system tables for custom monitoring dashboards. See [[Cloud-Platforms/Fabric/01-Architecture#Fabric Capacities]] for CU consumption management.

**Common Misconceptions:**
- Fabric Data Factory Pipelines are not identical to Azure Data Factory pipelines — while they share the same conceptual model and visual interface, Fabric pipelines lack some ADF features (Azure Integration Runtime for on-premises connectivity, SSIS package execution, some advanced trigger types); they are a Fabric-native reimplementation, not a direct migration path.
- Copy Activity does not transform data — it moves data as-is from source to destination; transformation logic requires a Notebook Activity, Dataflow Gen2 Activity, or a stored procedure call; a common mistake is expecting Copy Activity to filter or rename columns during ingestion.

**Interview Answer Skeleton:**
- **What it is:** A Fabric-native low-code pipeline orchestration engine that sequences data movement (Copy Activity with 150+ connectors), computation (Notebook and Dataflow activities), and control flow logic into capacity-CU-billed DAG pipelines monitored through the Monitor Hub.
- **Why it matters / trade-offs:** Provides orchestration without a separate ADF instance or integration runtime, simplifying the platform footprint; the trade-off is that Fabric Pipelines have fewer connectors and trigger types than full Azure Data Factory, making complex integration scenarios require ADF or third-party orchestrators.
- **Example or context:** A data engineering team builds a metadata-driven pipeline that reads a configuration table listing 200 source tables, runs a ForEach activity to copy each table from an Azure SQL Database to a Fabric Lakehouse using Copy Activity, then triggers a Notebook Activity to apply Delta MERGE logic for upserts — all without writing pipeline code.

**Free Resources:**
- [Fabric Data Factory Overview](https://learn.microsoft.com/en-us/fabric/data-factory/data-factory-overview) — pipeline activities, connectors, control flow, and monitoring
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Copy Activity reference, Linked Service configuration, and pipeline scheduling

---

## Spark on Fabric

**Status:** ⬜ Not Started

**Definition:** Fabric provides a managed Apache Spark environment with Starter Pools (pre-warmed Spark clusters available in ~10–30 seconds), automatic Spark runtime management, and deep integration with OneLake. The Fabric Spark runtime is a Microsoft-managed open-source Spark distribution optimised for Delta Lake on OneLake, with automatic runtime upgrades managed by the platform.

**Key Mental Model:** Spark on Fabric is Spark without the infrastructure management — sessions start in seconds, scale automatically, and write directly to OneLake Delta tables without cluster provisioning or configuration.

**How It Works:**
- **Starter Pools** are Microsoft-maintained pools of pre-initialised Spark containers that are allocated to notebook or job sessions on demand — because the Spark drivers and executors are already running in warm containers, session attachment takes 10–30 seconds vs 3–5 minutes for a cold-start Spark cluster on Azure Synapse or Databricks.
- When a notebook or Spark job session ends, the Spark containers are returned to the pool rather than terminated — keeping them warm for the next session; this means the pool has continuous container cost, covered by the Fabric SaaS model (not billed separately to customers).
- Fabric provides **custom Spark pools** for workloads that require specific node sizes, autoscale configurations, or longer-running sessions — custom pools provision dedicated VM clusters from the Microsoft-managed infrastructure and take longer to start than Starter Pools but offer higher resource limits.
- Delta tables written by Spark to the Lakehouse `/Tables/` directory are automatically registered by the Lakehouse metadata service — the Lakehouse periodically scans the Delta transaction log of new directories and adds discovered tables to the SQL Analytics Endpoint's metadata, making them queryable via T-SQL within seconds of the Spark write completing.
- Fabric Spark integrates with **Microsoft Purview** for data lineage — Spark job metadata (input and output datasets, transformation logic) is automatically sent to Purview as lineage events, enabling cross-platform lineage graphs that show data flowing from Spark into Lakehouse tables and downstream to Power BI reports. See [[Cloud-Platforms/Fabric/05-Administration]] for Purview governance integration.

**Common Misconceptions:**
- Starter Pools do not guarantee unlimited concurrency — each Fabric capacity has a CU-based limit on total Spark concurrency; running many simultaneous Spark sessions on a small capacity will queue sessions until CUs are available, despite the pool's pre-warmed state.
- Fabric Spark is not a separate billing item — Spark CU consumption is pooled with all other Fabric workloads (Pipelines, Warehouse queries, Power BI refreshes) against the same capacity; a runaway Spark job can throttle all other Fabric experiences in the workspace.

**Interview Answer Skeleton:**
- **What it is:** A Microsoft-managed Spark runtime on Fabric that provides sub-30-second session starts via pre-warmed Starter Pools, automatic Delta table registration in Lakehouse metadata, and CU-pooled billing alongside all other Fabric workloads.
- **Why it matters / trade-offs:** Eliminates Spark cluster provisioning and lifecycle management while integrating natively with OneLake and Purview; the trade-off is that Spark sessions share CU budget with other Fabric experiences, requiring capacity sizing that accounts for peak concurrent Spark usage alongside BI and pipeline workloads.
- **Example or context:** A data engineering team runs 10 Spark notebooks simultaneously during morning ETL — with Starter Pools, all 10 sessions are active within 30 seconds; without Starter Pools on Azure Synapse, each session would queue for VM provisioning, causing staggered starts and overall slower ETL completion.

**Free Resources:**
- [Fabric Spark Compute Documentation](https://learn.microsoft.com/en-us/fabric/data-engineering/spark-compute) — Starter Pools, custom pools, sizing, and autoscale configuration
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Spark runtime versioning, OneLake integration, and Purview lineage for Spark

---

## Dataflows Gen2

**Status:** ⬜ Not Started

**Definition:** Dataflows Gen2 are low-code/no-code ETL tools in Fabric based on Power Query. They provide a visual transformation canvas with 150+ connectors that can load data into Lakehouses or Warehouses. Dataflows Gen2 add a staging layer not present in Gen1, enabling efficient incremental refresh and data landing in OneLake-backed destinations.

**Key Mental Model:** Dataflows are the drag-and-drop ETL builder — connect to a source, visually apply transformations (filter, join, rename), and load to a destination, all without writing code.

**How It Works:**
- Dataflows Gen2 use the **Power Query engine** (the same M formula language underlying Excel Power Query and Power BI dataflows) to define transformations visually; each step in the Power Query editor is compiled to an execution plan that runs on Fabric-managed compute.
- Unlike Dataflows Gen1 (which only loaded data into Power BI datasets), Gen2 adds **data destination support**: transformed data can be written directly to Fabric Lakehouse tables, Warehouse tables, or Azure SQL Database, using either full-load (overwrite) or merge (upsert) write modes.
- **Staging in OneLake**: Gen2 uses an intermediate staging layer in OneLake between source extraction and destination load — source data is first written to a staging location, then the transformation query runs over the staged data. This decouples source latency from transformation compute and improves failure recovery (re-run from staging, not re-extract from source).
- Dataflows run on Fabric-managed **Mashup/Power Query compute**, which is separate from the Spark pool and does not require Spark knowledge — transformations defined in Power Query M execute on a proprietary compute engine that scales according to data volume.
- Gen2 dataflows support **incremental refresh** when the source supports query folding (predicate push-down to the source) — on each refresh, only rows added or changed since the last watermark are extracted from the source, significantly reducing extract time and source system load for large tables. See [[Cloud-Platforms/Fabric/02-Data-Engineering#Data Factory Pipelines]] for orchestrating Dataflow runs within pipeline sequences.

**Common Misconceptions:**
- Power Query query folding is not guaranteed — if the source connector does not support fold (e.g., non-relational APIs, CSV files), incremental refresh cannot push date filters to the source and will always do a full extract; checking fold diagnostic output is essential before relying on incremental refresh for large datasets.
- Dataflows Gen2 are not suitable for very large-scale transformations — Power Query compute is optimised for medium-scale ETL (millions of rows); for billion-row transformations with complex logic, Spark notebooks or Warehouse T-SQL procedures perform significantly better.

**Interview Answer Skeleton:**
- **What it is:** A low-code ETL tool using the Power Query/M engine with 150+ source connectors, an OneLake-based staging layer, and destination support for Lakehouse and Warehouse tables — with incremental refresh via query folding for change-data extraction.
- **Why it matters / trade-offs:** Democratises ETL development for non-Spark engineers and business users; the trade-off is that Power Query compute has lower throughput than Spark for large datasets, and complex multi-table transformation logic is harder to maintain in visual M code than in SQL or Python.
- **Example or context:** A business analyst uses a Dataflows Gen2 to pull data from an Azure SQL Database (leveraging query folding for incremental refresh) through Power Query transformations, and loads the result into a Fabric Lakehouse table daily — no Spark or T-SQL knowledge required, and the incremental extract reduces source load from 2 hours to 3 minutes.

**Free Resources:**
- [Fabric Dataflows Gen2 Overview](https://learn.microsoft.com/en-us/fabric/data-factory/dataflows-gen2-overview) — architecture, staging, destinations, and incremental refresh patterns
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Power Query M language reference and dataflow best practices

---

## Delta Format on OneLake

**Status:** ⬜ Not Started

**Definition:** All managed tables in Fabric's Lakehouse, Warehouse, and KQL EventHouse are stored as Delta-Parquet files on OneLake. This universal format means any Fabric experience can read tables created by another experience directly — a Lakehouse Delta table is immediately queryable from the Warehouse SQL endpoint and Power BI Direct Lake mode without any ETL or data movement.

**Key Mental Model:** Delta on OneLake is a shared language — all Fabric experiences speak Delta, so data written by one can be read by any other without translation or movement.

**How It Works:**
- Every Delta table in Fabric consists of Parquet data files and a `_delta_log/` transaction log directory in OneLake; the Delta protocol (open-source Apache standard) ensures any Delta-compatible reader (Spark, Trino, Power BI) can open the table by reading the transaction log to determine the current set of valid Parquet files.
- The **Warehouse T-SQL engine** reads Lakehouse Delta tables using the same OneLake paths — when a cross-database query references a Lakehouse table, the Warehouse query engine resolves the OneLake path for that table and reads the Delta Parquet files directly, using **zero-copy shortcuts** rather than data replication.
- **Direct Lake mode** in Power BI reads Delta Parquet column segment files directly from OneLake into the VertiPaq in-memory engine — rather than importing a full copy of the data (Import mode) or sending every DAX query to the source (DirectQuery mode), Direct Lake loads only the column segments needed for the current query, balancing freshness and performance.
- When Spark writes a new Delta commit to OneLake, the transaction log entry is available to all readers immediately — the SQL Analytics Endpoint, Warehouse cross-database queries, and Power BI Direct Lake all see the new data within seconds of the commit completing, without any explicit refresh or cache invalidation step.
- The **Delta protocol versioning** in OneLake is standardised — Fabric maintains Delta protocol compatibility across experiences, so a Delta table written by a Spark 3.4 job is readable by the T-SQL engine and Power BI without version negotiation or conversion. See [[Cloud-Platforms/Fabric/03-Data-Warehouse]] for how Warehouse uses Delta on OneLake for cross-database queries.

**Common Misconceptions:**
- "Delta format on OneLake" does not mean all experiences use the same query engine — Spark, T-SQL, and VertiPaq are completely different engines that happen to all read the same Delta-Parquet files; query performance characteristics differ dramatically between experiences for the same underlying data.
- Not all files in OneLake are Delta tables — the `/Files/` section of a Lakehouse can contain arbitrary files (CSV, JSON, Parquet, images) that are not Delta-formatted and cannot be queried via the SQL Endpoint or Direct Lake; only files in the `/Tables/` path as Delta-formatted are treated as managed tables.

**Interview Answer Skeleton:**
- **What it is:** An open Delta-Parquet storage standard universally used by all Fabric experiences (Spark, T-SQL, VertiPaq, KQL) as the table format on OneLake, enabling zero-copy cross-experience data access via shared transaction log reads rather than data replication.
- **Why it matters / trade-offs:** Eliminates ETL between Fabric experiences and enables a single source of truth in OneLake shared by all workloads; the trade-off is that Delta's transaction log becomes a consistency contract — any experience that writes in a non-compliant way (e.g., directly replacing Parquet files without updating the transaction log) breaks the table for all other readers.
- **Example or context:** A data engineer writes Silver Lakehouse Delta tables via Spark; a Warehouse analyst immediately runs T-SQL cross-database queries against those tables via the SQL Analytics Endpoint; Power BI reports access the same tables via Direct Lake — three experiences, one physical copy of data, no ETL between them.

**Free Resources:**
- [Delta Tables in Fabric Lakehouse](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-and-delta-tables) — Delta table lifecycle, auto-discovery, and cross-experience access
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — OneLake Delta format compatibility and cross-experience query patterns

---
