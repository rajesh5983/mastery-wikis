# Microsoft Fabric — Power BI

---

## Direct Lake Mode

**Status:** ⬜ Not Started

**Definition:** Direct Lake is a new Power BI connectivity mode that reads Delta Parquet files directly from OneLake — combining the speed of Import mode (in-memory columnar storage) with the freshness of DirectQuery (no scheduled refresh required). Data is loaded into memory on demand without an ETL import step.

**Mental Model:** Direct Lake is the best of both worlds for Power BI — data stays in OneLake (always fresh, like DirectQuery), but Power BI loads it into memory on demand (fast, like Import) without a scheduled copy.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/get-started/direct-lake-overview — Microsoft Direct Lake mode overview and architecture documentation

---

## Semantic Models

**Status:** ⬜ Not Started

**Definition:** A Semantic Model (formerly Power BI Dataset) defines the business logic layer on top of raw data — measures, calculated columns, relationships, hierarchies, and row-level security — that Power BI reports connect to. In Fabric, semantic models connect directly to Lakehouse and Warehouse tables via Direct Lake.

**Mental Model:** A semantic model is the translation layer between raw data and business language — it defines what "Revenue" means, how Customer relates to Order, and who is allowed to see which rows.

**Free Resources:** https://learn.microsoft.com/en-us/power-bi/connect-data/service-datasets-understand — Microsoft semantic model documentation covering measures, relationships, and RLS

---

## DAX

**Status:** ⬜ Not Started

**Definition:** DAX (Data Analysis Expressions) is the formula language used in Power BI, Analysis Services, and Fabric semantic models for defining measures, calculated columns, and custom tables. DAX is an expression language designed for tabular data with implicit row context and filter context semantics.

**Mental Model:** DAX is the formula language of Power BI — like Excel formulas but for data models. CALCULATE() is the engine that changes filter context; SUMX() and AVERAGEX() iterate row by row. Understanding context propagation is the key to mastery.

**Free Resources:** https://learn.microsoft.com/en-us/dax/ — Microsoft DAX reference documentation with function descriptions and examples

---

## Power BI Premium vs Fabric

**Status:** ⬜ Not Started

**Definition:** Power BI Premium (P-SKUs) was the previous enterprise tier of Power BI with dedicated capacity. Microsoft Fabric (F-SKUs) supersedes it, offering all Premium features plus all Fabric experiences (Lakehouse, Warehouse, Data Engineering, etc.) in a single capacity. P-SKUs are being retired in favour of F-SKUs.

**Mental Model:** Fabric F-SKUs are the upgrade from Premium — you get everything Power BI Premium offered, plus all the data engineering and ML capabilities of Fabric, under one capacity bill.

**Free Resources:** https://learn.microsoft.com/en-us/power-bi/enterprise/service-premium-faq — Microsoft Power BI Premium vs Fabric comparison and migration documentation

---

## Real-Time Intelligence

**Status:** ⬜ Not Started

**Definition:** Real-Time Intelligence in Fabric (formerly Synapse Real-Time Analytics) combines event streams (Eventstream for ingestion from Kafka, Event Hubs, IoT Hub), KQL Databases (Kusto for time-series and log analytics), and Real-Time Dashboards for near-zero latency analytics on streaming data.

**Mental Model:** Real-Time Intelligence is Fabric's streaming analytics stack — a conveyor belt of events (Eventstream), a specialised time-series storage (KQL Database), and a live dashboard that updates in seconds.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/real-time-intelligence/overview — Microsoft Fabric Real-Time Intelligence overview documentation

---

## Copilot in Power BI

**Status:** ⬜ Not Started

**Definition:** Copilot in Power BI uses LLMs to assist with report building — generating DAX measures from natural language descriptions, summarising report insights, creating report pages from prompts, and answering questions about data in conversational language. Available in Fabric-capacity workspaces.

**Mental Model:** Copilot is a Power BI assistant who speaks natural language — describe the measure you need, and it writes the DAX; describe the chart you want, and it builds the visual.

**Free Resources:** https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-introduction — Microsoft Copilot in Power BI documentation covering capabilities and usage
