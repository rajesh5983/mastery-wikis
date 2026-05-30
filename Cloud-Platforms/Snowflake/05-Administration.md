# Snowflake — Administration

---

## RBAC (Role-Based Access Control)

**Status:** ⬜ Not Started

**Definition:** Snowflake uses a hierarchical RBAC model where privileges are granted to roles, and roles are granted to users or other roles. The role hierarchy (SYSADMIN, USERADMIN, SECURITYADMIN, ACCOUNTADMIN) provides separation of duties. Object ownership determines default access, and privilege inheritance flows up through the role hierarchy.

**Key Mental Model:** Snowflake RBAC is a hierarchy of permission bundles — a Manager role inherits everything a Staff role can do, plus additional privileges. Users wear roles like hats, and the hat they're wearing determines what doors open.

**How It Works:**
- Snowflake's built-in system roles form a mandatory hierarchy: ACCOUNTADMIN sits at the top (account-wide administration, billing, replication), SYSADMIN manages objects (databases, warehouses, schemas), SECURITYADMIN manages roles and grants, USERADMIN creates and manages users, and PUBLIC is assigned to all users automatically. Each child role inherits privileges from its parent.
- Object ownership is critical: when a role creates an object, that role owns it and has all privileges on it. Ownership can be transferred. Privileges on owned objects are only effective if the role also has USAGE on the parent database and schema — creating an object does not automatically grant traversal rights through the namespace hierarchy.
- Custom roles are created under SYSADMIN by convention (`CREATE ROLE analyst; GRANT ROLE analyst TO ROLE sysadmin`). Placing custom roles under SYSADMIN ensures SYSADMIN can manage objects owned by those roles. Custom roles not placed under SYSADMIN create orphaned privilege branches that SYSADMIN cannot see.
- `FUTURE GRANTS` eliminate repetitive privilege management: `GRANT SELECT ON FUTURE TABLES IN SCHEMA raw TO ROLE analyst` automatically grants SELECT on any table created in that schema in the future — without requiring a new GRANT statement per table. This is essential in fast-moving schemas where new tables are created frequently.
- The `SHOW GRANTS TO ROLE role_name` and `SHOW GRANTS OF ROLE role_name` commands audit who has which privileges and which users hold which roles. For compliance, Snowflake provides `ACCESS_HISTORY` in the Account Usage views — tracking which users queried which tables, useful for data access auditing.

**Common Misconceptions:**
- ACCOUNTADMIN should be the default working role for administrators — ACCOUNTADMIN should only be used for specific account-level tasks (billing review, replication configuration, account parameter changes); daily object management should use SYSADMIN; using ACCOUNTADMIN routinely increases the blast radius of accidental actions and violates least-privilege principles.
- Granting privileges to a parent database grants access to all objects within it — in Snowflake, USAGE on a database only allows traversal (seeing the database exists); SELECT on individual tables, schemas, or views must be granted separately; there is no inheritable access from database-level grants.

**Interview Answer Skeleton:**
- **What it is:** Snowflake's hierarchical privilege model where privileges attach to roles (not users directly), roles inherit from parent roles, and object ownership determines default access — with system roles (ACCOUNTADMIN, SYSADMIN, SECURITYADMIN, USERADMIN) providing separation of duties for account management.
- **Why it matters / trade-offs:** Proper RBAC design is the foundation of Snowflake data governance — misconfigured roles create either overly permissive access (data breach risk) or overly restrictive access (productivity friction). FUTURE GRANTS are the key to sustainable privilege management as schemas grow. The trade-off is complexity: the namespace hierarchy (account → database → schema → object) requires USAGE grants at each level, which surprises teams coming from simpler database systems.
- **Example or context:** A data platform team designs a three-role pattern: TRANSFORMER (owns and writes to the dbt target schema), REPORTER (SELECT on all reporting views), ANALYST (SELECT on curated schemas, no raw access). FUTURE GRANTS on each schema handle new objects automatically. SECURITYADMIN owns role management; SYSADMIN owns object management — these responsibilities never cross, keeping the audit trail clean.

