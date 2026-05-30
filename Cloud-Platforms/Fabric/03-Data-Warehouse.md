# Microsoft Fabric — Data Warehouse

---

## Synapse Data Warehouse

**Status:** ⬜ Not Started

**Definition:** The Fabric Data Warehouse is a fully managed T-SQL data warehouse built on top of OneLake, offering distributed query processing for large-scale analytics. Unlike traditional warehouses, it stores data as Delta Parquet on OneLake, enabling cross-experience access without data copies.

**Mental Model:** The Fabric Warehouse is a SQL-native window onto OneLake — it speaks T-SQL to analysts and BI tools, but underneath it's Delta files that other Fabric experiences can also read directly.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-warehouse/data-warehousing — Microsoft Fabric Data Warehouse overview documentation

---

## T-SQL in Fabric

**Status:** ⬜ Not Started

**Definition:** Fabric's Data Warehouse supports a broad subset of T-SQL (Transact-SQL) including DML (SELECT, INSERT, UPDATE, DELETE), DDL (CREATE TABLE, ALTER, DROP), stored procedures, views, and common T-SQL functions. This enables SQL Server-experienced engineers and BI developers to work productively without learning new syntax.

**Mental Model:** T-SQL in Fabric is a familiar dialect — if you know SQL Server or Azure Synapse, the core SQL works as expected, with modern distributed warehouse semantics underneath.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-warehouse/tsql-surface-area — Microsoft Fabric T-SQL surface area documentation covering supported statements and functions

---

## Cross-Database Queries

**Status:** ⬜ Not Started

**Definition:** Fabric's zero-copy cross-database queries allow a Warehouse to query tables from other Warehouses and Lakehouses in the same workspace using three-part name syntax (`[workspace].[lakehouse].[table]`), all reading from shared OneLake storage without data movement.

**Mental Model:** Cross-database queries are reading from any shelf in the shared library — you can query the sales Lakehouse from within the finance Warehouse in one SQL statement, without copying data between them first.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-warehouse/query-warehouse — Microsoft Fabric cross-database query documentation

---

## Warehouse vs Lakehouse

**Status:** ⬜ Not Started

**Definition:** In Fabric, choose a Warehouse when you need full T-SQL DML support (UPDATE, DELETE, MERGE), traditional warehouse semantics, stored procedures, and governed schema management. Choose a Lakehouse when you need Spark-based processing, file operations, schema-on-read flexibility, or mixed Python and SQL workflows.

**Mental Model:** Warehouse is for the SQL-first analyst or BI engineer — familiar T-SQL, full DML, governed. Lakehouse is for the data engineer or data scientist — code-first, Spark-native, more flexible with raw and semi-processed data.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-vs-data-warehouse — Microsoft comparison guide for Warehouse vs Lakehouse in Fabric

---

## COPY INTO

**Status:** ⬜ Not Started

**Definition:** COPY INTO is a T-SQL command in Fabric Warehouse for efficiently loading data from OneLake files (CSV, Parquet, JSON) directly into warehouse tables. It handles schema mapping, error tolerance, and incremental loading, and is the primary pattern for bulk data ingestion into the Fabric Warehouse.

**Mental Model:** COPY INTO is the loading dock for the Warehouse — it moves files sitting in OneLake into organised warehouse tables efficiently, handling format conversion and error tolerance automatically.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-warehouse/ingest-data-copy — Microsoft Fabric COPY INTO documentation covering syntax and options

---

## Auto-Scale

**Status:** ⬜ Not Started

**Definition:** Fabric Warehouse compute automatically scales up to handle concurrent query bursts using the shared capacity pool, and scales down when demand drops. Unlike virtual warehouse models (Snowflake) where you manually choose warehouse size, Fabric scales dynamically within your purchased capacity.

**Mental Model:** Fabric auto-scale is an elastic staffing model — when queries pile up, more resources are pulled from the shared capacity pool automatically, and released when the rush is over.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/data-warehouse/burstable-capacity — Microsoft Fabric burstable capacity and auto-scale documentation
