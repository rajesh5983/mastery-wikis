# Layer 3 — Data Modeling

> **Framework:** Designing data structures for analytical workloads and business intelligence.

---

## OLTP vs OLAP

**Status:** ⬜ Not Started

**Definition:** OLTP (Online Transaction Processing) systems are optimised for fast single-row reads and writes with full ACID guarantees — powering real-time application operations like order placement or account updates. OLAP (Online Analytical Processing) systems are optimised for scanning millions of rows, aggregating across multiple dimensions, and serving analytical queries — powering dashboards, cohort analyses, and business intelligence reports.

**Key Mental Model:** OLTP is a cash register — it must ring up one item in under 100ms. OLAP is a ledger auditor — it must scan every transaction across all registers for the whole year to find patterns, and a few seconds of latency is acceptable.

**How It Works:**
- OLTP databases use row-oriented storage: all columns for a single row are stored together on disk, making single-row reads and writes fast. Fetching one customer record reads one contiguous block. This layout is inefficient for analytical queries that scan one column across millions of rows, requiring reading all blocks.
- OLAP databases use columnar storage (Parquet-backed in BigQuery, Snowflake, Redshift Spectrum): all values for a single column are stored together, enabling column pruning (only read the columns you SELECT) and SIMD-accelerated vectorised processing across millions of values in a single CPU cycle burst.
- OLTP systems enforce ACID transactionality with row-level locking. Every UPDATE acquires locks, maintains undo logs, and participates in a transaction log. This overhead makes OLTP unsuitable for bulk analytical loads or queries that touch millions of rows, which would hold locks for seconds and block concurrent application writes.
- OLAP systems favour append-only bulk loads over row-level updates. Columnar formats like Parquet are immutable — updates require rewriting entire files. Open table formats (Delta Lake, Iceberg) add a transaction log on top to provide limited row-level update semantics. See [[DE-Engineer/06-Platform]] for details.
- Separating OLAP from OLTP (the Lambda/Kappa architecture pattern, or simply warehouse + replication) protects the production OLTP system from analytical query load. A long-running analytical query on a Postgres production database can cause table bloat, vacuum lag, and connection exhaustion that degrades application response times. See [[DE-Engineer/04-Pipeline]] for ingestion patterns.

**Common Misconceptions:**
- A fast enough OLTP database can serve analytical queries — at scale, row-oriented storage and ACID overhead make OLTP 10–100x slower than columnar OLAP systems for multi-million-row analytical scans. This is a fundamental storage architecture difference, not a tuning problem.
- OLAP systems can replace OLTP for transactional workloads — OLAP systems lack the row-level locking, foreign key enforcement, and sub-millisecond single-row write latency that application backends require. They are complementary, not interchangeable.

**Interview Answer Skeleton:**
- **What it is:** Two fundamentally different database storage architectures — row-oriented OLTP optimised for transactional single-row operations, and columnar OLAP optimised for aggregate analytical scans — each making trade-offs that make them unsuitable for the other's workload.
- **Why it matters / trade-offs:** Querying a production OLTP system for analytics causes performance degradation on both sides: the analytical query is slow (row-oriented storage), and the running query contends with application writes. The trade-off of maintaining a separate OLAP system is cost and data latency.
- **Example or context:** A Postgres e-commerce database handles thousands of order inserts per minute. Running a dashboard query that joins orders, customers, and products across 50M rows would take minutes, block vacuuming, and slow order inserts during the query. The solution is replicating to BigQuery or Snowflake and running analytics there.

**Free Resources:**
- [Kimball Group Dimensional Modelling Resources](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources) — foundational dimensional modelling techniques covering OLAP design, star schemas, and warehouse architecture
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference with detailed OLTP vs OLAP comparison and storage architecture explanations

---

## Star Schema, Snowflake Schema, Fact and Dimension Tables

**Status:** ⬜ Not Started

**Definition:** A star schema organises a data warehouse into a central fact table (events or transactions with numeric measures) surrounded by denormalised dimension tables (customer, product, date). A snowflake schema further normalises dimensions into sub-tables, reducing redundancy at the cost of additional joins. The choice between them is a trade-off between storage efficiency and query performance.

**Key Mental Model:** A fact table is the receipt — it records that a transaction happened with its numeric values (quantity, revenue, duration). Dimension tables are the reference cards attached to that receipt — they describe the who (customer), what (product), where (store), and when (date) of the event in rich descriptive detail.

