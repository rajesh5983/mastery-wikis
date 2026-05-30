# Microsoft Fabric — Administration

---

## Capacity Management

**Status:** ⬜ Not Started

**Definition:** Fabric Capacity Management involves monitoring CU (compute unit) consumption across all workloads, identifying which workspaces and experiences are consuming most capacity, and using throttling controls to prevent burst workloads from affecting other users on the shared capacity.

**Mental Model:** Capacity management is balancing the office electricity budget — monitoring who's using the most power, setting limits for heavy consumers, and ensuring the lights don't go out for everyone when one person runs a server farm.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/enterprise/metrics-app — Microsoft Fabric Capacity Metrics App documentation for monitoring and optimisation

---

## Workspace Governance

**Status:** ⬜ Not Started

**Definition:** Fabric Workspace governance covers workspace creation policies (who can create workspaces), workspace roles (Admin, Member, Contributor, Viewer), item permissions, and linking workspaces to Microsoft 365 groups for identity management. Workspace roles control access to all items within the workspace.

**Mental Model:** Workspace governance is the access management system for the Fabric environment — who can create spaces, who can edit content in each space, and how access is linked to the organisation's Active Directory groups.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/fundamentals/roles-workspaces — Microsoft Fabric workspace roles and governance documentation

---

## OneLake Security

**Status:** ⬜ Not Started

**Definition:** OneLake security provides POSIX-style ACLs for folders and files in OneLake, workspace-level access for Fabric items, and row-level security within semantic models. Data access can be controlled at the storage level (OneLake ACLs), the item level (Fabric permissions), or the query level (RLS in semantic models).

**Mental Model:** OneLake security has three layers — the storage perimeter (ACLs on folders), the Fabric item gates (workspace permissions), and the query filters (RLS/CLS). Each layer adds a check, and all must be passed to see data.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/security/security-overview — Microsoft Fabric security overview covering all protection layers

---

## Purview Integration

**Status:** ⬜ Not Started

**Definition:** Microsoft Purview integrates with Fabric to provide enterprise data governance — automated data discovery and cataloguing of Fabric items, data classification and sensitivity labelling, lineage tracking, and compliance reporting. Sensitivity labels applied in Purview propagate to Power BI reports and exports.

**Mental Model:** Purview is the compliance officer's view of Fabric — it automatically discovers what data exists, classifies it, and ensures that sensitive data is labelled and tracked wherever it moves.

**Free Resources:** https://learn.microsoft.com/en-us/purview/purview-portal — Microsoft Purview documentation covering Fabric integration and data governance features

---

## Fabric Admin Portal

**Status:** ⬜ Not Started

**Definition:** The Fabric Admin Portal is the central administration hub for Fabric tenant settings — capacity management, workspace auditing, tenant-wide feature controls (enabling/disabling AI features, external sharing, export capabilities), and monitoring the Fabric usage metrics across the organisation.

**Mental Model:** The Fabric Admin Portal is the control panel for the entire Fabric environment — tenant-wide switches, the capacity health dashboard, and the audit log that shows who did what across all workspaces.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/admin/admin-overview — Microsoft Fabric Admin Portal documentation covering tenant settings and capacity management

---

## Domain Management

**Status:** ⬜ Not Started

**Definition:** Domains in Fabric are logical groupings of workspaces by business unit or subject area (e.g., Finance, Marketing, HR), with designated domain admins who can manage settings for their domain's workspaces. Domains enable federated data governance — central IT sets guardrails, business units manage within their domain.

**Mental Model:** Domain management is a franchise model for governance — corporate IT sets the brand standards and non-negotiable rules, but each franchise owner (domain admin) manages their location independently within those rules.

**Free Resources:** https://learn.microsoft.com/en-us/fabric/governance/domains — Microsoft Fabric domain management documentation covering setup and federated governance
