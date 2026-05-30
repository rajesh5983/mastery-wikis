# Layer 6 — Platform

> **Framework:** Cloud data platforms and production operations for scalable analytics infrastructure.

---

## Cloud Storage and Compute Fundamentals

**Status:** ⬜ Not Started

**Definition:** Cloud storage (S3, Azure Blob, GCS) provides scalable, cheap object storage for raw and processed data. Cloud compute (VMs, containers, managed clusters) provides processing power. The key architectural shift in modern data platforms is that storage and compute are decoupled — you scale them independently and pay separately.

**Mental Model:** Separating storage and compute is like separating a library from its reading rooms. The books (data) stay in the library forever; the reading rooms (compute) can expand or close based on how many readers show up today.

**Common Misconceptions:**
- Separating storage and compute always increases costs — separation reduces costs by allowing compute to be shut down when idle; object storage is extremely cheap at rest.
- Cloud object storage is just cheaper block storage — object storage has different access patterns (optimised for large sequential reads, not random row access); treat it differently.

**Interview Skeleton:**
- What it is: the foundational cloud infrastructure pattern enabling scalable, cost-effective data platforms
- Why it matters: the storage/compute separation is the architectural basis for Snowflake, BigQuery, and Databricks
- Example: explain why Snowflake can scale a virtual warehouse up to handle a burst query without affecting storage costs

**Free Resources:** https://aws.amazon.com/s3/features/ — AWS S3 documentation on object storage architecture, durability, and access patterns

---

## Snowflake, BigQuery, Redshift, and Databricks

**Status:** ⬜ Not Started

**Definition:** These are the four dominant cloud data platforms. Snowflake is a multi-cloud SQL warehouse with independently scalable virtual warehouses. BigQuery is Google's serverless warehouse with per-query pricing. Redshift is AWS's columnar warehouse. Databricks is a unified analytics platform built on Apache Spark with Delta Lake and Unity Catalog.

**Mental Model:** Snowflake is a professional kitchen — well-organised, powerful, you pay for time at the counter. BigQuery is a restaurant where you pay per dish ordered, not per hour in the kitchen. Databricks is a combined kitchen and food research lab — SQL, ML, and streaming in one space.

**Common Misconceptions:**
- BigQuery is always cheapest because of per-query pricing — poorly optimised queries on large unpartitioned tables generate enormous BigQuery costs; partition and cluster tables to control spend.
- Databricks is only for ML/AI workloads — Databricks is a full data engineering platform with SQL analytics, streaming pipelines, and governance through Unity Catalog.

**Interview Skeleton:**
- What it is: the leading cloud data platforms, each with different pricing models, architecture, and optimal workload profiles
- Why it matters: platform choice shapes cost, performance, and how the entire data stack is architected for years
- Example: compare Snowflake's virtual warehouse model with BigQuery's serverless model — when would you choose each?

**Free Resources:** https://docs.snowflake.com/en/user-guide/intro-key-concepts — Snowflake introduction to key architectural concepts

---

## IAM, Security, and Governance Basics

**Status:** ⬜ Not Started

**Definition:** IAM (Identity and Access Management) controls who can access which resources. In data platforms, governance extends to column-level security, row-level filters, data masking, and audit logging. Data engineers must implement least-privilege access and understand compliance requirements (GDPR, HIPAA, SOC 2).

**Mental Model:** IAM is a building's keycard system — some cards open only the lobby, others open specific labs, a few open everything. Data governance adds a camera in every room that records who accessed what, when, and from where.

**Common Misconceptions:**
- IAM is the security team's responsibility, not data engineering's — data engineers provision service account access, table permissions, and column masking for every pipeline they build; security sets policy, engineers implement it.
- Read-only access is always safe — over-permissioned read access to PII tables violates compliance rules even if no one misuses it.

**Interview Skeleton:**
- What it is: the mechanisms for controlling, auditing, and governing access to data assets across the platform
- Why it matters: data breaches and compliance failures start with poor access controls on systems engineers built
- Example: describe how you'd implement column-level masking for PII in Snowflake and audit who accessed unmasked data

