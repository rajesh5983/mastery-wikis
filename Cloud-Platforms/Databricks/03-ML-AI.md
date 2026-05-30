# Databricks — ML and AI

---

## MLflow

**Status:** ⬜ Not Started

**Definition:** MLflow is the open-source ML lifecycle platform built into Databricks. It tracks experiments (parameters, metrics, artifacts), manages the model registry (versioning, staging, production promotion), and provides a unified API for logging models from any framework (scikit-learn, PyTorch, HuggingFace, etc.).

**Mental Model:** MLflow is the experiment notebook and model filing system for ML teams — every training run is logged, every model version is tracked, and promoting a model from staging to production is a controlled, audited step.

**Free Resources:** https://mlflow.org/docs/latest/index.html — MLflow official documentation covering tracking, model registry, and deployment

---

## Feature Store

**Status:** ⬜ Not Started

**Definition:** The Databricks Feature Store is a centralised repository for computed features — pre-calculated, validated feature values that can be shared across ML models and pipelines. It ensures feature consistency between training and serving, reduces duplicate computation, and tracks feature lineage.

**Mental Model:** The feature store is a shared ingredient pantry for ML models — teams compute expensive features once, store them centrally, and multiple models reuse the same validated ingredients rather than each computing their own version.

**Free Resources:** https://docs.databricks.com/en/machine-learning/feature-store/index.html — Databricks Feature Store documentation covering creation, publishing, and training/serving consistency

---

## Model Serving

**Status:** ⬜ Not Started

**Definition:** Databricks Model Serving provides serverless real-time inference endpoints for MLflow-registered models. Endpoints auto-scale, support A/B testing between model versions, and integrate with Databricks' monitoring and governance. Foundation Model APIs provide serverless access to LLMs like DBRX and Llama on Databricks infrastructure.

**Mental Model:** Model Serving is the deployment counter for ML models — you register a model in MLflow, click deploy, and Databricks provisions an auto-scaling REST endpoint with no infrastructure management required.

**Free Resources:** https://docs.databricks.com/en/machine-learning/model-serving/index.html — Databricks Model Serving documentation covering endpoint creation, configuration, and monitoring

---

## AutoML

**Status:** ⬜ Not Started

**Definition:** Databricks AutoML automatically trains and evaluates multiple ML models on a provided dataset, producing a comparison of models with their metrics and generating the best-performing model's notebook so engineers can inspect and customise the approach. It is a transparent glass-box AutoML, not a black-box.

**Mental Model:** AutoML is a rapid prototyping assistant — it tries many approaches quickly and shows you what worked and why, giving you a starting point rather than a magic black box you can't interpret.

**Free Resources:** https://docs.databricks.com/en/machine-learning/automl/index.html — Databricks AutoML documentation covering supported problem types and output interpretation

---

## Foundation Model APIs

**Status:** ⬜ Not Started

**Definition:** Databricks Foundation Model APIs provide serverless, pay-per-token access to open LLMs (Llama 3, DBRX, Mixtral) hosted on Databricks infrastructure, billed through the existing Databricks account. This enables LLM-powered applications without separate API providers or infrastructure.

**Mental Model:** Foundation Model APIs are the LLM vending machine inside Databricks — you call the same endpoint style as external APIs, but the models run on Databricks infrastructure, the cost flows through your existing Databricks account, and data stays within your cloud.

**Free Resources:** https://docs.databricks.com/en/machine-learning/foundation-models/index.html — Databricks Foundation Model API documentation covering supported models and usage

---

## Vector Search

**Status:** ⬜ Not Started

**Definition:** Databricks Vector Search is a managed vector database integrated with Unity Catalog that stores and indexes embedding vectors for semantic similarity search. It is purpose-built for RAG applications within Databricks, automatically syncing with Delta Lake source tables as they update.

**Mental Model:** Vector Search is the semantic search index that lives inside your Databricks environment — instead of building a separate vector database, you point it at a Delta table of documents and it maintains a continuously updated searchable index.

**Free Resources:** https://docs.databricks.com/en/generative-ai/vector-search.html — Databricks Vector Search documentation covering index creation, sync, and query patterns