**Free Resources:**
- [Snowflake Access Control Documentation](https://docs.snowflake.com/en/user-guide/security-access-control-overview) — Snowflake access control documentation covering the RBAC model, built-in roles, object ownership, and FUTURE GRANTS
- [Snowflake Security Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for configuring RBAC, creating custom roles, and auditing access with ACCESS_HISTORY

---

## Resource Monitors

**Status:** ⬜ Not Started

**Definition:** Resource Monitors track credit consumption for virtual warehouses and trigger notifications or automatic actions (SUSPEND, SUSPEND_IMMEDIATE) when spending reaches defined thresholds. They are the primary tool for preventing runaway cost from oversized warehouses or long-running queries.

**Key Mental Model:** Resource Monitors are the circuit breakers on Snowflake spending — when a warehouse crosses its credit threshold, the monitor trips and shuts it down before the bill gets out of control.

**How It Works:**
- A Resource Monitor defines a credit quota for a specified period (daily, weekly, monthly, or a custom date range). The quota is the total credit budget; when consumption reaches a defined percentage of the quota, the monitor triggers configured actions.
- Actions are configured at multiple thresholds: for example, at 75% consumption — notify the administrator via email; at 90% — notify again; at 100% — SUSPEND the warehouse (completes in-flight queries, blocks new ones); or SUSPEND_IMMEDIATE (kills all queries immediately). Each threshold can have independent actions.
- Resource Monitors can be account-level (covering all warehouse credits in the account) or warehouse-level (covering only specified warehouses). Warehouse-level monitors are more granular — assign different quotas to development, production, and data science warehouses independently.
- The quota resets at the start of each period. At the beginning of each month (for monthly monitors), the consumed credits counter resets to zero and the warehouse can run again even if it was suspended at the end of the previous period.
- Resource Monitors monitor credit consumption only — they do not control storage costs or Cortex/Snowpipe serverless costs. Comprehensive cost management requires combining Resource Monitors with the `WAREHOUSE_METERING_HISTORY` and `STORAGE_USAGE` Account Usage views for a complete spend picture.

**Common Misconceptions:**
- Resource Monitors provide real-time credit alerts — Resource Monitor thresholds are checked periodically (not in real-time per-query); there can be a delay between a threshold being crossed and the notification or action being triggered, meaning a warehouse can slightly overshoot its quota before being suspended.
- SUSPEND and SUSPEND_IMMEDIATE are equivalent — SUSPEND waits for currently running queries to complete before suspending the warehouse (preferred for production); SUSPEND_IMMEDIATE kills all running queries instantly (appropriate for runaway cost emergencies but causes query failures that must be handled by callers).

**Interview Answer Skeleton:**
- **What it is:** Snowflake credit consumption tracking objects that attach to warehouses or the account, define a credit budget per period, and trigger notifications and automated suspend actions at configurable consumption thresholds — preventing uncontrolled compute spend.
- **Why it matters / trade-offs:** Resource Monitors are the first line of defence against accidental runaway costs from misconfigured warehouses, runaway queries, or unexpected workload spikes — they provide both visibility (email alerts) and automatic remediation (SUSPEND). The trade-off is blunt action: a suspended warehouse blocks all users, so threshold configuration must balance cost protection against operational impact.
- **Example or context:** A data science team's development warehouse has a Resource Monitor with a 500-credit monthly quota: 75% threshold sends an email alert to the team lead, 90% sends a second alert, 100% triggers SUSPEND. Mid-month, a model training loop accidentally runs overnight and hits 90% by morning — the alert fires, the team catches it, and the SUSPEND at 100% prevents the warehouse from running through the rest of the month's budget.

**Free Resources:**
- [Snowflake Resource Monitor Documentation](https://docs.snowflake.com/en/user-guide/resource-monitors) — Snowflake Resource Monitor documentation covering quota configuration, threshold actions, and account vs warehouse-level monitors
- [Snowflake Cost Management Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for configuring Resource Monitors and analysing spend with Account Usage views

---

## Account Replication

**Status:** ⬜ Not Started

**Definition:** Snowflake Account Replication synchronises databases, share objects, users, roles, and grants across Snowflake accounts in different regions or cloud providers. It enables business continuity (failover to a secondary account) and multi-region read replicas for globally distributed teams.

**Key Mental Model:** Account Replication is Snowflake's disaster recovery and global distribution feature — a live copy of your environment in another region, ready to take over if the primary account has an outage.

**How It Works:**
- Replication is configured at the database or account level using Replication Groups (for databases) or Failover Groups (for business continuity, including users, roles, grants, and warehouses). `CREATE REPLICATION GROUP primary_group ... OBJECT_TYPES = DATABASES, ROLES, USERS, GRANTS ... REPLICATION_SCHEDULE = '10 MINUTES'` defines what to replicate and how often.
- Secondary accounts are added to the replication group: `ALTER REPLICATION GROUP ... ADD target_account.region`. Snowflake then replicates the specified objects to the target account on the configured schedule. The secondary account can query replicated databases but cannot write to them — they are read-only replicas.
- Failover is a manual or automated action: `ALTER FAILOVER GROUP ... PRIMARY` on the secondary account promotes it to primary, making it writable. DNS entries or connection strings must be updated (or a redirect policy configured) to route application traffic to the new primary. Recovery Point Objective (RPO) depends on replication schedule frequency; Recovery Time Objective (RTO) includes promotion time plus traffic redirection.
- Replication is asynchronous — the secondary lags the primary by up to the replication interval. For a 10-minute replication schedule, up to 10 minutes of committed changes may not be reflected on the secondary at any given time. This is the RPO for disaster recovery scenarios.
- Replication costs are billed based on the data transferred (egress between regions) plus compute used during replication refresh. High-churn tables (frequent updates, large byte volumes) incur higher replication costs; immutable or slowly changing tables are cheap to replicate.

**Common Misconceptions:**
- Account Replication provides zero RPO — replication is asynchronous with a configurable interval; there is always a potential data loss window equal to the replication lag at the time of failure; zero RPO is not achievable with Snowflake's current replication model.
- Failover is automatic in all scenarios — Snowflake does not automatically promote a secondary to primary during an outage; failover requires a manual `ALTER FAILOVER GROUP` command (or an automated external trigger) on the secondary account; fully automated failover requires additional tooling around Snowflake's replication API.

**Interview Answer Skeleton:**
- **What it is:** Snowflake's cross-region, cross-cloud object replication mechanism using Replication and Failover Groups — synchronising databases, roles, users, and grants to secondary accounts on a configurable schedule, enabling disaster recovery failover and globally distributed read access.
- **Why it matters / trade-offs:** Account Replication provides enterprise-grade business continuity for Snowflake workloads — the secondary account can take over production queries within minutes of a regional outage. The trade-offs are asynchronous lag (RPO equals the replication interval), manual failover initiation, and replication costs proportional to data change volume.
- **Example or context:** A financial services firm runs their primary Snowflake account on AWS us-east-1 with a Failover Group replicating to AWS eu-west-1 every 5 minutes. During an AWS us-east-1 outage, the operations team promotes eu-west-1 to primary in 3 minutes, updates the JDBC connection pool config, and trading desks reconnect automatically — with at most 5 minutes of unrecoverable transaction data (the RPO).

**Free Resources:**
- [Snowflake Account Replication Documentation](https://docs.snowflake.com/en/user-guide/account-replication-intro) — Snowflake account replication documentation covering Replication Groups, Failover Groups, RPO, RTO, and failover procedures
- [Snowflake Business Continuity Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for configuring Failover Groups and practising failover and failback procedures

---

## Private Link

**Status:** ⬜ Not Started

**Definition:** Snowflake Private Link (AWS PrivateLink, Azure Private Link) provides a private network connection to Snowflake that does not traverse the public internet. Traffic stays within the cloud provider's backbone, improving security and satisfying data residency requirements that prohibit public internet exposure.

**Key Mental Model:** Private Link is a private underground tunnel between your cloud environment and Snowflake — all data travels through it, never touching the public internet even though Snowflake is a SaaS platform.

**How It Works:**
- On AWS, Private Link creates a VPC Endpoint in your VPC that routes Snowflake traffic through AWS's private network to Snowflake's service endpoint in the same region. The VPC Endpoint receives a private IP address within your VPC's CIDR range — DNS resolves `account.snowflakecomputing.com` to this private IP within the VPC.
- On Azure, an Azure Private Endpoint is created in your VNet, connecting to Snowflake's Private Link Service. The setup is similar: Snowflake's hostname resolves to the private endpoint's IP within the VNet, and all traffic stays on Azure's private backbone.
- Private Link setup requires coordination: the customer provides their AWS account ID or Azure subscription ID to Snowflake support, who allowlists the endpoint. The customer then creates the VPC/VNet endpoint using the Snowflake-provided service name. DNS override (using a Route 53 private hosted zone on AWS or Azure Private DNS zone) resolves Snowflake's hostname to the private IP within the VPC.
- After Private Link is established, Snowflake can enforce that all connections use Private Link by blocking public internet access to the account: `ALTER ACCOUNT SET NETWORK_POLICY = private_link_only_policy`. This ensures no accidental public internet exposure of Snowflake data, even if a client misconfigures their network settings.
- Private Link applies to data plane traffic (queries, data loading, Snowpipe) but not all Snowflake management plane operations. Some administrative APIs may still require internet connectivity; verify with Snowflake documentation for specific operations when planning a fully air-gapped setup.

**Common Misconceptions:**
- Private Link encrypts traffic between your network and Snowflake — Snowflake already encrypts all traffic in transit using TLS regardless of whether Private Link is used; Private Link's security benefit is routing isolation (traffic does not traverse the public internet), not additional encryption.
- Private Link is sufficient for all compliance requirements — Private Link satisfies the "no public internet" network control, but compliance frameworks (PCI-DSS, HIPAA, SOC 2) require additional controls (IP allowlisting, MFA, audit logging, encryption at rest) that are separate from Private Link configuration.

**Interview Answer Skeleton:**
- **What it is:** A private network connectivity option for Snowflake using cloud provider backbone networking (AWS PrivateLink, Azure Private Link) — routing all Snowflake traffic through private endpoints within the customer's VPC/VNet, ensuring data never traverses the public internet between the customer environment and Snowflake.
- **Why it matters / trade-offs:** Private Link satisfies "no public internet exposure" requirements mandated by financial services regulators, healthcare data custodians, and government cloud policies — enabling Snowflake adoption in security-sensitive environments. The trade-off is setup complexity: VPC endpoint creation, DNS configuration, and Snowflake support coordination are required; multi-account environments need separate endpoints per Snowflake account.
- **Example or context:** A healthcare analytics platform must ensure patient data never traverses the public internet. They configure AWS PrivateLink between their data processing VPC and Snowflake, enforce private-only access via a network policy on the Snowflake account, and document the configuration for HIPAA audit evidence — all query and Snowpipe traffic stays on AWS private backbone between their VPC and Snowflake's managed infrastructure.

**Free Resources:**
- [Snowflake Private Link Documentation](https://docs.snowflake.com/en/user-guide/admin-security-privatelink) — Snowflake Private Link documentation covering AWS and Azure setup, DNS configuration, and enforcement via network policies
- [Snowflake Network Security Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for configuring Private Link and network policies to restrict Snowflake access to private endpoints

---

## Parameter Management

**Status:** ⬜ Not Started

**Definition:** Snowflake Parameters are configuration settings that control account-wide behaviour (session defaults, query timeout, data retention), and can be set at account, user, session, or object level. Understanding parameter inheritance (object → user → account) is key for tuning and troubleshooting.

**Key Mental Model:** Parameters are the tuning knobs of Snowflake — set them at the account level as defaults, override at user or session level for exceptions, and set on individual tables or warehouses for specific requirements.

**How It Works:**
- Parameter scope hierarchy determines which value applies: object-level (most specific) overrides user-level, which overrides session-level, which overrides account-level (most general). When a session starts, it inherits account defaults; when a user connects, user-level overrides take effect; an `ALTER SESSION SET parameter = value` overrides for that session; object-level settings (e.g., `DATA_RETENTION_TIME_IN_DAYS` on a table) apply only to that object.
- Session parameters control query execution behaviour: `STATEMENT_TIMEOUT_IN_SECONDS` kills queries exceeding the limit; `QUERY_TAG` attaches a tag to all queries run in the session (useful for cost attribution in `QUERY_HISTORY`); `TIMEZONE` controls timestamp interpretation; `USE_CACHED_RESULT` enables or disables result caching.
- Object parameters apply to specific Snowflake objects: `DATA_RETENTION_TIME_IN_DAYS` on a table or database overrides the account default retention period; `MAX_CONCURRENCY_LEVEL` on a warehouse controls how many concurrent SQL statements the warehouse handles before queuing; `STATEMENT_QUEUED_TIMEOUT_IN_SECONDS` controls how long a query waits in the warehouse queue before failing.
- `SHOW PARAMETERS FOR SESSION` shows all current session parameter values and whether they are at default or overridden. `SHOW PARAMETERS FOR ACCOUNT` shows account-level settings. `SHOW PARAMETERS IN WAREHOUSE warehouse_name` shows warehouse-specific parameter overrides.
- Network policies are a special class of parameter-adjacent object: they define IP allowlists/blocklists and are attached to users or the account via `ALTER USER ... SET NETWORK_POLICY = ...`. They control which source IPs can authenticate to Snowflake, complementing Private Link for network security.

**Common Misconceptions:**
- Session parameters persist across connections — session parameters set with `ALTER SESSION SET` apply only to the current session; when the connection closes and a new one opens, session parameters revert to user-level or account-level defaults; persistent changes require `ALTER USER` or `ALTER ACCOUNT`.
- Changing account-level parameters immediately affects all active sessions — account-level parameter changes apply to new sessions; sessions already established retain their previously inherited parameter values until they reconnect.

**Interview Answer Skeleton:**
- **What it is:** A hierarchical configuration system (account → user → session → object) where Snowflake parameters control behaviour like query timeout, result caching, data retention, and timezone — with more specific scope levels overriding account defaults for targeted tuning.
- **Why it matters / trade-offs:** Parameter management is the primary mechanism for account-wide defaults and per-user/session tuning without code changes — critical for enforcing query timeouts (cost control), tagging queries for attribution, and per-table retention configuration. The trade-off is that the four-level hierarchy can make it non-obvious which value is in effect; systematic use of `SHOW PARAMETERS` is necessary for diagnosing unexpected behaviour.
- **Example or context:** A platform team sets `STATEMENT_TIMEOUT_IN_SECONDS = 3600` at the account level (1-hour hard limit for all queries). The data science team overrides it at the user level with `ALTER USER ds_user SET STATEMENT_TIMEOUT_IN_SECONDS = 14400` for their long-running training queries. ETL jobs set `QUERY_TAG = 'etl_pipeline_name'` in session to attribute costs per pipeline in `QUERY_HISTORY` dashboards — no application code changes required.

**Free Resources:**
- [Snowflake Parameters Documentation](https://docs.snowflake.com/en/sql-reference/parameters) — Snowflake parameters reference covering all parameters, their scope levels, default values, and configuration syntax
- [Snowflake Account Administration Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for configuring account, session, and object parameters for performance tuning and governance

---

## Cost Management

**Status:** ⬜ Not Started

**Definition:** Snowflake cost management involves warehouse right-sizing (choose the smallest warehouse that meets latency SLA), auto-suspend (warehouses stop when idle), clustering keys (reduce bytes scanned), result caching (avoid re-running identical queries), and query history analysis to find expensive queries that need optimisation.

**Key Mental Model:** Snowflake cost management is like managing a fleet of vehicles — right-size each vehicle for its route (warehouse sizing), don't leave engines running when parked (auto-suspend), and fix the vehicles that burn the most fuel (optimise expensive queries).

**How It Works:**
- Snowflake charges for compute in credits per second — a running warehouse of any size accrues credits while running. Auto-suspend (`AUTO_SUSPEND = 60` seconds) is the single most impactful default setting: a warehouse that auto-suspends after 60 seconds of inactivity costs nothing between query bursts. Without auto-suspend, an idle warehouse runs continuously.
- Warehouse sizing follows a doubling pattern (XS = 1 credit/hour, S = 2, M = 4, L = 8, XL = 16). For most analytical queries, query execution time is roughly halved when the warehouse doubles in size — meaning cost (credits × time) is nearly constant. Right-sizing is therefore less about cost and more about latency: use the smallest warehouse that meets the query latency SLA.
- The `QUERY_HISTORY` and `WAREHOUSE_METERING_HISTORY` views in `INFORMATION_SCHEMA` and `ACCOUNT_USAGE` schema are the primary cost analysis tools. Sorting `QUERY_HISTORY` by `CREDITS_USED_CLOUD_SERVICES` or `EXECUTION_TIME` identifies expensive queries; `BYTES_SCANNED` compared to `BYTES_PROCESSED_FROM_RESULT_CACHE` shows result cache hit rates.
- Clustering keys improve query performance on large tables by co-locating related micro-partitions. `ALTER TABLE orders CLUSTER BY (order_date)` reorganises the table to group rows with the same order_date into adjacent micro-partitions — queries filtering on order_date scan far fewer micro-partitions, reducing bytes billed. Clustering has maintenance cost (automatic reclustering credits) that must be weighed against the scan reduction benefit.
- Cloud services cost (Snowflake's control plane — metadata operations, query compilation, authentication) is charged when it exceeds 10% of daily compute credits. Heavy use of `SHOW`, `DESCRIBE`, metadata queries, or very frequent small queries can push cloud services costs above the free tier threshold. Consolidating metadata queries and avoiding query patterns that over-use the control plane keeps cloud services costs within the free tier.

**Common Misconceptions:**
- Larger warehouses always cost more — because larger warehouses run queries faster, the total credits used (credits per hour × hours) for a given query is often similar between warehouse sizes; the decision between S and L is usually about query latency vs concurrency needs, not cost; the exception is queries that cannot parallelise (sequential operations), where larger warehouses provide speed at higher cost.
- Result cache is always used automatically — result cache is only used when the exact same SQL is resubmitted by any user with the same effective privileges, the underlying table data has not changed, and the `USE_CACHED_RESULT` session parameter is enabled (default: true); any change to the query, including whitespace in some cases, produces a cache miss.

**Interview Answer Skeleton:**
- **What it is:** A multi-lever approach to controlling Snowflake spend: auto-suspend for idle warehouses (eliminate wasted compute), right-sizing for latency SLA (not raw cost reduction), clustering keys for bytes scanned reduction (storage-compute trade-off), result caching (avoid re-execution), and query history analysis to find and fix the most expensive queries.
- **Why it matters / trade-offs:** Unmanaged Snowflake costs scale with data volume and query activity — without active cost management, organisations routinely overspend by 2-5x. The trade-offs are non-obvious: right-sizing for cost often means accepting higher latency; clustering has maintenance overhead; result caching has strict invalidation conditions. The most impactful single change is enabling auto-suspend on all warehouses.
- **Example or context:** A data platform team audits `QUERY_HISTORY` and finds 15 queries consuming 40% of compute credits — all from a single dashboard refreshing every 30 minutes with full table scans on a 500M row orders table. Fixes applied: clustering on order_date (reduces bytes scanned by 80%), dashboard caching enabled in BI tool (reduces Snowflake execution frequency), and a dedicated XS warehouse for BI with a Resource Monitor. Monthly compute spend drops by 35%.

**Free Resources:**
- [Snowflake Cost Management Documentation](https://docs.snowflake.com/en/user-guide/cost-understanding-overall) — Snowflake cost management documentation covering credits, storage billing, cloud services costs, and optimisation strategies
- [Snowflake Cost Optimisation Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for analysing warehouse spend with Account Usage views and applying right-sizing and clustering optimisations
