# Snowflake — Data Engineering

---

## Snowpipe

**Status:** ⬜ Not Started

**Definition:** Snowpipe is Snowflake's continuous micro-batch ingestion service that automatically loads new files from cloud storage (S3, Azure Blob, GCS) as they arrive, using event notifications to trigger loads within seconds rather than waiting for a scheduled COPY INTO batch.

**Mental Model:** Snowpipe is an always-on loading dock — the moment a new file lands in the cloud storage bucket, Snowpipe is notified and loads it into the target table automatically, without a scheduled pipeline needing to poll.

**Free Resources:** https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro — Snowpipe documentation covering setup, event notification configuration, and monitoring

---

## Streams and Tasks

**Status:** ⬜ Not Started

**Definition:** A Snowflake Stream is a CDC (change data capture) object that tracks row-level DML changes (INSERT, UPDATE, DELETE) on a table. A Task is a scheduled or event-triggered SQL or stored procedure execution. Together, Streams + Tasks enable lightweight ELT pipelines entirely within Snowflake.

**Mental Model:** A Stream is a change log that records what happened to a table; a Task is a scheduled worker that processes that change log. Together they are a built-in mini-Airflow for simple Snowflake-native pipelines.

**Free Resources:** https://docs.snowflake.com/en/user-guide/streams-intro — Snowflake Streams documentation covering stream types and consumption patterns

---

## Dynamic Tables

**Status:** ⬜ Not Started

**Definition:** Dynamic Tables are Snowflake's declarative pipeline feature — you define the target transformation as a SELECT query, set a target lag (e.g., 1 minute, 1 hour), and Snowflake automatically keeps the table up to date by incrementally refreshing it. They replace complex Streams + Tasks logic for many use cases.

**Mental Model:** Dynamic Tables are materialised views with a freshness guarantee — you declare what the table should look like (the query) and how stale it can be (the lag), and Snowflake figures out the refresh schedule and incremental computation.

**Free Resources:** https://docs.snowflake.com/en/user-guide/dynamic-tables-intro — Snowflake Dynamic Tables documentation covering definition, lag, and incremental refresh

---

## Iceberg Tables

**Status:** ⬜ Not Started

**Definition:** Snowflake Iceberg Tables store data in open Apache Iceberg format on your own cloud storage rather than in Snowflake's proprietary storage. This allows Snowflake to query data that other engines (Spark, Flink, Trino) also access, enabling a multi-engine lakehouse architecture with Snowflake as one of the query engines.

**Mental Model:** Iceberg Tables are Snowflake's "bring your own storage" option — data lives in your bucket in open format, Snowflake reads it as a first-class citizen, but you're not locked in to Snowflake-only access.

**Free Resources:** https://docs.snowflake.com/en/user-guide/tables-iceberg — Snowflake Iceberg Tables documentation covering setup, integration, and use cases

---

## Stages and File Formats

**Status:** ⬜ Not Started

**Definition:** Stages are named cloud storage locations in Snowflake — either internal (Snowflake-managed storage) or external (your S3/ADLS/GCS bucket). File Formats define how files should be parsed (CSV delimiter, JSON strip nulls, Parquet compression). COPY INTO uses stages and file formats to load data into tables.

**Mental Model:** A Stage is the loading bay — where files wait before being loaded. A File Format is the instruction sheet for how to read each file type. Together they make COPY INTO statements reusable and configurable.

**Free Resources:** https://docs.snowflake.com/en/user-guide/data-load-overview — Snowflake data loading overview covering stages, file formats, and COPY INTO

---

## Time Travel

**Status:** ⬜ Not Started

**Definition:** Snowflake Time Travel allows querying data as it existed at any point within the retention period (1 day by default, up to 90 days on Enterprise). You can query historical data with `AT (TIMESTAMP => ...)`, restore accidentally dropped or modified tables, and create clones from any historical point.

**Mental Model:** Time Travel is an undo button for your data — not just one level back, but any point within the retention window. Accidentally deleted a table? Travel back 30 minutes and restore it.

**Free Resources:** https://docs.snowflake.com/en/user-guide/data-time-travel — Snowflake Time Travel documentation covering syntax, use cases, and retention configuration
