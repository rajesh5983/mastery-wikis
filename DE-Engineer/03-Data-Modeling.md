# Layer 3 — Data Modeling

> **Framework:** Designing data structures for analytical workloads and business intelligence.

---

## OLTP vs OLAP

**Status:** ⬜ Not Started

**Definition:** OLTP (Online Transaction Processing) systems are optimised for fast single-row reads and writes — like a bank account update or order placement. OLAP (Online Analytical Processing) systems are optimised for scanning millions of rows and aggregating across dimensions — like a quarterly revenue report or cohort analysis.

**Mental Model:** OLTP is a cash register — it needs to ring up one item fast. OLAP is a ledger auditor — it needs to scan every transaction in the year to find patterns.

**Common Misconceptions:**
- You can use an OLTP database for analytics if it's fast enough — at scale, row-oriented storage and ACID overhead make OLTP 100x slower than columnar OLAP systems for analytical queries.
- OLAP systems can replace OLTP — OLAP systems are not designed for high-frequency transactional writes; they are separate tools serving separate purposes.

**Interview Skeleton:**
- What it is: two fundamentally different database designs optimised for different access and write patterns
- Why it matters: choosing the wrong system type causes performance and architectural problems that are expensive to fix later
- Example: explain why a Postgres production database should not be queried directly for dashboards, and what you'd build instead

**Free Resources:** https://www.guru99.com/oltp-vs-olap.html — Side-by-side comparison of OLTP and OLAP characteristics and use cases

---

## Star Schema, Snowflake Schema, Fact and Dimension Tables

**Status:** ⬜ Not Started

**Definition:** A star schema organises a data warehouse into a central fact table (events or transactions) surrounded by dimension tables (customer, product, date). A snowflake schema normalises the dimensions into sub-tables. Fact tables store measurable events with numeric values; dimension tables store descriptive attributes about those events.

**Mental Model:** A fact table is the receipt — it records that something happened with numeric values. Dimension tables are the reference cards — they describe the who, what, where, and when of that event.

**Common Misconceptions:**
- Snowflake schema is always better because it's more normalised — normalisation reduces redundancy but increases join complexity; extra joins hurt analytical query performance.
- Fact tables should include all descriptive details — descriptive columns belong in dimensions; keeping them in the fact table creates a wide, hard-to-maintain table.

**Interview Skeleton:**
- What it is: dimensional modelling patterns that organise analytics data for fast querying and intuitive business reporting
- Why it matters: the schema determines join patterns, aggregation speed, and how easily new business questions can be answered
- Example: design a schema for an e-commerce order fact table with customer, product, and date dimensions — justify grain and key choices

**Free Resources:** https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/ — Kimball Group dimensional modelling reference techniques

---

## Primary Keys, Surrogate Keys, and SCDs

**Status:** ⬜ Not Started

**Definition:** A primary key uniquely identifies each row. A surrogate key is a system-generated integer or UUID used as the primary key instead of the natural business key. Slowly Changing Dimensions (SCDs) are strategies for handling dimension attribute changes over time — SCD Type 1 overwrites old values, Type 2 adds new rows with validity date ranges.

**Mental Model:** A surrogate key is like a library barcode — not the book's ISBN (natural key), but a system-generated label that never changes even if the book gets a new edition. SCD Type 2 is like keeping all previous versions of a contact in a Rolodex with "valid from" and "valid to" dates.

**Common Misconceptions:**
- Natural keys are always safer than surrogate keys — natural keys can change (customer email, product SKU), cascading expensive updates; surrogate keys are stable by design.
- SCD Type 2 is always the right choice — it creates more rows and more complex queries; only use it when historical accuracy of dimension attributes genuinely matters to the business.

**Interview Skeleton:**
- What it is: mechanisms for uniquely identifying rows and tracking changes to dimension attributes over time
- Why it matters: incorrect key strategy leads to join fanout bugs, broken historical queries, or lost change history
- Example: describe how you'd implement SCD Type 2 for a customer dimension where address changes, including the merge logic

