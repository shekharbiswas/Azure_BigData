# Week 4, Day 2 - Reliability
### Concepts

---

## Today in One Minute

Pipelines fail. A cluster is slow to start, a file is missing, a number is wrong. Today you make sure the pipeline retries when it should, stops when something is clearly wrong, and tells you about it.

---

## 1. Retries

A retry runs a failed activity again after a short wait. It fixes temporary problems, such as a cluster that took too long to start. It does not fix a bug in your code.

In Data Factory, each activity has Retry and Retry interval settings. A good start is 2 retries with 60 seconds between them.

---

## 2. Failure Path

You can connect an activity to another one that runs only when it fails (the red arrow).

One catch: if you handle a failure with a normal activity, Data Factory may mark the whole run as Succeeded. To keep the run marked as Failed, end the failure path with a **Fail** activity. It also lets you write a clear message, such as which date failed.

---

## 3. Row-Count Checks

A pipeline can finish without errors and still be wrong. A simple check catches many problems.

| Check | What it tells you |
|---|---|
| Rows read for the date is greater than 0 | The data actually arrived |
| Rows read = rows kept + rows removed | Nothing was lost without being counted |
| Rows in silver for the date = rows written | The write did what you expected |
| One number in gold matches silver | Gold was built from the right data |

If a check fails, raise an error. A loud failure is better than a quiet wrong answer.

---

## 4. Logging

Print short, clear messages in the notebook so you can read what happened later:

```
[silver] load_date=2019-10-05 raw=1200000 removed=3000 written=1197000
```

---

## 5. Alerts

Azure Monitor can watch your Data Factory and send an email when a pipeline run fails.

```
Failed run -> Monitor alert rule -> Action group -> Email
```

An alert can take a few minutes to arrive.

---

## Key Terms

| Term | Meaning |
|---|---|
| Retry | Run a failed step again after a wait |
| Failure path | Steps that run only when something fails |
| Fail activity | Ends the run as Failed with your own message |
| Row-count check | Compare counts between steps |
| Alert rule | A rule that sends a message when something happens |
| Action group | Who gets notified, and how |