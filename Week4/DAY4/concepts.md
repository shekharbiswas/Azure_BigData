# Week 4, Day 4 - Serve and Measure
### Concepts

---

## Today in One Minute

Gold tables are only useful if people can query them. Today you answer your business questions with SQL, make one thing faster and measure it, and work out roughly what a run costs.

---

## 1. Serving the Gold Layer

Serving means making gold tables easy to query. For this project, SQL on the Delta tables is enough.

To query a Delta folder with SQL, register it as a table that points at the folder:

```sql
CREATE TABLE IF NOT EXISTS gold_daily_events
USING DELTA
LOCATION 'abfss://gold@<storage-account>.dfs.core.windows.net/<folder>';
```

The data stays where it is. The table is just a name for it.

You can run SQL in a notebook with `%sql` on your existing cluster. No extra service is needed.

---

## 2. Small Files

Each MERGE or small write adds new files. Many small files slow reads down, because Spark has to open each one.

`OPTIMIZE` joins small files into larger ones. You used it on gold in Week 3. Today you try it on silver.

---

## 3. Measuring a Change

Performance work is only real if you measure.

1. Run a query and note the time.
2. Change one thing.
3. Run the same query again and note the time.

Rules: change only one thing, use the same cluster, and run each query twice, using the second time (the first run warms things up).

---

## 4. Cost per Run

A rough estimate is enough.

```
cost per run  =  run time in hours  x  cluster cost per hour
```

Cluster cost per hour is the VM price plus the Databricks charge (DBU) for that VM size. Storage and Data Factory costs for one run are tiny by comparison.

Azure Cost Management shows actual spending by day, but it can lag by up to a day.

---

## Key Terms

| Term | Meaning |
|---|---|
| Serving | Making data easy to query |
| External table | A table name pointing to Delta files in storage |
| Small files problem | Many tiny files make reads slow |
| OPTIMIZE | Joins small files into bigger ones |
| DBU | Databricks billing unit |