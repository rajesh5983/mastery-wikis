# Databricks — Administration

---

## Unity Catalog Administration

**Status:** ⬜ Not Started

**Definition:** Unity Catalog administration covers metastore setup (one per region), workspace attachment, privilege granting (GRANT/REVOKE on catalogs, schemas, tables), data access configuration (storage credentials, external locations), and enabling data lineage and audit logging.

**Mental Model:** Unity Catalog administration is setting up the library's card catalog system — deciding who gets a library card (access), what sections they can enter (privileges), and installing the system that tracks who checked out what (lineage and audit).

**Free Resources:** https://docs.databricks.com/en/data-governance/unity-catalog/manage-privileges/index.html — Unity Catalog privilege management documentation

---

## Cluster Policies

**Status:** ⬜ Not Started

**Definition:** Cluster policies define rules that constrain what users can configure when creating clusters — limiting instance types, DBU spend, auto-termination settings, and required tags. Policies enforce cost governance and security standards without requiring manual approval for every cluster creation.

**Mental Model:** Cluster policies are the purchase policy for compute — employees can order from the menu (policy-approved configs), but cannot order the most expensive items without approval.

**Free Resources:** https://docs.databricks.com/en/admin/clusters/cluster-policies.html — Databricks cluster policies documentation

---

## Instance Pools

**Status:** ⬜ Not Started

**Definition:** Instance Pools maintain a set of idle, ready-to-use cloud VM instances that clusters can acquire immediately rather than waiting for cloud provider provisioning. This reduces cluster start times from 5–10 minutes to 30–60 seconds for clusters that draw from the pool.

**Mental Model:** Instance Pools are a waiting room of pre-warmed VMs — when a cluster needs to start, it takes VMs from the room instead of waiting for them to be delivered from the cloud provider.

**Free Resources:** https://docs.databricks.com/en/compute/pool-best-practices.html — Databricks instance pool best practices and configuration guide

---

## Budget Policies

**Status:** ⬜ Not Started

**Definition:** Budget Policies in Databricks attach spending limits to principals (users, service principals, groups) or workspaces, triggering alerts or enforcement actions when DBU consumption exceeds thresholds. This provides proactive cost governance rather than reviewing bills after overspending occurs.

**Mental Model:** Budget policies are the spending alerts on a corporate credit card — you set the limit, and when someone is approaching it, you get a notification before the bill arrives, not after.

**Free Resources:** https://docs.databricks.com/en/admin/account-settings/budgets.html — Databricks budget policies and account-level cost management documentation

---

## Databricks CLI and API

**Status:** ⬜ Not Started

**Definition:** The Databricks CLI provides command-line access to workspace administration, job management, cluster operations, and file system operations. The REST API enables programmatic management of all Databricks resources and is the foundation for CI/CD integration and infrastructure-as-code approaches.

**Mental Model:** The CLI and API are the administrator's remote control — instead of clicking through the UI, you script and automate everything from cluster creation to secret management to job deployment.

**Free Resources:** https://docs.databricks.com/en/dev-tools/cli/index.html — Databricks CLI documentation covering installation, authentication, and commands

---

## Workspace Security

**Status:** ⬜ Not Started

**Definition:** Databricks workspace security covers IP access lists (restrict access by IP range), private link (no traffic over public internet), customer-managed keys (encrypt workspace data with your own KMS keys), and network isolation (no internet egress from clusters by default in some configurations).

**Mental Model:** Workspace security is a series of concentric security rings — access lists at the perimeter, private networking inside, encryption at the data layer, and audit logs throughout.

**Free Resources:** https://docs.databricks.com/en/security/index.html — Databricks security documentation covering network, access, and encryption controls
