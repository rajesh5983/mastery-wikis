# Microsoft Fabric — Administration

---

## Capacity Management

**Status:** ⬜ Not Started

**Definition:** Fabric Capacity Management involves monitoring compute unit (CU) consumption across all workloads in a capacity, identifying which workspaces and experiences drive the most utilisation, and using throttling and smoothing controls to prevent burst workloads from degrading performance for other users sharing the same capacity pool.

**Key Mental Model:** Capacity management is balancing the office electricity budget — monitoring who's consuming the most power, setting controls for heavy users, and ensuring the lights don't go out for everyone when one workload runs a surge job.

**How It Works:**
- Each Fabric capacity is backed by a fixed pool of CUs (compute units) allocated to the capacity's SKU tier. When a Fabric experience (Spark job, Warehouse query, Pipeline run) executes, it draws CUs from this pool. The Capacity Metrics App (a pre-built Power BI report) shows real-time and historical CU consumption broken down by workspace, experience, and operation type.
- **Throttling** activates automatically when cumulative CU consumption exceeds the capacity's smoothing window limit. Fabric applies a 10-minute and 24-hour smoothing window — burst usage above the throttling threshold is allowed briefly but triggers rejection or queuing for subsequent requests until consumption falls back within bounds.
- **Autoscale** (available on F64+ SKUs) allows temporary CU bursting above the purchased tier for short durations, billed as pay-as-you-go for the overage period. This prevents hard throttling for rare peak events without permanently over-provisioning the capacity tier.
- The Capacity Metrics App provides a `CU % used` time-series chart that shows throttling events (coloured bands when usage exceeds 100%). Post-incident analysis uses this chart to identify which workspace, experience, and time window caused the throttling, guiding right-sizing decisions.
- **Workspace capacity assignment** determines which capacity a workspace draws from. Workspaces can be reassigned between capacities at runtime — providing an operational mechanism to isolate high-CU workloads by moving them to a dedicated capacity during peak periods. See [[Cloud-Platforms/Fabric/05-Administration#Domain Management]] for organising workspaces at scale.

**Common Misconceptions:**
- CU throttling means the capacity is permanently undersized — Fabric's smoothing window means that brief burst spikes trigger throttling even on appropriately sized capacities; investigate whether the workload is truly sustained (requiring a larger SKU) or just spiky (requiring scheduling or query optimisation).
- All Fabric experiences consume CUs equally — Spark jobs, Power BI refreshes, Warehouse queries, and Pipeline activities have different CU consumption profiles and throttling characteristics; the Capacity Metrics App breaks down consumption by experience type to isolate the actual contributor.

**Interview Answer Skeleton:**
- **What it is:** The practice of monitoring shared CU pool utilisation using the Capacity Metrics App, controlling throttling behaviour, and making SKU or workload scheduling decisions to balance cost against performance SLAs.
- **Why it matters / trade-offs:** Fabric's shared capacity model means one runaway workload affects all other workspaces; capacity management prevents contention without requiring dedicated compute per workload. The trade-off is that shared capacity introduces non-deterministic latency — a workspace that ran a report in 30 seconds at low utilisation may take 3 minutes during throttling from another workspace's heavy load.
- **Example or context:** A team's daily 8am Power BI dataset refresh competes with a scheduled Spark pipeline that runs at the same time — the Capacity Metrics App shows throttling at 8:05am. The fix is to stagger the Spark pipeline start time to 8:30am, ensuring the refresh completes within its CU budget before the Spark job draws from the same pool.

**Free Resources:**
- [Microsoft Fabric Capacity Metrics App](https://learn.microsoft.com/en-us/fabric/enterprise/metrics-app) — official documentation for the Capacity Metrics App, throttling behaviour, and smoothing window mechanics
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — capacity planning guidance, SKU tier comparison, and autoscale configuration reference

---

## Workspace Governance

**Status:** ⬜ Not Started

**Definition:** Fabric workspace governance defines who can create workspaces (controlled via tenant settings), the role-based permission model within each workspace (Admin, Member, Contributor, Viewer), item-level permissions for sharing specific Fabric items, and how workspaces are linked to Microsoft 365 groups for identity and licence management.

**Key Mental Model:** Workspace governance is the access management system for the Fabric environment — it defines who can create new projects, who can modify content within a project, and how those access decisions are connected to the organisation's existing Microsoft 365 identity infrastructure.

**How It Works:**
- Workspace roles form a four-tier hierarchy: **Admin** (full control, can add/remove members and delete the workspace), **Member** (can create, edit, and share items, and add Contributors), **Contributor** (can create and edit items but not share them outside the workspace), **Viewer** (read-only access to items and their outputs).
- Role assignments can be made to individual users, security groups, or Microsoft 365 groups (formerly Office 365 groups). Using security groups rather than individual users is the governance best practice — role changes are made in Azure AD/Entra ID, not in each workspace individually, reducing administrative overhead.
- **Item-level sharing** allows specific Fabric items (a single report, a specific Lakehouse) to be shared with users who have no workspace role at all — granting read access without exposing the entire workspace. Item sharing is governed separately from workspace roles and appears in the item's "Share" dialog.
- The **workspace creation policy** (set in the Tenant Admin Portal) controls who can create new workspaces: all users, specific security groups, or no users (admin only). Restricting workspace creation prevents sprawl but creates a central IT bottleneck; a balanced approach is to allow creation by a designated set of data stewards per business unit.
- **Microsoft 365 group-linked workspaces** (classic workspace model) inherit membership from the M365 group and create a SharePoint site and Teams channel alongside the Fabric workspace. The current model (v2 workspaces) decouples Fabric workspaces from M365 groups, giving more granular control. See [[Cloud-Platforms/Fabric/05-Administration#Domain Management]] for organising workspaces into domains.

**Common Misconceptions:**
- Workspace Admin is the same as Fabric Tenant Admin — Workspace Admin is a per-workspace role with no authority outside that workspace; Fabric Tenant Admin is an organisation-wide role in the Microsoft 365 Admin Center with control over all tenants settings.
- Sharing a Power BI report with "Viewer" access is the same as giving workspace Viewer role — workspace Viewer role grants read access to all items in the workspace; item-level sharing grants read access only to the shared item, with no visibility into other workspace content.

**Interview Answer Skeleton:**
- **What it is:** The four-tier workspace role model (Admin/Member/Contributor/Viewer) combined with item-level sharing and tenant-wide workspace creation policies, managed through security group assignments integrated with Microsoft Entra ID.
- **Why it matters / trade-offs:** Well-designed workspace governance enables self-service analytics while preventing data access sprawl; the trade-off is that granular item-level sharing becomes difficult to audit at scale — each item has its own sharing list, which requires a tool like Purview to centralise access visibility.
- **Example or context:** A Finance domain has 12 workspaces. Rather than adding individuals to each, the data team creates an "Finance-Fabric-Viewer" Entra ID security group, adds it as Viewer to all 12 workspaces, and manages Finance employee access through group membership in Entra ID — access is granted/revoked centrally in one place rather than in each workspace individually.

**Free Resources:**
- [Fabric Workspace Roles](https://learn.microsoft.com/en-us/fabric/fundamentals/roles-workspaces) — official documentation covering the role hierarchy, permissions per role, and item-level sharing mechanics
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — workspace governance best practices, tenant settings for workspace creation, and M365 group integration

---

## OneLake Security

**Status:** ⬜ Not Started

**Definition:** OneLake security operates at three levels: POSIX-style ACLs on folders and files within OneLake (storage-layer access control), workspace and item permissions in Fabric (experience-layer access control), and row-level and column-level security within semantic models and Warehouse (query-layer access control). All three layers can apply simultaneously to a single request.

**Key Mental Model:** OneLake security has three concentric gates — the storage fence (OneLake ACLs on folders/files), the Fabric item gates (workspace and item permissions), and the query filters (RLS/CLS inside semantic models or Warehouse policies). A request must pass all applicable gates to see data.

**How It Works:**
- OneLake ACLs are managed via the OneLake File Explorer or ADLS Gen2 APIs. They follow the POSIX model: each folder/file has an owner, group, and "other" entry, each with read (r), write (w), execute (x) permissions. ACL inheritance propagates from parent folders to child items when set as "default" ACLs — this is the mechanism for bulk access grants at the folder level.
- Fabric workspace and item permissions are managed through the Fabric portal and do not require ADLS Gen2 API access. Workspace roles automatically grant access to the underlying OneLake paths for items in that workspace — a workspace Contributor can write to Lakehouse tables without a separate OneLake ACL grant.
- **Row-Level Security (RLS)** in Power BI semantic models defines DAX filter rules per role (e.g., `[Region] = USERPRINCIPALNAME()`) that restrict which rows a user sees when querying the semantic model. RLS is evaluated after the workspace/item access check — the user must have access to the semantic model item first, then RLS restricts the data visible within it.
- **Column-Level Security (CLS)** in Fabric Warehouse restricts access to specific columns using `DENY SELECT` on column names within a T-SQL permission grant. Combined with object-level security (denying access to specific tables), CLS enables fine-grained query-level access control within the same Warehouse without duplicating data.
- **Sensitivity labels** applied via Microsoft Purview flow through OneLake items and are enforced during export (e.g., a "Confidential" label prevents exporting a Power BI report to CSV, or applies encryption to Excel exports). Labels are inherited from Lakehouse tables to downstream reports automatically when configured. See [[Cloud-Platforms/Fabric/05-Administration#Purview Integration]] for label propagation mechanics.

**Common Misconceptions:**
- OneLake ACLs and Fabric item permissions are the same thing — they operate at different layers and are managed separately; an OneLake ACL change does not automatically update Fabric item permissions, and vice versa. Both may need updating when changing access.
- RLS in semantic models prevents data from being queried via the Warehouse SQL endpoint — RLS is applied only at the semantic model layer; users who connect directly to the Warehouse via SQL bypass semantic model RLS entirely. Warehouse-level column and row security (T-SQL DENY) is needed to protect data at the SQL endpoint layer.

**Interview Answer Skeleton:**
- **What it is:** A layered security model combining POSIX-style ACLs on OneLake storage paths, Fabric workspace/item permission assignments, and query-layer RLS/CLS policies — each layer independently controllable but jointly enforced for any given data access request.
- **Why it matters / trade-offs:** Defence in depth — multiple independent layers prevent accidental data exposure when one layer is misconfigured. The trade-off is complexity: administrators must understand which layer governs which access pattern, and access issues require checking all three layers to diagnose.
- **Example or context:** A Lakehouse has customer PII data. The workspace Contributor role allows Spark pipelines to write to it. A Power BI report over the same Lakehouse uses a Direct Lake semantic model with RLS restricting each regional analyst to their own region's customer rows. A direct SQL connection to the Lakehouse SQL endpoint bypasses RLS — a separate Warehouse view with DENY on PII columns is created for SQL-based access.

**Free Resources:**
- [OneLake Security Overview](https://learn.microsoft.com/en-us/fabric/security/security-overview) — official documentation covering all three security layers, ACL mechanics, and integration with Purview sensitivity labels
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — RLS configuration for semantic models, column-level security in Warehouse, and OneLake ACL setup guides

---

## Purview Integration

**Status:** ⬜ Not Started

**Definition:** Microsoft Purview integrates with Fabric to provide enterprise data governance at the metadata layer: automated discovery and cataloguing of all Fabric items (Lakehouses, Warehouses, reports, datasets), data classification with built-in and custom sensitive information types, sensitivity label propagation from source data through to downstream reports, and end-to-end data lineage tracing from ingestion source to Power BI visual.

**Key Mental Model:** Purview is the compliance officer's view of the entire Fabric environment — it automatically discovers what data assets exist across all workspaces, classifies sensitive content, tracks where it flows, and ensures that a "Highly Confidential" label on a Lakehouse table follows the data through all its downstream transformations.

**How It Works:**
- Purview's **Fabric scanner** (part of the Fabric Admin Portal's "Information Protection" settings) periodically scans all workspaces in the tenant, cataloguing every Lakehouse, Warehouse, Power BI dataset, report, and dataflow as a data asset in Purview's unified data map. The scan runs on a configurable schedule (default: weekly).
- **Sensitivity labels** are created and managed in Microsoft Purview (formerly Microsoft Information Protection). Labels are assigned to data assets either manually (by data owners via the Fabric portal or Purview) or automatically via auto-labelling policies based on detected sensitive information types (credit card numbers, national IDs, health data patterns detected by Purview's built-in classifiers).
- **Label inheritance** propagates sensitivity labels downstream: a label on a Lakehouse table is automatically applied to a Power BI semantic model built on that table, then to reports published from that model. Label inheritance is controlled by the tenant-level "Downstream content inheritance" setting and can be configured to apply labels automatically or to flag without auto-applying.
- **Data lineage** in Purview maps data flow relationships between Fabric items: a Dataflow Gen2 that reads from a Lakehouse and writes to a Warehouse is represented as a lineage edge. Lineage lets governance teams trace where a specific data element originated and where it is consumed, enabling impact analysis for schema changes.
- **Compliance reporting** in Purview generates reports on sensitivity label coverage, unlabelled assets, and access patterns — feeding audit requirements for GDPR, HIPAA, and SOC 2 without manual inventory work. See [[Cloud-Platforms/Fabric/05-Administration#OneLake Security]] for how Purview labels integrate with OneLake storage-layer security.

**Common Misconceptions:**
- Purview integration automatically secures data — Purview classifies and labels data and enforces label-based export restrictions (e.g., preventing unencrypted Excel exports of Confidential data); it does not replace OneLake ACLs or Fabric item permissions, which must be configured independently.
- Purview lineage covers all data flows in Fabric — lineage tracking in Purview captures Fabric-native data movements (Dataflows, Pipelines, Power BI refreshes) but does not automatically capture lineage for custom Spark code or external tools writing to OneLake unless lineage events are explicitly emitted via the Purview API.

**Interview Answer Skeleton:**
- **What it is:** Microsoft Purview's automated scanning, classification, sensitivity label application, and lineage tracking layer over all Fabric data assets — providing governance visibility and compliance enforcement at the tenant level without manual inventory work.
- **Why it matters / trade-offs:** Purview provides the audit trail and sensitivity enforcement that enterprises require for GDPR, HIPAA, and internal governance programmes. The trade-off is that full Purview integration requires Purview licences (separate from Fabric) and scanner configuration, and auto-labelling policies require careful calibration to avoid false-positive sensitivity classifications.
- **Example or context:** A healthcare organisation uses Fabric to process patient data. Purview's auto-labelling policy detects health information identifiers (ICD codes, patient IDs) in Lakehouse tables and applies a "Highly Confidential - Health Data" label automatically. The label propagates to downstream Power BI reports, preventing unapproved export of patient-level data to Excel — all without manual intervention on each individual asset.

**Free Resources:**
- [Microsoft Purview Documentation](https://learn.microsoft.com/en-us/purview/purview-portal) — Purview data map, sensitivity labels, auto-labelling policies, and Fabric integration setup
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — Purview integration configuration in Fabric Admin Portal, lineage mechanics, and label inheritance settings

---

## Fabric Admin Portal

**Status:** ⬜ Not Started

**Definition:** The Fabric Admin Portal is the central administration interface for the Fabric tenant — it controls tenant-wide feature enablement (AI features, external sharing, export formats), capacity assignments, usage metric monitoring, workspace auditing via the audit log, and governs data sharing policies across the entire organisation.

**Key Mental Model:** The Fabric Admin Portal is the master control panel for the entire Fabric deployment — tenant-wide on/off switches for every feature, the health dashboard for each capacity, and the audit trail that records every administrative and data access event across all workspaces.

**How It Works:**
- **Tenant settings** in the Admin Portal are the top-level governance layer: each setting controls a specific capability and can be scoped to "Enabled for the whole organisation," "Disabled for the whole organisation," or "Enabled/Disabled for specific security groups." Settings cover areas including AI features, R/Python visual execution, external data sharing, data export formats, service principal access, and third-party integration permissions.
- **Capacity management** within the Admin Portal assigns workspaces to capacities, allows capacity administrators to pause/resume capacities, configures autoscale settings, and shows real-time utilisation. Capacity admin is delegatable — a capacity can have its own admin without granting that admin access to the Tenant Admin Portal.
- The **audit log** records every significant event in Fabric: user logins, report views, dataset refreshes, workspace role changes, item shares, API calls, and capacity configuration changes. Audit logs are stored in Microsoft 365's audit system and queryable via the Microsoft Purview Audit search or the Office 365 Management APIs for programmatic export to SIEM systems.
- **Usage metrics** at the tenant level aggregate Power BI report views, refresh counts, and capacity utilisation trends for executive-level governance reporting. Workspace-level usage metrics are available separately and are accessible to workspace admins without requiring Tenant Admin Portal access.
- The Admin Portal API allows programmatic administration: listing workspaces, assigning workspaces to capacities, managing users, and configuring tenant settings via REST API. This enables Infrastructure-as-Code patterns for Fabric administration using tools like Terraform or custom Python scripts. See [[Cloud-Platforms/Fabric/05-Administration#Workspace Governance]] for workspace-level access controls governed via the Admin Portal.

**Common Misconceptions:**
- Tenant Admin access is needed to manage a capacity — capacity administrators can manage their own capacity (suspend, resume, view utilisation, assign workspaces) without Tenant Admin access; the Admin Portal gives Tenant Admins visibility across all capacities.
- Disabling a tenant setting affects existing content immediately — most tenant settings disable the creation of new instances of a feature but do not retroactively remove existing feature usage; disabling "Users can publish apps" prevents new app publishing but does not unpublish existing apps.

**Interview Answer Skeleton:**
- **What it is:** The tenant-level administration hub that controls Fabric's feature flags (tenant settings), assigns workspaces to capacities, provides audit log access, and governs external sharing and data export policies across the entire organisation.
- **Why it matters / trade-offs:** The Admin Portal is the enforcement point for organisational data governance and compliance policies in Fabric — without proper Admin Portal configuration, features like external sharing or AI integrations may be enabled beyond the organisation's risk tolerance. The trade-off is that tenant settings are coarse-grained (all-or-security-group); fine-grained per-workspace feature control is not supported.
- **Example or context:** A financial services organisation evaluating Fabric disables "Users can export data to Excel" for all users via tenant settings, and enables AI Copilot features only for the "Fabric-AI-Pilot" security group while the data classification and governance review is in progress — the Admin Portal makes both changes in two settings without any workspace-level configuration work.

**Free Resources:**
- [Fabric Admin Overview](https://learn.microsoft.com/en-us/fabric/admin/admin-overview) — tenant settings reference, admin portal navigation, audit log access, and capacity management documentation
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — admin portal API reference, tenant settings full index, and governance best practices for Fabric deployment

---

## Domain Management

**Status:** ⬜ Not Started

**Definition:** Domains in Microsoft Fabric are logical groupings of workspaces by business unit, subject area, or team (Finance, Marketing, HR, Data Platform). Each domain has designated domain admins who manage domain-level settings, including which workspaces belong to the domain, default sensitivity labels for the domain's data assets, and delegated governance responsibilities.

**Key Mental Model:** Domain management implements a federated governance model — the Fabric Tenant Admin sets the non-negotiable tenant-wide rules, and domain admins manage their domain's workspaces with autonomy within those boundaries. Like a franchise: corporate sets the brand standards, each franchisee runs their location.

**How It Works:**
- Domains are created in the Fabric Admin Portal by Tenant Admins. Each domain has a display name, description, and a list of domain admins (users or security groups). Domain admins can be added without granting them Tenant Admin access — domain admin is a delegated role scoped to that domain's workspaces.
- Workspaces are assigned to domains either by Tenant Admins or by domain admins (if the tenant setting allows domain admin workspace assignment). A workspace can belong to only one domain; reassignment moves all domain-level settings with it.
- **Domain-level default sensitivity labels** allow domain admins to set a default label applied to all new items created within the domain's workspaces — ensuring Finance domain items are automatically labelled "Confidential - Finance" without relying on individual data owners to apply labels manually.
- **Delegated tenant settings** allow Tenant Admins to grant domain admins the ability to configure specific tenant settings for their domain's workspaces — for example, allowing the Data Science domain admin to enable "Users can use R and Python visuals" for their workspaces without enabling it tenant-wide. This is the mechanism for controlled feature experimentation.
- Data portal views (upcoming in Fabric) aggregate all data assets within a domain into a discoverable domain-specific catalogue — allowing data consumers to browse Finance data assets independently of Marketing data assets, with the domain admin curating what is visible. See [[Cloud-Platforms/Fabric/05-Administration#Purview Integration]] for how Purview's data map integrates with Fabric domain organisation.

**Common Misconceptions:**
- Domain admins can access all workspace content in their domain — domain admin is an administrative role (add/remove workspaces from the domain, configure domain settings); it does not grant the domain admin data access to workspace content. A domain admin who is not a workspace member cannot read Lakehouse tables in their domain's workspaces.
- Domains replace workspace-level governance — domains add an organisational layer above workspaces; workspace roles (Admin, Member, Contributor, Viewer) remain the mechanism for controlling who can access and modify specific content. Domains govern organisational structure and delegated administration, not data-level access.

**Interview Answer Skeleton:**
- **What it is:** A Fabric organisational construct that groups workspaces by business domain, delegates administrative authority to domain admins, applies domain-level default sensitivity labels, and enables delegated tenant settings configuration — implementing federated data governance at the sub-tenant level.
- **Why it matters / trade-offs:** Domains enable large organisations to scale Fabric governance without centralising all administration in a single Tenant Admin team — business units own their data domain while IT retains tenant-wide guardrails. The trade-off is added complexity: a two-tier admin model (tenant + domain) requires clear boundary documentation and handoff processes to avoid governance gaps.
- **Example or context:** A 5,000-person retailer has separate Fabric domains for Finance, Supply Chain, Marketing, and Data Platform. Each domain has 2–3 domain admins from the business unit who manage workspace assignments and default labels for their area. The central IT Fabric Tenant Admin sets tenant-wide policies (no public sharing, no CSV export without Confidential label) and delegates everything else to domain admins — reducing Tenant Admin workload while keeping compliance guardrails in place.

**Free Resources:**
- [Fabric Domain Management](https://learn.microsoft.com/en-us/fabric/governance/domains) — official documentation on domain creation, workspace assignment, domain admin delegation, and delegated tenant settings
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric) — governance architecture for large-scale Fabric deployments including domain strategy and federated administration patterns
