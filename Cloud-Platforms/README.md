# Cloud Platforms

A comparative learning space for the three dominant enterprise data platforms: **Databricks**, **Microsoft Fabric**, and **Snowflake**. Each platform is studied both independently and in relation to the others.

---

## Comparison Philosophy

These three platforms overlap significantly in what they can do, but they differ fundamentally in *how* they are designed, *what* they optimise for, and *who* they are built for. The goal is not to crown a winner but to understand:

- When each platform is the natural fit
- Where each platform struggles
- How to evaluate them for a specific organisation's workload

Study the comparison matrix first (`00-Comparison-Matrix.md`), then dive into each platform's individual files for depth.

---

## Platform Index

| Platform | Architecture Style | Strength | Notes |
|----------|--------------------|----------|-------|
| [Databricks](Databricks/) | Lakehouse on open formats | ML/AI + Data Engineering | Open source core (Delta Lake, MLflow) |
| [Fabric](Fabric/) | Unified SaaS on OneLake | Microsoft ecosystem integration | One capacity for all workloads |
| [Snowflake](Snowflake/) | Multi-cloud SQL warehouse | Governed, scalable SQL analytics | Data sharing and collaboration |

---

## Cross-Platform Concepts

- [[Cloud-Platforms/00-Comparison-Matrix]] — side-by-side feature comparison
- [[DE-Engineer/06-Platform]] — layer 6 of the DE framework for platform concepts
- [[AI-Engineer/10-Inference-Deployment]] — cloud AI options on each platform
