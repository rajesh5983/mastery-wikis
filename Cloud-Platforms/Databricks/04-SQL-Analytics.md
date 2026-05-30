# Databricks — SQL Analytics

---

## SQL Warehouses (Serverless vs Classic)

**Status:** ⬜ Not Started

**Definition:** A Databricks SQL Warehouse is a compute cluster optimised for SQL queries, BI tool connections (Power BI, Tableau), and dbt. Classic warehouses run on your cloud account's VMs. Serverless warehouses are managed by Databricks on Databricks-owned infrastructure, with faster startup times and per-query billing.

**Mental Model:** Classic SQL Warehouses are like renting a car — you choose the model, you manage it. Serverless is like calling a taxi — it appears when you need it, disappears when you're done, and you pay only for the ride.

**Free Resources:** https://docs.databricks.com/en/compute/sql-warehouse/index.html — Databricks SQL Warehouse documentation covering types, sizing, and serverless configuration

---

## Databricks SQL

**Status:** ⬜ Not Started

**Definition:** Databricks SQL is the SQL analytics product within the Databricks platform — a web-based SQL editor, dashboard builder, and alert system for running queries against Delta Lake tables. It connects to SQL Warehouses and integrates with Unity Catalog for governance.

**Mental Model:** Databricks SQL is the data analyst's workbench within Databricks — write SQL, build charts, share dashboards, and set alerts, all without leaving the platform.

**Free Resources:** https://docs.databricks.com/en/sql/index.html — Databricks SQL documentation covering the editor, dashboards, alerts, and query history

---

## Query Federation

**Status:** ⬜ Not Started

**Definition:** Query Federation in Databricks (via Lakehouse Federation) allows SQL queries to read from external data sources — Snowflake, BigQuery, MySQL, PostgreSQL, Redshift — without copying data. Results are joined with internal Delta Lake tables in a single query.

**Mental Model:** Query Federation is a universal translator — you write one SQL query that reaches into multiple external systems simultaneously, as if they were all local tables, without a data migration project.

**Free Resources:** https://docs.databricks.com/en/query-federation/index.html — Databricks Lakehouse Federation documentation covering supported sources and connection setup

---

## Materialized Views

**Status:** ⬜ Not Started

**Definition:** Materialized views in Databricks pre-compute and store the results of a query as a Delta table, automatically refreshing when underlying data changes. This trades storage for query performance — queries against the materialized view are fast because they read from a pre-computed snapshot rather than recalculating.

**Mental Model:** A materialized view is a pre-cooked meal in the fridge — making it fresh every time is slower, but pulling out the pre-cooked version is instant. It's stale only if you haven't updated it since the ingredients changed.

**Free Resources:** https://docs.databricks.com/en/sql/language-manual/sql-ref-syntax-ddl-create-materialized-view.html — Databricks materialized view documentation

---

## Cost Controls

**Status:** ⬜ Not Started

**Definition:** Databricks cost controls include cluster auto-termination (shut down idle compute), cluster policies (restrict instance types and sizes), budget policies (DBU spending limits per workspace or principal), and monitoring through the Cost Management dashboard and system tables.

**Mental Model:** Cost controls are the spending limits and automatic shutoffs on your Databricks environment — idle clusters don't run up bills, rogue large clusters can't be created without approval, and every DBU spent is visible.

**Free Resources:** https://docs.databricks.com/en/admin/clusters/cluster-policies.html — Databricks cluster policies documentation covering cost control and governance for compute
