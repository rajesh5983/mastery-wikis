# Layer 5 — Scale

> **Framework:** Distributed systems and performance engineering for large-scale data workloads.

---

## Distributed Processing Fundamentals

**Status:** ⬜ Not Started

**Definition:** Distributed processing splits a large computation across multiple machines (nodes) working in parallel. Each node processes a partition of the data, and results are combined. This enables processing datasets larger than any single machine's memory or storage can hold.

**Mental Model:** Distributed processing is like a barn raising — one person can't lift the beams alone, but ten people working on coordinated sections can raise the whole barn in a day.

**Common Misconceptions:**
- Distributed is always faster — distribution adds network overhead and coordination cost; for small datasets, a single machine is faster and simpler.
- All distributed systems work the same way — Spark (batch, in-memory), Flink (stream), Kafka (messaging), and HDFS (storage) each have fundamentally different architectures and trade-offs.

**Interview Skeleton:**
- What it is: splitting computation across many machines to process data that exceeds single-node capacity
- Why it matters: modern data volumes require horizontal scaling; understanding the fundamentals predicts how systems behave under load and failure
- Example: explain the driver/executor model in Spark and what happens when an executor fails mid-job

**Free Resources:** https://spark.apache.org/docs/latest/cluster-overview.html — Apache Spark cluster mode overview explaining the distributed architecture

---

## Partitioning, Skew, Shuffle, and Memory Trade-offs

**Status:** ⬜ Not Started

**Definition:** Partitioning divides data into chunks processed independently. Skew occurs when some partitions are much larger than others, causing slow tasks that bottleneck the entire job. Shuffle is the expensive network operation of redistributing data across nodes, triggered by joins and aggregations. Memory trade-offs determine what stays in RAM vs spills to disk.

**Mental Model:** Skew is one checkout lane at the supermarket with 50 people while the others are empty — the overall throughput is limited by the slowest lane. Shuffle is the postal system — fast within a building, slow across cities.

**Common Misconceptions:**
- More partitions are always better — too many small partitions create scheduling overhead; the sweet spot is partitions of roughly 100–200MB each.
- Shuffles can be eliminated with clever code — some operations (group by, join on non-co-partitioned data) require shuffles; the goal is to minimise them, not eliminate them entirely.

**Interview Skeleton:**
- What it is: the mechanics of how data is distributed and moved in a distributed compute engine and the performance implications of each
- Why it matters: most Spark performance problems are caused by skew, excessive shuffles, or memory spills to disk
- Example: diagnose a slow Spark join — how do you detect skew, and what strategies (salting, broadcast joins, AQE) would you apply?

**Free Resources:** https://spark.apache.org/docs/latest/sql-performance-tuning.html — Spark SQL performance tuning guide covering partitioning and Adaptive Query Execution

---

## Streaming Basics, Kafka, and Event Pipelines

**Status:** ⬜ Not Started

**Definition:** Stream processing handles data as it arrives, event by event or in micro-batches, rather than waiting to accumulate a large batch. Apache Kafka is a distributed message broker that durably stores and replicates events at scale. Event pipelines consume from Kafka topics, apply transformations, and write to downstream sinks.

**Mental Model:** Kafka is a conveyor belt at a factory — producers put items on it, consumers pick items off at their own pace, and the belt keeps moving regardless. Streaming is processing each item as it passes rather than waiting for a full batch.

**Common Misconceptions:**
- Streaming is always better than batch — streaming adds significant complexity; use it only when business requirements genuinely demand low latency (minutes, not hours).
- Kafka guarantees exactly-once processing by default — Kafka provides exactly-once semantics only with specific producer/consumer configuration (idempotent producers + transactional consumers).

**Interview Skeleton:**
- What it is: infrastructure for processing data as it arrives rather than in scheduled batch windows
- Why it matters: fraud detection, real-time dashboards, and CDC all require streaming; knowing when to stream vs batch is a senior engineering decision
- Example: describe a Kafka-based clickstream pipeline including consumer groups, offset management, and how you handle consumer lag

**Free Resources:** https://developer.confluent.io/learn-kafka/ — Confluent's free Kafka fundamentals course covering architecture and patterns

