# Week 4, Day 2 - Reliability
### Tasks

---

**Goal for today:** retries are on, the notebooks check their own numbers, and a failed run sends you an email.

**Time:** about 6 hours.

**Output:** a failed test run, and the alert email you received.

Keep adding to `week4-notes.md`.

---

## Part A - Warm-up 

### Task 1: Check your understanding
Read `concepts.md`, then answer in your notes, in one line each:
1. Which kind of problem does a retry fix, and which kind does it not fix?
2. Why end a failure path with a Fail activity?
3. Why can a run succeed and still be wrong?

---

## Part B - Retries 

### Task 2: Turn on retries
In your Data Factory pipeline, open each activity (copy, silver notebook, gold notebook). In General, set Retry to `2` and Retry interval to `60` seconds. Publish.

---

## Part C - Checks and Logging 

### Task 3: Add checks to the silver notebook
Add these around your existing steps. Use your own DataFrame and path names.

```python
raw_count = raw_df.filter(F.to_date("event_time") == load_date).count()
clean_count = df.count()
removed = raw_count - clean_count
print(f"[silver] load_date={load_date} raw={raw_count} removed={removed} kept={clean_count}")

if raw_count == 0:
    raise Exception(f"No data found for {load_date}")
```

After the MERGE:

```python
silver_count = (spark.read.format("delta").load(silver_path)
                .filter(F.col("event_date") == load_date).count())
print(f"[silver] rows in silver for {load_date}: {silver_count}")

if silver_count != clean_count:
    raise Exception(f"Silver count {silver_count} does not match {clean_count}")
```

### Task 4: Add one check to the gold notebook
Pick one number that must match between silver and gold. For example, the total events for a day in your daily gold table must equal the silver row count for that day. Print both numbers and raise an exception if they differ.

---

## Part D - Failure Path 

### Task 5: Add a Fail activity
In the pipeline, drag a **Fail** activity onto the canvas. Connect the silver notebook activity to it with the red On failure arrow. Set the message to:

```
Silver step failed for @{pipeline().parameters.load_date}
```

Publish.

---

## Part E - Alert 

### Task 6: Create an alert rule
In the portal, open your Data Factory, then Monitoring, then Alerts, then New alert rule.
1. Condition: the metric **Failed pipeline runs metrics**, threshold greater than 0.
2. Action group: create one, with a notification by email to your own address.
3. Name the rule `alert-adf-failed-runs` and create it.

---

## Part F - Break It on Purpose 

### Task 7: Run a failing test
Start the pipeline with `load_date = 2019-12-01`. There is no data for that date, so the notebook should raise the "No data found" error.

1. Watch the run in Monitor. You should see the retries, then the run marked Failed.
2. Wait for the alert email. It can take a few minutes.
3. Take a screenshot of the failed run and of the email.

### Task 8: Run a good date again
Run the pipeline with `load_date = 2019-10-03`. Confirm it succeeds and the printed counts look right.

---

## End-of-Day Checklist

- [ ] Every activity has retries set
- [ ] The silver notebook prints counts and fails if there is no data or the counts differ
- [ ] The gold notebook checks one number against silver
- [ ] A Fail activity keeps failed runs marked as Failed
- [ ] I received an alert email from a failed test run
- [ ] A normal run still succeeds
- [ ] I saved the two screenshots for Friday

---

## Tomorrow

Day 3 moves secrets into Key Vault, gives Data Factory access without keys, and puts your code on GitHub.