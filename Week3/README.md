# Week 3 - Project 1: Core Azure Pipeline
### Module: Big Data with Azure

---

## Overview

This is the first project week. No new topics are taught. You combine what you learned in Weeks 1 and 2 into one working pipeline that takes raw files all the way to analysis-ready gold tables, started with a single trigger. By Friday, you'll have a bronze, silver and gold pipeline running from Azure Data Factory, plus a short write-up of your design choices.

**Prerequisites:** Weeks 1 and 2 completed. A working Azure account with a budget alert, and comfort with SQL, Python and basic PySpark.

**Tools used this week:** Azure portal, Azure CLI, ADLS Gen2, Azure Data Factory, Azure Databricks (PySpark and Delta Lake), Azure Key Vault, VS Code.

---

## Structure

Each day has one file:
- **`tasks.md`** - the day's work, the expected output, and an end-of-day checklist

```
week-3/
├── README.md          (this file)
├── day-1/
│   └── tasks.md
├── day-2/
│   └── tasks.md
├── day-3/
│   └── tasks.md
├── day-4/
│   └── tasks.md
└── day-5/
    └── tasks.md
```

---

## The Pipeline at a Glance

```
Source files
  -> ADF copy activity        -> bronze/  raw files, unchanged, date-partitioned
  -> Databricks silver notebook -> silver/  clean Delta tables
  -> Databricks gold notebook   -> gold/    business tables (Delta)

One ADF pipeline chains all three steps, started by a schedule trigger.
```

### What belongs in each layer

| Layer | Contents | Format |
|---|---|---|
| Bronze | Raw data exactly as received, in date-partitioned folders | Original files |
| Silver | Schema enforced, types fixed, nulls handled, duplicates and bad records removed | Delta |
| Gold | Aggregated business tables, joined with reference data | Delta |

### Project rules

- Never edit bronze. If silver is wrong, fix the notebook and reprocess from bronze.
- Use the naming convention from Week 1: `<type>-<project>-<env>`.
- Keep all resources in one resource group and one region.
- Keep secrets in Key Vault. Never put keys or connection strings in code or in GitHub.
- Set auto-terminate on every Databricks cluster.

---

## Day-by-Day Breakdown

### Day 1 - Plan and Provision
Pick the dataset and write down the business questions the gold tables should answer. Sketch the architecture. Create the resource group, ADLS Gen2 with bronze, silver and gold containers, Data Factory, Databricks workspace and Key Vault. Confirm the budget alert is active.
**Output:** Architecture sketch and a working empty environment.

### Day 2 - Bronze
Build an ADF pipeline with linked services, parameterized datasets and a copy activity. Land the raw files in date-partitioned folders without changing them.
**Output:** Raw data in `bronze/` with a run history in ADF.

### Day 3 - Silver
Write a Databricks notebook that enforces the schema, fixes types, handles nulls, removes duplicates and filters bad records. Add basic data quality checks. Write the result as Delta, partitioned sensibly.
**Output:** Clean Delta tables in `silver/` and a short data quality report.

### Day 4 - Gold
Aggregate silver into business tables such as daily totals, top categories and trends. Join reference data where useful. Write as Delta and run OPTIMIZE.
**Output:** Analysis-ready tables in `gold/`.

### Day 5 - Chain, Schedule, Present
Chain everything in one ADF pipeline: copy, then the silver notebook, then the gold notebook. Add a schedule trigger and test a full run. Write the summary and give an informal walkthrough.
**Output:** Working pipeline and a short write-up.

---

## Project 1 Deliverable

- A working bronze, silver and gold pipeline triggered from ADF
- A short written summary covering:
  - Partitioning choices
  - File formats
  - Cleaning rules
  - What each gold table answers

---

## By End of Week 3, You Should Be Able To:

- [ ] Design a bronze, silver and gold layout for a real dataset
- [ ] Build an ADF copy activity with linked services and parameterized datasets
- [ ] Clean and validate data in a Databricks notebook and write it as Delta
- [ ] Build gold tables that answer specific business questions
- [ ] Chain ADF and Databricks steps into one pipeline with a schedule trigger
- [ ] Explain your design choices in a short write-up

---

## Looking Ahead

Week 4 is the capstone. You take this pipeline and make it behave like a production one: incremental loading, retries and alerts, secrets in Key Vault, code in GitHub, and a queryable gold layer. Keep your Week 3 resources and code, because Week 4 builds directly on them.