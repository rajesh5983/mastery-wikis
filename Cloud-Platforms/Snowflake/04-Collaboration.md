# Snowflake — Collaboration

---

## Data Sharing

**Status:** ⬜ Not Started

**Definition:** Snowflake Data Sharing allows a data provider to share live, read-only access to Snowflake tables, views, or dynamic data mashups with consumer accounts — without copying or moving data. Consumers query the shared data using their own virtual warehouses; providers control access and can revoke it instantly.

**Mental Model:** Snowflake Data Sharing is giving someone a read-only key to a specific room in your data warehouse — they can look at everything in that room from their own computer, but you control the key and can revoke it anytime, and no data is physically moved.

**Free Resources:** https://docs.snowflake.com/en/user-guide/data-sharing-intro — Snowflake Data Sharing documentation covering shares, consumers, and governance

---

## Marketplace

**Status:** ⬜ Not Started

**Definition:** The Snowflake Marketplace is a data exchange where providers publish live data products (weather data, financial data, geolocation, demographic data) that any Snowflake customer can acquire and query alongside their own data. Data is delivered via shares — no extraction, no ETL, live and always current.

**Mental Model:** The Snowflake Marketplace is an app store for data — browse, subscribe, and immediately query third-party datasets in your own Snowflake environment, as if you loaded it yourself, but without any integration work.

**Free Resources:** https://app.snowflake.com/marketplace — Snowflake Marketplace (accessible with Snowflake account); documentation at https://docs.snowflake.com/en/user-guide/data-marketplace-intro

---

## Clean Rooms

**Status:** ⬜ Not Started

**Definition:** Snowflake Clean Rooms enable two organisations to perform joint data analysis on overlapping data (audience overlap, attribution, cohort analysis) without either party seeing the other's raw data. Analysis runs in a controlled environment with defined analysis templates, preserving privacy through differential privacy and aggregation rules.

**Mental Model:** A Clean Room is a secure meeting room where two organisations can compare notes on shared customers without either showing their full customer list to the other — the analysis happens in the middle, and only aggregate insights leave the room.

**Free Resources:** https://docs.snowflake.com/en/user-guide/cleanrooms/introduction — Snowflake Clean Rooms documentation covering setup, templates, and privacy controls

---

## Data Exchange

**Status:** ⬜ Not Started

**Definition:** A Snowflake Data Exchange is a private marketplace — organisations can create a controlled, invitation-only environment for sharing data products within a specific ecosystem (e.g., all subsidiaries, all partners in a supply chain). It provides the discoverability of the Marketplace with access control for private audiences.

**Mental Model:** A Data Exchange is a private farmers' market — a curated group of producers and consumers sharing data products within a controlled community, rather than the open public marketplace.

**Free Resources:** https://docs.snowflake.com/en/user-guide/data-exchange — Snowflake Data Exchange documentation covering setup and participant management

---

## Cross-Cloud Auto-Fulfillment

**Status:** ⬜ Not Started

**Definition:** Cross-cloud Auto-Fulfillment allows Snowflake data providers to publish a share once, and consumers on different cloud providers or regions can access it without the provider manually replicating data. Snowflake automatically replicates shared data to the consumer's region/cloud transparently.

**Mental Model:** Auto-Fulfillment is Snowflake's logistics layer for data sharing — you list a data product once, and Snowflake handles delivery to any consumer regardless of which cloud or region they're on.

**Free Resources:** https://docs.snowflake.com/en/user-guide/data-sharing-auto-fulfillment — Snowflake cross-cloud auto-fulfillment documentation
