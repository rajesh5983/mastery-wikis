# Layer 6 — Platform

> **Framework:** Cloud data platforms and production operations for scalable analytics infrastructure.

---

## Cloud Storage and Compute Fundamentals

**Status:** ⬜ Not Started

**Definition:** Cloud object storage (S3, Azure Data Lake Storage, GCS) provides virtually unlimited, durable, cheap storage for raw and processed data at roughly $0.02/GB/month. Cloud compute (VMs, managed clusters, serverless functions) provides processing capacity on demand. The foundational architectural principle of modern data platforms is storage-compute separation — each scales independently, and you pay separately for each.

**Key Mental Model:** Storage-compute separation is like separating a library from its reading rooms. The books (data) live in the library indefinitely at minimal cost; reading rooms (compute clusters) can expand to handle a rush of readers and then disappear entirely during quiet hours — you never pay for empty reading rooms.

**How It Works:**
- Object storage systems (S3, GCS, ADLS Gen2) are designed for large sequential reads and writes, not random row access. Internally they store data as immutable objects identified by keys (paths), with no concept of directories or mutable updates at the byte level. This immutability is what makes atomic multi-file transactions possible via external table format transaction logs (Delta Lake, Iceberg) without the storage layer itself needing to support transactions.
- S3-compatible storage provides 11 nines of object durability through multi-Availability Zone replication and internally detects silent bit rot via checksum verification. This durability exceeds what most enterprises could achieve with on-premises SAN storage, eliminating the need for separate backup infrastructure for data lake contents.
- Compute clusters in cloud platforms (Snowflake virtual warehouses, BigQuery slots, Databricks clusters) read data from object storage over a high-bandwidth internal network. This remote read architecture has higher per-query latency than local disk access (~50ms vs ~0.1ms for NVMe), which modern platforms address through local SSD caching layers that keep hot data close to compute.
- Auto-scaling and auto-suspend are the cost control mechanisms enabled by compute-storage separation: a Snowflake virtual warehouse can be suspended after N minutes of inactivity (stopping compute billing entirely) while data remains fully accessible in S3 storage at storage-only cost. This is impossible with tightly coupled architectures like traditional on-premises MPP databases.
- Spark on cloud object storage (EMR on S3, Databricks on ADLS) splits the read path: executor tasks request data blocks from the storage system over the internal network, with S3 Select or equivalent predicate pushdown reducing bytes transferred for filtered reads. File format (Parquet with predicate pushdown vs CSV) dramatically affects the bytes transferred from storage to compute per query. See [[DE-Engineer/05-Scale]] for file format details.

**Common Misconceptions:**
- Separating storage and compute always increases per-query costs compared to co-located architectures — compute-storage separation reduces total cost of ownership by eliminating idle compute costs. In a traditional co-located MPP database, you pay for compute 24/7 even when no queries run. Separation means paying for compute only during actual query execution.
- Cloud object storage is just cheaper SAN/NAS block storage with S3 API — object storage has fundamentally different access semantics: eventually consistent (though S3 is now strongly consistent after 2020), optimised for large sequential access, no partial-object update support, and different durability mechanisms. Treating it like block storage (many small random writes, frequent metadata operations) causes serious performance problems.

**Interview Answer Skeleton:**
- **What it is:** The foundational cloud infrastructure architecture that decouples data storage (object storage — cheap, durable, infinite) from compute processing (ephemeral clusters that scale to zero), enabling both independent scaling and the economic model of modern cloud data platforms.
- **Why it matters / trade-offs:** Storage-compute separation is the architectural basis for every major cloud data platform (Snowflake, BigQuery, Databricks, Redshift Spectrum). The trade-off vs co-located architectures is slightly higher per-query I/O latency, addressed by local SSD caching in all major platforms.
- **Example or context:** A Snowflake virtual warehouse scales up from XS to 2XL for a heavy end-of-month reporting batch, processes 10TB in 8 minutes, then auto-suspends. The data remains in S3 at storage cost only; the compute cost applies only during that 8-minute window. A traditional on-premises MPP cluster would have paid for compute capacity around the clock.

