# Module: Big Data with Azure (4 Weeks)

---

## Module Overview

This module teaches how data is stored, processed and delivered at scale on Azure. The first two weeks build the skills: Azure and data lake fundamentals, then distributed processing with Spark and pipeline orchestration. The last two weeks are project weeks, where one complete Azure pipeline is built, run and then hardened.

Four weeks, one continuous build: Azure foundations -> data lake and big data concepts -> Spark processing and orchestration -> Project 1 (core pipeline) -> Capstone (production-style pipeline). Two projects are built into the module, so there is working proof of progress, not only exercises.

**Tech stack used throughout:** Azure Data Lake Storage Gen2, Azure Data Factory, Azure Databricks (PySpark, Delta Lake), Azure Key Vault, Azure Monitor, GitHub, SQL (Databricks SQL or Synapse serverless) for serving.

---

## Prerequisites

- Comfortable writing SQL, including joins, aggregations, and window functions
- Basic Python and pandas
- No prior Azure or Spark experience required. Week 1 starts from the ground up on that side.

---

## Cost and Safety Rules (apply all four weeks)

- Create one resource group per project and delete it when the project ends.
- Set a budget alert on the subscription on day 1.
- Use the smallest Databricks cluster (single node) and set auto-terminate to 20 minutes or less.
- Keep all secrets in Key Vault. Never commit keys, connection strings or passwords to GitHub.

---

## Weekly Breakdown

### Week 1 - Azure and Big Data Foundations
**Topics:** Cloud basics (subscriptions, resource groups, regions), access control (RBAC, service principals, managed identity), cost management, Azure CLI and portal. Big data concepts: volume/velocity/variety, distributed processing, partitioning, batch vs streaming, data lake vs data warehouse, medallion architecture (bronze, silver, gold). Storage: Blob and ADLS Gen2, hierarchical namespace, containers and folder design, file formats (CSV, JSON, Parquet) and why Parquet wins at scale.
**Hands-on tie-in:** Create a storage account with ADLS Gen2, set up bronze/silver/gold containers, upload a raw dataset, and read and write files from Python using the Azure SDK.
**By end of week:** Can set up a secure data lake on Azure, explain the medallion layout, and compare file formats with real size and speed numbers from their own data.

### Week 2 - Processing at Scale: Spark, Delta Lake and Data Factory
**Topics:**
- Azure Databricks: workspace, clusters, notebooks, connecting to ADLS securely with secrets from Key Vault.
- PySpark: DataFrames, transformations vs actions, joins, aggregations, window functions, partitioning and shuffles, reading the Spark UI at a basic level.
- Delta Lake: why it replaces plain Parquet, ACID writes, schema enforcement, MERGE (upsert), time travel, OPTIMIZE.
- Azure Data Factory: linked services, datasets, pipelines, copy activity, parameters, triggers, running a Databricks notebook from a pipeline, monitoring runs.
**Hands-on tie-in:** Take the raw dataset from Week 1, clean it in PySpark, write it as a Delta table, then build an ADF pipeline that copies new files in and runs the notebook.
**By end of week:** Can process a multi-million row dataset in PySpark, write and update Delta tables, and orchestrate the steps with a parameterized ADF pipeline.

### Week 3 - Project 1: Core Azure Pipeline
Build the first complete version of the pipeline described in the next section. The goal is a pipeline that runs end to end from raw files to gold tables with one trigger.

**Day plan**

| Day | Work | Output |
|---|---|---|
| 1 | Pick the dataset and business questions. Design the architecture. Provision the resource group, ADLS Gen2 (bronze/silver/gold), Data Factory, Databricks workspace, Key Vault. Set naming rules and the budget alert. | Architecture sketch and a working empty environment |
| 2 | Bronze: ADF pipeline with linked services, parameterized datasets and a copy activity. Land raw files in date-partitioned folders, unchanged. | Raw data in `bronze/` with a run history in ADF |
| 3 | Silver: Databricks notebook that enforces the schema, fixes types, handles nulls, removes duplicates, filters bad records, and adds basic data quality checks. Write as Delta, partitioned sensibly. | Clean Delta tables in `silver/` and a short data quality report |
| 4 | Gold: aggregate into business tables (for example daily totals, top categories, trends), join reference data, write as Delta. Run OPTIMIZE. | Analysis-ready tables in `gold/` |
| 5 | Chain it all in one ADF pipeline: copy -> silver notebook -> gold notebook. Add a schedule trigger, test a full run, write the summary, give an informal walkthrough. | Working pipeline and a short write-up |

