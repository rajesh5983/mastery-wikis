# Databricks — Architecture

---

## Lakehouse Architecture

**Status:** ⬜ Not Started

**Definition:** The Databricks Lakehouse combines the low-cost, flexible storage of a data lake with the reliability, governance, and performance of a data warehouse. It stores data in open formats (Delta Lake) on cloud object storage, then layers ACID transactions, schema enforcement, and SQL performance on top. This eliminates the need to maintain separate lake and warehouse systems with redundant data copies.

**Key Mental Model:** A lakehouse is a reservoir with a water treatment plant attached — the reservoir (object storage) holds everything cheaply, and the treatment plant (Delta Lake + Databricks) makes it clean, queryable, and reliable.

**How It Works:**
- Databricks is deployed on a customer's cloud account in two logical planes: a **control plane** (Databricks-managed, handles orchestration, UI, job management) and a **data plane** (customer-managed VMs and object storage where data and compute actually live).
- All data resides in cloud object storage (S3, ADLS Gen2, GCS) in open Parquet files governed by Delta Lake transaction logs — no proprietary binary format locks in data.
- The Databricks Runtime is an optimised Apache Spark distribution that runs inside the customer data plane, executing notebooks, jobs, and SQL queries against those storage files.
- SQL Analytics and ML workloads share the same underlying Delta tables, eliminating ETL pipelines between a lake and a separate warehouse system.
- [[Cloud-Platforms/Databricks/02-Data-Engineering]] and [[Cloud-Platforms/Databricks/03-ML-AI]] both read and write to this same storage layer, achieving single-copy governance.

**Common Misconceptions:**
- Many engineers assume the Lakehouse means Databricks *is* the storage — in reality, Databricks never owns the data; it runs compute on top of storage the customer controls in their own cloud account.
- "Lakehouse replaces the warehouse" is an oversimplification; for heavy BI/reporting workloads, dedicated SQL Warehouses with Photon still outperform generic notebook clusters against the same Delta tables.

**Interview Answer Skeleton:**
- **What it is:** A unified platform architecture where a single copy of data in open-format object storage serves batch ETL, streaming, SQL analytics, and ML workloads — governed by Delta Lake ACID semantics and Unity Catalog.
- **Why it matters / trade-offs:** Eliminates data duplication between lake and warehouse tiers, reducing storage costs and governance complexity; the trade-off is that operational/OLTP workloads still require separate purpose-built databases.
- **Example or context:** A large retailer replaces their Hadoop cluster + Redshift setup with a single Databricks Lakehouse — raw ingest, cleaned Delta tables, and BI dashboards all read the same data with no nightly copy jobs.

