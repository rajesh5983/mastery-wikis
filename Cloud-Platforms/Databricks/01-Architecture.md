# Databricks — Architecture

---

## Lakehouse Architecture

**Status:** ⬜ Not Started

**Definition:** The Databricks Lakehouse combines the low-cost, flexible storage of a data lake with the reliability, governance, and performance of a data warehouse. It stores data in open formats (Delta Lake) on cloud object storage, then layers ACID transactions, schema enforcement, and SQL performance on top.

**Mental Model:** A lakehouse is a reservoir with a water treatment plant attached — the reservoir (object storage) holds everything cheaply, and the treatment plant (Delta Lake + Databricks) makes it clean, queryable, and reliable.

**Free Resources:** https://docs.databricks.com/en/lakehouse/index.html — Databricks lakehouse architecture documentation

---

## Delta Lake

**Status:** ⬜ Not Started

**Definition:** Delta Lake is an open-source storage layer that brings ACID transactions, schema enforcement, time travel (versioned history), and scalable metadata management to data stored in Parquet files on cloud object storage. It is the default table format on Databricks.

**Mental Model:** Delta Lake is the transaction log for your data files — like a database WAL (write-ahead log), it records every change so you can roll back, audit, or query any historical version.

**Free Resources:** https://docs.delta.io/latest/index.html — Delta Lake open source documentation covering transactions, time travel, and schema evolution

---

## Unity Catalog

**Status:** ⬜ Not Started

**Definition:** Unity Catalog is Databricks' unified governance layer — a centralised metastore for data assets (tables, volumes, views), ML models, files, and dashboards across all workspaces. It provides fine-grained access control, data lineage, and audit logging.

**Mental Model:** Unity Catalog is the central library system for your entire Databricks environment — one catalogue that knows where everything is, who can access it, and where it came from.

**Free Resources:** https://docs.databricks.com/en/data-governance/unity-catalog/index.html — Unity Catalog documentation covering setup, privileges, and lineage

---

## Cluster Types

**Status:** ⬜ Not Started

**Definition:** Databricks has three main compute types: All-Purpose Clusters (interactive development, notebooks, multi-user), Job Clusters (single-job automated pipelines that start and terminate per run), and SQL Warehouses (optimised for SQL Analytics, BI tools, and dbt). Serverless versions exist for both notebooks and SQL.

**Mental Model:** All-Purpose clusters are a shared workshop open all day. Job clusters are delivery vans that go out for one trip and return. SQL Warehouses are a dedicated service counter just for queries.

**Free Resources:** https://docs.databricks.com/en/compute/index.html — Databricks compute documentation covering cluster types, configurations, and sizing

---

## Photon Engine

**Status:** ⬜ Not Started

**Definition:** Photon is a native vectorised query engine written in C++ that accelerates Apache Spark SQL and DataFrame operations on Databricks. It provides 2–10x query speedup for SQL workloads through vectorised execution without any code changes required.

**Mental Model:** Photon is a turbocharger for Spark SQL — the same queries run faster without any changes to the code, because the execution engine processes data in vectorised batches rather than row by row.

**Free Resources:** https://docs.databricks.com/en/compute/photon.html — Databricks Photon documentation covering supported operations and performance benchmarks

---

## Storage-Compute Separation

**Status:** ⬜ Not Started

**Definition:** In Databricks, data lives in cloud object storage (S3, ADLS, GCS) independently of the compute clusters that process it. Clusters read from and write to external storage; shutting down a cluster does not delete data. This enables elastic scaling and eliminates the coupling that caused cost inefficiency in traditional on-premise Hadoop clusters.

**Mental Model:** Data is in the cloud filing cabinet; compute is the team working at desks. The team can grow or shrink without touching the filing cabinet — and the filing cabinet is always there even when everyone goes home.

**Free Resources:** https://docs.databricks.com/en/lakehouse/index.html — Databricks architecture overview explaining storage-compute separation
