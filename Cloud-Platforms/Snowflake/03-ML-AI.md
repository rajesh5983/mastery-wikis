# Snowflake — ML and AI

---

## Snowpark

**Status:** ⬜ Not Started

**Definition:** Snowpark is Snowflake's developer framework for writing non-SQL data processing logic in Python, Java, or Scala that runs directly within Snowflake's engine. Code is compiled to efficient SQL operations, executed on Snowflake compute, and operates on Snowflake data without moving it outside the platform.

**Mental Model:** Snowpark brings the code to the data — instead of exporting data to a Python notebook and processing it externally, you write Python that executes inside Snowflake where the data already lives.

**Free Resources:** https://docs.snowflake.com/en/developer-guide/snowpark/index — Snowpark documentation covering Python, Java, and Scala APIs

---

## Cortex LLM Functions

**Status:** ⬜ Not Started

**Definition:** Snowflake Cortex LLM Functions provide serverless access to frontier LLMs (Llama, Mistral, Mixtral, Claude) directly from SQL using functions like COMPLETE(), SENTIMENT(), SUMMARIZE(), TRANSLATE(), and EXTRACT_ANSWER(). LLM inference runs on Snowflake compute with no external API calls.

**Mental Model:** Cortex LLM Functions bring AI into SQL — you call COMPLETE('mistral-7b', 'Summarise this: ' || review_text) just like any SQL function, and Snowflake runs the inference internally on Snowflake infrastructure.

**Free Resources:** https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions — Snowflake Cortex LLM functions documentation covering available models and functions

---

## ML Functions

**Status:** ⬜ Not Started

**Definition:** Snowflake ML Functions provide built-in, no-code machine learning capabilities accessed via SQL: FORECAST (time-series prediction), ANOMALY_DETECTION, CLASSIFICATION, CONTRIBUTION_EXPLORER (feature importance), and SENTIMENT. These run on Snowflake compute without exporting data.

**Mental Model:** ML Functions are AI appliances in Snowflake's kitchen — pre-built, ready to use, no setup required. You call a function on your data and get predictions or anomalies back as a result set.

**Free Resources:** https://docs.snowflake.com/en/guides-overview-ml-functions — Snowflake ML Functions overview covering all available functions and their use cases

---

## Feature Store

**Status:** ⬜ Not Started

**Definition:** The Snowflake Feature Store (via Snowpark ML) provides a centralised repository for ML features computed in Snowflake. It manages feature definitions, entity linkage, point-in-time retrieval for training sets, and feature serving for online inference — all within Snowflake's governance framework.

**Mental Model:** The Snowflake Feature Store is the ingredient library for ML models — features are computed and stored once in Snowflake, then any model training job retrieves the right features at the right historical point in time.

**Free Resources:** https://docs.snowflake.com/en/developer-guide/snowflake-ml/feature-store/overview — Snowflake Feature Store documentation covering entities, features, and dataset generation

---

## Document AI

**Status:** ⬜ Not Started

**Definition:** Snowflake Document AI uses vision models to extract structured data from unstructured documents (PDFs, images, invoices, contracts) stored in Snowflake Stages. You build a custom extraction model by providing examples, and the model runs inference on new documents within Snowflake.

**Mental Model:** Document AI is a trained document reader inside Snowflake — you show it examples of what to extract (invoice totals, vendor names, dates), and it learns to extract those fields from new documents automatically.

**Free Resources:** https://docs.snowflake.com/en/user-guide/snowflake-cortex/document-ai/overview — Snowflake Document AI documentation covering model building and inference

---

## Arctic LLM

**Status:** ⬜ Not Started

**Definition:** Snowflake Arctic is Snowflake's open-source LLM optimised for enterprise tasks — SQL generation, coding, and structured data extraction. It uses a Mixture-of-Experts (MoE) architecture with a dense transformer and expert layers, achieving strong performance on enterprise benchmarks at relatively low inference cost.

**Mental Model:** Arctic is Snowflake's purpose-built LLM for data work — trained on SQL, code, and structured data tasks rather than general conversation, making it efficient and effective for data engineering and analytics use cases.

**Free Resources:** https://www.snowflake.com/en/data-cloud/arctic/ — Snowflake Arctic model page with architecture overview and benchmark comparisons
