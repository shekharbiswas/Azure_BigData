# Week 3, Day 5 - Schedule, Verify, Present
### Concepts

---

## Today in One Minute

The pipeline is built. Today you prove it works as one piece. You switch every step on, run it from start to finish, put it on a schedule, check that the results are correct, and explain what you built in a short write-up and an informal walkthrough.

```
Trigger (schedule)
   -> pl_ecommerce_main
        -> run_bronze   (copy source to bronze)
        -> run_silver   (clean to Delta)
        -> run_gold     (business tables)
```

---

## 1. Triggers: What Starts a Pipeline

A pipeline does nothing by itself. Something has to start it.

| Way to start | When to use it |
|---|---|
| **Debug** | Testing a draft while you build |
| **Trigger now** | Running the published pipeline once by hand |
| **Schedule trigger** | Running at set times, such as every day at 02:00 |
| Tumbling window trigger | Running for fixed time slices, with history and backfill |
| Event trigger | Running when a file arrives in storage |

Today you use **Trigger now** for the first full test and a **schedule trigger** for the second. Event and tumbling window triggers come up when you need incremental loading in Week 4.

---

## 2. Schedule Triggers

A schedule trigger has four settings:

| Setting | Example | Notes |
|---|---|---|
| Start date and time | Today, 20 minutes from now | Must be in the future |
| Time zone | India Standard Time | Choose it on purpose. The default is UTC. |
| Recurrence | Every 1 day | Or hours, weeks, months |
| Run at | 02:00 | Daily runs often go at night, when nobody is using the data |

Two facts that catch people out:

- **Triggers only work after you publish.** Creating one in the editor is not enough. Always click **Publish all**.
- **A trigger is a separate object from the pipeline.** One trigger can start several pipelines, and one pipeline can have several triggers. The trigger carries the **parameter values** for the run.

**A schedule left on keeps costing money.** Every run starts a cluster. After your test, stop the trigger (Manage, then Triggers, then Stop). Keeping the pipeline is free. Running it daily is not.

---

## 3. What a Daily Run Means for This Dataset

Our source files never change. A daily run therefore reads the same two files and rebuilds the same tables every time. That is fine for today, because it proves something important: the pipeline is **idempotent** from top to bottom. Run it three times and you get the same result, with no duplicates.

In a real project, new files arrive every day and you only want to process the new ones. That is **incremental loading**, which is Week 4. Note in your write-up that the current pipeline reloads everything each time.

---

## 4. Verifying an End-to-End Run

A green pipeline run means the steps finished without errors. It does not mean the data is right. After the full run, check:

| Check | Expected |
|---|---|
| ADF Monitor | All three steps Succeeded |
| `bronze/` | Same two files, sizes equal to `source` |
| `silver/` row count | Equals your Day 3 data quality report |
| `gold/` revenue | Equals your Day 4 numbers |
| Run time | Recorded, for the write-up |

If today's numbers match the ones you wrote earlier, you have shown that the pipeline gives the same result every time. This **reproducibility** is one of the main reasons for automating a pipeline.

---

## 5. Writing the Project Summary

The summary is for a reader who has never seen your project, such as a hiring manager or a teammate. It should let them understand your decisions in five minutes. A decision is stronger when it has a **reason**.

Weak: "I partitioned by date."
Strong: "Silver is partitioned by `event_date` because most questions filter by day, and it gives about 61 folders of roughly 1.8 million rows each, which is a healthy size."

Good summaries cover:

- **What and why:** the dataset, the questions, the result
- **Design choices with reasons:** partitioning, file formats, cleaning rules
- **What each gold table answers**
- **Numbers:** rows in, rows out, run times
- **Honest limits:** what the pipeline does not do yet (for example, no incremental loading, no alerts)

Short is good. Two pages is enough.

---

## 6. The Informal Walkthrough

A walkthrough is a five-minute talk where you show the project to someone. It is practice for explaining your work in interviews and team meetings.

A simple order that works:

1. **The problem (30 seconds).** What data, what questions.
2. **The architecture (1 minute).** Show your diagram. Follow one file from source to gold.
3. **The pipeline running (1 minute).** Show the ADF Monitor with the green run.
4. **One result (1 minute).** Run one query on a gold table and answer one business question.
5. **A decision and a mistake (1 minute).** One design choice with its reason, and one thing that went wrong and how you fixed it.
6. **What is next (30 seconds).** Week 4: incremental loading, alerts, GitHub.

Tips: speak in plain words, show the real screens, and do not read your notes aloud. Practice once with a timer. Recording yourself on screen is optional, but it shows quickly where you ramble.

---

## 7. Tidy Up Before You Leave

| Item | Action |
|---|---|
| Schedule trigger | **Stop it** after the test |
| Databricks cluster | Terminated |
| Azure resources | **Keep all of them.** Week 4 builds on them. |
| Cost | Open Cost Management and look at the week's spend |
| Files | Pipeline JSON, notebooks, notes, report, and summary saved in the project folder |

---

## Key Terms

| Term | Meaning |
|---|---|
| Trigger | What starts a pipeline |
| Schedule trigger | A trigger that starts a pipeline at set times |
| Trigger run | One firing of a trigger, shown in Monitor |
| Pipeline run | One execution of a pipeline, however it was started |
| Idempotent | Running again gives the same result |
| Reproducible | The same inputs and code give the same numbers |
| Incremental loading | Processing only new data (Week 4) |
| Walkthrough | A short informal talk that explains a project |