# Snowflake — Collaboration

---

## Data Sharing

**Status:** ⬜ Not Started

**Definition:** Snowflake Data Sharing allows a data provider to share live, read-only access to Snowflake tables, views, or dynamic data mashups with consumer accounts — without copying or moving data. Consumers query the shared data using their own virtual warehouses; providers control access and can revoke it instantly.

**Key Mental Model:** Snowflake Data Sharing is giving someone a read-only key to a specific room in your data warehouse — they can look at everything in that room from their own computer, but you control the key and can revoke it anytime, and no data is physically moved.

**How It Works:**
- A Share is a named Snowflake object in the provider account. The provider adds database objects (tables, views, secure views, dynamic tables) to the share with `GRANT SELECT ON TABLE ... TO SHARE share_name`. The Share can also include schemas and databases, granting access to entire namespaces.
- On the consumer side, a shared database is created from the share: `CREATE DATABASE shared_db FROM SHARE provider_account.share_name`. This database appears in the consumer's account like a local database, but all data reads transparently access the provider's storage — no copy exists in the consumer account.
- Security is maintained through Snowflake's multi-tenancy isolation. Consumers can only read objects explicitly added to the share. They cannot access the provider's other tables, even in the same database. Secure views on the provider side can filter rows dynamically based on `CURRENT_ACCOUNT()` — allowing row-level, per-consumer data filtering from a single share.
- Data is always live — consumers see the current state of the provider's tables, including any changes made after the share was created. There is no cache expiry or replication lag because consumers read from the same underlying micro-partitions as the provider.
- Access is revocable instantly: `REVOKE SELECT ON TABLE ... FROM SHARE share_name` immediately removes consumer access — no ETL teardown, no data deletion required because no data was copied.

