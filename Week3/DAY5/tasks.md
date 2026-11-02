# Week 3, Day 5 - Schedule, Verify, Present
### Tasks

---

**Goal for today:** the full pipeline runs end to end from Data Factory, runs again from a schedule trigger, gives the same numbers as before, and is explained in a short write-up and a five-minute walkthrough.

**Time:** about 6 hours. The full pipeline runs take a while, so write your summary while they run.

**Output:** a working `pl_ecommerce_main` with a schedule trigger, `project-1-summary.md`, and a practiced walkthrough.

---

## Part A - Warm-up (15 minutes)

### Task 1: Check your understanding
Answer in your notes, in one or two lines each:
1. What is the difference between Debug, Trigger now and a schedule trigger?
2. Why does a trigger have no effect until you publish?
3. Why does a green pipeline run not prove that the data is correct?

---

## Part B - Switch Everything On (30 minutes)

### Task 2: Reactivate the steps
1. Open `pl_ecommerce_main` in Data Factory Studio.
2. Click `run_bronze`, open the **General** tab and set **Activity state** back to **Activated**. Do the same for `run_silver`.
3. Check that the green arrows still connect `run_bronze`, `run_silver` and `run_gold` in that order.
4. Click **Validate**, then **Publish all**.

### Task 3: Review before you run
Walk through each activity and tick these off in your notes:
- [ ] `run_bronze` passes `files` from the pipeline parameter
- [ ] `run_silver` and `run_gold` pass both `storage_account` and `secret_scope`
- [ ] The notebook paths point to the right notebooks
- [ ] The cluster `cluster-bigdata-dev` still has auto-terminate set

---

## Part C - First Full Run (75 minutes, mostly waiting)

### Task 4: Run it end to end
1. Click **Add trigger**, then **Trigger now**.
2. Keep the default parameter values and click **OK**.
3. Open **Monitor**, then **Pipeline runs**. Click your run to see the three steps.

Expect about 45 to 90 minutes in total. The cluster start takes about 5 minutes, and the silver step is the longest. You do not need to watch it. The status refreshes with the **Refresh** button.

**Use the waiting time for Part F (the write-up) and Part G (walkthrough notes).**

### Task 5: Record the run
When the run finishes, write down in your notes:
- Total duration
- Duration of each step (open the run, hover over each activity)
- The status of all three steps

If a step failed, click the error icon, read the message, fix the cause and run again. Write down what went wrong in your notes. A real problem and its fix is useful material for the walkthrough.

---

## Part D - Schedule Trigger (40 minutes)

### Task 6: Create the trigger
1. In `pl_ecommerce_main` click **Add trigger**, then **New/Edit**, then **New**.
2. Name: `tr_daily_ecommerce`
3. Type: **Schedule**
4. Start date: today, at a time about 15 minutes from now
5. Time zone: **India Standard Time**
6. Recurrence: every **1 Day**
7. Check **Start trigger on creation**.
8. Click **OK**. When it asks for pipeline parameters, enter the same values as the defaults:

| Parameter | Value |
|---|---|
| `files` | the same JSON array as before |
| `storage_account` | `stbigdatadev001` |
| `secret_scope` | `kv-scope` |

9. Click **Publish all**. The trigger is not active until you publish.

### Task 7: Wait for it to fire
Go to **Monitor**, then **Trigger runs**. After the start time you should see one run of `tr_daily_ecommerce`. Then open **Pipeline runs** and confirm a new full run started with the trigger as the source.

Use the wait for Part E or F.

### Task 8: Stop the trigger
Once you see the pipeline run start, stop the trigger so it does not run again tomorrow:

1. Open **Manage**, then **Triggers**.
2. Click `tr_daily_ecommerce` and set it to **Stopped**.
3. Click **Publish all**.

Write one line in your notes about why a trigger left on after a project ends is a problem.

---

## Part E - Verify the Results (40 minutes)

Do these after the first full run has finished. If the scheduled run is still going, wait for it to end before you query the tables, so you are not reading while it is writing.

### Task 9: Check bronze
```powershell
az login
$sa = "stbigdatadev001"
az storage blob list --account-name $sa --container-name bronze --prefix ecommerce --auth-mode key --query "[].{name:name, bytes:properties.contentLength}" -o table
```

There must be exactly two files with the same sizes as in `source`. Nothing new should be there from the extra runs.

### Task 10: Check silver and gold against your earlier numbers
In Databricks, open a new notebook named `checks_final` (or use a scratch cell) and run:

```python
sa = "stbigdatadev001"
spark.conf.set(f"fs.azure.account.key.{sa}.dfs.core.windows.net",
               dbutils.secrets.get("kv-scope", "storage-account-key"))

silver = spark.read.format("delta").load(f"abfss://silver@{sa}.dfs.core.windows.net/ecommerce_events")
rej = spark.read.format("delta").load(f"abfss://silver@{sa}.dfs.core.windows.net/ecommerce_rejected")
daily = spark.read.format("delta").load(f"abfss://gold@{sa}.dfs.core.windows.net/daily_summary")

print("silver rows:  ", silver.count())
print("rejected rows:", rej.count())
display(daily.agg({"revenue": "sum", "total_events": "sum"}))
```

