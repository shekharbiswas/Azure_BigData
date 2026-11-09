# Week 4 - Capstone: Production-Style Pipeline
### Module: Big Data with Azure

---

## Overview

This is the second project week. No new topics are taught. You take the pipeline from Week 3 and make it behave like a real one: it loads only new data, handles failures, keeps secrets safe, and serves results you can query. By Friday, you'll have a finished repo, a README and a short demo.

**Prerequisites:** Week 3 completed. Keep your Week 3 resources and code, because everything this week builds on them.

**Tools used this week:** Azure Data Factory, Azure Databricks (PySpark and Delta Lake), Azure Key Vault, Azure Monitor, GitHub, Databricks SQL, VS Code.

---

## Structure

Each day has one file:
- **`tasks.md`** - the day's work, the expected output, and an end-of-day checklist

```
week-4/
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

## What Changes From Week 3

```
Week 3:  full run, every time, no checks, secrets set up by hand
Week 4:  new files only, safe to re-run, alerts on failure, secrets in Key Vault,
         code in GitHub, gold tables you can query
```

### Project rules

- Do not rebuild. Improve the Week 3 pipeline in place.
- Bronze stays untouched, as before.
- Keep all secrets in Key Vault. Never put keys or connection strings in code or in GitHub.
- Keep auto-terminate on every Databricks cluster.
- Delete the resource group after the demo on Day 5.

---

## Day-by-Day Breakdown

### Day 1 - Incremental Loading
Add a date parameter to the ADF pipeline so it copies only new files. Change the silver notebook to use Delta MERGE instead of full rewrites. Run the pipeline twice for the same date and confirm nothing is duplicated.
**Output:** A pipeline that is safe to re-run.

### Day 2 - Reliability
Add retries to the ADF activities and a failure path. Add simple row-count checks between bronze, silver and gold, and print clear log messages in the notebooks. Create an Azure Monitor alert for failed runs, then break something on purpose to test it.
**Output:** A failed test run and the alert you received.

### Day 3 - Secrets and GitHub
Move any remaining keys or connection strings into Key Vault. Use a managed identity or service principal with only the access it needs. Push the notebooks and ADF definitions to GitHub.
**Output:** A repo with the pipeline code and no secrets in it.

### Day 4 - Serve and Measure
Write a few SQL queries on the gold tables that answer your business questions. Compare run time before and after one change, such as file size or partitioning. Note the approximate cost of one run.
**Output:** Working queries and a short timing note.

### Day 5 - Document and Present
Write the README, add the architecture diagram, and give a short demo. Delete the resource group when you are done.
**Output:** Finished repo and a presented project.

---

## Capstone Deliverable

A GitHub repo containing:

- ADF pipeline definitions and Databricks notebooks
- SQL queries for the gold layer
- A README covering the approach, how to run it, and the cost per run
- A short demo, live or recorded

Suggested layout:

```
azure-pipeline/
  adf/            pipeline, dataset and trigger definitions
  notebooks/      bronze to silver, silver to gold
  sql/            gold queries
  docs/           architecture diagram, data quality report
  README.md
```

---

## By End of Week 4, You Should Be Able To:

- [ ] Load only new data and re-run the pipeline safely
- [ ] Use Delta MERGE to update silver without duplicates
- [ ] Add retries, checks and an alert for failed runs
- [ ] Keep secrets in Key Vault and code in GitHub
- [ ] Query the gold layer and explain your results
- [ ] Present the project and its cost clearly

---

## Looking Ahead

This completes the module. Keep the repo as proof of work, and delete the Azure resources so no cost continues.