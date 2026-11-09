# Week 4, Day 1 - Incremental Loading
### Tasks

---

**Goal for today:** the pipeline takes a `load_date`, processes only that date, and gives the same silver result when run twice.

**Time:** about 6 hours.

**Output:** an updated ADF pipeline and silver notebook, and a note showing the re-run test.

Keep a notes file called `week4-notes.md`. You will use it in Friday's presentation.

---

## Part A - Warm-up 

### Task 1: Check your understanding
Read `concepts.md`, then answer in your notes, in one line each:
1. Why is a full load a problem as data grows?
2. What does "safe to re-run" mean?
3. Why does the MERGE match include `event_date`?

---

## Part B - Add the Date Parameter 

### Task 2: Add a pipeline parameter
Open your Week 3 pipeline in Data Factory.
1. Click an empty area of the canvas, open the Parameters tab, and add a parameter `load_date` of type String, with a default of `2019-10-01`.
2. Open your silver notebook activity. In Settings, under Base parameters, add `load_date` with the value `@pipeline().parameters.load_date`.
3. If your copy activity already writes into date folders, you can use the same parameter in the folder path. If it copies whole files, leave the copy step as it is.

Publish the changes.

---

## Part C - Change the Silver Notebook 

### Task 3: Read the date and filter
At the top of your silver notebook add:

```python
from delta.tables import DeltaTable
from pyspark.sql import functions as F

dbutils.widgets.text("load_date", "2019-10-01")
load_date = dbutils.widgets.get("load_date")
```

After your existing cleaning steps, keep only that date and add the partition column:

```python
df = (clean_df   # use the name of your cleaned DataFrame from Week 3
      .withColumn("event_date", F.to_date("event_time"))
      .filter(F.col("event_date") == load_date))
```

### Task 4: Replace the full write with MERGE
Use your own silver path. Replace the old write with:

```python
target = DeltaTable.forPath(spark, silver_path)

(target.alias("t")
  .merge(
      df.alias("s"),
      """t.event_date = s.event_date
         AND t.event_time = s.event_time
         AND t.event_type = s.event_type
         AND t.product_id = s.product_id
         AND t.user_id = s.user_id
         AND t.user_session = s.user_session""")
  .whenMatchedUpdateAll()
  .whenNotMatchedInsertAll()
  .execute())
```

Notes:
- Your silver table must be partitioned by `event_date`. If it is not, add the column and rewrite silver once with `.partitionBy("event_date")`, then use MERGE from then on.
- If the silver table does not exist yet, write the first date with a normal Delta write and use MERGE after that.

---

## Part D - Test It 

### Task 5: Run the same date twice
Count the rows in silver per date with this:

```python
spark.read.format("delta").load(silver_path).groupBy("event_date").count().orderBy("event_date").show(5)
```

1. Run the pipeline from Data Factory with `load_date = 2019-10-01`. Note the count for that date.
2. Run it again with the same date. The count must be exactly the same.
3. Run it with `2019-10-02`. A new date should appear and the first date should be unchanged.

Write the three counts in your notes.

If the count grows on the second run, check that the five key columns in the MERGE match your data.

---

## Part E - Wrap-up 

### Task 6: Save your work
- Copy the updated notebook into your project folder.
- Add a short note to `week4-notes.md`: what you changed and the three counts from Task 5.

---

## End-of-Day Checklist

- [ ] The pipeline has a `load_date` parameter that reaches the notebook
- [ ] The silver notebook processes only the given date
- [ ] Silver is updated with MERGE, not a full rewrite
- [ ] Running the same date twice gives the same row count
- [ ] I saved the counts in `week4-notes.md`

---

## Tomorrow

Day 2 adds retries, simple checks between layers, and an alert that emails you when a run fails.