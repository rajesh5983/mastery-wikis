# Microsoft Fabric — Data Engineering

---

## Lakehouse

**Status:** ⬜ Not Started

**Definition:** The Fabric Lakehouse is a data engineering experience that combines Delta Lake table management with a SQL analytics endpoint on top of OneLake storage. Every Lakehouse has two views: the Lakehouse explorer (for file and table management) and an automatically provisioned SQL endpoint (for T-SQL queries and Power BI connection).

**Mental Model:** A Fabric Lakehouse is a warehouse with an automatic front-counter service — the warehouse (OneLake + Delta tables) is where the data lives, and the service counter (SQL endpoint) serves SQL queries without any separate setup.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview — Microsoft Fabric Lakehouse documentation

---

## Notebooks

**Status:** ⬜ Not Started

**Definition:** Fabric Notebooks provide an interactive Python, Scala, R, and Spark SQL development environment with native integration with OneLake, Lakehouses, and the Fabric runtime. Notebooks support collaborative editing, scheduled runs via Fabric Pipelines, and version history through Azure DevOps integration.

**Mental Model:** Fabric Notebooks are the data scientist's workbench — write code, run it against live data in OneLake, collaborate in real-time, and schedule the notebook as a pipeline job when it's ready for production.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-engineering/how-to-use-notebook — Microsoft Fabric Notebooks documentation covering development and scheduling

---

## Data Factory Pipelines

**Status:** ⬜ Not Started

**Definition:** Fabric Data Factory provides low-code pipeline orchestration with 150+ connectors for data ingestion and movement. Pipelines are the orchestration layer in Fabric — they schedule and sequence activities (Notebook runs, Dataflow executions, stored procedure calls) with retry and monitoring built in.

**Mental Model:** Data Factory Pipelines are the assembly line manager in Fabric — they coordinate the sequence of activities, handle failures, and ensure data moves from source to destination on schedule.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-factory/data-factory-overview — Microsoft Fabric Data Factory documentation covering pipelines and activities

---

## Spark on Fabric

**Status:** ⬜ Not Started

**Definition:** Fabric provides a managed Apache Spark environment with starter pools (pre-warmed Spark clusters available in ~10 seconds), automatic Spark tuning, and integration with OneLake. The Fabric Spark runtime is based on Microsoft's open-source Spark runtime with optimisations for Delta Lake on OneLake.

**Mental Model:** Spark on Fabric is Spark without the infrastructure management — sessions start in seconds, scale automatically, and write directly to OneLake Delta tables without cluster provisioning or configuration.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-engineering/spark-compute — Microsoft Fabric Spark compute documentation covering pools, sizing, and optimisation

---

## Dataflows Gen2

**Status:** ⬜ Not Started

**Definition:** Dataflows Gen2 are low-code/no-code ETL tools in Fabric based on Power Query. They provide a visual transformation interface with 150+ connectors that can load data into Lakehouses or Warehouses. Dataflows Gen2 are ideal for citizen data engineers and structured ETL without writing code.

**Mental Model:** Dataflows are the drag-and-drop ETL builder — connect to a source, visually apply transformations (filter, join, rename), and load to a destination, all without writing code.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-factory/dataflows-gen2-overview — Microsoft Fabric Dataflows Gen2 documentation

---

## Delta Format on OneLake

**Status:** ⬜ Not Started

**Definition:** All managed tables in Fabric's Lakehouse, Warehouse, and KQL Database are stored as Delta Parquet files on OneLake. This means any experience can read tables created by another experience directly — a Lakehouse table is immediately queryable from the Warehouse SQL endpoint without copying data.

**Mental Model:** Delta on OneLake is a shared language — all Fabric experiences speak Delta, so data written by one can be read by any other without translation or movement.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-and-delta-tables — Microsoft documentation on Delta tables in Fabric Lakehouse
