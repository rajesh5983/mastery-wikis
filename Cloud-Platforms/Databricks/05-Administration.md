# Databricks — Administration

---

## Unity Catalog Administration

**Status:** ⬜ Not Started

**Definition:** Unity Catalog administration covers metastore setup (one per cloud region), workspace attachment, privilege granting (GRANT/REVOKE on catalogs, schemas, tables), data access configuration (storage credentials, external locations), and enabling data lineage and audit logging. It is the governance control plane for all Databricks workspaces in an account.

**Key Mental Model:** Unity Catalog administration is setting up the library's card catalog system — deciding who gets a library card (access), what sections they can enter (privileges), and installing the system that tracks who checked out what (lineage and audit).

**How It Works:**
- Unity Catalog uses a **three-level privilege hierarchy**: `CATALOG → SCHEMA → TABLE/VOLUME`. A privilege granted at the catalog level (e.g., `GRANT USE CATALOG`) is inherited by all schemas and tables within that catalog; more specific grants at lower levels can further restrict or expand access for sub-hierarchies.
- When a user executes a query, the Databricks Runtime enforces Unity Catalog privileges at the moment the query accesses each object — it checks the user's identity (resolved via SSO or PAT token), evaluates all applicable grants and denies, and raises an `PERMISSION_DENIED` error before any data is read if the check fails.
- **Storage Credentials** are cloud-managed identity references (IAM Role on AWS, Managed Identity on Azure) registered in Unity Catalog; **External Locations** map a storage credential to a specific cloud storage path (e.g., `s3://my-bucket/prod/`) — clusters access that path only via Unity Catalog, preventing direct credential bypass.
- **Audit logs** are written to a Unity Catalog-managed Delta table in the customer's storage account; every data access event, privilege change, and catalog operation is recorded with user identity, timestamp, and object reference — queryable via SQL for compliance reporting.
- Unity Catalog **lineage** is captured automatically at query execution time — the runtime instruments queries to record which input tables contributed to which output tables, with the graph stored in the Unity Catalog backend and visualisable in the Data Explorer UI. See [[Cloud-Platforms/Databricks/01-Architecture#Unity Catalog]] for the three-level namespace model.

**Common Misconceptions:**
- Unity Catalog metastores are regional, not global — a metastore in `us-east-1` cannot be shared with workspaces in `eu-west-1`; cross-region governance requires either Delta Sharing or a deliberate multi-metastore strategy with cross-catalog references.
- Adding a user to a workspace does not automatically grant them data access — workspace membership controls login; Unity Catalog GRANT statements must be explicitly issued to give access to specific catalogs, schemas, or tables.

**Interview Answer Skeleton:**
- **What it is:** The account-level governance administration layer for Databricks, managing a regional metastore's privilege hierarchy (catalog → schema → table), storage credential bindings, and automatic lineage and audit log capture for all workspace activity.
- **Why it matters / trade-offs:** Centralises governance across multiple workspaces that previously each had isolated Hive metastores; the trade-off is the upfront migration effort to move workspace-local tables into the Unity Catalog namespace and the operational overhead of managing external location and credential bindings.
- **Example or context:** A platform admin onboards a new analytics team — they create a catalog `analytics_prod`, grant `USE CATALOG` to the team's group, then grant `SELECT` on specific schemas; the audit log captures every table access, and lineage shows downstream reports derived from each source table.

**Free Resources:**
- [Unity Catalog Privilege Management](https://docs.databricks.com/en/data-governance/unity-catalog/manage-privileges/index.html) — GRANT/REVOKE syntax, inheritance model, and storage credential setup
- [Databricks Academy](https://academy.databricks.com) — free data governance courses covering Unity Catalog administration, external locations, and audit log configuration

---

## Cluster Policies

**Status:** ⬜ Not Started

**Definition:** Cluster policies define rules that constrain what users can configure when creating clusters — limiting instance types, DBU spend, auto-termination settings, and required tags. Policies enforce cost governance and security standards without requiring manual approval for every cluster creation request.

**Key Mental Model:** Cluster policies are the purchase policy for compute — employees can order from the menu (policy-approved configs), but cannot order the most expensive items without approval.

**How It Works:**
- A cluster policy is a JSON document that specifies **fixed values** (the user cannot override), **allowed values** (a restricted set of choices), or **range constraints** (min/max bounds) for every configurable cluster attribute — instance type, number of workers, autoscale bounds, runtime version, init scripts, and environment variables.
- Policies are **enforced at cluster creation time** by the Databricks control plane — when a user submits a `Create Cluster` request, the control plane validates all specified attributes against the assigned policy before provisioning any VMs; invalid configurations are rejected immediately with an error.
- Policies are assigned to **users, groups, or service principals** in the Unity Catalog IAM model; a user with no policy assigned cannot create All-Purpose clusters at all (if admins have disabled the default policy); multiple policies can be assigned to a single user, who then selects one at cluster creation.
- **Auto-termination enforcement** via policy is a key cost control: a policy can set `autotermination_minutes` as a fixed value, ensuring all user clusters self-terminate after a maximum idle period regardless of what the user configured — preventing "forgotten cluster" cost accumulation.
- Policies can require **cost attribution tags** (e.g., `CostCenter`, `Project`) as fixed or mandatory fields, ensuring all cluster spend is automatically tagged for chargeback reports pulled from Unity Catalog system tables. See [[Cloud-Platforms/Databricks/04-SQL-Analytics#Cost Controls]] for the broader cost governance model.

**Common Misconceptions:**
- Cluster policies do not apply to SQL Warehouses — SQL Warehouse governance uses a separate warehouse policy and size-restriction mechanism; cluster policies only govern All-Purpose and Job cluster creation.
- Policies do not retroactively modify existing clusters — a cluster created before a policy was applied or created by an admin remains as-is; the policy only constrains new cluster creation going forward.

**Interview Answer Skeleton:**
- **What it is:** JSON-based configuration constraints applied to cluster creation in the Databricks control plane that enforce fixed values, allowed value sets, or range limits on cluster attributes — validated before any VM is provisioned and assigned to users or groups via IAM.
- **Why it matters / trade-offs:** Prevents unconstrained compute spending and enforces security standards (approved runtime versions, required init scripts) without requiring human approval for every cluster; the trade-off is that overly restrictive policies can block legitimate large-scale workloads and require admin intervention to create exceptions.
- **Example or context:** A platform team deploys a `data-analyst-policy` that fixes instance type to `m5.xlarge`, caps workers at 4, requires 30-minute auto-termination, and mandates a `team` tag — analysts can create self-service clusters within these guardrails, and the monthly cost report automatically attributes every DBU to a team.

**Free Resources:**
- [Databricks Cluster Policies Documentation](https://docs.databricks.com/en/admin/clusters/cluster-policies.html) — policy JSON syntax, attribute types, and assignment reference
- [Databricks Academy](https://academy.databricks.com) — free administration courses covering cluster governance, policy design patterns, and cost controls

---

## Instance Pools

**Status:** ⬜ Not Started

**Definition:** Instance Pools maintain a set of idle, ready-to-use cloud VM instances that clusters can acquire immediately rather than waiting for cloud provider provisioning. This reduces cluster start times from 5–10 minutes to 30–60 seconds for clusters that draw from the pool, and enables use of spot/preemptible instances with managed fallback.

**Key Mental Model:** Instance Pools are a waiting room of pre-warmed VMs — when a cluster needs to start, it takes VMs from the room instead of waiting for them to be delivered from the cloud provider.

**How It Works:**
- When a pool is created, Databricks immediately requests the cloud provider to provision and hold the specified **minimum idle instances** — these VMs are started, registered with the Databricks control plane, and kept in an idle state (with Databricks agent installed but no Spark context running).
- When a cluster configured to use a pool is created, the Databricks control plane assigns idle pool VMs to the cluster instead of requesting new VMs from the cloud provider API — only the Spark context and cluster setup steps run, not the VM provisioning step, reducing startup from minutes to seconds.
- Pools support **spot/preemptible instances** with an on-demand fallback configuration: the pool attempts to acquire spot instances first (at 60–80% cost savings); if spot capacity is unavailable, it falls back to on-demand instances, providing cost efficiency without the risk of pipeline failures from spot interruption.
- The pool maintains VMs between cluster uses: when a cluster releases VMs back to the pool (on scale-down or termination), those VMs return to the idle pool rather than being terminated, so the next cluster can reuse them immediately — reducing both startup time and cloud provisioning API calls.
- **Maximum capacity** and **idle instance auto-termination timeout** are configurable: the pool terminates VMs that have been idle beyond the timeout, preventing indefinite cost accumulation from pool VMs that are never claimed. See [[Cloud-Platforms/Databricks/05-Administration#Cluster Policies]] for enforcing pool usage through policies.

**Common Misconceptions:**
- Instance Pools do not reduce costs by themselves — idle pool VMs are billed at the full VM rate even when no cluster is using them; cost savings come from faster job turnaround (less wall-clock time per job) and spot instance usage, not from holding idle VMs.
- A cluster using a pool is not faster than a non-pool cluster for compute execution — the pool only reduces startup latency; once running, cluster performance is identical to a non-pool cluster of the same instance type.

**Interview Answer Skeleton:**
- **What it is:** A pre-provisioned set of idle cloud VMs managed by the Databricks control plane that clusters can acquire instantly, bypassing the cloud provider's VM provisioning latency, with support for spot instances and configurable idle termination.
- **Why it matters / trade-offs:** Reduces cold-start latency for short-lived Job Clusters from 5–8 minutes to under 60 seconds, enabling tighter SLAs for automated pipelines; the trade-off is idle VM costs for pools that are maintained but infrequently claimed.
- **Example or context:** A data pipeline team has hourly Job Cluster runs — without a pool, each run spends 6 minutes waiting for VMs before Spark even starts; with an instance pool of 10 pre-warmed VMs, each run acquires VMs in 30 seconds and starts processing immediately, reducing total pipeline duration and shortening the window of VM cost per run.

**Free Resources:**
- [Databricks Instance Pool Best Practices](https://docs.databricks.com/en/compute/pool-best-practices.html) — pool configuration, spot fallback, and sizing guidance
- [Databricks Academy](https://academy.databricks.com) — free administration courses covering compute optimisation, pools, and cost management

---

## Budget Policies

**Status:** ⬜ Not Started

**Definition:** Budget Policies in Databricks attach spending limits to principals (users, service principals, groups) or workspaces, triggering alerts or enforcement actions when DBU consumption exceeds thresholds. This provides proactive cost governance at the account level rather than reviewing overspend after the billing cycle closes.

**Key Mental Model:** Budget policies are the spending alerts on a corporate credit card — you set the limit, and when someone is approaching it, you get a notification before the bill arrives, not after.

**How It Works:**
- Budget policies are configured at the **Databricks Account level** (not workspace level) through the Account Console or Account API; each budget defines a scope (specific workspaces, tags, or the entire account), a time period (monthly rolling or fixed date range), and a DBU or dollar threshold.
- When consumption crosses a configured percentage threshold (e.g., 80% and 100%), the Databricks Account service sends **email alerts** to configured recipients; for automated responses, the budget alert can trigger a webhook that calls an external system or a Databricks Workflow.
- Budget data is also available in Unity Catalog system tables (`system.billing.usage`) which record every DBU consumption event with timestamp, workspace, cluster, SKU, and associated tags — enabling custom budget dashboards with SQL queries rather than relying solely on the Account Console UI.
- **Tag-based budgets** aggregate spend across multiple workspaces by matching cluster tags — if all data engineering clusters are tagged `team=data-eng`, a budget policy can sum their spend across workspaces and alert when the engineering team collectively exceeds their monthly allocation.
- Budget policies interact with cluster policies — cluster policies enforce required cost-centre tags at creation time, ensuring every cluster is categorised for budget attribution; without required tags, budget aggregation by team or project is unreliable. See [[Cloud-Platforms/Databricks/05-Administration#Cluster Policies]] for tag enforcement at cluster creation.

**Common Misconceptions:**
- Budget policies are alerting mechanisms by default, not hard enforcement limits — reaching the budget threshold sends an alert but does not automatically stop running workloads; hard enforcement (blocking new cluster creation) requires additional automation built on top of the alert webhook.
- DBU-based budgets are not equivalent to dollar budgets — DBU rates vary by SKU (Jobs, All-Purpose, SQL, Model Serving all have different DBU costs), so a DBU budget threshold does not directly translate to a predictable dollar amount without knowing the workload mix.

**Interview Answer Skeleton:**
- **What it is:** Account-level consumption thresholds tied to workspaces, principals, or tag-based scopes that generate alerts when DBU spending crosses configured percentages, with spend data queryable via Unity Catalog system tables for custom reporting.
- **Why it matters / trade-offs:** Provides visibility into cost trends before billing surprises and enables chargeback reporting; the trade-off is that default budget policies are alerting-only, not enforcement — automated hard limits require webhook-based automation or cluster policy restrictions.
- **Example or context:** A platform team sets a monthly budget policy on each business unit's tag — when the `team=marketing` tag group approaches 80% of its DBU budget mid-month, an alert fires and a linked Databricks Workflow automatically applies a more restrictive cluster policy to that group for the rest of the month.

**Free Resources:**
- [Databricks Budget Policies Documentation](https://docs.databricks.com/en/admin/account-settings/budgets.html) — budget configuration, alert thresholds, and account-level cost management
- [Databricks Academy](https://academy.databricks.com) — free administration courses covering account-level cost governance and system table reporting

---

## Databricks CLI and API

**Status:** ⬜ Not Started

**Definition:** The Databricks CLI provides command-line access to workspace administration, job management, cluster operations, secret management, and file system operations. The REST API enables programmatic management of all Databricks resources and is the foundation for CI/CD integration, Terraform-based infrastructure-as-code, and automated deployment pipelines.

**Key Mental Model:** The CLI and API are the administrator's remote control — instead of clicking through the UI, you script and automate everything from cluster creation to secret management to job deployment.

**How It Works:**
- The Databricks CLI (v0.200+) is built on the Databricks Go SDK and communicates with Databricks REST API endpoints using OAuth (machine-to-machine via service principal) or Personal Access Tokens; the CLI configuration stores named profiles in `~/.databrickscfg` to manage multiple workspace connections.
- Every CLI command maps directly to a Databricks REST API call — `databricks jobs run-now` issues a `POST /api/2.1/jobs/run-now`; this parity means the CLI can be used for prototyping and the equivalent API call used in application code or CI/CD scripts without learning different command structures.
- **Bundles** (Databricks Asset Bundles / DABs) are YAML-defined resource definitions for jobs, pipelines, clusters, and permissions that the CLI deploys using `databricks bundle deploy` — this is Databricks' native infrastructure-as-code tool, enabling version-controlled, environment-specific deployments without Terraform for simple cases.
- The **Databricks Terraform provider** (`databricks/databricks`) wraps the REST API and supports all major resources (workspaces, clusters, jobs, Unity Catalog objects, secrets) — infrastructure teams can provision entire Databricks environments declaratively, with state tracking in a Terraform backend.
- **Service principals** (non-human identities for automation) authenticate to the API via OAuth client credentials or service principal tokens; they can be assigned workspace roles and Unity Catalog privileges independently of human users, enabling least-privilege automation identities for CI/CD pipelines. See [[Cloud-Platforms/Databricks/05-Administration#Unity Catalog Administration]] for service principal permission management.

**Common Misconceptions:**
- The old Databricks CLI (v0.100 and earlier) and the new CLI (v0.200+) are substantially different — the new CLI uses a completely rewritten Go-based architecture with different command syntax and authentication; documentation references to the old CLI are not applicable to the new version.
- The REST API does not expose direct Spark execution — you cannot submit arbitrary Spark code via the REST API synchronously; you submit a job run or notebook run and poll for completion, which means REST API-based automation is inherently asynchronous.

**Interview Answer Skeleton:**
- **What it is:** A Go-based CLI and REST API ecosystem for programmatic Databricks resource management, underpinning Databricks Asset Bundles for native IaC deployment and supporting OAuth/service-principal authentication for CI/CD automation.
- **Why it matters / trade-offs:** Enables fully automated, version-controlled deployment of Databricks jobs, pipelines, and permissions; the trade-off is that Databricks' rapid release cadence means CLI/API versions can diverge from documentation, requiring careful version pinning in CI/CD pipelines.
- **Example or context:** A data platform team manages all Databricks jobs, DLT pipelines, and cluster policies as YAML in a Git repository — on each merge to main, a GitHub Actions workflow runs `databricks bundle deploy --target prod`, updating all resources atomically and reverting if the deployment fails.

**Free Resources:**
- [Databricks CLI Documentation](https://docs.databricks.com/en/dev-tools/cli/index.html) — CLI installation, authentication, bundle deployment, and command reference
- [Databricks Academy](https://academy.databricks.com) — free DevOps courses covering CI/CD integration, Databricks Asset Bundles, and API-driven automation

---

## Workspace Security

**Status:** ⬜ Not Started

**Definition:** Databricks workspace security covers IP access lists (restrict workspace access by IP range), Private Link (route all traffic over the cloud provider's private network), customer-managed keys (encrypt workspace metadata and DBFS with customer-owned KMS keys), and network isolation (no internet egress from clusters in locked-down configurations).

**Key Mental Model:** Workspace security is a series of concentric security rings — access lists at the perimeter, private networking inside, encryption at the data layer, and audit logs throughout.

**How It Works:**
- **IP Access Lists** are enforced at the Databricks control plane level — when a user or API client connects to the workspace, the control plane checks the source IP against the configured allowlist before authenticating; requests from non-allowed IPs receive a `403` response before any workspace operation is attempted.
- **Private Link** (AWS PrivateLink / Azure Private Link) replaces the public internet route between the customer's VPC/VNet and the Databricks control plane with a private endpoint — all traffic (REST API, cluster-to-control-plane heartbeats, data access) flows over the cloud provider's backbone network, never traversing the public internet.
- **Customer-Managed Keys (CMK)** encrypt workspace metadata stored in the Databricks control plane (notebook content, job configurations, cluster configs) and optionally DBFS data in the customer's storage — the encryption key is held in the customer's KMS (AWS KMS, Azure Key Vault), so Databricks cannot access workspace data if the customer revokes the key.
- **Network isolation** via VPC/VNet injection places the cluster VMs inside the customer's private network with no public IP assignment; egress can be restricted by customer-managed Network Security Groups or AWS Security Groups, preventing cluster VMs from making outbound internet calls (useful for preventing data exfiltration).
- All security configuration changes (IP list updates, Private Link toggles, CMK key rotations) are captured in the Unity Catalog audit log alongside data access events, providing a unified security event trail. See [[Cloud-Platforms/Databricks/05-Administration#Unity Catalog Administration]] for audit log configuration.

**Common Misconceptions:**
- Private Link does not automatically encrypt the data in the Delta tables — it secures the network path between the customer's environment and Databricks; data-at-rest encryption in object storage is governed by the cloud provider's storage encryption settings and optionally CMK, not by Private Link.
- Disabling public internet access on clusters does not block all outbound traffic — traffic to the cloud provider's internal services (S3, ADLS, Secrets Manager) routes through private endpoints or VPC-internal paths and is not affected; the restriction targets external internet destinations.

**Interview Answer Skeleton:**
- **What it is:** A layered workspace security model combining network-level controls (IP access lists, Private Link, no-public-IP clusters), encryption controls (CMK for metadata and DBFS), and cluster network isolation — all audited through Unity Catalog event logs.
- **Why it matters / trade-offs:** Required for enterprise regulated industries (financial services, healthcare) to meet compliance requirements (PCI DSS, HIPAA, SOC2); the trade-off is significant operational complexity — Private Link and CMK require cloud-side infrastructure (private endpoints, KMS policies) to be pre-configured before workspace creation.
- **Example or context:** A bank deploys Databricks with VPC injection (no public IPs on clusters), Private Link to the control plane, and CMK encryption using their existing AWS KMS key — security auditors can verify that no Databricks cluster can reach the public internet and that Databricks personnel cannot access workspace metadata without the bank's encryption key.

**Free Resources:**
- [Databricks Security Documentation](https://docs.databricks.com/en/security/index.html) — IP access lists, Private Link, CMK, and network isolation configuration guides
- [Databricks Academy](https://academy.databricks.com) — free security and compliance courses covering enterprise workspace hardening patterns

---
