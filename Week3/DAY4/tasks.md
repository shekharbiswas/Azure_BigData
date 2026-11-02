# Week 3, Day 4 - Gold and Chaining
### Tasks

---

**Goal for today:** five gold tables in Delta that answer your business questions, checked against silver and optimized, plus a chained Data Factory pipeline (copy, silver notebook, gold notebook) with the gold step tested.

**Time:** about 6 hours.

**Output:** notebook `gold_ecommerce`, Delta tables in `gold/`, pipeline `pl_ecommerce_main` in Data Factory, and your business question answers in `project-notes.md`.

---

## Part A - Warm-up (15 minutes)

### Task 1: Check your understanding
Answer in your notes, in one or two lines each:
1. What is the grain of `daily_summary`? Of `product_performance`?
2. Why is revenue not the sum of every price in silver?
3. Why do we use a left join to the reference table?
4. Why do we not partition small gold tables?

---

## Part B - Design the Gold Tables (30 minutes)

### Task 2: Match questions to tables
Open the business questions you wrote on Day 1. Create this table in your notes. Adjust the questions if yours were different, but keep the five tables.

| Gold table | Grain | Business question it answers | Columns |
|---|---|---|---|
| `daily_summary` | One day | | |
| `category_funnel` | One category | | |
| `revenue_by_category_brand` | One month, category, brand | | |
| `hourly_traffic` | One hour of day | | |
| `product_performance` | One product | | |

Fill in the question and the main columns for each. If one of your Day 1 questions is not covered, note which table you would add.

---

## Part C - Set Up the Gold Notebook (25 minutes)

### Task 3: Start the cluster
In Databricks open **Compute** and start `cluster-bigdata-dev`. It takes about 5 minutes. Check that auto-terminate is still 20 minutes.

### Task 4: Create the notebook
Create a notebook named `gold_ecommerce` (Python), attached to the cluster.

**Cell 1 - settings and helpers:**

```python
from pyspark.sql import functions as F

dbutils.widgets.text("storage_account", "stbigdatadev001")
dbutils.widgets.text("secret_scope", "kv-scope")

sa = dbutils.widgets.get("storage_account")
scope = dbutils.widgets.get("secret_scope")

spark.conf.set("spark.sql.session.timeZone", "UTC")
spark.conf.set(
    f"fs.azure.account.key.{sa}.dfs.core.windows.net",
    dbutils.secrets.get(scope, "storage-account-key"),
)

silver_path = f"abfss://silver@{sa}.dfs.core.windows.net/ecommerce_events"
gold_base = f"abfss://gold@{sa}.dfs.core.windows.net"

def count_if(event):
    return F.sum((F.col("event_type") == event).cast("int"))

def rate(numerator, denominator):
    # percentage, empty when the denominator is 0
    return F.round(F.when(F.col(denominator) > 0, F.col(numerator) / F.col(denominator) * 100), 2)

def write_gold(df, name):
    (df.write.format("delta").mode("overwrite")
       .option("overwriteSchema", "true").save(f"{gold_base}/{name}"))
    print(f"written: {name}")

events = spark.read.format("delta").load(silver_path)
```

**Cell 2 - check that silver is readable:**

```python
print(events.count())
display(events.limit(5))
```

The count must match the silver row count in your Day 3 data quality report.

---

## Part D - Build the Gold Tables (120 minutes)

Each task is one or two new cells. After each write, look at the result with `display` and write one line in your notes about what you see.

### Task 5: `daily_summary`
```python
daily = (
    events.groupBy("event_date")
    .agg(
        F.count("*").alias("total_events"),
        count_if("view").alias("views"),
        count_if("cart").alias("carts"),
        count_if("remove_from_cart").alias("cart_removals"),
        count_if("purchase").alias("purchases"),
        F.countDistinct("user_id").alias("unique_users"),
        F.sum(F.when(F.col("event_type") == "purchase", F.col("price")).otherwise(0)).alias("revenue"),
    )
    .withColumn("view_to_purchase_rate", rate("purchases", "views"))
    .withColumn("day_of_week", F.date_format("event_date", "EEEE"))
    .orderBy("event_date")
)
write_gold(daily, "daily_summary")
display(daily)
```

