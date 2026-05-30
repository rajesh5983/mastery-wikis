# Databricks — SQL Analytics

---

## SQL Warehouses (Serverless vs Classic)

**Status:** ⬜ Not Started

**Definition:** A Databricks SQL Warehouse is a compute cluster optimised for SQL queries, BI tool connections (Power BI, Tableau), and dbt. Classic warehouses provision VMs in the customer's cloud account and use Photon for vectorised execution. Serverless warehouses run on Databricks-owned GPU and CPU infrastructure, start in seconds, and are billed per-query with no idle VM cost.

**Key Mental Model:** Classic SQL Warehouses are like renting a car — you choose the model, you manage it. Serverless is like calling a taxi — it appears when you need it, disappears when you're done, and you pay only for the ride.

**How It Works:**
- **Classic SQL Warehouses** provision EC2/Azure VMs into the customer's data plane VPC; the Databricks control plane instructs the cloud provider to start the specified instance type and count, which typically takes 3–8 minutes. The Photon C++ engine runs on these VMs and executes all SQL queries.
- **Serverless SQL Warehouses** bypass the customer's cloud VM provisioning entirely — compute runs on Databricks-managed infrastructure with pre-warmed container pools, enabling cold start in under 10 seconds. The customer's data plane still holds the Delta Lake files; only compute is offloaded.
- Photon powers SQL execution on both Classic and Serverless variants — it reads Delta Parquet column chunks directly from object storage into native memory, executes vectorised scans/joins/aggregations in C++, and writes results back to object storage or returns them to the client.
- **Auto-stop** terminates a Classic warehouse after a configurable idle period (default 10 minutes); Serverless warehouses scale to zero immediately when there are no active queries, eliminating idle cost entirely.
- **Cluster sizing** (`2X-Small` to `4X-Large`) controls how many nodes are in the warehouse — larger sizes process more data in parallel and handle higher concurrency, but cost proportionally more DBUs per hour. For serverless, Databricks auto-selects appropriate compute size based on query demand. See [[Cloud-Platforms/Databricks/05-Administration]] for cluster policies that enforce warehouse size limits.

**Common Misconceptions:**
- Serverless SQL Warehouses are not always cheaper than Classic — for sustained high-concurrency workloads running 24/7, Classic warehouses on reserved VM pricing can be significantly cheaper than per-DBU Serverless billing; Serverless excels for intermittent BI dashboards and development workloads.
- SQL Warehouses are not just All-Purpose clusters with a different name — they use a fundamentally different execution stack (Photon C++ vs JVM Spark) and are architecturally optimised for concurrent short queries, not for long-running Spark ML training jobs.

**Interview Answer Skeleton:**
- **What it is:** Two variants of SQL-optimised compute: Classic (customer-cloud VMs with Photon, 3–8 minute cold start) and Serverless (Databricks-managed pre-warmed infrastructure, sub-10-second start, instant scale-to-zero), both executing queries via Photon against Delta Lake tables.
- **Why it matters / trade-offs:** Serverless dramatically reduces BI dashboard latency and eliminates idle cost; the trade-off is higher per-DBU cost at sustained load and reduced control over compute configuration compared to Classic warehouses.
- **Example or context:** A BI team with 50 analysts who query dashboards during business hours switches from a Classic SQL Warehouse (idle overnight at full VM cost) to Serverless — the warehouse scales to zero each evening, cutting compute cost by 60% while analysts see faster first-query response in the morning.