**How It Works:**
- The grain of the fact table defines what one row represents — one order line item, one click event, one daily snapshot — and every design decision flows from this definition. A fact table at order-line grain means one row per product per order, allowing product-level aggregations without double-counting order-level metrics.
- Foreign keys in the fact table point to surrogate keys in dimension tables. The join between fact and dimension tables is how descriptive attributes (customer name, product category) are associated with numeric measures at query time. In a star schema these are single-hop joins; in a snowflake schema, dimension-to-sub-dimension joins add additional hops.
- Columnar OLAP systems like Snowflake and BigQuery distribute tables across compute nodes. When a fact table is joined to a large dimension table, the database may shuffle (redistribute) one or both tables so that matching keys land on the same node — this is a broadcast join (replicate small table to all nodes) or hash join (redistribute both by key). Star schemas with small dimensions favour broadcast joins and avoid expensive shuffles.
- Degenerate dimensions are dimension attributes stored directly in the fact table because they have no other attributes (e.g., order_number, invoice_id). They are not separate dimension tables because there is nothing else to say about them — they are identifiers, not descriptive categories.
- Conformed dimensions are dimension tables shared across multiple fact tables — a single date dimension used by both a sales fact and a returns fact table. This ensures consistent attribute definitions (fiscal_week, is_holiday) across all fact models, preventing metric disagreements caused by different teams computing date attributes independently. See [[DE-Engineer/03-Data-Modeling]] and [[DE-Engineer/06-Platform]].

**Common Misconceptions:**
- Snowflake schema is superior because it is more normalised — normalisation reduces redundancy but increases join depth. In columnar OLAP systems with modern query planners, the extra joins in a snowflake schema often produce slower queries than the equivalent denormalised star schema with slightly redundant dimension data.
- Fact tables should include all descriptive fields for convenience — adding descriptive columns to the fact table creates a wide, duplicated, hard-to-maintain structure. Descriptive attributes belong in dimension tables; measures belong in the fact table. Violating this creates what Kimball called a "fact table that acts as a dimension."

**Interview Answer Skeleton:**
- **What it is:** A dimensional modelling pattern that organises analytical data into a central numeric event table (fact) connected to descriptive reference tables (dimensions), enabling intuitive business querying by joining measures to attributes.
- **Why it matters / trade-offs:** The schema structure determines join patterns, aggregation correctness, and how easily new business questions can be answered without model changes. A star schema with well-defined grain enables self-service BI. The trade-off vs snowflake is that star schemas store some redundant data in dimensions for simpler, faster queries.
- **Example or context:** An e-commerce order fact table at order-line grain stores: order_line_id (surrogate PK), order_id (degenerate dimension), customer_key, product_key, date_key, quantity, unit_price, discount_amount. Customer name, product category, and fiscal week live in their respective dimension tables — fetched only when needed via join, not duplicated in every fact row.

