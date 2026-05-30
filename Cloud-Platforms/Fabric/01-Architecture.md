# Microsoft Fabric — Architecture

---

## OneLake Architecture

**Status:** ⬜ Not Started

**Definition:** OneLake is a single, unified data lake per Fabric tenant that automatically stores all data created by any Fabric experience (Lakehouse, Warehouse, KQL Database, etc.). All experiences share the same underlying ADLS Gen2 storage, eliminating data silos between services without any data movement or copying.

**Key Mental Model:** OneLake is the single filing cabinet for the entire Fabric office — all departments store files there automatically, so anyone with the right access can find data from any experience in one place.

**How It Works:**
- OneLake is built directly on **Azure Data Lake Storage Gen2** (ADLS Gen2) using a hierarchical namespace within a single Azure subscription managed by Microsoft; each Fabric tenant gets one OneLake instance, and each workspace within that tenant has its own folder path under `onelake.dfs.fabric.microsoft.com/<workspace-id>/`.
- Data written by any Fabric experience (Lakehouse tables, Warehouse tables, KQL ingestion, EventHouse streams) is stored as Delta-Parquet files in OneLake's ADLS Gen2 backing store — the format is the same regardless of which experience wrote it, enabling any experience to read another's data without conversion.
- **Shortcuts** allow OneLake to reference data stored outside Fabric (Azure Blob Storage, ADLS Gen2, S3, GCS) or in another Fabric workspace as virtual paths within OneLake — the data stays in the source location but appears as a native OneLake path to all Fabric experiences, without copying.
- The OneLake endpoint `onelake.dfs.fabric.microsoft.com` is compatible with the ADLS Gen2 REST API and Azure Storage SDK, meaning existing tools that can read ADLS Gen2 (Azure Data Factory, Azure Databricks, Power BI) can connect to OneLake using the same client libraries with just a URL change.
- All OneLake access is subject to Fabric workspace permissions and OneLake ACLs managed through the Fabric Admin Portal — permissions cascade from workspace-level roles (Admin, Member, Contributor, Viewer) down to individual items and files. See [[Cloud-Platforms/Fabric/05-Administration]] for permission and capacity administration.

**Common Misconceptions:**
- OneLake is not a separate storage product that Microsoft bills for independently — storage costs in Fabric are billed as standard ADLS Gen2 storage rates in the background; the OneLake abstraction is a governance and access layer on top of that storage, not a distinct storage engine.
- "All experiences share the same data" does not mean all experiences can query each other's tables directly without configuration — a Warehouse cannot query a Lakehouse Delta table without setting up a Lakehouse connection or shortcut within the Warehouse; sharing requires explicit object references even though the underlying storage is the same.

**Interview Answer Skeleton:**
- **What it is:** A tenant-wide ADLS Gen2-backed storage layer that provides a unified hierarchical namespace for all Fabric experiences, with shortcut references to external storage, OneLake-compatible API access, and workspace-level ACL governance.
- **Why it matters / trade-offs:** Eliminates the multi-storage-account sprawl of Azure PaaS architectures where each service (Synapse, ADF, Data Lake) managed its own storage; the trade-off is that all Fabric workloads in a tenant share one logical lake, requiring careful workspace and folder organisation to prevent governance complexity at scale.
- **Example or context:** A retail organisation uses Fabric Lakehouse for data engineering, Fabric Warehouse for T-SQL reporting, and Power BI for dashboards — all three read from the same Delta files in OneLake with no ETL between experiences, and a shortcut to S3 makes raw supplier data appear as a native OneLake path.