**Common Misconceptions:**
- Data Sharing works between any Snowflake accounts globally — standard Data Sharing requires the provider and consumer to be in the same cloud provider and region; cross-region and cross-cloud sharing requires Cross-Cloud Auto-Fulfillment (see [[Cloud-Platforms/Snowflake/04-Collaboration#Cross-Cloud Auto-Fulfillment]]) which replicates data to the consumer's region automatically.
- Consumers need their own Snowflake account to access shared data — Snowflake Reader Accounts allow providers to create managed consumer accounts for organisations that do not have their own Snowflake subscription; the provider pays for the reader account's compute.

**Interview Answer Skeleton:**
- **What it is:** A zero-copy data access mechanism where a Snowflake provider grants consumers read access to specific database objects (tables, views) via a Share object — consumers query live data from the provider's storage using their own compute, with no data physically duplicated.
- **Why it matters / trade-offs:** Data Sharing eliminates the extract-and-deliver pattern for B2B data distribution — no ETL pipelines, no file transfers, no replication lag. Consumers always see current data. The trade-off is that both parties need Snowflake accounts (or the provider must provision a Reader Account), limiting reach compared to file-based delivery.
- **Example or context:** A financial data vendor maintains their market data in Snowflake and creates a share per client with a secure view filtering to that client's subscribed tickers. Each client account gets a shared database with live market data — the vendor updates data once, all clients see it immediately, and revoking a lapsed subscription takes one SQL command.

**Free Resources:**
- [Snowflake Data Sharing Documentation](https://docs.snowflake.com/en/user-guide/data-sharing-intro) — Snowflake Data Sharing documentation covering shares, consumers, secure views, and Reader Accounts
- [Snowflake Data Sharing Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for creating and consuming a Snowflake data share with live data access

---

## Marketplace

**Status:** ⬜ Not Started

**Definition:** The Snowflake Marketplace is a data exchange where providers publish live data products (weather data, financial data, geolocation, demographic data) that any Snowflake customer can acquire and query alongside their own data. Data is delivered via shares — no extraction, no ETL, live and always current.

**Key Mental Model:** The Snowflake Marketplace is an app store for data — browse, subscribe, and immediately query third-party datasets in your own Snowflake environment, as if you loaded it yourself, but without any integration work.

**How It Works:**
- Marketplace listings are Shares published by providers. A consumer browses listings, requests access (free listings are instant; paid listings require provider approval or a commercial agreement), and once access is granted, creates a shared database in their account. The entire process from discovery to queryable data takes minutes.
- Data in a Marketplace listing is live — it reflects the provider's current data with no caching or replication lag. When a weather data provider updates their forecast dataset, all consumers with that listing see the update immediately on their next query, without any action on the consumer's part.
- Providers can offer personalised listings — a provider can create a listing visible only to specific consumer accounts, delivering a custom dataset or filtered view of their data. This enables private commercial data products distributed through the Marketplace infrastructure without custom sharing setup.
- Listings support multiple regions and clouds via Cross-Cloud Auto-Fulfillment. A provider lists once; Snowflake handles delivering the data product to consumers in any region by replicating the underlying share objects automatically.
- Data products in the Marketplace can be free (public datasets, trial samples, community data) or paid. Snowflake facilitates the commercial relationship for paid listings through its own billing infrastructure — providers set pricing, Snowflake collects payment, and the data access is managed automatically based on subscription status.

**Common Misconceptions:**
- Marketplace data is exported and loaded into your Snowflake account — Marketplace data is never copied into your account; it remains in the provider's storage and is accessed live through a shared database; your account only stores the pointer (the database object), not the data itself.
- All Marketplace listings are free — the Marketplace includes both free and paid listings; free-tier and trial listings exist, but high-value commercial data (financial reference data, premium demographic datasets) requires a commercial agreement with the provider.

**Interview Answer Skeleton:**
- **What it is:** Snowflake's public data exchange where providers publish live datasets as Marketplace listings — consumers browse, request access, and query third-party data directly in their Snowflake environment via the same Share mechanism as direct data sharing, with no ETL or data movement.
- **Why it matters / trade-offs:** The Marketplace dramatically accelerates data enrichment use cases — instead of negotiating data delivery contracts, building ETL pipelines, and managing refresh schedules, a team can augment their data with external datasets (weather, geolocation, financial) in minutes. The trade-off is dependency on provider availability and pricing; if a provider changes or removes a listing, downstream queries break.
- **Example or context:** A retail demand forecasting team enriches their sales training data with Weather Source weather data from the Marketplace — joining local store sales to daily temperature and precipitation by ZIP code without a single API call or file download. The weather data is always current; the join query runs entirely within Snowflake against live provider data.

**Free Resources:**
- [Snowflake Marketplace Documentation](https://docs.snowflake.com/en/user-guide/data-marketplace-intro) — Snowflake Marketplace documentation covering browsing, acquiring listings, and using shared data in your account
- [Snowflake Marketplace Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for finding and using a Snowflake Marketplace data product in your own queries

---

## Clean Rooms

**Status:** ⬜ Not Started

**Definition:** Snowflake Clean Rooms enable two organisations to perform joint data analysis on overlapping data (audience overlap, attribution, cohort analysis) without either party seeing the other's raw data. Analysis runs in a controlled environment with defined analysis templates, preserving privacy through differential privacy and aggregation rules.

**Key Mental Model:** A Clean Room is a secure meeting room where two organisations can compare notes on shared customers without either showing their full customer list to the other — the analysis happens in the middle, and only aggregate insights leave the room.

**How It Works:**
- A Clean Room is a Snowflake-managed environment where a host (data provider) defines the data they contribute and the analysis templates (SQL queries) that participants are permitted to run. Participants join the clean room with their own data and can only execute the predefined template queries — no ad-hoc access to the underlying data of either party.
- Privacy controls are enforced at the query level. Analysis templates include minimum aggregation thresholds (e.g., results with fewer than 50 matching records are suppressed) and differential privacy noise injection to prevent re-identification. These rules are embedded in the templates and cannot be bypassed by participants.
- Snowflake Clean Rooms are built on the Snowflake Native App Framework — the clean room runs as a managed application within the host's Snowflake account, and participant data is never transmitted to the host; matching occurs within the clean room's isolated compute boundary.
- Common analysis templates: audience overlap (what percentage of my customers match your customers?), attribution analysis (which of my ad impressions preceded conversions in your purchase data?), cohort analysis (how do our shared customers behave differently from non-overlapping segments?). Each template is pre-approved and auditable.
- All analysis runs are logged. Both parties can audit which templates were executed, when, and by whom — providing transparency and compliance evidence for data collaboration agreements that may require regulatory review.

**Common Misconceptions:**
- Clean Rooms are only for advertising use cases — while audience overlap and attribution are common media use cases, Clean Rooms apply to any regulated data collaboration: healthcare organisations sharing patient outcomes research, financial institutions benchmarking credit risk, retailers and CPG brands analysing shopper behaviour jointly.
- Clean Rooms are difficult to set up and require data movement — Snowflake Clean Rooms run natively within Snowflake with both parties' data staying in their respective accounts; no data extraction or transfer is required; the entire workflow is SQL-based through the Native App interface.

**Interview Answer Skeleton:**
- **What it is:** A Snowflake-managed isolated computation environment where two parties contribute data, run pre-approved analysis templates, and receive aggregate results — with privacy controls (minimum thresholds, differential privacy) that prevent either party from accessing the other's raw data.
- **Why it matters / trade-offs:** Clean Rooms enable data collaboration that was previously impossible due to privacy regulations (GDPR, HIPAA, CCPA) or competitive sensitivity — parties gain mutual analytical value without the legal and reputational risk of raw data sharing. The trade-off is limited analytical flexibility: only pre-approved templates run; ad-hoc questions require negotiating new template definitions.
- **Example or context:** A retailer and a consumer goods brand run an audience overlap analysis in a Clean Room: the retailer contributes hashed customer purchase history; the brand contributes hashed marketing exposure data. The clean room returns "34% overlap, with overlapping customers showing 2.1x higher brand category spend" — without the retailer seeing the brand's exposure list or the brand seeing the retailer's customer purchase records.

**Free Resources:**
- [Snowflake Clean Rooms Documentation](https://docs.snowflake.com/en/user-guide/cleanrooms/introduction) — Snowflake Clean Rooms documentation covering setup, templates, privacy controls, and participant management
- [Snowflake Clean Rooms Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for creating a Clean Room, defining analysis templates, and running a joint audience analysis

---

## Data Exchange

**Status:** ⬜ Not Started

**Definition:** A Snowflake Data Exchange is a private marketplace — organisations can create a controlled, invitation-only environment for sharing data products within a specific ecosystem (e.g., all subsidiaries, all partners in a supply chain). It provides the discoverability of the Marketplace with access control for private audiences.

**Key Mental Model:** A Data Exchange is a private farmers' market — a curated group of producers and consumers sharing data products within a controlled community, rather than the open public marketplace.

**How It Works:**
- An organisation (the exchange host) creates a Data Exchange — a named, private catalog of data products visible only to invited accounts. The host configures the exchange governance: who can publish listings, which consumer accounts can request access, and whether listings require host approval before going live.
- Providers within the exchange publish listings using the same Sharing mechanism as the public Marketplace — they create a Share, attach it to a listing description, and publish to the exchange. Consumers see listings in their private exchange catalog and request access through the exchange interface.
- A Data Exchange is implemented on Snowflake's Marketplace infrastructure but with an invitation-only access model. The host manages a member list of provider and consumer accounts — accounts not on the list cannot discover or access the exchange.
- Data Exchange is appropriate for: corporate data hubs (a parent company distributing reference data to subsidiaries), supply chain ecosystems (a manufacturer sharing operational data with logistics partners), or industry consortia (a group of organisations sharing anonymised benchmark data). The key requirement is a defined, manageable set of participants.
- Governance within a Data Exchange is the host's responsibility — defining listing standards, approving new participants, auditing access patterns, and revoking access for departing members. The host account must design governance policies before onboarding participants.

**Common Misconceptions:**
- A Data Exchange is the same as just creating multiple Shares — a Data Exchange provides a discoverable catalog, managed access request workflow, and centralised governance across multiple providers and consumers; individual Shares require bilateral relationship management without the catalog and workflow infrastructure.
- Data Exchange is limited to sharing within a single Snowflake region — Data Exchange listings support Cross-Cloud Auto-Fulfillment, allowing exchange members in different regions or cloud providers to access listings without the host managing per-consumer replication.

**Interview Answer Skeleton:**
- **What it is:** A host-managed, invitation-only private catalog of Snowflake data products — built on the Marketplace infrastructure but restricted to approved participant accounts — providing discoverability, access governance, and listing management for a defined data-sharing ecosystem.
- **Why it matters / trade-offs:** Data Exchanges enable large-scale data distribution within a controlled community without the operational overhead of managing individual bilateral sharing relationships — the exchange provides a central catalog and access workflow. The trade-off is that the host takes on governance responsibility; participant onboarding, listing approval, and access auditing require ongoing operational effort.
- **Example or context:** A global financial institution creates a Data Exchange for their 12 regional subsidiaries. The central data team publishes FX reference rates, regulatory reporting schemas, and shared customer master data as exchange listings. Each subsidiary requests access through the exchange catalog; the central team approves and the data is immediately available in the subsidiary's Snowflake account — no file transfers, no ETL, and all updates are instantaneous.

**Free Resources:**
- [Snowflake Data Exchange Documentation](https://docs.snowflake.com/en/user-guide/data-exchange) — Snowflake Data Exchange documentation covering host setup, member management, and listing governance
- [Snowflake Data Sharing and Exchange Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart covering the end-to-end flow of publishing and consuming data products in a Snowflake Data Exchange

---

## Cross-Cloud Auto-Fulfillment

**Status:** ⬜ Not Started

**Definition:** Cross-cloud Auto-Fulfillment allows Snowflake data providers to publish a share once, and consumers on different cloud providers or regions can access it without the provider manually replicating data. Snowflake automatically replicates shared data to the consumer's region/cloud transparently.

**Key Mental Model:** Auto-Fulfillment is Snowflake's logistics layer for data sharing — you list a data product once, and Snowflake handles delivery to any consumer regardless of which cloud or region they're on.

**How It Works:**
- Without Auto-Fulfillment, Snowflake Data Sharing requires the provider and consumer to be in the same cloud provider and region — a provider on AWS us-east-1 cannot directly share with a consumer on Azure East US. Cross-Cloud Auto-Fulfillment removes this constraint by automating cross-region/cross-cloud replication.
- When a consumer in a different region or cloud requests access to a listing with Auto-Fulfillment enabled, Snowflake provisions a replication group from the provider account, replicates the shared database objects to infrastructure in the consumer's cloud/region, and creates the consumer-facing share from the replicated copy — transparently.
- Replication is managed by Snowflake's Marketplace infrastructure, not the provider. The provider does not configure, monitor, or pay for the cross-region replication directly — replication costs are built into Snowflake's Marketplace fee structure for listings using Auto-Fulfillment.
- Refresh latency depends on the replication cadence, which is managed by Snowflake based on listing type and provider data update frequency. Data freshness for Auto-Fulfillment consumers may lag the provider's data by minutes to hours, unlike same-region sharing which is always live.
- Providers opt into Auto-Fulfillment per listing. Once enabled, the listing becomes discoverable globally across all Snowflake regions and clouds in the Marketplace. Without Auto-Fulfillment, the listing is only visible to consumers in the same cloud/region as the provider.

**Common Misconceptions:**
- Auto-Fulfillment is instantaneous like same-region sharing — Auto-Fulfillment involves cross-region replication, which introduces latency; consumers in different regions see data refreshed on a replication schedule (minutes to hours), not the live-read-from-provider model of same-region sharing.
- Providers configure and manage the cross-region replication — Snowflake's Marketplace infrastructure manages all cross-cloud replication for Auto-Fulfillment listings; providers simply enable the feature on their listing; the replication infrastructure is fully managed by Snowflake.

**Interview Answer Skeleton:**
- **What it is:** A Snowflake-managed replication layer that automatically delivers Marketplace and Data Exchange listings to consumers in any cloud provider or region — eliminating the same-cloud/same-region constraint of standard Snowflake Data Sharing without provider-side replication management.
- **Why it matters / trade-offs:** Auto-Fulfillment allows data providers to reach a global Snowflake customer base with a single listing, regardless of cloud provider fragmentation — the Marketplace becomes a true global distribution channel. The trade-off is data freshness: cross-region consumers see replicated data with a lag rather than the live access of same-region sharing, which matters for time-sensitive data products.
- **Example or context:** A geolocation data provider hosts their data on AWS us-east-1 and publishes a Marketplace listing with Auto-Fulfillment enabled. Customers on Azure West Europe and GCP Asia Pacific subscribe and get a shared database in their own region — Snowflake replicates updates to each region automatically. The provider manages one listing; Snowflake handles the rest.

**Free Resources:**
- [Snowflake Cross-Cloud Auto-Fulfillment Documentation](https://docs.snowflake.com/en/user-guide/data-sharing-auto-fulfillment) — Snowflake cross-cloud auto-fulfillment documentation covering enablement, replication mechanics, and freshness behaviour
- [Snowflake Marketplace and Sharing Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart covering Marketplace listing creation with cross-cloud Auto-Fulfillment enabled