**Free Resources:**
- [Kimball Group Dimensional Modelling Resources](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources) — the canonical reference for star schema design, fact/dimension patterns, and conformed dimensions
- [dbt Documentation](https://docs.getdbt.com) — covers dimensional modelling implementation in dbt, including staging, dimension, and fact layer conventions

---

## Primary Keys, Surrogate Keys, and SCDs

**Status:** ⬜ Not Started

**Definition:** A primary key uniquely identifies each row in a table. A surrogate key is a system-generated integer or UUID used as the primary key in place of the natural business key, providing stability when business keys change. Slowly Changing Dimensions (SCDs) are strategies for tracking how dimension attributes change over time — SCD Type 1 overwrites old values, SCD Type 2 adds new rows with validity date ranges to preserve history.

**Key Mental Model:** A surrogate key is like a library barcode — not the book's ISBN (the natural key, which can change with new editions), but a system-assigned label that is permanently stable. SCD Type 2 is like keeping all previous versions of a contact card in a Rolodex with "active from" and "active to" dates on each version.

**How It Works:**
- Surrogate keys are generated at load time (auto-increment integers or UUIDs) and never updated. When a customer's email address changes, only the dimension record changes — the surrogate key stays the same, and all historical fact rows continue to correctly reference the original dimension record without any update to the fact table.
- SCD Type 1 is a simple overwrite: when an attribute changes, the dimension row is updated in place. This loses history — all historical fact rows joined to this dimension will now show the current attribute value, not the value at the time of the transaction. Appropriate when historical accuracy of the attribute is not required (e.g., correcting a typo in a customer name).
- SCD Type 2 adds a new row for each change with an effective_start_date, an effective_end_date (NULL or a far-future sentinel value for the current record), and an is_current flag. Fact rows carry the surrogate key of the dimension version that was current at transaction time — enabling "what was the customer's segment when they made this purchase?" queries even if the segment has since changed.
- The merge (UPSERT) logic for SCD Type 2: compare incoming dimension records to current active records; if an attribute has changed, expire the current row (set effective_end_date = today - 1, is_current = false) and insert a new current row with the updated attributes. Delta Live Tables and dbt snapshots automate this pattern. See [[DE-Engineer/06-Platform]] for Delta Lake CDC support.
- SCD Type 3 adds new columns for the previous value (e.g., previous_segment alongside current_segment) — this tracks exactly one historical value without the row explosion of Type 2, but cannot track more than one prior state. Rarely used in modern warehouses.

**Common Misconceptions:**
- Natural keys are safer and more transparent than surrogate keys — natural keys like customer email or product SKU appear stable but change in practice (company mergers, re-platforming, email changes). Using natural keys as foreign keys in fact tables requires cascading updates across billions of fact rows when the key changes. Surrogate keys decouple fact table stability from source system key changes.
- SCD Type 2 should be the default for all dimension attributes — SCD Type 2 multiplies dimension rows over time and makes queries more complex (you must always filter on is_current = true or use a validity date range join). It should only be applied to attributes where historical accuracy genuinely drives business decisions, such as customer tier or pricing segment.

**Interview Answer Skeleton:**
- **What it is:** Key design patterns for uniquely identifying dimension records (surrogate keys) and tracking how slowly changing attributes evolve over time (SCD types), enabling accurate historical reporting in dimensional models.
- **Why it matters / trade-offs:** Incorrect key strategy causes join fanout (duplicate fact rows), lost change history, or expensive cascading updates. The trade-off of SCD Type 2 is query complexity and row count growth; the trade-off of SCD Type 1 is loss of historical accuracy. The business requirement determines which is appropriate.
- **Example or context:** A customer dimension tracks loyalty_tier (Bronze/Silver/Gold). With SCD Type 2, a query for "revenue attributed to Gold customers at the time of purchase" correctly uses the tier valid on the transaction date. With SCD Type 1, the same query would incorrectly attribute all historical purchases to the customer's current tier, overstating Gold revenue.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — covers dbt snapshots, which implement SCD Type 2 with automated merge logic and configurable unique key and change detection strategies
- [Kimball Group Dimensional Modelling Resources](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources) — definitive reference for SCD types, surrogate key patterns, and dimension design trade-offs

---

## Define Grain, Partitions, and Ownership

**Status:** ⬜ Not Started

**Definition:** Grain is the precise, unambiguous definition of what one row in a fact table represents — the finest atomic unit of the business event being modelled. Partitioning physically divides a large table into independent segments by a column value (typically date), allowing the query engine to skip irrelevant partitions entirely. Ownership defines which team is accountable for a table's data quality, schema contracts, and SLA compliance.

**Key Mental Model:** Grain is the atomic unit — get it wrong and every aggregation is wrong, irreversibly. Partitioning is organising a warehouse by year — you only open the 2024 aisle instead of walking through every aisle. Ownership is the on-call roster — when data breaks, someone specific is paged.

**How It Works:**
- Grain must be stated as a concrete sentence: "one row represents one order line item on a customer order." Ambiguity — "one row represents an order" — causes fan-out bugs when an order has multiple line items, double-counting revenue in joins. Grain defines both what metrics can be measured at this table and what dimension keys must be present.
- Partition pruning works by the query planner inspecting partition metadata before reading any data. A query with `WHERE order_date >= '2024-01-01'` on a date-partitioned table reads only the matching partitions; without the filter predicate on the partition column, all partitions are scanned. Queries must reference the partition column in WHERE or JOIN conditions for pruning to activate.
- In BigQuery, partitions are defined on a DATE/TIMESTAMP column or an ingestion time pseudocolumn. Partition pruning is automatic when the filter references the partition column directly — applying a function like DATE_TRUNC on it may defeat pruning. Snowflake uses micro-partitions and clustering keys instead of explicit partitions, achieving similar range skipping based on min/max metadata per micro-partition.
- Partition key selection has a correctness dimension beyond performance: if a pipeline inserts data late (an event from yesterday arrives today), inserting into today's partition causes the yesterday partition to be incomplete. Using the event timestamp as the partition key with a configurable lookback window (reading N days of partitions per run) is more correct than using insert timestamp for event-based models.
- Data ownership encoded in a data catalogue (dbt docs, Atlan, DataHub) with explicit owner fields creates a contractual accountability boundary. Without explicit ownership, schema changes, late data, and quality failures fall into organisational ambiguity — the table is "everyone's and no one's." See [[DE-Engineer/06-Platform]] for data catalogue integration.

**Common Misconceptions:**
- Grain can be adjusted later with a backfill — changing grain is a breaking change to the data model. All downstream models, dashboards, and metrics built on the existing grain must be rebuilt from scratch with historical data reprocessed. Grain decisions are among the most expensive to change; define them correctly at design time.
- Partitioning always speeds up queries — partition pruning only activates when the filter predicate references the partition column directly. A query without a date filter on a date-partitioned table still performs a full table scan across all partitions. Over-partitioning (partitioning by hour on low-volume tables) also creates small-file performance problems.

**Interview Answer Skeleton:**
- **What it is:** Three foundational design decisions for production fact tables — grain (what one row means, determining correctness), partitioning (physical layout for query performance), and ownership (accountability for quality and SLA).
- **Why it matters / trade-offs:** Wrong grain causes silent double-counting in every downstream metric. Wrong partitioning causes full table scans that defeat the purpose of partitioning. Unclear ownership means data quality failures go undetected for days. These decisions are cheap to get right upfront and expensive to fix retroactively.
- **Example or context:** A subscription events table: grain = "one row represents one state transition for one subscription" (not one subscription, not one customer). Partition key = event_date (not load_date) to enable correct late-arriving event handling. Owner = the data engineering team for the subscriptions domain, with a freshness SLA of 6 hours from source event time.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — covers grain definition, model design, incremental models with partition logic, and data ownership through dbt docs and meta fields
- [Kimball Group Dimensional Modelling Resources](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources) — includes detailed guidance on grain declaration, fact table design, and dimensional ownership conventions

---

## Normalisation vs Denormalisation

**Status:** ⬜ Not Started

**Definition:** Normalisation organises data into multiple related tables by decomposing redundant data into separate entities, enforcing referential integrity and reducing update anomalies — the standard pattern for OLTP transactional databases. Denormalisation intentionally reintroduces redundancy by flattening related tables into wider single tables, eliminating join cost at query time — the standard pattern for OLAP analytical systems.

**Key Mental Model:** Normalisation is a well-organised apartment — every item has exactly one proper place, nothing is duplicated, changing an item's properties requires updating it in one location. Denormalisation is packing a suitcase for a trip — you duplicate what you need for convenience, accepting the redundancy because read access speed matters more than storage tidiness.

**How It Works:**
- Third Normal Form (3NF) requires that every non-key column depends only on the primary key, not on other non-key columns. A customer table with both city and city_zip_code violates 3NF because zip_code depends on city, not customer_id — the solution is a separate cities table. This eliminates update anomalies: changing a city's zip code requires updating one row in cities, not thousands in customers.
- Denormalisation for OLAP pre-computes joins at write time. A wide order fact table that includes product_category, customer_country, and fiscal_quarter directly (copied from dimension tables) allows aggregation queries to avoid dimension joins entirely. The trade-off is that updating product_category now requires rewriting all historical fact rows — acceptable in append-heavy OLAP workloads, catastrophic in OLTP.
- In columnar OLAP systems, the physical cost of joining large tables is primarily the shuffle (redistributing data across compute nodes). Denormalising frequently joined small dimensions into the fact table eliminates these shuffles entirely. For example, embedding a low-cardinality date dimension's attributes (day_of_week, is_holiday, fiscal_quarter) directly in the fact table removes a mandatory join from every analytical query.
- dbt materialisation choices implement this trade-off: staging models are normalised (one source per model, minimal transformation), intermediate models build grain-correct base tables, mart models are intentionally denormalised wide tables for BI consumption — pre-joining dimensions for read performance. See [[DE-Engineer/04-Pipeline]] for dbt model layer design.
- Over-denormalisation creates maintenance problems: wide fact tables with dozens of embedded dimension attributes are difficult to extend (adding a new attribute requires a schema migration), and if business definitions change, all copies of that attribute must be updated consistently. The practical balance is to denormalise stable, frequently queried attributes and preserve joins for volatile or rarely queried ones.

**Common Misconceptions:**
- Denormalisation is always bad practice signalling poor design — in analytical systems, denormalised wide tables are often the architecturally correct design choice. Data warehouse design is intentionally different from application database design, and applying OLTP normalisation rules to OLAP models causes unnecessarily complex queries and poor performance.
- Full normalisation (3NF) is the goal in a data warehouse — data warehouses are not designed for 3NF; dimensional models intentionally denormalise for read performance. Applying 3NF to a warehouse creates a snowflake schema that performs worse and is harder to query than a denormalised star schema.

**Interview Answer Skeleton:**
- **What it is:** A design spectrum from fully normalised (minimal redundancy, maximum update consistency, complex read joins) to fully denormalised (maximum redundancy, simplest reads, expensive updates) — the right position depends on whether the workload is OLTP or OLAP.
- **Why it matters / trade-offs:** Normalised schemas are the correct default for OLTP to prevent update anomalies. Denormalised schemas are correct for OLAP to avoid join overhead at analytical query time. Applying the wrong approach to the wrong workload is a common architectural mistake that is expensive to reverse.
- **Example or context:** A product hierarchy (product → subcategory → category → department) in a normalised schema requires a 3-hop join from the fact table to get the department for an order. Flattening subcategory, category, and department into the product dimension table (denormalisation) reduces this to a single join, making department-level revenue queries faster and simpler.

**Free Resources:**
- [Kimball Group Dimensional Modelling Resources](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources) — comprehensive guidance on when to normalise vs denormalise in dimensional models and the trade-offs of each choice
- [dbt Documentation](https://docs.getdbt.com) — covers dbt model layer design, staging vs mart denormalisation patterns, and materialisation strategies

---

## Data Quality Checks and Business Logic

**Status:** ⬜ Not Started

**Definition:** Data quality checks are automated tests that validate expected properties of pipeline outputs — row counts, null rates, uniqueness constraints, referential integrity, value range bounds, and freshness windows. Business logic encodes the organisation's agreed metric definitions directly into the data model — for example, "an active customer is one with a completed order in the last 90 days" — ensuring consistent interpretation across all downstream consumers.

**Key Mental Model:** Data quality checks are smoke alarms — you install them once at the right places and they alert you before a small data problem becomes a wrong board-level decision. Business logic in the model is a shared dictionary — every team reads the same definition instead of each team computing their own version.

**How It Works:**
- dbt tests are defined in YAML alongside model definitions and run after each model build. Generic tests (unique, not_null, accepted_values, relationships) cover most quality needs without custom SQL. Singular tests are custom SQL assertions that return rows when the condition is violated — zero rows means the test passes.
- Freshness checks validate data timeliness by querying source tables' maximum timestamp and comparing it to an expected freshness threshold. A source freshness failure means the upstream pipeline has stalled before dbt's models even run — catching this early prevents dbt from building downstream models on stale data and surfacing incorrect results.
- Great Expectations and dbt-expectations extend dbt's native test framework with statistical checks — checking that a column's mean falls within expected bounds, that row counts are within 20% of yesterday's count, or that a ratio metric stays within historical norms. These catch data drift that boolean tests miss.
- Business logic centralisation in dbt models (or equivalent transformation layers) prevents "metric disagreement" — the organisational dysfunction where different teams compute the same metric differently and get different numbers. The rule: if two teams disagree on the revenue number, it means the definition of "revenue" lives in two places. Centralise it in one model.
- Row count anomaly detection (comparing today's fact table rows to a rolling average of prior days) catches silent pipeline failures — a load job that succeeded structurally but loaded no data, or 10x the expected rows due to a deduplication bug. These don't trigger schema errors but cause wrong downstream metrics.

**Common Misconceptions:**
- Data quality is the responsibility of the data source team, not the pipeline team — data engineers own the quality of what they produce. Even if upstream data is imperfect, the pipeline team is responsible for handling, documenting, and alerting on quality issues rather than silently propagating bad data downstream.
- Business logic should live in the BI tool (Tableau calculated fields, Looker LookML measures) rather than the data model — when logic lives only in the visualisation layer, the same metric is computed differently in different dashboards, creating disagreements that require weeks of forensic investigation. The single source of truth for a metric definition belongs in the transformation layer.

**Interview Answer Skeleton:**
- **What it is:** Automated validation tests on pipeline outputs to catch quality regressions early, combined with centralised encoding of business metric definitions in the transformation layer so all consumers interpret data consistently.
- **Why it matters / trade-offs:** Undetected data quality failures erode stakeholder trust in a way that is very slow to rebuild. Decentralised business logic causes endless metric disagreements between teams. The trade-off of comprehensive testing is maintenance overhead — tests must be updated when business definitions change or models are refactored.
- **Example or context:** A dbt fact table model gets: not_null and unique tests on the primary key, a relationships test checking every customer_key exists in the customer dimension, an accepted_values test on order_status, and a custom singular test asserting no order has revenue < 0. The "active customer" definition (ordered in last 90 days) is encoded once in a `dim_customers` model field, referenced by every downstream mart rather than recomputed in each one.

**Free Resources:**
- [dbt Documentation](https://docs.getdbt.com) — comprehensive coverage of dbt tests, source freshness checks, singular tests, and data quality patterns in transformation pipelines
- [Data Engineering Wiki](https://dataengineering.wiki) — community reference covering data quality frameworks, business logic centralisation, and testing strategies for production pipelines

---