**Free Resources:**
- [Databricks Academy](https://academy.databricks.com) — free courses covering cloud storage architecture, compute-storage separation, and how Databricks and Delta Lake interact with object storage
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference covering cloud infrastructure fundamentals, object storage behaviour, and platform architecture patterns

---

## Snowflake, BigQuery, Redshift, and Databricks

**Status:** ⬜ Not Started

**Definition:** These four platforms dominate cloud data engineering. Snowflake is a multi-cloud SQL warehouse with independently scalable virtual warehouses per workload. BigQuery is Google's serverless, fully managed warehouse with automatic scaling and per-query billing. Redshift is AWS's provisioned columnar warehouse with Spectrum for external S3 queries. Databricks is a unified analytics platform combining Spark, Delta Lake, SQL analytics, and ML on one governed compute layer.

**Key Mental Model:** Snowflake is a professional kitchen — powerful, well-organised, you pay by the hour for counter time. BigQuery is a restaurant where you pay per dish ordered, not for kitchen time. Databricks is a combined kitchen and food research laboratory — SQL, ML training, streaming, and governance in one integrated space.

**How It Works:**
- Snowflake's virtual warehouse (VW) architecture isolates workloads: each VW is an independent cluster of compute nodes that reads from shared storage. Multiple VWs can simultaneously query the same tables without resource contention — the ETL VW running heavy transformations doesn't compete with the BI VW serving dashboard queries. VW sizing (XS to 6XL) doubles compute resources per tier, and Snowflake's query profile shows where each query spent its time for targeted optimisation.
- BigQuery uses a serverless, slot-based compute model: queries are automatically decomposed into parallel tasks assigned to Google's internal compute pool. On-demand billing charges per TB scanned; reserved slots provide predictable cost for high-query-volume workloads. Partition pruning and column pruning directly reduce scanned bytes and therefore cost — unpartitioned, wide tables can be extremely expensive at scale.
- Redshift uses a shared-nothing MPP architecture with data distributed across fixed compute nodes by a distribution key. The distribution key determines which node stores each row, affecting join performance: if the join key matches the distribution key, the join is local (no data movement); if not, Redshift must redistribute data (hash redistribution), consuming network bandwidth and time. Choosing the correct dist key and sort key is a critical schema design decision for Redshift performance.
- Databricks Unity Catalog provides a three-level namespace (catalog.schema.table) with centralised access control, column-level masking, row-level filters, audit logging, and data lineage tracking across all Databricks workspaces and engines. Delta Lake's transaction log provides ACID writes, time travel (SELECT * FROM table VERSION AS OF 10), and CDC change feed. These together make Databricks a full lakehouse governance platform, not just a Spark execution environment.
- Query acceleration layers across platforms: Snowflake's result cache returns identical query results instantly without compute cost; Snowflake's local disk cache (SSD) warms frequently accessed data; BigQuery BI Engine caches specific tables in-memory for sub-second dashboard queries; Databricks SQL Photon engine is a vectorised C++ execution engine that accelerates SQL queries 2–4x versus standard Spark SQL. See [[DE-Engineer/05-Scale]] for distributed query mechanics.

**Common Misconceptions:**
- BigQuery is always the cheapest option because of per-query billing — poorly optimised BigQuery queries on unpartitioned, unclusterd tables scanning terabytes of data can generate enormous per-query costs. Partition and cluster tables, use column projection, and leverage BI Engine for high-frequency dashboard queries to control spend. BigQuery's total cost of ownership depends heavily on query optimisation discipline.
- Databricks is primarily an ML/AI platform and is unsuitable for standard data engineering workloads — Databricks is a full data engineering platform: Delta Live Tables automates CDC and incremental pipeline orchestration, Databricks SQL provides warehouse-grade SQL analytics, and Unity Catalog provides enterprise governance. Many organisations use it as their primary data engineering platform with no ML workloads at all.

**Interview Answer Skeleton:**
- **What it is:** The four dominant cloud data platforms, each with distinct compute architectures (virtual warehouse isolation vs serverless auto-scaling vs MPP fixed nodes vs Spark clusters), pricing models, and optimal workload profiles — selecting the right one shapes the entire data stack for years.
- **Why it matters / trade-offs:** Platform choice determines tooling ecosystem, cost model, governance capability, and scalability ceiling. Snowflake excels for mixed SQL workloads with workload isolation needs; BigQuery for Google Cloud shops with variable query volumes; Databricks for organisations combining data engineering, ML, and streaming on one platform; Redshift for AWS-native shops with stable, well-understood workloads.
- **Example or context:** Choosing between Snowflake and BigQuery: a company with highly variable query volumes (quiet during the week, massive batch runs on weekends) benefits from BigQuery's serverless auto-scaling and per-query cost model. A company with predictable high-volume daily analytics benefits from Snowflake's provisioned virtual warehouses with predictable hourly costs and result caching for repeated dashboard queries.

**Free Resources:**
- [Databricks Academy](https://academy.databricks.com) — free courses on Databricks, Delta Lake, Unity Catalog, and Databricks SQL — covers the platform end-to-end
- [dbt Documentation](https://docs.getdbt.com) — covers dbt adapter configurations for Snowflake, BigQuery, Redshift, and Databricks, illustrating platform-specific SQL behaviours and optimisation patterns

---

## IAM, Security, and Governance Basics

**Status:** ⬜ Not Started

**Definition:** IAM (Identity and Access Management) controls which identities (users, service accounts, roles) can access which data platform resources and what operations they can perform. Data governance extends IAM with column-level security (masking PII fields per role), row-level filters (limiting visible rows by data domain ownership), audit logging, and compliance controls for GDPR, HIPAA, and SOC 2.

**Key Mental Model:** IAM is a building's keycard system — some cards open only the lobby, others open specific labs, a few open everything. Data governance adds a camera and access log in every room, recording who accessed which data, when, from where, and what they did with it.

**How It Works:**
- Role-based access control (RBAC) is the standard model: permissions are granted to roles, not directly to users. Users and service accounts are assigned roles. In Snowflake, roles form a hierarchy — ROLE_A can inherit permissions from ROLE_B. Least-privilege design means each pipeline service account has the minimum role required: read access on source tables, write access only to its own output schema.
- Column-level dynamic data masking allows the same query to return different data depending on the querying role. A column masked as `MASK('SSN', role => 'ANALYST')` returns '***-**-****' for analyst roles and the real value for 'DATA_ENGINEER' or 'COMPLIANCE' roles. The masking policy is applied transparently at query execution — no application code change required, and the physical data is not modified.
- Row-level security filters automatically append a WHERE predicate to every query based on the querying user's attributes. A row access policy WHERE region = current_user_region means a user in the 'EMEA' role automatically sees only EMEA rows, without any query modification. This is enforced by the database engine, not by application-level WHERE clauses that could be accidentally omitted.
- Service account security for pipeline automation: pipeline tools (Airflow, dbt, Fivetran) authenticate to cloud platforms via service accounts with IAM roles or OAuth2 client credentials. These accounts should be scoped to exactly the tables they need, without console login capability, and rotated on a regular schedule. Secrets (connection strings, API keys) must be stored in a secrets manager (AWS Secrets Manager, Vault) — never in code or environment variables in logs.
- Audit logging records every data access event (who, what, when, from where, how many rows returned) and is the primary evidence trail for compliance audits and breach investigations. Snowflake's QUERY_HISTORY, BigQuery's Cloud Audit Logs, and Databricks' audit logs all provide this at the platform level. Exporting and retaining audit logs to immutable storage is a SOC 2 Type II requirement.

**Common Misconceptions:**
- IAM and security are the security team's responsibility, not data engineers' — data engineers provision every service account, table grant, and column mask for every pipeline they build. The security team sets policies and does reviews; engineers implement the access control layer for their data products. An engineer who doesn't understand IAM ships systems with overly broad permissions that create compliance exposure.
- Read-only access to all tables is safe and appropriate for analysts — read access to unmasked PII tables (name, email, SSN, health records) violates GDPR and HIPAA even when the data is never exported or misused. Column-level masking and role-based data product access are the correct controls for analyst access to sensitive tables.

**Interview Answer Skeleton:**
- **What it is:** The identity, access control, and audit infrastructure that governs who can access which data assets, with what permissions, and with full auditability — implemented through RBAC, column-level masking, row-level security, and audit logging at the data platform layer.
- **Why it matters / trade-offs:** Data breaches and compliance failures typically originate from over-permissioned access, not sophisticated attacks. Every pipeline a data engineer builds creates access that persists indefinitely — getting IAM right at build time is far cheaper than remediating over-permissioned systems after a compliance audit or incident.
- **Example or context:** Implementing PII protection for a customer table in Snowflake: create a masking policy that returns NULL for email and phone for the ANALYST role and real values for the COMPLIANCE role. Grant SELECT on the customer table to ANALYST role. The masking policy is applied transparently — analysts see masked data without knowing the real values exist in the column. QUERY_HISTORY provides the audit trail.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — covers dbt governance features including model-level access controls, data contracts, and integration with platform IAM systems
- [Databricks Academy](https://academy.databricks.com) — covers Unity Catalog access controls, column masking, row-level security, and audit logging on the Databricks platform

---

## Schema Evolution and Metadata/Catalog Concepts

**Status:** ⬜ Not Started

**Definition:** Schema evolution is the managed process of modifying table structures (adding columns, changing types, renaming fields, deprecating columns) without silently breaking downstream pipeline consumers. A data catalog is a searchable, centralised inventory of data assets documenting their schemas, lineage (where data came from and flows to), ownership, quality metrics, and access patterns — the foundational infrastructure for data discoverability at scale.

**Key Mental Model:** Schema evolution is renovating a building while tenants are still living in it — adding a new floor is safe, but removing a load-bearing wall without notifying everyone first causes the structure to collapse. A data catalog is the building directory — tells you what exists on each floor, who owns each room, and who to call when something breaks.

**How It Works:**
- Backward-compatible schema changes (safe): adding a nullable column with a default value, widening a numeric type (INT to BIGINT), adding a new table or view. These changes don't break existing queries that don't reference the new column. Delta Lake and Iceberg support these via schema evolution with `mergeSchema = true`.
- Backward-incompatible schema changes (breaking): dropping a column, renaming a column, narrowing a type (FLOAT to INT), changing column semantics. These require a coordinated migration: add the new column, migrate consumers, deprecate the old column, then drop it. The migration window can be weeks or months depending on downstream consumer count.
- Delta Lake schema enforcement rejects writes with schema mismatches by default (schemaEvolution disabled). This prevents an upstream process from accidentally adding unexpected columns or changing types without an explicit schema migration. Schema evolution mode (mergeSchema = true) merges new columns automatically — appropriate for ELT pipelines ingesting semi-structured sources with organic schema growth.
- dbt documentation as a lightweight catalog: dbt docs generates a searchable HTML site from model YAML definitions, exposing column descriptions, test coverage, ownership meta fields, and the full DAG lineage graph. This is the minimum viable catalog for dbt-centric data stacks — adequate for teams with < 200 models, before investing in a dedicated catalog tool like DataHub, Atlan, or Alation.
- Data lineage in catalogs tracks the full transformation path: which source tables fed into which staging models, which staging models fed into which marts, and which dashboards or ML models consume each mart. Column-level lineage (tracking specific column derivations across transformations) enables impact analysis — before dropping a column, identify every downstream consumer that references it. Unity Catalog, OpenLineage, and Marquez provide lineage capture. See [[DE-Engineer/06-Platform]] for Unity Catalog specifics.

**Common Misconceptions:**
- Adding a new column is always a safe, non-breaking schema change — adding a NOT NULL column without a default value breaks any INSERT statement that doesn't explicitly provide the new column's value. In streaming pipelines where schema is inferred from source events (Kafka consumer, CDC feed), a new required field in the source schema can break the entire consumer pipeline silently until a null constraint violation surfaces.
- Data catalogs are documentation overhead that slow teams down — without a catalog, engineers spend hours searching for the right table, guessing ownership, and duplicating work building models that already exist. The time cost of undocumented schemas compounds: each new engineer must re-learn the same undocumented structures, and each schema change requires tribal knowledge to understand blast radius.

**Interview Answer Skeleton:**
- **What it is:** The discipline of managing table structure changes in a way that maintains backward compatibility for existing consumers, combined with catalog infrastructure that makes data assets discoverable, trustworthy, and understandable to the engineers and analysts who need them.
- **Why it matters / trade-offs:** Unmanaged schema changes are one of the most common causes of production pipeline failures. A column rename that seems trivial in the source system can simultaneously break 15 downstream dbt models, 8 dashboards, and 3 ML feature pipelines. The trade-off of maintaining a catalog is ongoing documentation effort; the trade-off of not maintaining one is hours of investigation time per incident.
- **Example or context:** Adding a new required column to a high-traffic fact table in Snowflake: (1) add the column as nullable with a NULL default (no breaking change); (2) backfill historical values; (3) update all INSERT/MERGE statements to populate the column; (4) after all writers are updated, add a NOT NULL constraint. This 4-step migration maintains backward compatibility at every stage — no downstream consumers break at any step.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — covers schema evolution handling, dbt docs as a catalog, data contracts, and governance in dbt projects
- [Databricks Academy](https://academy.databricks.com) — covers Unity Catalog, Delta Lake schema evolution, data lineage, and metadata management in the Databricks platform

---

## Data Contracts and Access Patterns

**Status:** ⬜ Not Started

**Definition:** A data contract is a versioned, formal agreement between a data producer team and its downstream consumers specifying the schema (column names, types, nullability), semantic definitions (what each field means), SLA commitments (freshness, completeness), and breaking change notification process. Access patterns define the approved mechanisms through which consumers interact with data — via views, APIs, or governed direct table access — establishing what guarantees they can depend on.

**Key Mental Model:** A data contract is an API contract for data — the producer commits to a stable interface and behaviour, consumers build on top of it, and breaking changes require a version bump and a migration period with advance notice. Allowing direct access to internal implementation tables is equivalent to exposing private class fields as a public API.

**How It Works:**
- Data contracts have three layers: (1) technical schema contract (column names, types, constraints — enforced by schema evolution rules and CI checks); (2) semantic contract (what the data means — enforced by documentation and tests); (3) SLA contract (freshness, availability, accuracy — enforced by monitoring and alerting). All three must be maintained for a contract to be trustworthy.
- dbt implements technical contracts through model-level `contract: enforced: true` configuration, which validates that the compiled SQL produces columns matching the declared column types and constraints at model build time. This fails fast in CI pipelines before bad schema reaches production, rather than breaking consumers at runtime.
- Consumer isolation through views: exposing a view (not the underlying table) to downstream consumers decouples the physical table structure from the contract. The producer can refactor the underlying model (rename internal columns, change implementation) without breaking consumers, as long as the view continues returning the contracted columns with the contracted types. Views are the data equivalent of an API facade pattern.
- Semantic versioning for breaking changes: a contract version bump (v1 → v2) with a deprecation window (maintain v1 and v2 simultaneously for 30 days) allows consumers to migrate on their own timeline without emergency responses. This requires the producer to maintain parallel model versions during the migration window — additional operational overhead that is the cost of managing external consumers.
- Event-driven data contracts (Schema Registry): for Kafka-based pipelines, Confluent Schema Registry enforces Avro or Protobuf schema compatibility on every producer write. Full compatibility modes (backward, forward, full) define which schema changes are allowed. A producer attempting to publish events that break the registered schema receives a hard rejection, preventing schema-induced consumer failures before they reach production. See [[DE-Engineer/05-Scale]] for Kafka fundamentals.

**Common Misconceptions:**
- Data contracts are bureaucratic process overhead that slows delivery — without contracts, schema changes silently break downstream consumers who discover the failure only when their pipeline fails or their dashboard shows wrong numbers. Contracts shift this from an unplanned incident (expensive, urgent, reputation-damaging) to a managed process (planned migration, advance notice, coordinated deployment).
- Allowing direct table access is acceptable when the table is "internal" — any table accessed by more than one team is effectively a public API regardless of intent. When teams access internal tables directly, every refactoring of the producer's model becomes a breaking change for every consumer, creating a coordination bottleneck that slows the entire organisation.

**Interview Answer Skeleton:**
- **What it is:** Formal, versioned agreements between data producer and consumer teams specifying schema, semantics, and SLAs — combined with controlled access mechanisms (views, APIs, schema registries) that decouple the contract from the underlying implementation and make breaking changes an explicit, managed process.
- **Why it matters / trade-offs:** At scale with multiple teams sharing data assets, uncoordinated schema changes cause cascading pipeline failures across the organisation. Contracts prevent this by making breaking changes visible and coordinated. The trade-off is producer overhead — maintaining backward-compatible interfaces and migration windows requires discipline and engineering time.
- **Example or context:** A customer events table shared with five downstream teams: contract specifies 25 columns with types and nullability, freshness SLA of < 4 hours, and a 30-day deprecation notice for any breaking column change. Access is via a versioned view (`customer_events_v2`). When the producer needs to rename an internal column, they update the view to alias the new column name to the contracted name — consumers see no change. The contract is the view definition, not the table definition.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — covers dbt data contracts, model contracts with enforcement, and governing access to data models through exposure and access configuration
- [Databricks Academy](https://academy.databricks.com) — covers Unity Catalog data sharing, access control, and governing data contracts across teams and catalogs in the Databricks platform

---

## Performance Tuning and Cost Optimisation

**Status:** ⬜ Not Started

**Definition:** Performance tuning identifies and resolves query execution bottlenecks — missing clustering keys, suboptimal join strategies, excessive data scans, and memory spills — to reduce query latency and pipeline runtime. Cost optimisation reduces cloud spend through compute right-sizing, auto-suspend, query result caching, partition pruning, storage tiering, and eliminating redundant computation.

**Key Mental Model:** Performance tuning is diagnosing a slow car — read the diagnostics (query profile / EXPLAIN plan), identify the bottleneck (full table scan, sort spill, network shuffle), then apply the targeted fix (add clustering key, increase memory, add filter). Cost optimisation is turning the engine off when the car is parked and choosing the right car for each trip.

**How It Works:**
- Snowflake query profile is the primary diagnostic tool: it shows a visual DAG of execution steps with time and byte statistics per node. The most common bottlenecks — TableScan consuming large percentages of total time, Disk Spill nodes indicating memory exhaustion, and large remote disk read percentages indicating cold cache — each point to specific optimisation actions: add clustering key, increase warehouse size, or trigger a cache warm-up query.
- Clustering keys in Snowflake reorder micro-partitions on disk by the specified column values, improving range scan efficiency. A table clustered on order_date means queries filtering by date range skip micro-partitions outside the range. Clustering depth (a metric in SYSTEM$CLUSTERING_INFORMATION) indicates how well the table is clustered — depth > 1 means significant clustering benefit; depth approaching total micro-partitions means minimal benefit.
- BigQuery cost control levers: partition by DATE/TIMESTAMP columns + cluster by high-cardinality filter columns reduces bytes scanned. Column projection (SELECT only needed columns) reduces scanned bytes directly. Materialised views precompute expensive aggregations and are automatically used by the query planner when queries can be satisfied from the view — zero application code change required.
- Auto-suspend and auto-resume in Snowflake: configuring a virtual warehouse to suspend after N minutes of inactivity eliminates idle compute billing (which can be the majority of total compute cost for intermittent workloads). Auto-resume on query submission adds ~3-5 seconds of cold-start latency — acceptable for batch pipelines, potentially noticeable for interactive dashboards (use a separate always-on small warehouse for BI tools).
- Databricks cost optimisation: use spot/preemptible instances for fault-tolerant batch workloads (50-80% cost reduction vs on-demand), configure autoscaling to right-size clusters to actual parallelism needs, use Delta Lake file compaction (OPTIMIZE command) to reduce small-files overhead that inflates read latency and compute cost, and leverage Photon vectorised engine for SQL-heavy workloads to reduce cluster-hours required.

**Common Misconceptions:**
- Scaling up compute (larger warehouse, bigger cluster) is the first response to slow queries — the majority of query performance problems are caused by missing clustering keys, poorly written SQL, or excessive data scans — not compute shortage. Scaling compute on a poorly optimised query increases cost proportionally without addressing the root cause. Profile first, tune the query and schema, then scale compute if the bottleneck is genuinely CPU/memory.
- Storage costs are negligible in cloud data platforms — petabyte-scale object storage at $0.02/GB/month is $20,000/month per PB. Storing uncompressed raw data, maintaining excessive time travel history (Delta/Iceberg default is 30 days of retained versions), and not purging obsolete intermediate tables accumulates to significant monthly spend. Storage governance (TTLs, compression, archival tiers) is a real cost engineering lever.

**Interview Answer Skeleton:**
- **What it is:** The practice of diagnosing query execution plans to identify bottlenecks (full scans, memory spills, missing indexes/clustering), applying targeted fixes (clustering keys, materialised views, query rewrites), and implementing cost controls (auto-suspend, partition pruning, spot instances) to minimise cloud spend.
- **Why it matters / trade-offs:** Unoptimised platforms waste money and miss SLAs simultaneously. Engineers who understand cost/performance trade-offs — and can explain the root cause of a slow query from a query profile — are significantly more valuable than those who only write correct SQL. The trade-off of aggressive optimisation is additional schema complexity (clustering keys require maintenance, materialised views add refresh cost).
- **Example or context:** A Snowflake query scans a 10TB orders table in 45 seconds. Query profile shows TableScan at 92% of query time with 0% partition pruning. Investigation: the table has no clustering key, and the WHERE clause filters on order_date. Fix: define `CLUSTER BY (order_date)` and allow auto-clustering to run. After clustering reaches depth ~1.1, the same query scans < 5% of partitions and completes in 3 seconds — without changing the query or increasing warehouse size.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — covers dbt performance patterns, materialisation strategies, and optimising model build performance across cloud warehouses
- [Databricks Academy](https://academy.databricks.com) — free courses on Spark performance tuning, Delta Lake OPTIMIZE, cluster configuration, and cost management on Databricks

---