Copy the numbers into this table in your notes:

| Check | Day 3 or 4 value | Today's value | Same? |
|---|---|---|---|
| Silver rows | | | |
| Rejected rows | | | |
| Gold total events | | | |
| Gold total revenue | | | |

They must be identical. If they are not, find out why before you write the summary. A common reason is that the cleaning code was changed after the earlier report was written.

### Task 11: Look at the Delta history
```python
display(spark.sql(f"DESCRIBE HISTORY delta.`abfss://gold@{sa}.dfs.core.windows.net/daily_summary`"))
```

You should see one version per run. Write down how many versions there are and why. This shows that each run replaced the table and did not add to it.

### Task 12: Confirm the cluster is stopped
Open **Compute** and check the state. If it is still running, click **Terminate**.

---

## Part F - Write the Summary (90 minutes)

### Task 13: Write `project-1-summary.md`
Use `project-notes.md`, `data-quality-report.md` and your tables from the earlier days. Use this outline:

```
# Project 1 - Core Azure Pipeline

## 1. Overview
Dataset, size, and the goal in 3 to 4 sentences.

## 2. Architecture
Insert architecture.png. One short paragraph following one file
from source to gold.

## 3. Design Choices
### Partitioning
- Bronze: by year and month. Why?
- Silver: by event_date. Why?
- Gold: not partitioned. Why?
### File formats
- Bronze: original CSV, copied as binary. Why?
- Silver and gold: Delta. Why?
### Cleaning rules
(table: column, rule, reason)

## 4. Gold Tables
| Table | Grain | Question it answers | Answer in one line |
|---|---|---|---|

## 5. Results
- Rows: raw, rejected, duplicates removed, silver
- Run time of each step and in total
- Result of the reproducibility check

## 6. Problems and Fixes
2 or 3 real problems you ran into and what you did.

## 7. Limits and Next Steps
- Reloads all data on every run (incremental loading next)
- No retries or alerts yet
- Storage key is used by Databricks (to improve)
- Code not yet in GitHub
```

Write each choice with its reason. Keep it to about two pages.

### Task 14: Read it back
Read the summary once from the top, as if you did not know the project. Fix anything unclear. Check that no key, token or connection string appears anywhere in it.

---

## Part G - Walkthrough (45 minutes)

### Task 15: Prepare
Write six bullet points for your walkthrough, one for each step in the order from `concepts.md`. For step 5 pick one real decision and one real problem.

Open these before you start, in separate browser tabs:
- Your architecture diagram
- Data Factory with `pl_ecommerce_main`
- Monitor with the green run
- Databricks with a query on a gold table

### Task 16: Practice
Give the walkthrough out loud with a timer, to a friend or to yourself. Aim for five minutes. Run one gold query live and say what the answer means for the business.

Afterwards write two lines in your notes: what went well and what you would change. If you recorded it, watch the first minute and check that you were clear.

---

## Part H - Save and Tidy Up (20 minutes)

### Task 17: Final project folder
Your folder should now look like this. Add anything that is missing.

```
project-1/
├── architecture.png
├── project-notes.md
├── project-1-summary.md
├── data-quality-report.md
├── .gitignore
├── adf/
│   ├── pl_ingest_bronze.json
│   └── pl_ecommerce_main.json
└── notebooks/
    ├── silver_ecommerce.ipynb
    └── gold_ecommerce.ipynb
```

Also export the trigger JSON from **Manage**, then **Triggers**, the **{ } Code** button, and save it as `adf/tr_daily_ecommerce.json`.

### Task 18: Cost and cleanup
- Open **Cost Management** and write down what the project has cost so far.
- Confirm the trigger is stopped and the cluster is terminated.
- Delete nothing in Azure. Week 4 builds directly on these resources.

---

## End-of-Day Checklist

- [ ] `pl_ecommerce_main` ran from start to finish with all three steps green
- [ ] I recorded the duration of each step
- [ ] A schedule trigger started a full run, and I then stopped the trigger
- [ ] Bronze still has two files with sizes equal to `source`
- [ ] Silver rows, rejected rows and gold totals are identical to my earlier numbers
- [ ] I can explain why re-running the pipeline does not create duplicates
- [ ] `project-1-summary.md` covers partitioning, file formats, cleaning rules and what each gold table answers, each with a reason
- [ ] I gave the five-minute walkthrough out loud at least once
- [ ] My project folder has the pipeline JSON, notebooks, notes, report, summary and `.gitignore`
- [ ] The cluster is terminated, the trigger is stopped, and no secrets are in my files

---

## Looking Ahead

Week 4 is the capstone. You will turn this pipeline into a production-style one: incremental loading instead of full reloads, retries and alerts, stronger secret handling, code in GitHub, and a queryable gold layer. Keep all your Week 3 resources, notebooks and notes.