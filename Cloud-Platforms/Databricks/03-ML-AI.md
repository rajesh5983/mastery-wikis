# Databricks — ML and AI

---

## MLflow

**Status:** ⬜ Not Started

**Definition:** MLflow is the open-source ML lifecycle platform built into Databricks. It tracks experiments (parameters, metrics, artifacts), manages the model registry (versioning, staging, production promotion), and provides a unified API for logging models from any framework (scikit-learn, PyTorch, HuggingFace, etc.). Databricks hosts a managed MLflow Tracking Server and Model Registry as first-class services within every workspace.

**Key Mental Model:** MLflow is the experiment notebook and model filing system for ML teams — every training run is logged, every model version is tracked, and promoting a model from staging to production is a controlled, audited step.

**How It Works:**
- When `mlflow.log_metric()` or `mlflow.autolog()` is called inside a training script, the MLflow client sends HTTP requests to the **Tracking Server** (hosted by Databricks at the workspace level) which persists run metadata — parameters, metrics, tags — to a backend store (PostgreSQL under the hood in Databricks Managed MLflow).
- Large artifacts (model weights, plots, datasets) are stored separately in **artifact storage** (DBFS or a configured S3/ADLS path) with only the artifact URI stored in the tracking backend, keeping metadata queries fast regardless of model file size.
- The **MLflow Model Registry** wraps the artifact storage path in a versioned lifecycle model: a model version transitions through `None → Staging → Production → Archived` states, with optional approval workflows; the registry URI is embedded in model serving endpoints so they always point to the right version.
- `mlflow.pyfunc` is the universal model flavour — any model packaged in `pyfunc` format can be served, scored, or deployed identically regardless of the underlying framework (sklearn, PyTorch, HuggingFace Transformers), because `pyfunc` wraps the `predict()` method in a standardised interface.
- MLflow's **autolog** feature instruments popular frameworks (sklearn, XGBoost, LightGBM, PyTorch Lightning) to automatically log hyperparameters, metrics, and model artifacts without manual `log_*` calls, reducing boilerplate in experiment code. See [[Cloud-Platforms/Databricks/03-ML-AI#Model Serving]] for how registered models become endpoints.

**Common Misconceptions:**
- MLflow tracking does not store training data — it stores metadata about runs and pointers to artifacts; data versioning requires a separate tool (Delta Lake time travel or a data versioning library like Delta Sharing).
- The open-source MLflow server and Databricks Managed MLflow are compatible at the API level, but Databricks adds Unity Catalog integration, workspace-level access control, and managed infrastructure — running your own open-source MLflow server loses these governance features.

**Interview Answer Skeleton:**
- **What it is:** An open-source ML lifecycle platform with a managed Databricks implementation providing experiment tracking (run metadata to a tracking server), model registry (versioned lifecycle management), and a `pyfunc` universal model format for framework-agnostic deployment.
- **Why it matters / trade-offs:** Standardises experiment tracking across teams and enables reproducibility; the trade-off is that MLflow's tracking API adds latency to training loops and its artifact store can become a storage cost concern for large models logged across many runs.
- **Example or context:** A data science team runs 200 hyperparameter tuning experiments with Hyperopt — MLflow autolog captures every trial's parameters and validation metrics automatically; the best run is registered to the Model Registry and promoted to Production after a review, triggering Model Serving to route traffic to the new version.