**Free Resources:** https://docs.snowflake.com/en/user-guide/security-access-control-overview — Snowflake access control and security overview

---

## Schema Evolution and Metadata/Catalog Concepts

**Status:** ⬜ Not Started

**Definition:** Schema evolution is the process of changing a table's structure (adding columns, changing types, renaming fields) without breaking downstream consumers. A data catalog is a searchable inventory of all data assets with their schemas, lineage, owners, and quality information.

**Mental Model:** Schema evolution is renovating a house while people are still living in it — add a new room freely, but don't knock down a load-bearing wall without warning everyone first. A data catalog is a library card catalog — tells you what exists, where it is, and who to contact.

**Common Misconceptions:**
- Adding a column is always a safe change — adding NOT NULL columns without defaults breaks existing pipelines that don't reference the new column; always add nullable columns or provide a default value.
- Data catalogs are nice-to-haves rather than infrastructure — without a catalog, engineers spend hours finding the right table and guessing ownership; catalogs multiply team productivity.

**Interview Skeleton:**
- What it is: practices and tools for managing how data schemas change over time and how data assets are discovered
- Why it matters: schema changes without coordination break downstream pipelines; lack of metadata creates tribal knowledge that blocks onboarding
- Example: describe how you'd handle adding a new required column to a high-traffic fact table in a production Snowflake environment

**Free Resources:** https://docs.getdbt.com/docs/collaborate/govern/about-dbt-governance — dbt governance documentation covering schema evolution and data contracts

---

## Data Contracts and Access Patterns

**Status:** ⬜ Not Started

**Definition:** A data contract is a formal, versioned agreement between a data producer and its consumers specifying the schema, semantics, SLAs, and ownership of a dataset. Access patterns define how downstream consumers are expected to query data — via views, APIs, or direct table access — and what guarantees they can rely on.

**Mental Model:** A data contract is like an API contract — the provider commits to a schema and behaviour, consumers can depend on it, and breaking changes require a version bump and a migration period. Direct table access makes internal schema a public API.

**Common Misconceptions:**
- Data contracts are bureaucratic overhead that slow teams down — without contracts, schema changes silently break pipelines; contracts shift this from an incident to a managed process.
- Direct table access is fine for all consumers — when internal schema is exposed directly, any refactoring immediately becomes a breaking change for every consumer.

**Interview Skeleton:**
- What it is: formalised agreements and controlled access mechanisms that decouple data producers from consumers
- Why it matters: at scale, uncoordinated schema changes cause widespread pipeline failures; contracts prevent this
- Example: describe the components of a data contract for a customer events table shared with five downstream teams

**Free Resources:** https://atlan.com/data-contracts/ — Atlan's guide to data contracts with implementation patterns

---

## Performance Tuning and Cost Optimisation

**Status:** ⬜ Not Started

**Definition:** Performance tuning identifies and fixes query execution bottlenecks — adding clustering keys, materialising frequent subqueries, optimising join order, right-sizing compute. Cost optimisation reduces cloud spend through auto-suspend, query result caching, partition pruning, and storage tiering.

**Mental Model:** Performance tuning is diagnosing a car — read the diagnostics (query profile), find the bottleneck (full table scan, memory spill), and fix it (add clustering, increase memory). Cost optimisation is turning the engine off when the car is parked.

**Common Misconceptions:**
- A bigger warehouse always fixes performance — many slow queries are caused by missing clustering keys or poor SQL, not compute shortage; throwing compute at bad SQL just costs more money.
- Storage costs are negligible — cloud storage at petabyte scale costs thousands of dollars per month; compression, tiering, and archiving matter.

**Interview Skeleton:**
- What it is: techniques for making data platform queries faster while keeping the cloud bill under control
- Why it matters: unoptimised platforms waste money and miss SLAs; engineers who understand cost/performance trade-offs are significantly more valuable
- Example: a Snowflake query scans a 10TB table in 45 seconds — walk through your investigation and optimisation process step by step

**Free Resources:** https://docs.snowflake.com/en/user-guide/warehouses-considerations — Snowflake warehouse sizing and performance considerations guide
