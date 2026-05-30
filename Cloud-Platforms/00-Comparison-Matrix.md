# Platform Comparison Matrix — Databricks vs Fabric vs Snowflake

> Use this matrix for quick reference when evaluating platform choices. For depth, visit each platform's individual files.

---

## Side-by-Side Comparison

| Dimension | Databricks | Microsoft Fabric | Snowflake |
|-----------|-----------|-----------------|-----------|
| **Compute model** | Clusters (All-Purpose, Job, SQL Warehouses); serverless option available | Capacity-based (F-SKUs); shared across all experiences; serverless for warehouse | Virtual Warehouses — independently scalable compute; multi-cluster for concurrency |
| **Storage architecture** | Delta Lake on cloud object storage (S3/ADLS/GCS); open Parquet + Delta format | OneLake — single unified storage lake per tenant; OneLake shortcuts for external data | Proprietary columnar micro-partition storage; separate from compute; Iceberg export available |
| **Governance approach** | Unity Catalog — unified metastore for data, ML models, and dashboards; column/row security | Microsoft Purview integration; workspace-level governance; domain management | Snowflake RBAC — hierarchical roles, column masking, row-level security, object tagging |
| **AI/ML native capabilities** | MLflow, Feature Store, Model Serving, Mosaic AI, Databricks Foundation Model APIs, Vector Search | Azure AI integration, Fabric AI Skills, Copilot across experiences, Real-Time Intelligence | Cortex LLM Functions (serverless inference), Snowpark ML, Document AI, Cortex Search |
| **Cost model** | DBU (Databricks Unit) per compute type + cloud storage costs; serverless DBUs for some services | Capacity units (CU) per F-SKU tier per hour; one bill covers all experiences in the capacity | Credit-based; credits per virtual warehouse size per hour; storage separate; on-demand or pre-purchased |
| **Ideal workload** | Complex ML pipelines, large-scale ETL, streaming, lakehouse architecture, multi-language data science | Microsoft 365-integrated analytics, Power BI semantic model workflows, unified SaaS for mixed workloads | Governed SQL analytics, data sharing with external partners, multi-cloud data exchange, BI-heavy organisations |
| **Key differentiator** | Open format lakehouse (no proprietary lock-in at storage layer); best-in-class ML/AI platform | Fully integrated SaaS experience within Microsoft ecosystem; one capacity bill for everything | Seamless data sharing and Marketplace; multi-cloud architecture; superior query concurrency management |

---

## When to Choose Each

**Choose Databricks when:**
- You have significant ML/AI workloads alongside data engineering
- Your team is Python/Spark-native
- You want open table formats and avoid storage lock-in
- You need streaming + batch in a unified platform

**Choose Microsoft Fabric when:**
- Your organisation is deeply invested in Microsoft 365, Azure, and Power BI
- You want a single capacity bill covering all data workloads
- Your primary stakeholders consume data through Power BI
- You prefer SaaS over infrastructure management

**Choose Snowflake when:**
- SQL-based analytics is the primary workload
- You need to share data with external partners securely
- You operate across multiple clouds and need a cloud-neutral warehouse
- Concurrency at scale for many simultaneous BI users is a requirement

---

## See Also
- [[Cloud-Platforms/Databricks/01-Architecture]]
- [[Cloud-Platforms/Fabric/01-Architecture]]
- [[Cloud-Platforms/Snowflake/01-Architecture]]