**Free Resources:**
- [MLflow Documentation](https://mlflow.org/docs/latest/index.html) — tracking, model registry, deployment, and pyfunc model flavour reference
- [Databricks ML Documentation](https://docs.databricks.com/en/machine-learning/index.html) — Databricks-specific MLflow managed features, Unity Catalog integration, and model serving

---

## Feature Store

**Status:** ⬜ Not Started

**Definition:** The Databricks Feature Store is a centralised repository for computed features — pre-calculated, validated feature values that can be shared across ML models and pipelines. It ensures feature consistency between training and serving, reduces duplicate computation, and tracks feature lineage. Features are backed by Delta Lake tables and governed through Unity Catalog.

**Key Mental Model:** The feature store is a shared ingredient pantry for ML models — teams compute expensive features once, store them centrally, and multiple models reuse the same validated ingredients rather than each computing their own version.

**How It Works:**
- Feature tables are registered Delta Lake tables with a declared **primary key** (entity identifier, e.g., `customer_id`) and optionally a **timestamp key** for time series features; the Feature Store API writes feature values to these tables using standard Delta MERGE operations.
- **Point-in-time correctness** is enforced during training dataset creation: when `FeatureStoreClient.create_training_set()` is called with a training spine (a DataFrame of entity keys with observation timestamps), the Feature Store performs a temporal join that retrieves the feature value that was valid *at the time of each observation*, preventing future feature leakage.
- The point-in-time join is implemented as a `ASOF JOIN` equivalent — for each row in the spine, the Feature Store finds the most recent feature row where `feature_timestamp <= observation_timestamp`, preserving the historical feature values that the model would have seen at prediction time.
- At serving time, the Feature Store client can look up fresh feature values by primary key from the online store (if configured) or from the Delta table directly, ensuring the same feature computation logic is used for both training and inference — eliminating training-serving skew.
- Feature tables are registered in Unity Catalog, so lineage tracking shows which models were trained on which feature versions, and access is governed by the same RBAC policies as other data assets. See [[Cloud-Platforms/Databricks/03-ML-AI#MLflow]] for how training datasets created by the Feature Store are linked to MLflow run metadata.

**Common Misconceptions:**
- The Feature Store does not automatically keep features fresh — teams must build and schedule their own feature engineering pipelines (via DLT, Workflows, or Structured Streaming) to populate the feature tables; the store is the governance and retrieval layer, not the computation engine.
- Point-in-time joins are not free computationally — for large spine DataFrames with many entity keys and many feature tables, the temporal join can be expensive and should be cached or pre-materialised for repeated training runs.

**Interview Answer Skeleton:**
- **What it is:** A Unity Catalog-governed Delta Lake layer with a specialised API that enforces point-in-time correct temporal joins when constructing training datasets, and provides consistent feature lookup at serving time to eliminate training-serving skew.
- **Why it matters / trade-offs:** Solves training-serving skew by using the same feature retrieval logic for both training and inference; the trade-off is operational complexity — teams must build and maintain feature engineering pipelines separately from the store itself.
- **Example or context:** A fraud detection team computes `avg_transaction_amount_7d` as a feature — the Feature Store ensures that when the model was trained on 2024 transaction data, it saw the 7-day average as it existed at each training example's timestamp, not today's average, preventing leakage from future transactions.

**Free Resources:**
- [Databricks Feature Store Documentation](https://docs.databricks.com/en/machine-learning/feature-store/index.html) — feature table creation, point-in-time joins, online store configuration, and training set construction
- [Databricks Academy](https://academy.databricks.com) — free ML engineering courses covering Feature Store patterns and training-serving consistency

---

## Model Serving

**Status:** ⬜ Not Started

**Definition:** Databricks Model Serving provides serverless real-time inference endpoints for MLflow-registered models. Endpoints auto-scale to zero, support traffic splitting between model versions for A/B testing, and integrate with Databricks' monitoring and Unity Catalog governance. Foundation Model APIs on the same infrastructure provide access to hosted LLMs.

**Key Mental Model:** Model Serving is the deployment counter for ML models — you register a model in MLflow, click deploy, and Databricks provisions an auto-scaling REST endpoint with no infrastructure management required.

**How It Works:**
- When an endpoint is created for an MLflow-registered model, Databricks provisions **serverless compute** (Databricks-managed, not customer-cloud VMs) and loads the `pyfunc` model artifact from the MLflow artifact store into memory on warm containers.
- Endpoints **scale to zero** when idle and scale up by adding container instances based on request queue depth — cold-start latency for custom models can be 10–30 seconds; Foundation Model API endpoints are pre-warmed and have near-instant availability.
- **Traffic routing** allows multiple model versions to be assigned percentage-based traffic weights on a single endpoint, enabling canary releases and A/B experiments where a small fraction of live traffic hits a challenger model version while the champion handles the majority.
- The serving endpoint exposes a REST API with a standardised `/invocations` path that accepts the same input formats supported by MLflow pyfunc (`dataframe_split`, `instances`, `inputs`), enabling direct integration with BI tools, application backends, or LangChain chains.
- **Inference Tables** can be enabled per endpoint to log all requests and responses to a Delta table in the customer's account, enabling model drift monitoring, quality auditing, and retraining dataset construction from production traffic. See [[Cloud-Platforms/Databricks/03-ML-AI#Vector Search]] for how endpoints power RAG applications.

**Common Misconceptions:**
- Model Serving endpoints are not GPU-by-default — custom model endpoints use CPU containers unless explicitly configured with GPU instance types; GPU endpoints are required for large deep learning models and have significantly higher DBU costs.
- "Serverless" does not mean instant cold starts for custom models — loading a large scikit-learn or PyTorch model from the artifact store into a fresh container takes time; for latency-sensitive applications, keeping a minimum of one warm instance provisioned is necessary.

**Interview Answer Skeleton:**
- **What it is:** A serverless, auto-scaling REST inference platform that deploys MLflow-registered models as HTTP endpoints with traffic splitting, inference logging, and integration with Databricks' governance layer — no infrastructure provisioning required.
- **Why it matters / trade-offs:** Eliminates custom Kubernetes deployment complexity for model serving; the trade-off is less control over container configuration compared to custom serving infrastructure and potential cold-start latency for infrequently-called endpoints.
- **Example or context:** A retail team deploys a product recommendation model to a Model Serving endpoint with a 90/10 traffic split — 90% goes to the current champion model, 10% to a newly trained challenger; after A/B metrics show the challenger outperforms, they shift to 100% with a single configuration update.

**Free Resources:**
- [Databricks Model Serving Documentation](https://docs.databricks.com/en/machine-learning/model-serving/index.html) — endpoint creation, traffic routing, inference tables, and GPU configuration
- [MLflow Deployment Documentation](https://mlflow.org/docs/latest/deployment/index.html) — MLflow model packaging and pyfunc deployment patterns

---

## AutoML

**Status:** ⬜ Not Started

**Definition:** Databricks AutoML automatically trains and evaluates multiple ML models on a provided dataset, producing a ranked comparison of models with their metrics and generating the best-performing model's training notebook for inspection and customisation. It is transparent glass-box AutoML — the generated notebook shows every preprocessing step and hyperparameter choice, not a sealed black box.

**Key Mental Model:** AutoML is a rapid prototyping assistant — it tries many approaches quickly and shows you what worked and why, giving you a starting point rather than a magic black box you can't interpret.

**How It Works:**
- AutoML takes a Delta Lake table or Spark DataFrame as input, automatically detects the problem type (classification, regression, forecasting) based on the target column's data type and cardinality, and performs **automated feature analysis** including missing value rates, cardinality, and distribution skew.
- Each trial trains a different model configuration (algorithm + hyperparameters) using HyperOpt under the hood — trials are parallelised across the cluster's worker nodes, with each trial logged as a separate MLflow run under an AutoML experiment.
- For each trial, AutoML generates a full **Python notebook** containing the exact preprocessing code, feature engineering steps, and model training code — these notebooks are stored in the workspace and serve as a reproducible starting point for further manual tuning.
- AutoML applies **data splitter best practices** automatically: stratified splits for imbalanced classification, time-based splits for forecasting, and cross-validation for small datasets — reducing common mistakes in baseline model evaluation.
- The AutoML results UI ranks all trials by the primary metric (e.g., validation F1, RMSE), shows a feature importance summary, and links directly to the best trial's MLflow run for artifact inspection. See [[Cloud-Platforms/Databricks/03-ML-AI#MLflow]] for how AutoML runs integrate with the MLflow tracking server.

**Common Misconceptions:**
- AutoML does not perform deep feature engineering — it applies standard preprocessors (scaling, encoding, imputation) but does not discover domain-specific features; it is a baseline generation tool, not a substitute for domain-driven feature engineering.
- The generated notebooks are meant to be the starting point, not the final model — AutoML trials are limited in their hyperparameter search space; significant gains are often achievable by taking the best notebook and running a deeper hyperparameter search manually.

**Interview Answer Skeleton:**
- **What it is:** An automated baseline model generation service that runs parallelised MLflow trials across multiple algorithms and hyperparameter configurations, outputs ranked results, and generates reproducible training notebooks for the best trials.
- **Why it matters / trade-offs:** Accelerates the path from data to a working baseline model and reduces the risk of common ML mistakes (bad splits, unscaled features); the trade-off is that AutoML results are starting points, not production models — they require domain knowledge, custom features, and deeper tuning.
- **Example or context:** A data scientist receives a new churn prediction dataset — instead of spending a week on baseline models, they run AutoML in 30 minutes, get a ranked list of XGBoost and LightGBM trials, inspect the generated notebook to understand the winning approach, then customise feature engineering and run a deeper Hyperopt search from that starting point.

**Free Resources:**
- [Databricks AutoML Documentation](https://docs.databricks.com/en/machine-learning/automl/index.html) — supported problem types, configuration options, and output notebook interpretation
- [Databricks Academy](https://academy.databricks.com) — free ML courses covering AutoML workflows and model customisation patterns

---

## Foundation Model APIs

**Status:** ⬜ Not Started

**Definition:** Databricks Foundation Model APIs provide serverless, pay-per-token access to open LLMs (Llama 3, DBRX, Mixtral) hosted on Databricks infrastructure, billed through the existing Databricks account. This enables LLM-powered applications without managing separate API providers, and data remains within the Databricks cloud environment.

**Key Mental Model:** Foundation Model APIs are the LLM vending machine inside Databricks — you call the same endpoint style as external APIs, but the models run on Databricks infrastructure, the cost flows through your existing Databricks account, and data stays within your cloud.

**How It Works:**
- Requests to Foundation Model API endpoints follow the OpenAI-compatible REST API format (chat completions, embeddings), so existing applications targeting OpenAI can switch to Databricks-hosted models by changing the base URL and API key with no code changes.
- The models run on **Databricks-managed GPU infrastructure** within the same cloud region as the customer's Databricks workspace, meaning input data sent to the API does not leave the cloud provider's network — critical for regulated industries with data residency requirements.
- **Pay-per-token billing** means there is no minimum reserved capacity; small workloads pay only for tokens consumed, while high-throughput workloads that need guaranteed capacity can switch to **Provisioned Throughput** endpoints with reserved token-per-second rates.
- The same Model Serving infrastructure that serves custom MLflow models also serves Foundation Model API endpoints, so the same endpoint monitoring, traffic routing, and inference table logging capabilities apply to LLM calls.
- Foundation Model APIs integrate with Databricks' AI Playground (an interactive chat UI) for prompt testing and with LangChain/LlamaIndex via the `langchain-databricks` provider, enabling RAG pipeline construction fully within the Databricks ecosystem. See [[Cloud-Platforms/Databricks/03-ML-AI#Vector Search]] for embedding and retrieval integration.

**Common Misconceptions:**
- Foundation Model APIs are not fine-tuned by Databricks for the customer's data — they serve the base model weights; fine-tuning on proprietary data requires a separate fine-tuning workflow using Databricks Model Training and then deployment as a custom endpoint.
- "Using Foundation Model APIs means Databricks trains on my data" is false — Databricks does not use customer inference data to train or update model weights; the models are static base checkpoints served via the API.

**Interview Answer Skeleton:**
- **What it is:** An OpenAI-compatible serverless LLM API hosted on Databricks-managed GPU infrastructure, providing pay-per-token access to open models with data residency within the customer's cloud region and billing through the existing Databricks account.
- **Why it matters / trade-offs:** Simplifies LLM adoption for teams already on Databricks by eliminating separate API accounts and keeping data within a governed environment; the trade-off is that hosted open models (Llama, Mixtral) may underperform frontier closed models (GPT-4, Claude) on complex reasoning tasks.
- **Example or context:** A healthcare analytics team builds a clinical note summarisation tool — using Databricks Foundation Model APIs instead of the OpenAI API ensures that patient data processed by the LLM never leaves their Azure region, satisfying their HIPAA data residency requirements.

**Free Resources:**
- [Databricks Foundation Model APIs Documentation](https://docs.databricks.com/en/machine-learning/foundation-models/index.html) — supported models, API format, provisioned throughput, and token billing
- [Databricks Academy](https://academy.databricks.com) — free generative AI courses covering LLM application patterns on Databricks

---

## Vector Search

**Status:** ⬜ Not Started

**Definition:** Databricks Vector Search is a managed vector database integrated with Unity Catalog that stores and indexes embedding vectors for semantic similarity search. It is purpose-built for RAG applications within Databricks, automatically syncing with Delta Lake source tables as they update and returning Unity Catalog-governed results.

**Key Mental Model:** Vector Search is the semantic search index that lives inside your Databricks environment — instead of building a separate vector database, you point it at a Delta table of documents and it maintains a continuously updated searchable index.

**How It Works:**
- A Vector Search index is backed by a **Delta Sync Index** or **Direct Vector Access Index**. For Delta Sync, the index monitors the source Delta table using Change Data Feed and incrementally updates the vector index as new embeddings are written, without requiring a full re-index.
- The index uses **HNSW (Hierarchical Navigable Small World)** approximate nearest-neighbour algorithm to structure the embedding space as a multi-layered proximity graph — at query time, the algorithm navigates from an entry point node through successively more refined graph layers to find approximate nearest neighbours in sub-linear time.
- When a similarity query is issued, the caller provides a query embedding vector (typically generated by calling the embedding model endpoint) along with a `num_results` and optional metadata filters; the index returns the top-k nearest neighbour vectors along with their associated Delta table columns (chunk text, document ID, etc.).
- **Metadata filtering** allows queries to combine semantic similarity with structured filters (e.g., return the 10 most similar documents where `document_type = 'contract'`), implemented by pre-filtering the HNSW candidate set by metadata before ranking by vector distance.
- Vector Search indexes are registered in Unity Catalog as first-class objects, so access to the index is governed by the same RBAC that protects the underlying Delta table — a user who cannot read the source table cannot retrieve results from its vector index. See [[Cloud-Platforms/Databricks/03-ML-AI#Foundation Model APIs]] for embedding generation integration.

**Common Misconceptions:**
- Vector Search does not automatically generate embeddings from raw text — the caller is responsible for running text through an embedding model (using the Foundation Model APIs or an external provider) before writing embedding vectors to the source Delta table; Vector Search stores and indexes pre-computed vectors.
- "Managed vector database" does not mean the index is always perfectly fresh — Delta Sync indexes have a configurable sync pipeline that runs asynchronously; for very latency-sensitive applications, there is a small window where newly added documents are not yet searchable.

**Interview Answer Skeleton:**
- **What it is:** A managed HNSW approximate nearest-neighbour index backed by a Delta Lake source table, governed by Unity Catalog, that syncs incrementally via Change Data Feed and supports hybrid metadata-filtered semantic similarity queries.
- **Why it matters / trade-offs:** Eliminates the need for a separate vector database service (Pinecone, Weaviate) for Databricks-based RAG pipelines and keeps retrieval within the governed data environment; the trade-off is that it is tightly coupled to the Databricks ecosystem and not suited for standalone vector search outside Databricks workloads.
- **Example or context:** A legal team builds a contract Q&A RAG application — document chunks are embedded using the Databricks embedding endpoint and stored in a Delta table; Vector Search maintains a live HNSW index that retrieves the top-5 relevant contract sections for each user question, which are then passed to the LLM endpoint for answer generation.

**Free Resources:**
- [Databricks Vector Search Documentation](https://docs.databricks.com/en/generative-ai/vector-search.html) — index types, Delta sync configuration, query API, and RAG integration patterns
- [Databricks Academy](https://academy.databricks.com) — free generative AI courses covering RAG architecture with Vector Search and Foundation Model APIs

---