**Free Resources:** https://www.databricks.com/glossary/slowly-changing-dimensions — Databricks explanation of SCD types with practical examples

---

## Define Grain, Partitions, and Ownership

**Status:** ⬜ Not Started

**Definition:** Grain is the precise definition of what one row in a fact table represents (one order line item, not one order). Partitioning divides a large table into smaller physical segments by a column — usually date — to speed up filtering queries. Ownership defines who is responsible for a table's data quality, schema changes, and SLA.

**Mental Model:** Grain is the atomic unit of your data — get this wrong and every aggregation built on top will be wrong. Partitioning is like organising filing cabinets by year — you only open the 2024 drawer instead of searching everything.

**Common Misconceptions:**
- Grain can be changed later without major impact — changing grain requires rebuilding historical data and updating all downstream models; define it correctly upfront.
- Partitioning always speeds up queries — partition pruning only kicks in if your WHERE clause filters on the partition column; queries without that filter still scan all partitions.

**Interview Skeleton:**
- What it is: foundational design decisions that determine correctness, performance, and maintainability of a data model
- Why it matters: wrong grain causes double-counting; wrong partitioning causes full table scans; unclear ownership causes undetected data quality failures
- Example: define the grain for a subscription events fact table and justify your partition key choice

**Free Resources:** https://docs.getdbt.com/terms/grain — dbt glossary definition of grain with practical guidance

---

## Normalisation vs Denormalisation

**Status:** ⬜ Not Started

**Definition:** Normalisation organises data into multiple related tables to reduce redundancy and enforce integrity — the OLTP pattern. Denormalisation flattens data into fewer, wider tables to speed up read queries by eliminating joins — the OLAP pattern.

**Mental Model:** Normalisation is a well-organised apartment — each item has its proper place, nothing is duplicated. Denormalisation is packing a suitcase for a trip — you put everything together even if it means duplication, because convenience matters more than tidiness.

**Common Misconceptions:**
- Denormalisation is always bad practice — in analytical systems, denormalised wide tables are often the correct design choice for query performance.
- Full normalisation is the goal in a data warehouse — data warehouses intentionally denormalise for read performance; 3NF is designed for OLTP, not OLAP.

**Interview Skeleton:**
- What it is: a spectrum of design choices trading storage efficiency for query performance and join complexity
- Why it matters: the right choice depends on workload type, query patterns, and write frequency
- Example: explain when you'd flatten a product hierarchy into the fact table vs keep it as a separate normalised dimension

**Free Resources:** https://www.vertabelo.com/blog/denormalization-when-why-and-how/ — Detailed article on when and how to denormalise for analytical workloads

---

## Data Quality Checks and Business Logic

**Status:** ⬜ Not Started

**Definition:** Data quality checks are automated tests that validate expected properties: row counts, null rates, uniqueness, referential integrity, value ranges, and freshness. Business logic encodes the organisation's definitions into the data model — for example, "an active customer has purchased in the last 90 days."

**Mental Model:** Data quality checks are smoke alarms — you install them once and they alert you before a small problem becomes a disaster. Business logic in the model is a shared dictionary — everyone uses the same definition rather than their own interpretation.

**Common Misconceptions:**
- Data quality is someone else's responsibility — data engineers own the quality of what they produce; bad pipeline output means wrong downstream decisions.
- Business logic should live only in the BI tool — if logic lives only in dashboards, the same metric gets calculated differently in different places, causing endless stakeholder disagreements.

**Interview Skeleton:**
- What it is: automated validation and standardised encoding of business rules that ensure data is correct and consistently interpreted
- Why it matters: data quality failures erode trust; inconsistent business logic causes metric disagreements that take months to resolve
- Example: describe the dbt tests you'd add to a fact table and where you'd encode the "active customer" definition

**Free Resources:** https://docs.getdbt.com/docs/build/data-tests — dbt documentation on writing and running data quality tests