---

## Throughput, Latency, and Cost Trade-offs

**Status:** ⬜ Not Started

**Definition:** Throughput is how much data a system processes per unit time (GB/hour). Latency is the delay between data being generated and being available for querying. Cost is the cloud spend required to achieve a given throughput and latency target. These three form a triangle — optimising one typically degrades the others.

**Mental Model:** Think of shipping packages. Same-day delivery (low latency) costs more and limits batch size (lower throughput). Weekly freight (high throughput) is cheap but slow. The right choice depends on what the receiver actually needs.

**Common Misconceptions:**
- Lower latency is always worth the cost — near-real-time streaming can cost 10x more than a daily batch job; validate that the business genuinely needs it before committing.
- Throughput and latency are independent — increasing batch sizes improves throughput but increases latency; for batch systems they are directly linked.

**Interview Skeleton:**
- What it is: the three competing dimensions of pipeline performance that must be balanced against actual business requirements
- Why it matters: engineers who ignore cost trade-offs ship technically correct but economically unsustainable systems
- Example: a stakeholder asks for real-time data — how do you evaluate whether hourly batch would satisfy their actual need?

**Free Resources:** https://www.databricks.com/blog/2022/06/21/streaming-data-architectures-a-technical-guide.html — Databricks guide on streaming architecture and latency/cost trade-offs

---

## Lake, Warehouse, and Lakehouse Choices

**Status:** ⬜ Not Started

**Definition:** A data lake stores raw, unstructured, and structured data cheaply in object storage (S3, ADLS, GCS). A data warehouse stores structured, transformed data optimised for SQL analytics (Snowflake, BigQuery, Redshift). A lakehouse combines both — open table formats (Delta Lake, Iceberg, Hudi) add ACID transactions and query performance to object storage.

**Mental Model:** A data lake is a reservoir — cheap, holds everything, but you need to know where to fish. A data warehouse is a bottled water facility — processed, clean, expensive, ready to drink. A lakehouse is a filtered tap — the reservoir's capacity with some of the warehouse's cleanliness.

**Common Misconceptions:**
- Lakehouses have made data warehouses obsolete — managed warehouses (Snowflake, BigQuery) still offer superior SQL performance, governance, and ease of use for most enterprise analytics workloads.
- All data should go into a data lake first — without governance and schema enforcement, data lakes become swamps where nobody can find or trust anything.

**Interview Skeleton:**
- What it is: three architectural patterns for storing and querying large datasets with different cost, performance, and governance profiles
- Why it matters: choosing the wrong storage architecture creates expensive migrations later
- Example: when would you recommend a lakehouse over a managed warehouse, and which open table format would you choose and why?

**Free Resources:** https://www.databricks.com/glossary/data-lakehouse — Databricks explanation of lakehouse architecture and Delta Lake

---

## Reliability and SLA Awareness

**Status:** ⬜ Not Started

**Definition:** Reliability is the probability that a pipeline delivers data correctly and on time. An SLA (Service Level Agreement) defines the agreed freshness, accuracy, and availability standards for a data product. SLA awareness means designing pipelines to meet those standards and alerting before they are breached.

**Mental Model:** An SLA is the commitment to have the newspaper on the doorstep by 7am. Reliability engineering is everything that makes that commitment achievable — spare delivery drivers, early alerts if the print run is delayed, a backup route.

**Common Misconceptions:**
- SLAs are defined by stakeholders, not engineering's concern — data engineers must understand and negotiate SLAs because they dictate architectural choices like streaming vs batch and redundancy requirements.
- 99% uptime sounds high — 99% allows 87 hours of downtime per year; most business-critical data products require 99.9% or higher.

**Interview Skeleton:**
- What it is: the practices and metrics that ensure data pipelines deliver trustworthy results within agreed time bounds consistently
- Why it matters: unreliable data destroys trust; once lost, trust takes months to rebuild
- Example: describe how you'd design a pipeline to guarantee a 6am SLA, including what happens when an upstream dependency is late

**Free Resources:** https://cloud.google.com/architecture/best-practices-for-operational-excellence — Google Cloud best practices for data pipeline reliability and operational excellence
