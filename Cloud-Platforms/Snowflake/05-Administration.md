# Snowflake — Administration

---

## RBAC (Role-Based Access Control)

**Status:** ⬜ Not Started

**Definition:** Snowflake uses a hierarchical RBAC model where privileges are granted to roles, and roles are granted to users or other roles. The role hierarchy (SYSADMIN, USERADMIN, SECURITYADMIN, ACCOUNTADMIN) provides separation of duties. Object ownership determines default access, and privilege inheritance flows up through the role hierarchy.

**Mental Model:** Snowflake RBAC is a hierarchy of permission bundles — a Manager role inherits everything a Staff role can do, plus additional privileges. Users wear roles like hats, and the hat they're wearing determines what doors open.

**Free Resources:** https://docs.snowflake.com/en/user-guide/security-access-control-overview — Snowflake access control overview covering the RBAC model and built-in roles

---

## Resource Monitors

**Status:** ⬜ Not Started

**Definition:** Resource Monitors track credit consumption for virtual warehouses and trigger notifications or automatic actions (SUSPEND, SUSPEND_IMMEDIATE) when spending reaches defined thresholds. They are the primary tool for preventing runaway cost from oversized warehouses or long-running queries.

**Mental Model:** Resource Monitors are the circuit breakers on Snowflake spending — when a warehouse crosses its credit threshold, the monitor trips and shuts it down before the bill gets out of control.

**Free Resources:** https://docs.snowflake.com/en/user-guide/resource-monitors — Snowflake Resource Monitor documentation covering thresholds, actions, and account-level monitors

---

## Account Replication

**Status:** ⬜ Not Started

**Definition:** Snowflake Account Replication synchronises databases, share objects, users, roles, and grants across Snowflake accounts in different regions or cloud providers. It enables business continuity (failover to a secondary account) and multi-region read replicas for globally distributed teams.

**Mental Model:** Account Replication is Snowflake's disaster recovery and global distribution feature — a live copy of your environment in another region, ready to take over if the primary account has an outage.

**Free Resources:** https://docs.snowflake.com/en/user-guide/account-replication-intro — Snowflake account replication documentation covering setup and failover

---

## Private Link

**Status:** ⬜ Not Started

**Definition:** Snowflake Private Link (AWS PrivateLink, Azure Private Link) provides a private network connection to Snowflake that does not traverse the public internet. Traffic stays within the cloud provider's backbone, improving security and satisfying data residency requirements that prohibit public internet exposure.

**Mental Model:** Private Link is a private underground tunnel between your cloud environment and Snowflake — all data travels through it, never touching the public internet even though Snowflake is a SaaS platform.

**Free Resources:** https://docs.snowflake.com/en/user-guide/admin-security-privatelink — Snowflake Private Link documentation covering setup for AWS and Azure

---

## Parameter Management

**Status:** ⬜ Not Started

**Definition:** Snowflake Parameters are configuration settings that control account-wide behaviour (session defaults, query timeout, data retention), and can be set at account, user, session, or object level. Understanding parameter inheritance (object → user → account) is key for tuning and troubleshooting.

**Mental Model:** Parameters are the tuning knobs of Snowflake — set them at the account level as defaults, override at user or session level for exceptions, and set on individual tables or warehouses for specific requirements.

**Free Resources:** https://docs.snowflake.com/en/sql-reference/parameters — Snowflake parameters reference documentation covering all parameters and scope levels

---

## Cost Management

**Status:** ⬜ Not Started

**Definition:** Snowflake cost management involves warehouse right-sizing (choose the smallest warehouse that meets latency SLA), auto-suspend (warehouses stop when idle), clustering keys (reduce bytes scanned), result caching (avoid re-running identical queries), and query history analysis to find expensive queries that need optimisation.

**Mental Model:** Snowflake cost management is like managing a fleet of vehicles — right-size each vehicle for its route (warehouse sizing), don't leave engines running when parked (auto-suspend), and fix the vehicles that burn the most fuel (optimise expensive queries).

**Free Resources:** https://docs.snowflake.com/en/user-guide/cost-understanding-overall — Snowflake cost management documentation covering credits, storage, and optimisation strategies