**Free Resources:**
- [Databricks SQL Warehouse Documentation](https://docs.databricks.com/en/compute/sql-warehouse/index.html) — Classic vs Serverless comparison, sizing guide, and auto-stop configuration
- [Databricks Academy](https://academy.databricks.com) — free courses covering SQL Warehouse optimisation, Photon, and cost management

---

## Databricks SQL

**Status:** ⬜ Not Started

**Definition:** Databricks SQL is the SQL analytics product within the Databricks platform — a web-based SQL editor, dashboard builder, and alert system for running queries against Delta Lake tables governed by Unity Catalog. It connects to SQL Warehouses and provides a collaborative environment for analysts who prefer SQL over notebooks.

**Key Mental Model:** Databricks SQL is the data analyst's workbench within Databricks — write SQL, build charts, share dashboards, and set alerts, all without leaving the platform.

**How It Works:**
- Queries written in the Databricks SQL editor are submitted to the selected SQL Warehouse via the Databricks SQL connector protocol; the warehouse executes the query using Photon against the Delta tables registered in Unity Catalog and returns results to the editor.
- **Query history** is recorded as a queryable system table in Unity Catalog (`system.query.history`), capturing query text, execution time, user, warehouse, and bytes scanned — enabling performance auditing and cost attribution without external APM tools.
- **Dashboards** are defined as a collection of visualisations backed by named queries; auto-refresh schedules can be set per dashboard, causing the backing queries to re-execute on the SQL Warehouse on a cron schedule without user intervention.
- **Alerts** evaluate a query's result against a threshold on a schedule (e.g., `if COUNT(*) where error_code IS NOT NULL > 0`) and send notifications via email, Slack, or webhook — providing data quality monitoring through SQL without a separate observability stack.
- Databricks SQL integrates with BI tools via standard JDBC/ODBC connectors; Power BI and Tableau can connect to a SQL Warehouse and query Unity Catalog tables using direct SQL pass-through, with Photon accelerating the underlying execution. See [[Cloud-Platforms/Databricks/04-SQL-Analytics#SQL Warehouses (Serverless vs Classic)]] for the compute layer powering queries.

**Common Misconceptions:**
- Databricks SQL is not a replacement for Power BI or Tableau — it is an analyst workbench for ad hoc queries and lightweight dashboards; complex enterprise BI with pixel-perfect reports, multi-page layouts, and advanced visual types still require dedicated BI tools.
- SQL queries in the editor are not automatically governed by row-level security without Unity Catalog row filters configured — simply using Databricks SQL does not enforce data masking; governance requires explicitly configured Unity Catalog policies.

**Interview Answer Skeleton:**
- **What it is:** A SQL-native analytics workbench within Databricks providing a query editor, auto-refresh dashboards, threshold alerting, and query history, all connecting to SQL Warehouses via Unity Catalog-governed table access.
- **Why it matters / trade-offs:** Gives analysts a self-service SQL environment within the governed Databricks ecosystem without needing separate BI tool infrastructure; the trade-off is limited visualisation sophistication compared to dedicated BI tools like Power BI.
- **Example or context:** A business analyst monitors daily revenue metrics via a Databricks SQL dashboard that auto-refreshes every 15 minutes — when revenue drops below threshold, a Databricks SQL Alert fires a Slack notification, eliminating the need for a separate monitoring tool.

**Free Resources:**
- [Databricks SQL Documentation](https://docs.databricks.com/en/sql/index.html) — editor, dashboards, alerts, query history, and BI tool connection guide
- [Databricks Academy](https://academy.databricks.com) — free SQL analytics courses covering dashboard creation, alerting, and analyst workflows

---

## Query Federation

**Status:** ⬜ Not Started

**Definition:** Query Federation in Databricks (via Lakehouse Federation) allows SQL queries to read from external data sources — Snowflake, BigQuery, MySQL, PostgreSQL, Redshift — without copying data into Databricks. Results can be joined with internal Delta Lake tables in a single SQL query through a unified Unity Catalog namespace.

**Key Mental Model:** Query Federation is a universal translator — you write one SQL query that reaches into multiple external systems simultaneously, as if they were all local tables, without a data migration project.

**How It Works:**
- Lakehouse Federation works by registering an external database system as a **foreign catalog** in Unity Catalog — the connection details (hostname, credentials, database name) are stored as a managed secret, and a catalog entry maps the external system's schemas and tables into the Unity Catalog namespace.
- When a query references a foreign table, the Databricks SQL Warehouse uses a **JDBC push-down** connector to translate the relevant parts of the SQL into the external system's dialect and execute them remotely — predicates, projections, and simple aggregations are pushed to the source to minimise data transfer.
- Data returned from the remote source is materialised in the Databricks worker memory as an in-memory partition and joined with local Delta table partitions using Spark's hash join or sort-merge join operators, depending on data size and statistics.
- Unity Catalog **lineage tracking** extends to foreign catalog queries — if a query reads from a PostgreSQL foreign table and writes to a Delta table, the lineage graph captures the cross-system data flow, giving governance visibility across system boundaries.
- Access to foreign catalog tables is governed by the same Unity Catalog RBAC as native tables — `GRANT SELECT ON TABLE foreign_catalog.schema.table TO user` controls who can query external data without granting access to the external system's credentials. See [[Cloud-Platforms/Databricks/01-Architecture#Unity Catalog]] for the privilege model.

**Common Misconceptions:**
- Lakehouse Federation does not replicate data — every query against a foreign table executes a live remote query at runtime; if the external system is slow or unavailable, the federated query fails or performs poorly, unlike materialized views that read from a local copy.
- Push-down optimisation has limits — complex window functions, lateral joins, or Databricks-specific SQL syntax may not translate to the external system's dialect, causing the entire remote table to be pulled into Databricks memory before filtering, which can be very expensive for large tables.

**Interview Answer Skeleton:**
- **What it is:** A live query federation layer that registers external databases (Snowflake, Postgres, BigQuery) as foreign catalogs in Unity Catalog, enabling JDBC push-down queries that join external data with local Delta tables without data movement.
- **Why it matters / trade-offs:** Eliminates copy-based ETL for exploratory cross-system queries and provides governed access to external data through the Unity Catalog namespace; the trade-off is that performance depends entirely on the external system's query speed and available predicates for push-down.
- **Example or context:** An analyst needs to join Salesforce opportunity data (in a PostgreSQL replica) with internal revenue data in a Delta table — Lakehouse Federation lets them write a single SQL query in Databricks SQL rather than waiting for a nightly ETL to copy the Salesforce data.

**Free Resources:**
- [Lakehouse Federation Documentation](https://docs.databricks.com/en/query-federation/index.html) — supported sources, foreign catalog setup, and push-down capabilities
- [Databricks Academy](https://academy.databricks.com) — free courses covering data federation patterns and Unity Catalog governance

---

## Materialized Views

**Status:** ⬜ Not Started

**Definition:** Materialized views in Databricks pre-compute and store the results of a query as a Delta table, automatically refreshing when underlying data changes. This trades storage for query performance — queries against the materialized view are fast because they read from a pre-computed snapshot. DLT-backed materialized views support incremental refresh for supported query patterns.

**Key Mental Model:** A materialized view is a pre-cooked meal in the fridge — making it fresh every time is slower, but pulling out the pre-cooked version is instant. It's stale only if you haven't updated it since the ingredients changed.

**How It Works:**
- A Databricks materialized view is implemented as a Delta Lake table managed by the DLT engine — the view definition is stored as a DLT pipeline, and the underlying Delta table is refreshed either on a schedule or when triggered manually via the Workflow or REST API.
- **Incremental refresh** is available for materialized views over append-only source tables with supported query patterns (aggregations without ordering, simple joins) — the DLT engine reads only the new Delta table rows since the last refresh using CDF offsets, computes the incremental aggregate delta, and merges it into the materialized view table rather than recomputing from scratch.
- **Full refresh** recomputes the entire result set from the source tables; this is required for queries with ordering, complex subqueries, or views over mutable sources where incremental logic cannot be safely inferred.
- The materialized view is registered in Unity Catalog and accessed like any Delta table — queries against it read from the pre-computed Delta files using Photon, with no awareness that the underlying data came from a complex multi-table aggregation.
- Materialized views can be chained — a view can be defined on top of another materialized view, forming a DLT pipeline DAG where each layer refreshes incrementally. See [[Cloud-Platforms/Databricks/02-Data-Engineering#Delta Live Tables (DLT)]] for the DLT pipeline mechanics that power materialized view refresh.

**Common Misconceptions:**
- Materialized views are not automatically real-time — they refresh on a trigger (manual, scheduled, or pipeline-driven); between refreshes, queries read stale data from the last refresh snapshot, not live source data.
- "Incremental refresh is always faster" is only true when the DLT engine can safely compute an incremental delta — complex aggregations with window functions, distinct counts, or median calculations cannot be incrementally maintained and always fall back to full refresh.

**Interview Answer Skeleton:**
- **What it is:** A pre-computed DLT-managed Delta table that stores the result of a defined query, supports incremental refresh for append-only aggregation patterns via CDF offsets, and is accessed via Unity Catalog like any standard Delta table.
- **Why it matters / trade-offs:** Dramatically accelerates queries over expensive aggregations for BI dashboards and reporting; the trade-off is refresh latency (data is only as fresh as the last refresh cycle) and storage cost for maintaining the pre-computed result alongside source tables.
- **Example or context:** A data team creates a materialized view that pre-aggregates daily sales by product and region from a 500M-row transactions Delta table — the BI dashboard queries the materialized view (returning in milliseconds) rather than re-scanning the full table on every page load.

**Free Resources:**
- [Databricks Materialized Views Documentation](https://docs.databricks.com/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view.html) — syntax, incremental refresh conditions, and refresh scheduling
- [Delta Live Tables Documentation](https://docs.databricks.com/en/delta-live-tables/index.html) — DLT pipeline mechanics underlying materialized view refresh

---

## Cost Controls

**Status:** ⬜ Not Started

**Definition:** Databricks cost controls include cluster auto-termination (shut down idle compute), cluster policies (restrict instance types and sizes via admin-defined templates), budget policies (DBU spending limits per workspace, tag, or principal), and cost monitoring through the Account Console, Cost Management dashboard, and Unity Catalog system tables.

**Key Mental Model:** Cost controls are the spending limits and automatic shutoffs on your Databricks environment — idle clusters don't run up bills, rogue large clusters can't be created without approval, and every DBU spent is visible.

**How It Works:**
- **Cluster policies** are JSON-defined templates that constrain the cluster configuration options available to users at cluster creation time — a policy can fix the instance type, set maximum autoscale workers, require auto-termination within a specific window, and tag all clusters for cost allocation; non-admin users can only create clusters that conform to an assigned policy.
- **Auto-termination** is a cluster-level setting (available in minutes of idle time) enforced by the Databricks control plane — the control plane monitors cluster activity and terminates idle clusters by instructing the cloud provider to release the VMs, stopping compute billing.
- **Budget policies** in the Account Console let admins set monthly DBU or dollar spend limits per workspace or SKU; when a limit is approached or exceeded, alert emails are sent and optionally policies can be configured to block new cluster creation.
- **Unity Catalog system tables** expose billing and usage data as queryable Delta tables (`system.billing.usage`) with columns for workspace, cluster, DBU type, and timestamp — enabling custom cost attribution dashboards and charge-back reports without exporting data to external tools.
- **Instance pools** reduce the time-to-interactive for clusters by pre-warming VMs; they also reduce cost indirectly by allowing clusters to start faster (shortening the interactive session) and by enabling spot/preemptible instance types with fallback logic. See [[Cloud-Platforms/Databricks/05-Administration]] for detailed policy administration.

**Common Misconceptions:**
- Auto-termination does not protect against long-running jobs — a cluster processing a 6-hour Spark job is not idle and will not auto-terminate during the job; cost overruns from long jobs require job-level timeout settings and budget alerts, not just auto-termination.
- Cluster policies are not retroactive — they constrain cluster creation; a cluster that was created before a policy was applied or that was created by an admin without the policy is not automatically brought into compliance; policies must be applied before cluster creation to be effective.

**Interview Answer Skeleton:**
- **What it is:** A multi-layer cost governance framework combining cluster policies (creation-time constraints), auto-termination (idle shutdown), budget policies (DBU spend limits with alerts), and system table billing data for chargeback and monitoring.
- **Why it matters / trade-offs:** Prevents unbounded compute spend in shared Databricks environments and enables cost attribution; the trade-off is that overly restrictive policies can create friction for data scientists who need flexibility to explore, requiring a balance between governance and productivity.
- **Example or context:** A data platform team applies a cluster policy to all non-admin users that caps All-Purpose clusters at 8 workers, requires auto-termination after 30 minutes, and tags every cluster with a cost centre — monthly chargeback reports are generated from `system.billing.usage` without any manual tracking.

**Free Resources:**
- [Databricks Cluster Policies Documentation](https://docs.databricks.com/en/admin/clusters/cluster-policies.html) — policy definition, assignment, and enforcement reference
- [Databricks Academy](https://academy.databricks.com) — free administration courses covering cost controls, system tables, and governance patterns

---