**Project 1 deliverable:** Working bronze -> silver -> gold pipeline triggered from ADF, plus a short written summary of design choices (partitioning, formats, cleaning rules) and what the gold tables answer.

### Week 4 - Capstone: Production-Style Pipeline
Take the Week 3 pipeline and make it behave like a real one: new data arrives daily, failures are caught, secrets are managed, and results are served and documented. Independent project work with mentor check-ins, not new topics.

**Day plan**

| Day | Work | Output |
|---|---|---|
| 1 | Incremental loading: process only new files (date or watermark parameter in ADF), and replace full rewrites with Delta MERGE in silver. Make the pipeline safe to re-run. | Re-runnable incremental pipeline |
| 2 | Reliability: retries and failure paths in ADF, logging in notebooks, row-count checks between layers, alert on failure through Azure Monitor. | Failure test passed with an alert received |
| 3 | Security and delivery: managed identity or service principal with least-privilege RBAC, all secrets in Key Vault, notebooks and ADF definitions in GitHub. | Repo with pipeline code and no exposed secrets |
| 4 | Performance and serving: tune partitioning and file sizes, compare run times before and after, expose gold tables through SQL queries or a simple dashboard. Review cost per run. | Timing comparison and a queryable gold layer |
| 5 | Documentation, README, final presentation. Delete the resource group after the demo. | Finished repo and a presented project |

**Capstone deliverable:** A portfolio-ready repo containing the data source reference, ADF pipeline definitions, Databricks notebooks, SQL or dashboard for serving, a README (approach, findings, how to run, cost per run), and a recorded or live demo.

**Optional stretch:** Add a streaming source with Event Hubs, or an orchestrated second dataset joined in gold.

---

## The Azure Pipeline (built in Weeks 3 and 4)

```
Source files (CSV/JSON, new files daily)
          |
Azure Data Factory   - copy activity, parameters, schedule trigger
          |
ADLS Gen2  bronze/   - raw data, never modified
          |
Azure Databricks     - PySpark notebook: clean, type, deduplicate, validate
          |
ADLS Gen2  silver/   - cleaned Delta tables
          |
Azure Databricks     - PySpark notebook: aggregate, join, model for questions
          |
ADLS Gen2  gold/     - business-ready Delta tables
          |
Serving: Databricks SQL or Synapse serverless SQL -> dashboard / queries

Across all stages: Key Vault (secrets), RBAC (access), Azure Monitor (alerts),
GitHub (version control), budget alert (cost)
```

**Stage by stage**

| Stage | Azure service | Input | Output | What to check |
|---|---|---|---|---|
| Ingest | Data Factory | Source files | `bronze/<dataset>/<yyyy>/<mm>/<dd>/` | File count and size match the source |
| Clean | Databricks (PySpark) | bronze files | silver Delta tables | Schema enforced, duplicates removed, bad rows counted and logged |
| Model | Databricks (PySpark) | silver tables | gold Delta tables | Totals in gold reconcile with silver |
| Orchestrate | Data Factory | Trigger | Ordered runs of all stages | Retries work, failure raises an alert |
| Serve | Databricks SQL or Synapse serverless | gold tables | Queries and dashboard | Business questions answered correctly |
| Secure and watch | Key Vault, RBAC, Monitor | Whole pipeline | No secrets in code, alerts on failure | Least-privilege access verified |

**Suggested datasets:** NYC Taxi trip data, or any public dataset with several million rows and a date column, so daily incremental loads can be simulated by splitting it into daily files.

**Suggested repo layout**

```
azure-pipeline/
  adf/            pipeline, dataset and trigger definitions (JSON)
  notebooks/      01_bronze_to_silver, 02_silver_to_gold
  sql/            serving queries
  docs/           architecture diagram, data quality report
  README.md
```

---

## Projects at a Glance

| | Project 1 (Week 3) | Capstone (Week 4) |
|---|---|---|
| **Scope** | Core pipeline, single dataset | Production-style pipeline |
| **Goal** | Raw files to gold tables with one trigger | Incremental, monitored, secure, documented |
| **Deliverable** | Working pipeline + short write-up | Full repo + README + demo |
| **Presentation** | Informal walkthrough | Presented like a real project: code, reasoning, results, cost |

---

## What You'll Be Able to Do After This Module

- Design and secure a data lake on Azure using the medallion layout
- Process large datasets with PySpark and Delta Lake on Databricks
- Build parameterized, scheduled pipelines in Azure Data Factory
- Load data incrementally, handle failures, and monitor runs
- Deliver gold tables for analysis and explain the cost and design trade-offs of the pipeline