### Task 6: `category_funnel`
```python
funnel = (
    events.groupBy("main_category")
    .agg(
        count_if("view").alias("views"),
        count_if("cart").alias("carts"),
        count_if("purchase").alias("purchases"),
    )
    .withColumn("view_to_cart_rate", rate("carts", "views"))
    .withColumn("cart_to_purchase_rate", rate("purchases", "carts"))
    .withColumn("view_to_purchase_rate", rate("purchases", "views"))
    .orderBy(F.desc("views"))
)
write_gold(funnel, "category_funnel")
display(funnel)
```

Write in your notes which category has the best and the worst view to purchase rate. Add one line on why these are event rates and not visitor rates.

### Task 7: Reference table
First look at the real category values:

```python
display(events.select("main_category").distinct().orderBy("main_category"))
```

Check the list against this mapping. Add any category that is missing and change any group you disagree with. Unmapped categories will become `Other`.

```python
mapping = [
    ("electronics", "Tech"), ("computers", "Tech"),
    ("appliances", "Home"), ("furniture", "Home"),
    ("country_yard", "Home"), ("construction", "Home"),
    ("apparel", "Fashion"), ("accessories", "Fashion"),
    ("kids", "Family"), ("medicine", "Family"),
    ("sport", "Lifestyle"), ("auto", "Lifestyle"), ("stationery", "Lifestyle"),
]
ref = spark.createDataFrame(mapping, ["main_category", "category_group"])
write_gold(ref, "ref_category_group")
```

### Task 8: `revenue_by_category_brand` (with the join)
```python
revenue = (
    events.filter(F.col("event_type") == "purchase")
    .withColumn("event_month", F.date_format("event_date", "yyyy-MM"))
    .groupBy("event_month", "main_category", "brand")
    .agg(
        F.count("*").alias("purchases"),
        F.sum("price").alias("revenue"),
        F.round(F.avg("price"), 2).alias("avg_price"),
    )
    .join(F.broadcast(ref), "main_category", "left")
    .fillna({"category_group": "Other"})
)
write_gold(revenue, "revenue_by_category_brand")
display(revenue.orderBy(F.desc("revenue")).limit(20))
```

Check that no row has a null `category_group`.

### Task 9: `hourly_traffic`
```python
hourly = (
    events.withColumn("event_hour", F.hour("event_time"))
    .groupBy("event_hour")
    .agg(
        F.count("*").alias("total_events"),
        count_if("view").alias("views"),
        count_if("purchase").alias("purchases"),
        F.countDistinct("event_date").alias("active_days"),
    )
    .withColumn("avg_events_per_day", F.round(F.col("total_events") / F.col("active_days"), 0))
    .orderBy("event_hour")
)
write_gold(hourly, "hourly_traffic")
display(hourly)
```

Times are in UTC. Add a note about this in your notes, and find the peak hour.

### Task 10: `product_performance`
This table keeps only products with at least 100 views, so "rarely bought" has meaning. Write the threshold in your notes.

```python
def known(col):
    # most common pattern: pick a real value over 'unknown'
    return F.coalesce(F.max(F.when(F.col(col) != "unknown", F.col(col))), F.lit("unknown"))

products = (
    events.groupBy("product_id")
    .agg(
        known("brand").alias("brand"),
        known("main_category").alias("main_category"),
        count_if("view").alias("views"),
        count_if("cart").alias("carts"),
        count_if("purchase").alias("purchases"),
        F.sum(F.when(F.col("event_type") == "purchase", F.col("price")).otherwise(0)).alias("revenue"),
    )
    .filter(F.col("views") >= 100)
    .withColumn("view_to_purchase_rate", rate("purchases", "views"))
)
write_gold(products, "product_performance")
display(products.orderBy(F.desc("views")).limit(20))
```

---

## Part E - Optimize and Validate (40 minutes)

### Task 11: Look at file counts before OPTIMIZE
```python
tables = ["daily_summary", "category_funnel", "revenue_by_category_brand",
          "hourly_traffic", "product_performance", "ref_category_group"]

def num_files(name):
    return spark.sql(f"DESCRIBE DETAIL delta.`{gold_base}/{name}`").select("numFiles").first()[0]

before = {t: num_files(t) for t in tables}
print(before)
```