**Free Resources:**
- [Databricks Lakehouse Architecture](https://docs.databricks.com/en/lakehouse/index.html) — official architecture overview explaining control plane, data plane, and storage separation
- [Databricks Academy](https://academy.databricks.com) — free courses including "Lakehouse Fundamentals" covering architecture concepts

---

## Delta Lake

**Status:** ⬜ Not Started

**Definition:** Delta Lake is an open-source storage layer that brings ACID transactions, schema enforcement, time travel (versioned history), and scalable metadata management to data stored in Parquet files on cloud object storage. It is the default table format on Databricks and forms the foundation of the Lakehouse architecture.

**Key Mental Model:** Delta Lake is the transaction log for your data files — like a database WAL (write-ahead log), it records every change so you can roll back, audit, or query any historical version.

**How It Works:**
- Every Delta table has a `_delta_log/` directory alongside its Parquet data files. Each committed transaction writes a JSON entry (or Parquet checkpoint at every 10 commits) to this log, recording which files were added or removed.
- ACID guarantees are achieved through **optimistic concurrency control**: writers validate their transaction against the current log version before committing; conflicting concurrent writes are detected and the losing writer retries or fails.
- **Time travel** works by replaying the transaction log to reconstruct the table state at any prior version or timestamp — no separate snapshot storage is needed.
- **Schema enforcement** rejects writes that do not match the registered schema at commit time; **schema evolution** (`mergeSchema`) allows controlled additions and can be explicitly enabled.
- **OPTIMIZE and Z-ORDER** compact small Parquet files and co-locate related rows on disk, reducing the number of files that pruning must scan for typical range query patterns. See [[Cloud-Platforms/Databricks/02-Data-Engineering]] for DLT's use of incremental Delta writes.

**Common Misconceptions:**
- Delta Lake is often confused with being Databricks-proprietary — it is Apache-licensed open source and runs on any Spark environment, including open-source Spark without a Databricks subscription.
- ACID on Delta Lake does not protect against concurrent reads seeing uncommitted data the same way a traditional RDBMS does at the row level; isolation is at the transaction/file level, and readers always see the last committed snapshot.

**Interview Answer Skeleton:**
- **What it is:** An open-source storage protocol that adds a JSON/Parquet transaction log on top of Parquet files in object storage, enabling ACID commits, time travel, and schema enforcement at scale.
- **Why it matters / trade-offs:** Replaces brittle partition-overwrite patterns with true transactional updates; the trade-off is that the transaction log becomes a bottleneck for extremely high-frequency micro-batch writes unless checkpointing is tuned correctly.
- **Example or context:** A data team runs `DELETE FROM events WHERE user_id = 123` on a Delta table for GDPR compliance — this rewrites only affected Parquet files and logs the change, something impossible on a plain Parquet lake.

**Free Resources:**
- [Delta Lake Documentation](https://docs.delta.io/latest/index.html) — open-source Delta Lake docs covering transactions, time travel, schema evolution, and optimisation
- [Databricks Delta Lake Guide](https://docs.databricks.com/en/delta/index.html) — Databricks-specific Delta features including OPTIMIZE, ZORDER, and Deletion Vectors

---

## Unity Catalog

**Status:** ⬜ Not Started

**Definition:** Unity Catalog is Databricks' unified governance layer — a centralised metastore for data assets (tables, volumes, views), ML models, files, and dashboards across all workspaces. It provides fine-grained access control, data lineage, and audit logging in a three-level namespace: `catalog.schema.table`.

**Key Mental Model:** Unity Catalog is the central library system for your entire Databricks environment — one catalogue that knows where everything is, who can access it, and where it came from.

**How It Works:**
- Privileges are defined in a **three-level hierarchy**: `CATALOG > SCHEMA > TABLE/VOLUME`. A privilege granted at the catalog level is inherited by all schemas and tables within it, while table-level grants override or restrict narrower access.
- At query execution time, the Databricks Runtime intercepts all data access calls and checks Unity Catalog privileges before allowing reads or writes — this enforcement happens inside the cluster, not at the storage layer, so it cannot be bypassed by direct object storage access when external locations are properly locked down.
- **Data lineage** is captured automatically: the runtime instruments each query to record which tables were read and written, storing lineage metadata in the Unity Catalog metastore without requiring manual tagging.
- **External Locations** and **Storage Credentials** allow Unity Catalog to manage access to specific cloud storage paths, meaning clusters need only one service principal credential (for Unity Catalog itself) rather than per-team storage credentials.
- Audit logs for all data access and privilege changes are written to a dedicated Delta table accessible to admins, enabling compliance reporting. See [[Cloud-Platforms/Databricks/05-Administration]] for cluster policy integration with Unity Catalog.

**Common Misconceptions:**
- Unity Catalog is not just a Hive Metastore replacement — it governs ML models, notebooks stored as files, dashboards, and volumes (arbitrary files), not only SQL tables.
- Enabling Unity Catalog does not automatically migrate legacy Hive tables; existing workspace-local Hive tables must be explicitly upgraded or recreated under the Unity Catalog namespace.

**Interview Answer Skeleton:**
- **What it is:** A centralised, cross-workspace governance metastore for Databricks that enforces fine-grained RBAC, tracks automated data lineage, and audits all access within a three-level `catalog.schema.table` namespace.
- **Why it matters / trade-offs:** Solves the multi-workspace governance problem where each workspace had its own isolated Hive metastore; the trade-off is migration complexity from legacy workspace metastores and the need for an account-level admin to set it up.
- **Example or context:** A financial services firm uses Unity Catalog column-level masking to let analysts query a `customers` table where PII columns return masked values, while the data engineering team sees raw data — enforced at query time without changing application code.

**Free Resources:**
- [Unity Catalog Documentation](https://docs.databricks.com/en/data-governance/unity-catalog/index.html) — setup, privilege model, lineage, and audit log configuration
- [Databricks Academy — Data Governance](https://academy.databricks.com) — free learning paths covering Unity Catalog administration and privilege management

---

## Cluster Types

**Status:** ⬜ Not Started

**Definition:** Databricks has three main compute types: All-Purpose Clusters (interactive development, notebooks, multi-user), Job Clusters (single-job automated pipelines that start and terminate per run), and SQL Warehouses (optimised for SQL Analytics, BI tools, and dbt). Serverless variants exist for both notebooks and SQL, offloading VM management to Databricks.

**Key Mental Model:** All-Purpose clusters are a shared workshop open all day. Job clusters are delivery vans that go out for one trip and return. SQL Warehouses are a dedicated service counter just for queries.

**How It Works:**
- When a cluster is created, the Databricks control plane calls the cloud provider API (EC2, Azure VMs, GCP Compute) to provision the specified number and type of VMs into the customer's data plane VPC/VNet — this negotiation typically takes 3–8 minutes for cold starts.
- **All-Purpose Clusters** maintain a live Spark context that multiple notebook users share via multiplexing; each user's commands are queued into a shared Spark job queue, meaning heavy queries from one user can starve others.
- **Job Clusters** are provisioned fresh at job start and torn down at job completion, ensuring no shared state between runs and enabling clean dependency management; they are the recommended pattern for production pipelines.
- **SQL Warehouses** (Classic) provision VMs in the customer's cloud account and use Photon for vectorised execution; **Serverless SQL Warehouses** run on Databricks-owned compute infrastructure and start in seconds rather than minutes, with costs based on DBU consumption rather than VM uptime.
- **Instance Pools** pre-warm a set of idle VMs in the customer cloud account so that clusters drawing from the pool can start in under 30 seconds by skipping the VM provisioning step. See [[Cloud-Platforms/Databricks/05-Administration]] for pool and policy management.

**Common Misconceptions:**
- SQL Warehouses are not just All-Purpose clusters with a SQL label — they use a completely different execution path (Photon C++ engine vs JVM Spark) and are optimised for concurrent, short-duration SQL queries rather than long-running Spark jobs.
- Serverless compute does not mean "free" — it still consumes DBUs; it means Databricks manages the underlying VMs so the customer pays per-query without managing cluster lifecycle.

**Interview Answer Skeleton:**
- **What it is:** Three distinct compute archetypes (All-Purpose, Job, SQL Warehouse) each optimised for different workloads, with Serverless variants that shift VM lifecycle management to Databricks.
- **Why it matters / trade-offs:** Choosing the wrong cluster type inflates costs and degrades performance; Job Clusters are cheaper for pipelines but have cold-start latency; SQL Warehouses deliver best SQL concurrency but are inappropriate for ML training.
- **Example or context:** A data platform team uses Job Clusters for nightly DLT pipelines (auto-terminated after each run), Serverless SQL Warehouses for BI dashboards (instant start, pay per query), and a small All-Purpose cluster per data scientist for exploratory work.

**Free Resources:**
- [Databricks Compute Documentation](https://docs.databricks.com/en/compute/index.html) — cluster types, sizing guidelines, autoscaling, and serverless configuration
- [Databricks Academy](https://academy.databricks.com) — free courses covering cluster selection and cost optimisation patterns

---

## Photon Engine

**Status:** ⬜ Not Started

**Definition:** Photon is a native vectorised query engine written in C++ that accelerates Apache Spark SQL and DataFrame operations on Databricks. It provides 2–10x query speedup for SQL workloads through vectorised execution without any code changes required from the user.

**Key Mental Model:** Photon is a turbocharger for Spark SQL — the same queries run faster without any changes to the code, because the execution engine processes data in vectorised batches rather than row by row.

**How It Works:**
- Photon replaces the JVM-based Spark execution engine for SQL and DataFrame operations by intercepting the compiled physical plan and substituting native C++ operator implementations for supported operations (scans, joins, aggregations, sorts, window functions).
- Execution is **vectorised**: rather than processing one row at a time through the operator tree, Photon processes data in columnar batches of thousands of rows simultaneously, saturating CPU SIMD (AVX2/AVX-512) instructions for maximum throughput.
- Photon operates transparently alongside standard Spark — if an operator is not yet supported by Photon, the query falls back to JVM Spark for that specific step and then returns to Photon for subsequent supported operators. The query plan in the Spark UI indicates which operators are Photon-accelerated.
- Photon is deeply integrated with Delta Lake's columnar Parquet file format — it reads Parquet column chunks directly into native memory, avoiding JVM object overhead and garbage collection pauses that limit JVM Spark throughput at scale.
- **SQL Warehouses always use Photon**; All-Purpose clusters require Photon-enabled runtimes (DBR 9.1 LTS and above on appropriate instance types). See [[Cloud-Platforms/Databricks/04-SQL-Analytics]] for Photon's role in warehouse performance.

**Common Misconceptions:**
- Photon does not accelerate all Spark workloads equally — Python UDFs and RDD-based code run on the JVM and cannot leverage Photon; performance gains are primarily in SQL scans, joins, and aggregations over Delta/Parquet data.
- Photon is not a separate product to license — it is included in the Databricks Runtime for SQL Warehouses and available on eligible All-Purpose cluster node types at no separate charge.

**Interview Answer Skeleton:**
- **What it is:** A native C++ vectorised execution engine that replaces JVM-based Spark operators for SQL and DataFrame workloads, enabling SIMD batch processing of columnar Parquet data.
- **Why it matters / trade-offs:** Delivers significant performance improvements for SQL-heavy ETL and BI workloads without code changes; the trade-off is that Python UDFs and arbitrary Spark RDD operations remain JVM-bound and do not benefit.
- **Example or context:** A data team migrates a slow Hive-on-YARN reporting pipeline to a Databricks SQL Warehouse — the same aggregate queries run 5x faster due to Photon's vectorised scan and hash join operators, with no SQL rewrites needed.

**Free Resources:**
- [Photon Documentation](https://docs.databricks.com/en/compute/photon.html) — supported operations, performance benchmarks, and configuration guidance
- [Databricks Academy](https://academy.databricks.com) — performance tuning learning paths covering Photon and query optimisation

---

## Storage-Compute Separation

**Status:** ⬜ Not Started

**Definition:** In Databricks, data lives in cloud object storage (S3, ADLS Gen2, GCS) independently of the compute clusters that process it. Clusters read from and write to external storage; shutting down a cluster does not delete data. This enables elastic scaling and eliminates the cost inefficiency of the coupled storage-compute model in traditional on-premise Hadoop clusters.

**Key Mental Model:** Data is in the cloud filing cabinet; compute is the team working at desks. The team can grow or shrink without touching the filing cabinet — and the filing cabinet is always there even when everyone goes home.

**How It Works:**
- Cloud object storage (S3, ADLS Gen2, GCS) acts as the durable persistence layer, storing Delta Lake Parquet files and transaction logs indefinitely regardless of cluster state.
- Clusters mount or directly access object storage paths via cloud-native credentials (IAM roles on AWS, Managed Identities on Azure) managed by Unity Catalog External Locations, so no data is co-located on cluster disk.
- **Autoscaling** works because adding worker nodes simply means more Spark executors reading from the same shared object storage paths — there is no data rebalancing step required as there would be with a Hadoop HDFS cluster.
- Cluster termination reclaims only the cloud VMs; the Delta Lake transaction log preserves the exact committed state, so any subsequent cluster can resume exactly where the previous one left off.
- The separation introduces a **network I/O cost**: Spark shuffle operations must write intermediate data to disk or cloud-native shuffle storage (DBR 12+ supports cloud-native shuffle), since there is no fast local HDFS to cache intermediate results. This is mitigated by Photon's reduced intermediate data volume. See [[Cloud-Platforms/00-Comparison-Matrix]] for how this compares to Snowflake's similar architecture.

**Common Misconceptions:**
- Storage-compute separation does not eliminate the need for local disk — Spark shuffle still writes temporary data to cluster-attached SSDs during large joins and sorts; the separation applies to durable storage, not intermediate computation.
- "Shut down clusters to save money" is correct for All-Purpose clusters but not sufficient for cost control — data storage costs in object storage accumulate continuously and can exceed compute costs for large datasets with long retention.

**Interview Answer Skeleton:**
- **What it is:** An architectural pattern where durable data resides permanently in cloud object storage and is decoupled from stateless compute clusters that can be elastically scaled, started, or stopped independently.
- **Why it matters / trade-offs:** Enables independent scaling of storage and compute, eliminates data loss risk from cluster failures, and allows multiple workloads to share the same data; the trade-off is higher latency for small random reads vs local SSDs and shuffle I/O bottlenecks on large aggregations.
- **Example or context:** A media company runs 500-node Spark clusters for hourly batch reprocessing, then scales to zero overnight — all data persists safely in S3 and the next day's cluster reads it immediately with no migration step.

**Free Resources:**
- [Databricks Architecture Overview](https://docs.databricks.com/en/lakehouse/index.html) — explains control plane, data plane, and storage separation model
- [Databricks Academy](https://academy.databricks.com) — free foundational courses covering cloud architecture and cost optimisation patterns

---
