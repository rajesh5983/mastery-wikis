# Microsoft Fabric — Architecture

---

## OneLake Architecture

**Status:** ⬜ Not Started

**Definition:** OneLake is a single, unified data lake per Fabric tenant that automatically stores all data created by any Fabric experience (Lakehouse, Warehouse, KQL Database, etc.). All experiences share the same underlying storage, eliminating data silos between services without any data movement.

**Mental Model:** OneLake is the single filing cabinet for the entire Fabric office — all departments store files there automatically, so anyone with the right access can find data from any experience in one place.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/onelake/onelake-overview — Microsoft OneLake overview covering architecture and data access patterns

---

## Fabric Capacities

**Status:** ⬜ Not Started

**Definition:** Fabric uses a capacity-based billing model — you purchase an F-SKU capacity (F2, F4, F8, ..., F2048) that provides a pool of compute units (CUs) shared across all Fabric workloads in attached workspaces. Larger capacities provide more CUs and higher concurrency limits.

**Mental Model:** A Fabric capacity is like a shared electricity meter for an office building — all workloads draw from the same total pool, and a larger meter allows more simultaneous usage.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/enterprise/licenses — Microsoft Fabric licensing and capacity documentation

---

## Fabric Experiences

**Status:** ⬜ Not Started

**Definition:** Microsoft Fabric is structured around distinct "experiences" — Data Engineering (Lakehouse + Spark), Data Factory (pipelines and dataflows), Data Warehouse (T-SQL warehouse), Data Science (notebooks + ML), Real-Time Intelligence (event streams and KQL), and Power BI — all running on shared OneLake and compute.

**Mental Model:** Fabric experiences are rooms in one building — each room (experience) is specialised for a different task, but they all share the same foundation (OneLake) and electricity supply (capacity).

**Free Resources:** https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview — Microsoft Fabric overview covering all experiences and their relationships

---

## SaaS vs PaaS

**Status:** ⬜ Not Started

**Definition:** Microsoft Fabric is a fully managed SaaS platform — you do not configure, patch, or scale infrastructure; Microsoft manages everything. This contrasts with Azure PaaS services (Azure Synapse, Azure Data Factory) where you manage individual service instances and their scaling configurations.

**Mental Model:** SaaS Fabric is like subscribing to Office 365 — Microsoft runs the software, you use it. Azure PaaS is like renting office space — you manage the equipment and infrastructure inside.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview — Fabric overview explaining the SaaS model and what is managed vs configurable

---

## Microsoft 365 Integration

**Status:** ⬜ Not Started

**Definition:** Fabric integrates natively with Microsoft 365 — data from SharePoint, Teams, Exchange, and Dynamics 365 is accessible via OneLake shortcuts and Microsoft Graph connectors. Power BI semantic models can be shared to Teams channels, and Fabric workspaces map to M365 groups.

**Mental Model:** Fabric is the data backbone of the Microsoft 365 ecosystem — it connects the analytical workloads to the collaboration tools, so the spreadsheet in Teams and the warehouse query in Fabric share the same governed data universe.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview — Microsoft Fabric documentation covering M365 integration patterns

---

## Tenant Settings

**Status:** ⬜ Not Started

**Definition:** Fabric Tenant Settings are admin controls configured by the Fabric Administrator for the entire organisation — enabling/disabling capabilities (who can create workspaces, which AI features are on), setting default capacity assignments, and controlling external sharing and export permissions.

**Mental Model:** Tenant settings are the master policy document for the entire Fabric deployment — the administrator sets the rules for what is allowed, and all workspaces operate within those boundaries.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/admin/tenant-settings-index — Microsoft Fabric tenant settings reference documentation