**Free Resources:**
- [OneLake Overview](https://learn.microsoft.com/en-us/fabric/onelake/onelake-overview) — OneLake architecture, shortcuts, API compatibility, and tenant structure
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — comprehensive Fabric platform documentation including OneLake data access patterns

---

## Fabric Capacities

**Status:** ⬜ Not Started

**Definition:** Fabric uses a capacity-based billing model — you purchase an F-SKU capacity (F2, F4, F8, up to F2048) that provides a pool of compute units (CUs) shared across all Fabric workloads in attached workspaces. Larger capacities provide more CUs, higher concurrency, and higher limits for individual workload operations.

**Key Mental Model:** A Fabric capacity is like a shared electricity meter for an office building — all workloads draw from the same total pool, and a larger meter allows more simultaneous usage.

**How It Works:**
- A capacity is a **reserved compute pool** represented as a set of CUs (Capacity Units) purchased from Microsoft on a per-second or reserved billing basis; workspaces are attached to a specific capacity, and all operations in those workspaces consume CUs from that capacity's pool.
- **Smoothing and bursting**: Fabric allows workloads to burst beyond the capacity's CU allocation for short periods; the platform tracks a rolling 24-hour consumption window and applies **throttling** if the cumulative consumption exceeds the capacity's allocation over that window — burst enables handling of peak workloads without purchasing peak capacity.
- When a workload (Spark job, Warehouse query, Pipeline activity) runs, the Fabric orchestration layer allocates CUs from the capacity pool for the duration of the operation — concurrent workloads compete for the same CU pool, so a large Spark job can starve smaller queries if capacity is insufficient.
- Fabric capacities can be **paused** (stopping CU consumption and billing while retaining all OneLake data) and **resumed** in minutes, similar to pausing a cloud VM — this is a cost management lever for development capacities that are not needed overnight.
- The Capacity Metrics app (a Power BI report available to capacity admins) shows CU consumption by operation, workload type, and workspace over time, enabling capacity right-sizing decisions and workload isolation analysis. See [[Cloud-Platforms/Fabric/05-Administration]] for capacity administration and throttling management.

**Common Misconceptions:**
- Purchasing a larger F-SKU does not guarantee faster individual queries — CUs represent throughput capacity for concurrent workloads, not single-query acceleration; a single query may not run faster on F64 vs F8 if it is not compute-bound.
- Capacity units and Power BI Premium Per-User (PPU) are different licensing models — Fabric capacities (F-SKUs) replace Power BI Premium Per-Capacity (P-SKUs) for Fabric features; PPU is a per-seat license that does not grant access to Fabric engineering experiences like Lakehouse or Warehouse.

**Interview Answer Skeleton:**
- **What it is:** A reserved pool of compute units (CUs) purchased as an F-SKU that all attached workspace workloads share, with burst-and-smooth throttling to handle peak demand and a Capacity Metrics app for monitoring utilisation and right-sizing.
- **Why it matters / trade-offs:** Simplifies procurement to a single capacity purchase covering all Fabric experiences; the trade-off is shared-pool resource contention — a runaway Spark job or large pipeline can consume all available CUs and throttle concurrent BI queries, requiring workload isolation through multiple capacities for production environments.
- **Example or context:** A company runs daily Spark data engineering jobs and concurrent Power BI report queries on the same F64 capacity — during morning ETL runs, analysts notice dashboard slowdowns; the platform team reviews the Capacity Metrics app and separates engineering workloads to a second capacity to guarantee BI query responsiveness.

**Free Resources:**
- [Fabric Licensing and Capacity](https://learn.microsoft.com/en-us/fabric/enterprise/licenses) — F-SKU tiers, CU allocation, and billing model
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — capacity metrics, throttling behaviour, and capacity management guidance

---

## Fabric Experiences

**Status:** ⬜ Not Started

**Definition:** Microsoft Fabric is structured around distinct "experiences" — Data Engineering (Lakehouse + Spark), Data Factory (pipelines and dataflows), Data Warehouse (T-SQL warehouse), Data Science (notebooks + ML), Real-Time Intelligence (event streams and KQL), and Power BI — all running on shared OneLake storage and a shared capacity CU pool.

**Key Mental Model:** Fabric experiences are rooms in one building — each room (experience) is specialised for a different task, but they all share the same foundation (OneLake) and electricity supply (capacity).

**How It Works:**
- Each Fabric experience is a distinct UI and compute layer within the Fabric SaaS platform; the **Data Engineering** experience provides a Spark runtime with Lakehouse items; the **Data Warehouse** experience provides a distributed T-SQL engine; **Power BI** provides the VertiPaq analytical engine — each is a separate compute pathway over the same OneLake storage.
- **Cross-experience data access** works because all Fabric experiences produce Delta-Parquet tables in OneLake — a Spark notebook in Data Engineering can read a table written by the Data Warehouse, and a Power BI semantic model can connect directly to a Lakehouse Delta table via Direct Lake mode without any ETL layer.
- The **Data Factory** experience in Fabric is a re-implementation of Azure Data Factory's pipeline and dataflow concepts within the Fabric SaaS model — pipelines use the same canvas UI but run on Fabric-managed compute using capacity CUs rather than separately provisioned ADF Integration Runtime instances.
- **Real-Time Intelligence** provides the KQL (Kusto Query Language) experience for high-throughput event stream ingestion and time-series analytics via EventHouse — KQL databases also store data in OneLake, making ingested event data queryable by other experiences immediately after ingestion.
- Experiences share **workspace-level governance**: a Fabric workspace contains items from multiple experiences (a Lakehouse, a Warehouse, a Pipeline, a Power BI report) all governed by the same workspace role assignments, eliminating the need for separate permission management per service. See [[Cloud-Platforms/Fabric/01-Architecture#OneLake Architecture]] for the shared storage layer.

**Common Misconceptions:**
- Fabric experiences are not built on the same compute engine — Spark (Data Engineering), distributed T-SQL (Warehouse), VertiPaq (Power BI), and Kusto (Real-Time Intelligence) are all distinct compute engines; "unified" refers to the storage layer and governance model, not a single query engine.
- Fabric does not automatically route queries to the best experience — engineers must choose the right experience for each workload; running T-SQL analytical queries in a Lakehouse SQL Endpoint is not the same as the Warehouse experience, and performance characteristics differ significantly.

**Interview Answer Skeleton:**
- **What it is:** A set of specialised SaaS compute experiences (Spark, T-SQL, VertiPaq, KQL, ML) that each operate on shared OneLake Delta-Parquet storage and a shared capacity CU pool, unified under workspace-level governance.
- **Why it matters / trade-offs:** Eliminates the service proliferation of Azure PaaS (separate ADF, Synapse Analytics, ADLS, Power BI Premium instances) into a single platform with unified billing and governance; the trade-off is that shared CU pools mean experiences contend for compute, requiring careful capacity planning for mixed workloads.
- **Example or context:** A data team ingests streaming clickstream events via Real-Time Intelligence (KQL), transforms them into a Gold layer using Spark in Data Engineering (Lakehouse), exposes the results via the Data Warehouse T-SQL endpoint, and builds Power BI reports via Direct Lake — all four experiences read/write the same OneLake storage without data movement.

**Free Resources:**
- [Microsoft Fabric Overview](https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview) — all experiences, their capabilities, and cross-experience data access patterns
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — detailed documentation for each individual Fabric experience

---

## SaaS vs PaaS

**Status:** ⬜ Not Started

**Definition:** Microsoft Fabric is a fully managed SaaS platform — you do not configure, patch, or scale infrastructure; Microsoft manages everything including Spark runtime upgrades and storage scaling. This contrasts with Azure PaaS services (Azure Synapse, Azure Data Factory, ADLS Gen2) where engineers provision, configure, and scale individual service instances independently.

**Key Mental Model:** SaaS Fabric is like subscribing to Office 365 — Microsoft runs the software, you use it. Azure PaaS is like renting office space — you manage the equipment and infrastructure inside.

**How It Works:**
- In Fabric's SaaS model, Microsoft manages the **control plane** (Spark cluster provisioning, T-SQL engine scaling, storage replication, security patching, runtime upgrades) entirely — tenants interact only with the user-facing APIs and UIs; there are no VMs, subscriptions, or networking configurations to manage for compute.
- **Starter pools** for Spark are pre-warmed containers maintained by Microsoft — when a Spark notebook or Lakehouse job runs, Fabric assigns a pre-started Spark context from the pool within seconds rather than provisioning Spark clusters from scratch, which would take 3–5 minutes on Azure PaaS.
- The Fabric SaaS model means **updates are continuous and automatic** — Microsoft deploys new features, runtime versions, and performance improvements to all tenants without requiring version selection or manual upgrades, unlike Synapse Analytics where the Spark runtime version is customer-selected and must be manually updated.
- **Tenant isolation** in the SaaS model is logical rather than physical — all tenants run on shared Microsoft-managed infrastructure with logical security boundaries; enterprises requiring physical isolation (dedicated VMs, private VNets for all Spark compute) may find Fabric's SaaS model insufficiently isolated compared to Azure PaaS with fully isolated VNets.
- OneLake storage is always hosted in the customer's selected home region and optionally replicated to secondary regions, but the compute infrastructure can span Microsoft's regional pool — organisations with strict data residency requirements must verify that transient compute data does not cross regional boundaries. See [[Cloud-Platforms/Fabric/05-Administration]] for tenant-level admin controls.

**Common Misconceptions:**
- "SaaS means less capable" is false — Fabric's SaaS model provides full Spark, T-SQL, KQL, and Power BI capabilities equivalent to or exceeding their Azure PaaS counterparts; the trade-off is reduced infrastructure configuration flexibility, not reduced analytical capability.
- Fabric SaaS does not eliminate all operations work — engineers still manage capacity sizing, workspace governance, data pipeline logic, and performance tuning; SaaS eliminates infrastructure operations (VM patching, cluster provisioning), not data platform operations.

**Interview Answer Skeleton:**
- **What it is:** A fully managed SaaS analytics platform where Microsoft owns the compute infrastructure lifecycle (provisioning, scaling, patching, upgrades), and customers interact exclusively with the data platform layer — workspaces, items, capacities, and data.
- **Why it matters / trade-offs:** Dramatically reduces infrastructure operations burden compared to Azure PaaS and enables faster provisioning of new analytics capabilities; the trade-off is reduced control over runtime versions, network configuration, and compute isolation compared to building on Azure PaaS components.
- **Example or context:** A team migrates from Azure Synapse Analytics (manually managing Spark pool versions, Integration Runtime, and ADLS Gen2 account configurations) to Fabric — they no longer manage any infrastructure, Spark starter pools eliminate cluster cold-start latency, and OneLake replaces their multi-account ADLS setup.

**Free Resources:**
- [Microsoft Fabric Overview](https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview) — SaaS model explanation and comparison with Azure PaaS services
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — comprehensive SaaS platform documentation including capacity, governance, and experience guides

---

## Microsoft 365 Integration

**Status:** ⬜ Not Started

**Definition:** Fabric integrates natively with Microsoft 365 — data from SharePoint, Teams, Exchange, and Dynamics 365 is accessible via OneLake shortcuts and Microsoft Graph connectors. Power BI semantic models can be shared to Teams channels, and Fabric workspaces are backed by Microsoft 365 groups for identity and access management.

**Key Mental Model:** Fabric is the data backbone of the Microsoft 365 ecosystem — it connects the analytical workloads to the collaboration tools, so the spreadsheet in Teams and the warehouse query in Fabric share the same governed data universe.

**How It Works:**
- **Fabric workspaces** are associated with Microsoft Entra ID (formerly Azure AD) security groups — adding a user to the M365 group grants them access to the Fabric workspace, so Fabric workspace membership follows the same identity lifecycle as Microsoft 365 (user onboarding/offboarding, group-based access review).
- **Microsoft Graph connectors** can ingest content from SharePoint, Teams, Exchange, and Dynamics 365 into OneLake, making M365 content searchable and queryable via Fabric Lakehouse and Warehouse experiences — enabling analytical queries that combine M365 operational data with enterprise data.
- Power BI reports embedded in **Teams tabs** render using the user's Power BI identity, respecting the same row-level security policies as the standalone Power BI service — analysts can view live data in their Teams workspace without leaving the collaboration tool.
- **OneLake Shortcuts to SharePoint** allow SharePoint document library files (CSVs, Parquet files) to appear as native OneLake folders queryable by Spark notebooks, without copying data out of SharePoint into a separate storage account.
- Fabric's **Copilot** features (in Power BI, Data Factory, and Data Science) are powered by Azure OpenAI Service within the Microsoft trust boundary — Copilot uses the user's M365 identity context to respect data access policies when generating insights or writing DAX/SQL. See [[Cloud-Platforms/Fabric/04-Power-BI]] for Power BI Copilot integration details.

**Common Misconceptions:**
- Fabric workspace access via M365 groups does not automatically grant access to individual OneLake items within the workspace — workspace-level roles control what actions a member can perform, but item-level permissions (row-level security in Power BI, table-level grants in Warehouse) must be configured separately.
- The integration with Teams does not make Teams a Fabric authoring environment — Teams is a consumption and sharing surface; data engineering, pipeline building, and model development happen in the Fabric portal, with results published to Teams for collaboration.

**Interview Answer Skeleton:**
- **What it is:** A deep integration layer connecting Fabric's data platform to Microsoft 365 through Entra ID group-based workspace access, Graph connector data ingestion, Power BI embedding in Teams, and OneLake shortcuts to SharePoint content.
- **Why it matters / trade-offs:** Enables organisations already on M365 to extend their identity, collaboration, and content management systems into the analytics platform without separate tooling; the trade-off is that non-Microsoft tooling (Slack, Google Workspace) has no native equivalent integration.
- **Example or context:** A consulting firm uses Teams as its primary collaboration tool — analysts view Power BI dashboards in Teams tabs (respecting RLS from the semantic model), data engineers query SharePoint-stored CSV files via OneLake shortcuts, and new employee Fabric access is provisioned automatically when they're added to the project's M365 group.

**Free Resources:**
- [Microsoft Fabric Overview](https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview) — M365 integration overview and workspace identity model
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — OneLake shortcuts, Graph connectors, and Teams embedding documentation

---

## Tenant Settings

**Status:** ⬜ Not Started

**Definition:** Fabric Tenant Settings are admin controls configured by the Fabric Administrator for the entire organisation — enabling or disabling platform capabilities (who can create workspaces, which AI/Copilot features are active), setting default capacity assignments, and controlling external sharing and data export permissions across all workspaces.

**Key Mental Model:** Tenant settings are the master policy document for the entire Fabric deployment — the administrator sets the rules for what is allowed, and all workspaces operate within those boundaries.

**How It Works:**
- Tenant settings are configured in the **Fabric Admin Portal** (`app.fabric.microsoft.com/admin`) by users assigned the Fabric Administrator role in Microsoft Entra ID; settings take effect immediately across all workspaces in the tenant without requiring workspace-level changes.
- Each tenant setting has a **scope**: settings can apply to the entire organisation, or be limited to specific security groups (e.g., "enable Copilot for the data-analytics group only") — this allows staged rollouts of new features without enabling them for all users simultaneously.
- **Workspace creation** settings control who can create new Fabric workspaces — by default all users can create workspaces, but this can be restricted to specific groups, preventing ungoverned workspace proliferation in large enterprises.
- **External sharing settings** control whether users can share Power BI reports, Fabric items, or OneLake data with external guest users (B2B sharing via Microsoft Entra B2B) — these are the primary controls for preventing accidental data exposure outside the organisation.
- **Export controls** (e.g., disable Excel export, disable PDF export, restrict live connection exports) prevent data exfiltration from Power BI reports to unmanaged endpoints — these settings are enforced at the Power BI service level before any export action is executed. See [[Cloud-Platforms/Fabric/05-Administration]] for capacity and workspace administration beyond tenant settings.

**Common Misconceptions:**
- Tenant settings do not override workspace or item-level permissions — they define the boundaries of what is *possible* at the platform level; workspace admins still control who has access within those boundaries. A tenant setting that disables external sharing prevents the option entirely, but does not grant additional access.
- Not all Fabric features are controlled by Tenant Settings — some features are enabled or disabled per capacity (via Capacity Settings in the Admin Portal), not per tenant; engineers troubleshooting missing features should check both Tenant Settings and Capacity Settings.

**Interview Answer Skeleton:**
- **What it is:** Organisation-wide platform configuration controls in the Fabric Admin Portal that gate feature availability (Copilot, workspace creation, AI features), external sharing, and export permissions, with security-group-based scoping for staged rollouts.
- **Why it matters / trade-offs:** Provides centralised governance for the entire Fabric deployment, enabling consistent policies across all workspaces; the trade-off is that overly restrictive tenant settings require admin intervention for legitimate use cases, creating bottlenecks if the governance model is too rigid.
- **Example or context:** A regulated financial firm disables all data export options and external sharing in Tenant Settings, then creates a specific security group for their certified BI team with export enabled — sensitive data cannot be extracted by regular users, while the reporting team retains necessary export capabilities for regulatory deliverables.

**Free Resources:**
- [Fabric Tenant Settings Reference](https://learn.microsoft.com/en-us/fabric/admin/tenant-settings-index) — comprehensive listing of all tenant settings with descriptions and impact
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — admin portal guide covering tenant settings, capacity settings, and governance controls

---
