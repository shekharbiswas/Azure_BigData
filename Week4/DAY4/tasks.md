# Week 4, Day 4 - Serve and Measure
### Tasks

---

**Goal for today:** your business questions answered with SQL, one timing comparison, and a rough cost per run.

**Time:** about 3 hours.

**Output:** a `sql/queries.sql` file, a timing note, and a cost estimate.

Keep adding to `week4-notes.md`.

---

## Part A - Warm-up (10 minutes)

### Task 1: Check your understanding
Read `concepts.md`, then answer in your notes, in one line each:
1. What does an external table do?
2. Why do many small files slow things down?
3. Why run each query twice when timing it?

---

## Part B - Query the Gold Layer (1 hour)

### Task 2: Register the gold tables
In a Databricks notebook, create a table for each gold table, if you have not already. Use your own storage account and folder names:

```sql
%sql
CREATE TABLE IF NOT EXISTS gold_daily_events
USING DELTA
LOCATION 'abfss://gold@stbigdatadev001.dfs.core.windows.net/daily_events';
```

### Task 3: Answer your business questions
Write one SQL query for each of your four or five business questions from Week 3 Day 1. Run each one and check the answer looks sensible.

Example (change the names to match your tables):

```sql
%sql
SELECT event_date,
       purchases,
       views,
       ROUND(100.0 * purchases / views, 2) AS view_to_purchase_pct
FROM gold_daily_events
ORDER BY event_date;
```

Save all queries in `sql/queries.sql`. Put the question as a comment above each query. In your notes, write one line per query on what the result shows.

---

## Part C - Measure One Change (45 minutes)

### Task 4: Time a query before and after OPTIMIZE
Pick one query that reads your silver table, such as a count per day.

1. Run it twice. Write down the second time.
2. Optimize silver:

```sql
%sql
OPTIMIZE delta.`abfss://silver@stbigdatadev001.dfs.core.windows.net/<your-silver-folder>`;
```

3. Run the same query twice again. Write down the second time.

Record it in your notes:

| Query | Before | After |
|---|---|---|
| (your query) | s | s |

If there is no difference, that is a fine result. Write that down and what you think the reason is.

---

## Part D - Cost per Run (30 minutes)

### Task 5: Estimate the cost
1. In Data Factory Monitor, run the pipeline for one date and note the total duration.
2. In Databricks, open your cluster and note the VM size and the DBU per hour shown there.
3. Find the VM price per hour for that size on the Azure pricing page for your region.
4. Calculate: run time in hours x (VM price + DBU price).

Write the result in your notes as an approximate figure, with the numbers you used. Tomorrow you can compare it to Cost Management if the data has arrived.

---

## Part E - Wrap-up (15 minutes)

### Task 6: Save your work
Copy `sql/queries.sql` into the repo and push it. Add the timing table and the cost estimate to `week4-notes.md`.

---

## End-of-Day Checklist

- [ ] Gold tables can be queried with SQL
- [ ] I have a query for each business question, saved in `sql/queries.sql`
- [ ] I measured one query before and after OPTIMIZE
- [ ] I have a rough cost per run
- [ ] The SQL file is pushed to GitHub

---

## Tomorrow

Presentation day at 3 PM. Day 5 is only preparation.