### Task 12: Run OPTIMIZE
```python
for t in tables:
    if t == "product_performance":
        spark.sql(f"OPTIMIZE delta.`{gold_base}/{t}` ZORDER BY (product_id)")
    else:
        spark.sql(f"OPTIMIZE delta.`{gold_base}/{t}`")

after = {t: num_files(t) for t in tables}
print(after)
```

Put the before and after numbers in a table in your notes. Explain in one sentence what changed and why.

### Task 13: Cross-check the numbers
```python
silver_rows = events.count()
silver_revenue = events.filter("event_type = 'purchase'").agg(F.sum("price")).first()[0]

d = spark.read.format("delta").load(f"{gold_base}/daily_summary")
r = spark.read.format("delta").load(f"{gold_base}/revenue_by_category_brand")

print("silver rows:               ", silver_rows)
print("daily_summary events:      ", d.agg(F.sum("total_events")).first()[0])
print("silver revenue:            ", silver_revenue)
print("daily_summary revenue:     ", d.agg(F.sum("revenue")).first()[0])
print("revenue_by_cat_brand:      ", r.agg(F.sum("revenue")).first()[0])
```

All matching pairs must be exactly equal. If not, find the cause before moving on. Common reasons: a filter in one table that is missing in another, or a join that dropped rows.

### Task 14: Answer your business questions
Create temporary views and run queries:

```python
for t in tables:
    spark.read.format("delta").load(f"{gold_base}/{t}").createOrReplaceTempView(t)
```

In a new cell, run each query below with `%sql` on the first line, then write the answer in `project-notes.md` next to the matching question.

```sql
-- Which categories earn the most revenue?
SELECT category_group, main_category, SUM(revenue) AS revenue
FROM revenue_by_category_brand GROUP BY 1, 2 ORDER BY revenue DESC LIMIT 10
```

```sql
-- Which brands earn the most revenue (excluding unknown)?
SELECT brand, SUM(revenue) AS revenue
FROM revenue_by_category_brand WHERE brand <> 'unknown' GROUP BY 1 ORDER BY revenue DESC LIMIT 10
```

```sql
-- Which products are viewed a lot but rarely bought?
SELECT product_id, brand, main_category, views, purchases, view_to_purchase_rate
FROM product_performance ORDER BY views DESC LIMIT 20
```

```sql
-- Busiest hours (UTC)
SELECT event_hour, avg_events_per_day FROM hourly_traffic ORDER BY avg_events_per_day DESC LIMIT 5
```

Add one query of your own for any question from Day 1 that these do not cover.

### Task 15: Export the notebook
Export `gold_ecommerce` as `.ipynb` to `notebooks/gold_ecommerce.ipynb` in your project folder. Check that it contains no key or token.

---

## Part F - Chain It in Data Factory (100 minutes)

### Task 16: Let Data Factory use Databricks
In PowerShell:

```powershell
az login
$rg = "rg-bigdata-dev"
$adf = "adf-bigdata-dev"
$principal = az datafactory show -g $rg -n $adf --query identity.principalId -o tsv
$dbw = az databricks workspace show -g $rg -n dbw-bigdata-dev --query id -o tsv
az role assignment create --assignee-object-id $principal --assignee-principal-type ServicePrincipal --role "Contributor" --scope $dbw
```

Wait 3 to 5 minutes.

### Task 17: Create the Databricks linked service
1. In Data Factory Studio open **Manage**, then **Linked services**, then **New**.
2. Open the **Compute** tab and choose **Azure Databricks**.
3. Name: `ls_databricks`
4. Select your workspace `dbw-bigdata-dev` from the Azure subscription list.
5. Cluster: **Existing interactive cluster**.
6. Authentication type: **Managed service identity**.
7. Choose cluster `cluster-bigdata-dev` from the list.
8. Click **Test connection**. It must succeed.
9. Click **Create**.

### Task 18: Build the main pipeline
1. Create a new pipeline named `pl_ecommerce_main`.
2. On the canvas, open the **Parameters** tab and add:

| Name | Type | Default |
|---|---|---|
| `files` | Array | the same JSON as in `pl_ingest_bronze` |
| `storage_account` | String | `stbigdatadev001` |
| `secret_scope` | String | `kv-scope` |

3. Add an **Execute Pipeline** activity. Name: `run_bronze`. Invoked pipeline: `pl_ingest_bronze`. Keep **Wait on completion** checked. Set the parameter `files` to the dynamic content `@pipeline().parameters.files`.

4. Add a **Notebook** activity (under Databricks). Name: `run_silver`.
   - Azure Databricks tab: linked service `ls_databricks`
   - Settings tab: Notebook path, use **Browse** to choose `silver_ecommerce`
   - Base parameters (add two):

| Name | Value |
|---|---|
| `storage_account` | `@pipeline().parameters.storage_account` |
| `secret_scope` | `@pipeline().parameters.secret_scope` |

5. Add a second **Notebook** activity. Name: `run_gold`. Same linked service and same two base parameters. Notebook: `gold_ecommerce`.

6. Connect them in order with the green **success** arrows: `run_bronze` to `run_silver` to `run_gold`. Drag from the green square on the right edge of one activity onto the next.

### Task 19: Test the gold step on its own
Running everything now would take over an hour. Skip the first two steps:

1. Click `run_bronze`, open the **General** tab, set **Activity state** to **Deactivated** and **Mark activity as** to **Succeeded**.
2. Do the same for `run_silver`.
3. Click **Debug**. Leave the default parameter values.
4. Watch the Output tab. Expect about 5 minutes waiting if the cluster is stopped, then the gold notebook runs.

When it succeeds, click the notebook icon (glasses) next to `run_gold` and open the run output link to see the notebook as it ran. Write down the run duration in your notes.

If it fails with a permission error on the linked service, wait a few more minutes for the Contributor role. If the notebook cannot find a widget, check that the base parameter names match the widget names exactly, including lowercase letters and underscores.

### Task 20: Add a clean ending to the silver notebook
When ADF runs the silver notebook, it should stop after writing silver and not repeat the slow checks from Day 3.

1. Open `silver_ecommerce`.
2. Directly below the cell that writes silver (Task 14 of Day 3), add a new cell with one line:

```python
dbutils.notebook.exit("silver ok")
```

3. The cells below it (look, count, quality checks) stay in the notebook. They are skipped when ADF runs it, and you can still run them by hand.

### Task 21: Publish
Switch the activities back on later, tomorrow. For now click **Publish all**, so `pl_ecommerce_main` and `ls_databricks` are saved.

---

## Part G - Wrap-up (30 minutes)

### Task 22: Save your work
- Export `pl_ecommerce_main` JSON (the **{ } Code** button) to `adf/pl_ecommerce_main.json` in your project folder.
- In `project-notes.md` add:
  - The grain table from Task 2, completed
  - The answers from Task 14
  - The before and after file counts from Task 11 and 12
  - The reasons why you chose the reference grouping
- Stop the cluster in Databricks (**Terminate**) and confirm it says Terminated. If you plan a long break, stop here.
- Delete nothing in Azure.

---

## End-of-Day Checklist

- [ ] I can say what one row means in each gold table
- [ ] Five gold tables and one reference table exist in `gold/`, all Delta
- [ ] Revenue is the same in silver, `daily_summary` and `revenue_by_category_brand`
- [ ] Total events in `daily_summary` equal the silver row count
- [ ] I ran OPTIMIZE and recorded the file counts before and after
- [ ] I wrote down an answer for each business question
- [ ] ADF has the Contributor role on the Databricks workspace, and `ls_databricks` tests successfully
- [ ] `pl_ecommerce_main` chains `run_bronze`, `run_silver` and `run_gold` with success arrows
- [ ] The gold step ran successfully from Data Factory
- [ ] The silver notebook ends with `dbutils.notebook.exit`
- [ ] The cluster is terminated and no keys are in my files

---

## Tomorrow

Day 5 finishes the project. You will switch all activities back on, run the whole pipeline end to end, add a schedule trigger, check that the results match your earlier numbers, and write the summary and walkthrough.