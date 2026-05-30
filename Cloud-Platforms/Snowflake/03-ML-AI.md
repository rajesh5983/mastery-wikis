# Snowflake — ML and AI

---

## Snowpark

**Status:** ⬜ Not Started

**Definition:** Snowpark is Snowflake's developer framework for writing non-SQL data processing logic in Python, Java, or Scala that runs directly within Snowflake's engine. Code is compiled to efficient SQL operations, executed on Snowflake compute, and operates on Snowflake data without moving it outside the platform.

**Key Mental Model:** Snowpark brings the code to the data — instead of exporting data to a Python notebook and processing it externally, you write Python that executes inside Snowflake where the data already lives.

**How It Works:**
- Snowpark provides a DataFrame API (modelled on Spark's API) in Python, Java, and Scala. Operations on Snowpark DataFrames are lazy — they build an execution plan that is compiled to SQL and submitted to Snowflake's engine. The local machine acts as the driver; Snowflake executes all the actual compute.
- Snowpark Python runs on Anaconda-managed Python environments inside Snowflake. You specify the required packages from the Snowflake Anaconda channel when creating UDFs, UDTFs, or Stored Procedures. This eliminates dependency management — the runtime environment is defined declaratively and provisioned by Snowflake.
- User-Defined Functions (UDFs) and User-Defined Table Functions (UDTFs) written in Python/Java/Scala are pushed down to Snowflake and executed at the row or table level within the query engine. A Python UDF calling scikit-learn for inference runs the model inside Snowflake's compute, not externally.
- Stored Procedures in Snowpark (as opposed to JavaScript stored procedures) use the Snowpark DataFrame API to build complex multi-step logic — full Python control flow, conditional branching, loops — while executing all data operations within Snowflake. They are the primary pattern for Snowpark-based ELT pipelines.
- Snowpark ML (a separate library built on Snowpark) provides scikit-learn-compatible estimators that run distributed training and inference inside Snowflake. Models trained with Snowpark ML are serialised and stored in Snowflake's Model Registry for versioned deployment.

**Common Misconceptions:**
- Snowpark is equivalent to Spark — Snowpark provides a Spark-like DataFrame API but runs on Snowflake's SQL engine (not a distributed Spark cluster); it lacks Spark's streaming capabilities and some advanced ML algorithms; it is best suited for batch transformations and model inference, not large-scale streaming or deep learning training.
- Snowpark Python runs locally — when you call `.collect()` or run a UDF, the execution happens inside Snowflake's compute; only the driver logic (building the plan) runs locally; network bandwidth to Snowflake is not a constraint because data is never pulled to the client during execution.

**Interview Answer Skeleton:**
- **What it is:** A DataFrame-based developer framework (Python, Java, Scala) that compiles code to SQL and executes it within Snowflake's engine — enabling Python ML workflows, UDFs, and complex stored procedures without exporting data outside the platform.
- **Why it matters / trade-offs:** Snowpark enables data scientists and engineers to use Python tooling on Snowflake data without the ETL overhead of exporting to an external compute environment, keeping data governance and compute costs within Snowflake. The trade-off is that the Anaconda channel constrains available packages and Snowpark ML cannot match the breadth of external ML frameworks for training complex models.
- **Example or context:** A fraud detection team trains a scikit-learn model using Snowpark ML directly on their transactions table in Snowflake, registers it in the Model Registry, and deploys it as a vectorised UDF. New transactions are scored at query time via `SELECT predict_fraud(features)` — no Sagemaker, no external inference endpoint, no data movement.

**Free Resources:**
- [Snowpark Developer Documentation](https://docs.snowflake.com/en/developer-guide/snowpark/index) — Snowpark documentation covering the Python, Java, and Scala APIs, UDF development, and Stored Procedures
- [Snowpark Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart guides for building Snowpark pipelines and ML workflows inside Snowflake

---

## Cortex LLM Functions

**Status:** ⬜ Not Started

**Definition:** Snowflake Cortex LLM Functions provide serverless access to frontier LLMs (Llama, Mistral, Mixtral, Claude) directly from SQL using functions like COMPLETE(), SENTIMENT(), SUMMARIZE(), TRANSLATE(), and EXTRACT_ANSWER(). LLM inference runs on Snowflake compute with no external API calls.

**Key Mental Model:** Cortex LLM Functions bring AI into SQL — you call COMPLETE('mistral-7b', 'Summarise this: ' || review_text) just like any SQL function, and Snowflake runs the inference internally on Snowflake infrastructure.

**How It Works:**
- Cortex LLM Functions are invoked as SQL functions within SELECT statements, operating row-by-row over table data. `SNOWFLAKE.CORTEX.COMPLETE('mistral-7b', 'Classify sentiment: ' || review)` runs LLM inference on each row's review text and returns the response as a string — batchable across millions of rows within a single SQL query.
- Task-specific functions are pre-built prompt templates: `SENTIMENT(text)` returns a score from -1 to 1; `SUMMARIZE(text)` generates a concise summary; `TRANSLATE(text, source_lang, target_lang)` translates text; `EXTRACT_ANSWER(context, question)` runs extractive QA. These require no prompt engineering and are optimised for their specific tasks.
- `COMPLETE()` accepts a model name and a full prompt string, giving maximum flexibility — you construct the system and user message, inject data from columns, and control the instruction precisely. Response format follows the provider's completion format (plain text or JSON if instructed).
- Billing is per-token: Cortex credits are consumed based on input + output tokens per function call. Running COMPLETE over a million-row table at 500 tokens per call consumes 500M tokens — significant cost that must be estimated before running at scale. Cortex credits differ from compute warehouse credits.
- Cortex functions run on Snowflake-managed infrastructure in the same region as your account. Data never leaves Snowflake's environment — this is the primary differentiator from calling an external LLM API, which would require an External Network Access integration and transmit data externally.

**Common Misconceptions:**
- Cortex LLM Functions support all frontier models — Cortex offers a curated set of models (Llama 3, Mistral, Mixtral, Arctic); GPT-4 and Claude are not available directly in Cortex; for those models, you would call their external APIs via External Functions, which does transmit data outside Snowflake.
- Cortex functions are free or included in compute credits — Cortex functions are billed separately in Cortex credits based on token consumption; running LLM inference at scale incurs substantial cost that must be planned for independently of warehouse compute budgets.

**Interview Answer Skeleton:**
- **What it is:** Serverless SQL functions (COMPLETE, SENTIMENT, SUMMARIZE, TRANSLATE, EXTRACT_ANSWER) that run LLM inference within Snowflake's managed infrastructure — enabling AI-augmented analytics directly in SQL without external API calls or data movement.
- **Why it matters / trade-offs:** Cortex LLM Functions eliminate the data export-and-inference pattern for AI enrichment at scale — classify, summarise, or extract from table data with a single SQL query, while data stays within Snowflake's governance boundary. The trade-off is cost at scale (token-based billing adds up quickly) and limited model choice compared to external APIs.
- **Example or context:** A support operations team runs `SELECT ticket_id, SNOWFLAKE.CORTEX.SENTIMENT(ticket_text) AS sentiment, SNOWFLAKE.CORTEX.SUMMARIZE(ticket_text) AS summary FROM support_tickets WHERE created_date = CURRENT_DATE` — enriching the day's 50,000 tickets with sentiment and summary in a single scheduled query, feeding a Power BI dashboard without any Python or API infrastructure.

**Free Resources:**
- [Snowflake Cortex LLM Functions Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions) — Snowflake Cortex LLM functions documentation covering available models, function syntax, and token billing
- [Snowflake Cortex AI Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for using Cortex LLM functions for text classification, summarisation, and extraction in SQL

---

## ML Functions

**Status:** ⬜ Not Started

**Definition:** Snowflake ML Functions provide built-in, no-code machine learning capabilities accessed via SQL: FORECAST (time-series prediction), ANOMALY_DETECTION, CLASSIFICATION, CONTRIBUTION_EXPLORER (feature importance), and SENTIMENT. These run on Snowflake compute without exporting data.

**Key Mental Model:** ML Functions are AI appliances in Snowflake's kitchen — pre-built, ready to use, no setup required. You call a function on your data and get predictions or anomalies back as a result set.

**How It Works:**
- ML Functions follow a two-step pattern: train a model object on historical data, then call the model on new data. `CREATE SNOWFLAKE.ML.FORECAST(...) FROM (SELECT date, revenue FROM training_data)` creates a trained forecasting model object; `CALL model!FORECAST(...)` then generates predictions — the model is persisted as a named object in the schema.
- `FORECAST` uses an ensemble of time-series algorithms (autoregression, seasonal decomposition) trained on the provided historical series. It outputs predicted values with confidence intervals for a specified horizon. Multiple series can be trained in a single call using a series identifier column.
- `ANOMALY_DETECTION` identifies statistical outliers in time-series or tabular data. You provide training data representing normal behaviour; the trained model then flags anomalous rows when called on new data. Both supervised (with explicit anomaly labels) and unsupervised modes are supported.
- `CLASSIFICATION` trains a binary or multi-class classifier (based on gradient-boosted trees) on labelled tabular data. The trained model object exposes `PREDICT()` for scoring new rows and `EXPLAIN()` for feature importance — providing interpretability alongside predictions.
- All ML Function model objects are Snowflake schema objects with full RBAC governance — access to train, use, or inspect a model is controlled by role grants, ensuring ML workflows comply with the same data governance model as tables and views.

**Common Misconceptions:**
- ML Functions are equivalent to custom ML model training — ML Functions are purpose-built, parameter-limited models for specific tasks; they offer no hyperparameter tuning, custom algorithm selection, or neural network architectures; for those needs, Snowpark ML or external ML platforms are necessary.
- ML Functions require ML expertise to use — ML Functions are explicitly no-code; the SQL interface abstracts all algorithm selection and training decisions; the primary skill required is understanding the business problem and data preparation, not ML methodology.

**Interview Answer Skeleton:**
- **What it is:** SQL-accessible, no-code ML models (FORECAST, ANOMALY_DETECTION, CLASSIFICATION, CONTRIBUTION_EXPLORER) that train on Snowflake data and produce predictions as SQL result sets — managed as Snowflake schema objects with full RBAC governance.
- **Why it matters / trade-offs:** ML Functions lower the barrier for ML-augmented analytics — data analysts with SQL skills can add forecasting and anomaly detection without a data science team or ML infrastructure. The trade-off is limited customisation: algorithm choice, hyperparameters, and model architecture are fixed; complex or specialised ML needs require Snowpark ML or external platforms.
- **Example or context:** A retail analytics team uses `SNOWFLAKE.ML.FORECAST` to train a revenue forecast model on 3 years of daily sales data by product category, then generates a 90-day forward forecast with confidence intervals — all in SQL, with the forecast model stored as a schema object that the BI team queries directly from Tableau via a SQL view.

**Free Resources:**
- [Snowflake ML Functions Documentation](https://docs.snowflake.com/en/guides-overview-ml-functions) — Snowflake ML Functions overview covering FORECAST, ANOMALY_DETECTION, CLASSIFICATION, and CONTRIBUTION_EXPLORER with syntax and examples
- [Snowflake ML Functions Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for building time-series forecasts and anomaly detection models using Snowflake ML Functions

---

## Feature Store

**Status:** ⬜ Not Started

**Definition:** The Snowflake Feature Store (via Snowpark ML) provides a centralised repository for ML features computed in Snowflake. It manages feature definitions, entity linkage, point-in-time retrieval for training sets, and feature serving for online inference — all within Snowflake's governance framework.

**Key Mental Model:** The Snowflake Feature Store is the ingredient library for ML models — features are computed and stored once in Snowflake, then any model training job retrieves the right features at the right historical point in time.

**How It Works:**
- A Feature Store is created as a Snowflake schema object. `FeatureStore(session, database, schema, warehouse)` initialises the store. Feature values are stored in Snowflake tables managed by the store, versioned and tracked as `FeatureView` objects linked to `Entity` definitions (e.g., `customer_id` as the entity for all customer-level features).
- `FeatureView` objects define the SQL or Snowpark transformation that computes a set of related features for an entity. The transformation is registered as a materialised or streaming view; the Feature Store schedules refresh on a configured cadence to keep features current.
- Point-in-time correct training dataset generation is the primary value: `feature_store.generate_dataset(spine_df, features)` joins the entity keys in the spine (with timestamps) to the feature history at the exact timestamp of each training example — preventing target leakage from future feature values bleeding into historical training rows.
- Feature serving for online inference reads the latest feature values by entity key. For low-latency serving, feature values can be published to an external key-value store (Redis, DynamoDB) via the Feature Store's publishing integration, keeping Snowflake offline and the KV store online.
- The Feature Store integrates with the Snowflake Model Registry — a trained model can reference the feature views it depends on, creating a lineage link between the model and the features used in training, enabling impact analysis when features change.

**Common Misconceptions:**
- The Snowflake Feature Store provides sub-millisecond online serving — the Snowflake Feature Store is primarily an offline store; online serving at low latency requires publishing to an external KV store; the store itself is optimised for batch feature generation and training dataset creation, not real-time request serving.
- Feature Stores eliminate the need for feature engineering — a Feature Store organises and governs computed features; the engineering work of defining, computing, and validating features is unchanged; the Feature Store ensures that work is reusable, versioned, and shared across teams rather than duplicated.

**Interview Answer Skeleton:**
- **What it is:** A Snowflake schema-managed repository (via Snowpark ML) of versioned ML features linked to entity definitions — providing point-in-time correct training dataset generation, feature reuse across teams, and lineage tracking between features and trained models.
- **Why it matters / trade-offs:** Feature Stores prevent the most common ML training correctness bug (target leakage from future features), centralise feature computation to eliminate duplication across teams, and enable model reproducibility by recording exactly which feature versions a model was trained on. The trade-off is setup overhead — defining entities, feature views, and refresh schedules requires ML engineering investment before the value accrues.
- **Example or context:** A lending platform's risk team registers customer behavioural features (30-day transaction count, average balance, days since last login) as a FeatureView on the customer entity. The credit scoring team generates training datasets with `generate_dataset(loan_applications_spine, [behavioural_features])` — point-in-time correct features at each application's timestamp — without duplicating the feature computation logic already owned by the risk team.

**Free Resources:**
- [Snowflake Feature Store Documentation](https://docs.snowflake.com/en/developer-guide/snowflake-ml/feature-store/overview) — Snowflake Feature Store documentation covering entities, feature views, dataset generation, and online serving integration
- [Snowpark ML Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for building ML pipelines with Snowpark ML including feature engineering and model training

---

## Document AI

**Status:** ⬜ Not Started

**Definition:** Snowflake Document AI uses vision models to extract structured data from unstructured documents (PDFs, images, invoices, contracts) stored in Snowflake Stages. You build a custom extraction model by providing examples, and the model runs inference on new documents within Snowflake.

**Key Mental Model:** Document AI is a trained document reader inside Snowflake — you show it examples of what to extract (invoice totals, vendor names, dates), and it learns to extract those fields from new documents automatically.

**How It Works:**
- Document AI models are created through a guided UI or SQL API where you define the extraction schema (field names, data types, whether values can be multi-value) and upload example documents with annotated ground-truth extractions. The model is trained on these examples within Snowflake using vision-language model fine-tuning.
- Once trained, the model is deployed as a Snowflake schema object. Inference is triggered via SQL: `SELECT SNOWFLAKE.CORTEX.DOCUMENT_AI!PREDICT(TO_FILE('@stage/invoice.pdf'))` returns a structured JSON object containing the extracted fields and their confidence scores.
- Source documents must be stored in Snowflake Stages (internal or external). The document processing pipeline is: upload to stage → call PREDICT → parse JSON output → insert structured results into target tables. This keeps the entire pipeline within Snowflake's governance boundary.
- Confidence scores on each extracted field allow routing: high-confidence extractions go directly to the target table; low-confidence extractions are flagged for human review in a separate queue. This human-in-the-loop pattern is essential for financial and legal document workflows where extraction errors have compliance consequences.
- Document AI is optimised for semi-structured documents with variable layouts (invoices from different vendors, contracts with varying clause positions). For highly standardised forms with fixed field positions, traditional OCR tools may be faster and cheaper.

**Common Misconceptions:**
- Document AI works equally well on all document types without training examples — Document AI requires representative training examples for each document type and field; a model trained on standard invoices will not generalise well to medical claim forms without additional training data for that document type.
- Document AI extracts data in real time as documents are uploaded — Document AI inference is a batch process triggered by SQL calls; integrating it into a real-time upload workflow requires an orchestration layer (Task, Snowpipe trigger) to schedule inference runs after document arrival.

**Interview Answer Skeleton:**
- **What it is:** A Snowflake-native document extraction capability that trains custom vision-language models on example documents and annotated fields, then runs inference via SQL against new documents stored in Snowflake Stages — returning structured JSON with per-field confidence scores.
- **Why it matters / trade-offs:** Document AI enables automated extraction from high-volume unstructured documents without building a separate ML pipeline or external OCR integration — the model training, inference, and output storage are all within Snowflake's governance boundary. The trade-off is that training data quality and coverage directly determine extraction accuracy; poor or insufficient examples produce unreliable models for edge-case documents.
- **Example or context:** A procurement team processes 10,000 supplier invoices monthly. Document AI is trained on 200 annotated invoices covering their top vendors. At month-end, a Task runs `PREDICT` against all new invoice files in the stage, inserts high-confidence extractions (confidence > 0.85) into the ERP feed table, and routes low-confidence rows to a human review dashboard — reducing manual data entry by 80% with a clear quality gate.

**Free Resources:**
- [Snowflake Document AI Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/document-ai/overview) — Snowflake Document AI documentation covering model building, inference, confidence scoring, and integration patterns
- [Snowflake Document AI Quickstart](https://quickstarts.snowflake.com) — hands-on quickstart for building and deploying a Document AI extraction model in Snowflake

---

## Arctic LLM

**Status:** ⬜ Not Started

**Definition:** Snowflake Arctic is Snowflake's open-source LLM optimised for enterprise tasks — SQL generation, coding, and structured data extraction. It uses a Mixture-of-Experts (MoE) architecture with a dense transformer and expert layers, achieving strong performance on enterprise benchmarks at relatively low inference cost.

**Key Mental Model:** Arctic is Snowflake's purpose-built LLM for data work — trained on SQL, code, and structured data tasks rather than general conversation, making it efficient and effective for data engineering and analytics use cases.

**How It Works:**
- Arctic uses a hybrid MoE architecture: a 10B parameter dense transformer handles general reasoning, plus 128 expert layers of 3.66B parameters each, with only the top-2 experts activated per token. Total parameters: ~480B; active parameters per forward pass: ~17B. This sparse activation enables high capacity at lower inference cost than a dense 480B model.
- Training focused on enterprise-relevant tasks: SQL generation (including complex multi-table queries and DDL), Python coding, instruction following with structured output (JSON schemas), and document comprehension. The training data mix prioritises these domains relative to general web text.
- Arctic weights are released under the Apache 2.0 licence — fully permissive for commercial use, self-hosting, and fine-tuning without Snowflake dependency. The weights are available on Hugging Face (snowflake/snowflake-arctic-instruct).
- Within Snowflake Cortex, Arctic is available as a selectable model in `SNOWFLAKE.CORTEX.COMPLETE('snowflake-arctic', ...)`. It is the lowest-cost model option in Cortex for tasks where its enterprise training alignment matches the use case.
- Self-hosting Arctic requires significant infrastructure: the full model needs distributed serving across multiple high-memory GPUs (A100 or H100 class). Tools like vLLM with MoE support handle the expert routing and distributed inference. For most enterprise users, Cortex consumption is more practical than self-hosting.

**Common Misconceptions:**
- Arctic is Snowflake's general-purpose LLM competing with GPT-4 — Arctic is deliberately specialised for enterprise data tasks; it is not designed to compete on general chat or creative writing benchmarks, and its benchmark strengths are concentrated on SQL, coding, and structured extraction tasks.
- Arctic's MoE architecture makes it faster than dense models of similar parameter count — the active parameter count (17B) does determine inference cost, but MoE models have expert routing overhead and load-balancing complexity that increases serving infrastructure requirements compared to equivalently-sized dense models.

**Interview Answer Skeleton:**
- **What it is:** Snowflake's open-source, enterprise-focused LLM using a hybrid Mixture-of-Experts architecture (480B total, 17B active parameters) — trained on SQL, coding, and structured extraction tasks, available via Snowflake Cortex or self-hosted under Apache 2.0.
- **Why it matters / trade-offs:** Arctic provides a Snowflake-native, cost-optimised LLM option for data and analytics tasks (SQL generation, query explanation, structured extraction) with permissive open-source licensing that allows self-hosting without API dependency. The trade-off is narrow specialisation — for general reasoning, long-context document analysis, or creative tasks, frontier models significantly outperform Arctic.
- **Example or context:** A Snowflake Cortex SQL assistant uses Arctic to generate SQL from natural language questions over a metadata-described schema, costing a fraction of GPT-4 per query. For straightforward single-table queries and aggregations, Arctic's SQL accuracy matches frontier models on that domain; for complex multi-CTE analytical queries, a frontier model is selected via a routing layer based on query complexity score.

**Free Resources:**
- [Snowflake Arctic Model Page](https://www.snowflake.com/en/data-cloud/arctic/) — Snowflake Arctic model page with architecture overview, benchmark comparisons, and access instructions
- [Snowflake Cortex AI Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions) — documentation for using Arctic and other models via Cortex LLM functions in SQL
