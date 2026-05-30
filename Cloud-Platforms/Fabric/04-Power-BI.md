# Microsoft Fabric — Power BI

---

## Direct Lake Mode

**Status:** ⬜ Not Started

**Definition:** Direct Lake is a Power BI connectivity mode that reads Delta Parquet column segment files directly from OneLake into the VertiPaq in-memory engine — combining the query speed of Import mode (in-memory columnar storage) with the data freshness of DirectQuery (no scheduled refresh step). Data is loaded into memory on demand per column segment, not imported as a full dataset copy.

**Key Mental Model:** Direct Lake is the best of both worlds for Power BI — data stays in OneLake (always fresh, like DirectQuery), but Power BI loads it into memory on demand (fast, like Import) without a scheduled copy.

**How It Works:**
- In Direct Lake mode, Power BI's VertiPaq engine does not hold a full pre-imported copy of the dataset. Instead, it maintains a metadata mapping from semantic model columns to specific Delta Parquet file column segments in OneLake; when a DAX query needs data from a column, VertiPaq loads that column's segment from the corresponding Parquet file directly into memory.
- **Column segment loading** is the key mechanism: Delta Parquet files are columnar, and VertiPaq reads individual column chunks (segments) rather than entire row groups; for a report visual that queries 3 out of 50 columns in a large table, only those 3 column segments are loaded from OneLake, not the full table.
- **Framing** is Direct Lake's equivalent of a semantic model refresh — when the underlying Delta table receives a new commit (new Parquet files via a Spark job or Pipeline), the VertiPaq engine performs a framing operation that updates its metadata to point to the new Delta snapshot; this is near-instantaneous compared to a full Import refresh and happens automatically.
- **Fallback to DirectQuery** occurs when a query requests aggregations or calculations that require data not yet loaded in memory and cannot be efficiently loaded on demand — in this case, Direct Lake transparently falls back to issuing a DirectQuery SQL call to the Lakehouse SQL Analytics Endpoint for that specific query, ensuring query correctness at the cost of DirectQuery latency.
- Direct Lake mode requires the semantic model to be in a Fabric-capacity workspace (F-SKU or P-SKU) and connected to a Fabric Lakehouse or Warehouse; it is not available for external data sources or for OneLake tables accessed via ADLS Gen2 shortcuts from outside Fabric. See [[Cloud-Platforms/Fabric/02-Data-Engineering#Delta Format on OneLake]] for the Delta layer that Direct Lake reads.

**Common Misconceptions:**
- Direct Lake does not eliminate all latency — the first query against a column that has not been loaded into VertiPaq memory triggers an on-demand segment load from OneLake, which has measurable latency; subsequent queries against the same column use the in-memory segment at full VertiPaq speed.
- Direct Lake does not support all semantic model features that Import mode supports — measures referencing certain unsupported DAX functions or calculated tables derived from complex M queries may force a fallback to DirectQuery or may not be supported in Direct Lake mode at all; teams migrating from Import mode must validate measure compatibility.

**Interview Answer Skeleton:**
- **What it is:** A VertiPaq loading mode that reads Delta Parquet column segments from OneLake on demand rather than maintaining a full pre-imported copy, using Delta framing for near-instant metadata refresh when the underlying table updates, with transparent DirectQuery fallback for unsupported patterns.
- **Why it matters / trade-offs:** Eliminates scheduled dataset refresh latency (large Import datasets can take hours to refresh), giving reports near-real-time freshness; the trade-off is that cold-query performance for unloaded columns has higher latency than a fully pre-warmed Import dataset, and fallback to DirectQuery for unsupported patterns can surprise teams with unexpected latency.
- **Example or context:** A retail team's Power BI sales dashboard previously used Import mode with a 4-hour refresh schedule — migrating to Direct Lake means Spark pipelines writing new sales data to the Lakehouse are visible in the dashboard within minutes of the Delta commit, with no scheduled refresh job to maintain.

**Free Resources:**
- [Direct Lake Mode Overview](https://learn.microsoft.com/en-us/fabric/get-started/direct-lake-overview) — architecture, framing, column segment loading, and fallback behaviour
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Direct Lake configuration, limitations, and migration from Import mode

---

## Semantic Models

**Status:** ⬜ Not Started

**Definition:** A Semantic Model (formerly Power BI Dataset) defines the business logic layer on top of raw data — measures, calculated columns, relationships, hierarchies, and row-level security — that Power BI reports connect to. In Fabric, semantic models connect to Lakehouse and Warehouse Delta tables via Direct Lake mode, serving as the governed analytical contract between data and report authors.

**Key Mental Model:** A semantic model is the translation layer between raw data and business language — it defines what "Revenue" means, how Customer relates to Order, and who is allowed to see which rows.

**How It Works:**
- A semantic model is defined using a **tabular model schema** — tables, columns, measures (DAX expressions), and relationships are defined in the model's `.pbix` or `.bim` file; the model is published to a Fabric workspace where it is stored as a semantic model item.
- **Relationships** define how tables are joined for DAX calculations — a one-to-many relationship between `DimCustomer` (one) and `FactSales` (many) means that filtering `DimCustomer` automatically filters `FactSales` when DAX evaluates measures, through the relationship's filter direction propagation.
- **DAX filter context** propagates from report visuals (slicers, filters) through the semantic model's relationship graph — when a slicer selects `Region = 'West'`, the filter context flows through active relationships to all related tables, and DAX measures evaluate `SUM(FactSales[Revenue])` only for rows where the related dimension rows match the filter.
- **Row-Level Security (RLS)** is defined as DAX filter expressions on dimension tables — `[Region] = USERPRINCIPALNAME()` restricts each user to rows matching their email-to-region mapping; RLS is enforced at the semantic model layer before any data is returned to the report visual, regardless of the underlying data source.
- **Direct Lake connection** binds the semantic model to specific Delta tables in a Lakehouse or Warehouse — the model's table definitions map to Delta table paths in OneLake, and VertiPaq loads column segments from those paths on demand using the Direct Lake framing mechanism. See [[Cloud-Platforms/Fabric/04-Power-BI#Direct Lake Mode]] for the storage loading mechanics.

**Common Misconceptions:**
- A semantic model is not a data copy — in Direct Lake mode, the model does not hold a separate duplicate of the data; it holds metadata (column types, measure definitions, relationships) and loads column segments from OneLake on demand. The data lives in OneLake, not in the semantic model.
- Semantic models and Fabric Warehouses are not interchangeable — the Warehouse serves T-SQL analytical queries; the semantic model serves DAX queries from Power BI with business logic (measures, hierarchies, RLS) that do not exist in the Warehouse schema; both are needed for a complete BI architecture.

**Interview Answer Skeleton:**
- **What it is:** A tabular metadata layer that defines DAX measures, table relationships, RLS policies, and hierarchies over underlying data sources, serving as the analytical contract between raw OneLake/Warehouse data and Power BI report visualisations.
- **Why it matters / trade-offs:** Centralises business metric definitions so all reports using the same semantic model use consistent, governed calculations; the trade-off is that poorly designed semantic models with complex bidirectional relationships or many-to-many relationships cause performance issues that manifest as slow DAX query times.
- **Example or context:** A finance team's semantic model defines `[Gross Margin %]` as a DAX measure with 15 steps of logic; all 20 Power BI reports built by different teams use this single semantic model — when the CFO changes the margin definition, the data team updates it in one place and all reports immediately reflect the new logic.

**Free Resources:**
- [Semantic Models Documentation](https://learn.microsoft.com/en-us/power-bi/connect-data/service-datasets-understand) — measures, relationships, RLS, and Direct Lake connection configuration
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — semantic model lifecycle management, endorsement, and cross-workspace sharing

---

## DAX

**Status:** ⬜ Not Started

**Definition:** DAX (Data Analysis Expressions) is the formula language used in Power BI, Analysis Services, and Fabric semantic models for defining measures, calculated columns, and custom tables. DAX operates on tabular data with implicit row context and filter context semantics — understanding how filter context is established and manipulated by `CALCULATE()` is fundamental to writing correct DAX.

**Key Mental Model:** DAX is the formula language of Power BI — like Excel formulas but for data models. CALCULATE() is the engine that changes filter context; SUMX() and AVERAGEX() iterate row by row. Understanding context propagation is the key to mastery.

**How It Works:**
- Every DAX measure evaluation begins with a **filter context** established by the report visual, slicers, and row/column headers; filter context is a set of active filters on the model's tables that restricts which rows each table exposes to the DAX expression.
- `CALCULATE()` is DAX's context modifier — it evaluates its first argument (an expression) in a modified filter context defined by its subsequent arguments (filter expressions); `CALCULATE([Revenue], Region = "West")` replaces the existing Region filter with `West` before evaluating `[Revenue]`, regardless of what the report visual's filter context says.
- **Iterator functions** (SUMX, AVERAGEX, MAXX, MINX, RANKX) iterate over a table row by row — for each row, they establish a **row context** (the values of all columns in that row are accessible), evaluate the second argument expression in that row context, and accumulate the result; they do not inherit filter context propagation automatically, requiring explicit `RELATED()` or `RELATEDTABLE()` to traverse relationships in row context.
- **Context transition** occurs when a row context is present inside `CALCULATE()` — the row context is automatically converted into an equivalent filter context before the expression is evaluated; this is essential for measures referenced within calculated columns, and is a common source of unexpected results for DAX practitioners.
- **Variables** (`VAR x = expr RETURN expr`) in DAX are evaluated at the point of declaration using the current filter and row context at that moment — they do not re-evaluate lazily when later referenced, enabling complex multi-step calculations and preventing repeated evaluation of expensive sub-expressions. See [[Cloud-Platforms/Fabric/04-Power-BI#Semantic Models]] for how DAX measures are used within semantic models.

**Common Misconceptions:**
- DAX `CALCULATE()` does not always add to filters — `CALCULATE([Revenue], ALL(DimDate))` removes all filters from `DimDate` rather than adding one; `CALCULATE()` arguments can both add and remove filters, and understanding which filters are preserved vs replaced is critical to writing correct year-over-year or share-of-total measures.
- Calculated columns and measures are fundamentally different in DAX — calculated columns are computed once at data refresh and stored in the model (row context, materialised); measures are computed on demand per query (filter context, not stored); using a calculated column when a measure is needed causes stale results and unnecessary storage.

**Interview Answer Skeleton:**
- **What it is:** A tabular formula language for Power BI semantic models where every expression evaluates within a filter context established by visual filters and modified by `CALCULATE()`, with row-level iteration via SUMX/AVERAGEX functions and context transition for calculated columns.
- **Why it matters / trade-offs:** Enables complex analytical calculations (time intelligence, ratios, rankings, semi-additive measures) that SQL cannot express within a report visual without pre-materialised aggregations; the trade-off is a steep learning curve around context propagation, and poorly written DAX (unnecessary ALL(), uncached virtual tables) can cause severely degraded report performance.
- **Example or context:** A financial analyst needs Year-over-Year growth: `[YoY Growth %] = DIVIDE([Revenue] - CALCULATE([Revenue], SAMEPERIODLASTYEAR(DimDate[Date])), CALCULATE([Revenue], SAMEPERIODLASTYEAR(DimDate[Date])))` — `CALCULATE()` with `SAMEPERIODLASTYEAR()` shifts the date filter context to the prior year while preserving all other slicer filters, something impossible to express in a single SQL window function with dynamic user slicer state.

**Free Resources:**
- [DAX Reference Documentation](https://learn.microsoft.com/en-us/dax/) — complete DAX function reference with syntax, parameters, and examples
- [Power BI Documentation](https://learn.microsoft.com/en-us/power-bi) — DAX measures, calculated columns, context transition, and time intelligence pattern guides

---

## Power BI Premium vs Fabric

**Status:** ⬜ Not Started

**Definition:** Power BI Premium (P-SKUs) was the previous enterprise capacity tier with dedicated compute for Power BI. Microsoft Fabric F-SKUs supersede it, offering all Premium features (large models, incremental refresh, XMLA endpoint, paginated reports) plus all Fabric experiences (Lakehouse, Warehouse, Data Engineering, ML) in a single unified capacity. P-SKUs are being retired in favour of F-SKUs.

**Key Mental Model:** Fabric F-SKUs are the upgrade from Premium — you get everything Power BI Premium offered, plus all the data engineering and ML capabilities of Fabric, under one capacity bill.

**How It Works:**
- Power BI Premium P-SKUs were Azure-hosted dedicated capacity units allocated exclusively to Power BI workloads — datasets, reports, dataflows, and paginated reports; P-SKUs did not include Fabric's data engineering experiences (Lakehouse, Warehouse, Spark, Pipelines).
- Fabric F-SKUs provide the same Power BI Premium capabilities (XMLA endpoint for external model connectivity, incremental refresh, large semantic models beyond 10GB, paginated reports, deployment pipelines) plus the full Fabric experience portfolio — the CU pool is shared across all workload types rather than isolated to Power BI.
- The **pricing equivalence** between P-SKUs and F-SKUs is approximately: P1 ≈ F64, P2 ≈ F128, P3 ≈ F256 — at equivalent capacity, F-SKUs add all Fabric engineering experiences at no additional cost, making them universally more cost-effective for organisations that would otherwise purchase both Premium and separate Azure PaaS data services.
- F-SKUs offer **pay-as-you-go hourly billing** as an option in addition to reserved annual pricing, whereas P-SKUs required annual reserved commitments — this gives Fabric customers more flexibility to scale capacity up/down or pause and resume without financial penalties.
- Existing P-SKU customers can **migrate to F-SKUs** using Microsoft's migration tooling; the Power BI workspaces, semantic models, and reports migrate with full feature parity, with the added benefit that the workspace can immediately use Lakehouse, Warehouse, and Pipeline capabilities without purchasing additional services. See [[Cloud-Platforms/Fabric/01-Architecture#Fabric Capacities]] for F-SKU capacity mechanics.

**Common Misconceptions:**
- F-SKUs and P-SKUs are not exactly the same compute under different branding — F-SKUs use the newer Fabric capacity infrastructure with smoothing/burst mechanics, while P-SKUs used a fixed-allocation model; Power BI performance characteristics can differ slightly between the two for identical CU counts.
- Migrating from P-SKU to F-SKU does not require rebuilding Power BI reports or semantic models — the migration is at the capacity level; existing report `.pbix` files and published semantic models move to the new F-SKU capacity without modification.

**Interview Answer Skeleton:**
- **What it is:** The replacement of Power BI Premium P-SKUs with Fabric F-SKUs — F-SKUs provide all Power BI Premium features plus the full Fabric engineering experience portfolio (Lakehouse, Warehouse, Spark, Pipelines) under a single shared-CU capacity, with hourly pay-as-you-go billing flexibility.
- **Why it matters / trade-offs:** Simplifies licensing by consolidating Power BI and data engineering platform costs into one capacity; the trade-off is shared CU competition between Power BI and engineering workloads, which could degrade BI performance during heavy Spark ETL runs if capacity is undersized.
- **Example or context:** A company paying for Power BI Premium P1 (≈ F64 equivalent) migrates to F64 — they retain identical Power BI capabilities, gain access to Lakehouse, Warehouse, and Spark experiences at no extra cost, and can use the same F64 capacity for both their data engineering pipelines and Power BI reports.

**Free Resources:**
- [Power BI Premium FAQ and Fabric Migration](https://learn.microsoft.com/en-us/power-bi/enterprise/service-premium-faq) — P-SKU vs F-SKU comparison, feature parity, and migration guidance
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — licensing, capacity SKU tiers, and F-SKU feature matrix

---

## Real-Time Intelligence

**Status:** ⬜ Not Started

**Definition:** Real-Time Intelligence in Fabric combines Eventstream (event ingestion from Kafka, Azure Event Hubs, IoT Hub), EventHouse (KQL databases for time-series and log analytics), and Real-Time Dashboards for near-zero latency analytics on high-throughput streaming data — all stored in OneLake for cross-experience access.

**Key Mental Model:** Real-Time Intelligence is Fabric's streaming analytics stack — a conveyor belt of events (Eventstream), a specialised time-series storage (EventHouse/KQL), and a live dashboard that updates in seconds.

**How It Works:**
- **Eventstream** ingests data from streaming sources (Azure Event Hubs, Kafka, IoT Hub, custom HTTP endpoints) and provides a no-code visual routing interface where stream data can be filtered, aggregated, or enriched before routing to destinations (EventHouse, Lakehouse, Warehouse, another Eventstream).
- **EventHouse** is a Fabric-managed KQL (Kusto Query Language) database optimised for high-throughput time-series ingestion and fast analytical queries on time-windowed data — it uses a columnar storage engine tuned for append-heavy patterns (log data, IoT telemetry, clickstream events) with sub-second query latency.
- KQL databases in EventHouse store data as **Delta Parquet on OneLake** — this means KQL-ingested event data is immediately queryable by Spark notebooks and the Fabric Warehouse via cross-database queries and shortcuts, enabling the same event data to serve both real-time KQL queries and batch analytics.
- **Real-Time Dashboards** are KQL-backed dashboards that auto-refresh every few seconds, updating visual tiles as new event data arrives in the EventHouse — unlike Power BI dashboards (which refresh on a schedule), Real-Time Dashboards use streaming KQL queries that fetch only new rows since the last refresh, enabling near-real-time visualisation.
- Eventstream's **event processing** layer supports stateful transformations (windowed aggregations, sessionisation, joins to reference tables) executed by Fabric's managed stream processing compute, enabling pre-aggregated summaries to be written to the Lakehouse for batch analytics while raw events stream to the EventHouse for real-time queries. See [[Cloud-Platforms/Fabric/01-Architecture#Fabric Experiences]] for how Real-Time Intelligence fits within the broader Fabric platform.

**Common Misconceptions:**
- KQL (Kusto Query Language) is not SQL and is not T-SQL compatible — it is a pipe-separated query language with a different syntax and query model; engineers familiar only with SQL need to learn KQL basics to query EventHouse data directly, though the Fabric Warehouse's cross-database T-SQL can query EventHouse tables for analysts who prefer SQL.
- Real-Time Dashboards in Fabric are not the same as Power BI dashboards with streaming datasets — they use a fundamentally different rendering pipeline optimised for sub-second refresh from KQL queries; Power BI streaming datasets have different latency and query semantics.

**Interview Answer Skeleton:**
- **What it is:** A Fabric streaming analytics stack comprising Eventstream (managed ingestion routing with stream processing), EventHouse (KQL-based columnar time-series database on OneLake), and Real-Time Dashboards (auto-refreshing KQL-backed visualisations with second-level latency).
- **Why it matters / trade-offs:** Provides end-to-end streaming analytics within Fabric without separate Azure Stream Analytics or third-party streaming tools, with event data stored in OneLake for cross-experience batch access; the trade-off is that KQL is a specialised language with a learning curve, and EventHouse is optimised for time-series/log patterns, not general OLAP analytics.
- **Example or context:** A logistics company ingests GPS telemetry from 10,000 trucks via Eventstream from IoT Hub — raw events land in EventHouse for real-time tracking dashboards refreshed every 5 seconds; Eventstream also routes pre-aggregated hourly summary events to a Fabric Lakehouse for batch route-optimisation analysis in Spark.

**Free Resources:**
- [Fabric Real-Time Intelligence Overview](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/overview) — Eventstream, EventHouse, KQL databases, and Real-Time Dashboard documentation
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — KQL reference, Eventstream connectors, and Real-Time Intelligence architecture patterns

---

## Copilot in Power BI

**Status:** ⬜ Not Started

**Definition:** Copilot in Power BI uses LLMs (Azure OpenAI Service) to assist with report building — generating DAX measures from natural language descriptions, summarising report insights, creating report pages from prompts, and answering questions about data in conversational language. Available in Fabric-capacity (F64+) workspaces.

**Key Mental Model:** Copilot is a Power BI assistant who speaks natural language — describe the measure you need, and it writes the DAX; describe the chart you want, and it builds the visual.

**How It Works:**
- When a Copilot prompt is submitted in Power BI, the Copilot service sends the request to **Azure OpenAI Service** within Microsoft's trust boundary along with **semantic model metadata** as context — table names, column names, measure definitions, and relationship information are injected into the LLM prompt so the model can generate contextually accurate DAX and report structures.
- For **DAX generation**, Copilot constructs a prompt that includes the user's natural language request, the relevant semantic model table and column names, and examples of existing measures — the LLM generates DAX code that the Copilot service validates syntactically before presenting to the user for review and acceptance.
- For **report page generation**, Copilot analyses the semantic model structure and the user's description ("create a page showing sales trends by region over time") and generates a set of visual configurations — chart types, axis fields, filters, and layout — that are applied to a new report page as a starting point for further customisation.
- **Report summarisation** uses the LLM to interpret the data displayed in report visuals at the time of the summarisation request — Copilot reads the current visual data (from DAX query results) and generates a natural language summary of key trends, outliers, and patterns, with the data processed within Azure OpenAI's managed boundary without leaving Microsoft's infrastructure.
- Copilot respects **semantic model RLS** — the data passed to Copilot for summarisation is filtered by the user's row-level security policies before being sent to the LLM; a user who cannot see certain rows in a report also cannot have those rows summarised by Copilot. See [[Cloud-Platforms/Fabric/04-Power-BI#Semantic Models]] for RLS configuration.

**Common Misconceptions:**
- Copilot does not have access to the underlying raw data tables in OneLake — it only receives semantic model metadata and the results of DAX queries on data that the current user is authorised to see; it cannot generate queries that bypass semantic model filters or RLS.
- "Copilot generates perfect DAX" is an overstatement — Copilot-generated DAX should be reviewed and tested; the LLM can produce syntactically valid but semantically incorrect measures, particularly for complex time intelligence, semi-additive measures, or context-dependent calculations; generated code is a starting point, not a production-ready measure.

**Interview Answer Skeleton:**
- **What it is:** An Azure OpenAI-powered assistant within Power BI that uses semantic model metadata as LLM context to generate DAX measures, build report pages, and summarise visual data — scoped to the user's authorised data view and processed within Microsoft's Azure OpenAI trust boundary.
- **Why it matters / trade-offs:** Accelerates report development for analysts without deep DAX expertise and enables non-technical stakeholders to get quick data insights; the trade-off is that LLM-generated DAX requires expert review before deployment, and Copilot's quality degrades significantly when semantic model metadata (table/column names, measure descriptions) is poorly documented.
- **Example or context:** A product analyst without DAX expertise needs a complex `[Rolling 90-Day Active Users]` measure — they describe it to Copilot in plain English, Copilot generates the DAX using `CALCULATE()`, `DATESINPERIOD()`, and `DISTINCTCOUNT()`, and a senior analyst reviews and approves it before publishing, saving several hours of research.

**Free Resources:**
- [Copilot in Power BI Documentation](https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-introduction) — capabilities, data privacy, configuration requirements, and best practices
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Copilot across Fabric experiences and Azure OpenAI integration architecture

---
