# Week 3, Day 3 - Silver
### Tasks

---

**Goal for today:** a Databricks notebook that reads the raw files from bronze, enforces the schema, handles nulls, removes duplicates and bad records, and writes clean Delta tables to silver, plus a short data quality report.

**Time:** about 6 hours.

**Output:** notebook `silver_ecommerce`, Delta tables `silver/ecommerce_events` and `silver/ecommerce_rejected`, and `data-quality-report.md`.

Keep adding to `project-notes.md`.

---

## Part A - Warm-up (20 minutes)

### Task 1: Check your understanding
Read `concepts.md`, then answer in your notes, in one or two lines each:
1. Why do we read every column as text first and cast afterwards?
2. Why do we quarantine bad rows instead of deleting them?
3. Why is `event_date` a better partition column than `user_id`?
4. What is the small-files problem and how do we avoid it today?

### Task 2: Write your cleaning rules first
Before any code, write this table in your notes and fill in the last column. This becomes part of your final write-up.

| Column | Rule | Reason |
|---|---|---|
| `category_code` | Fill empty with `unknown` | |
| `brand` | Fill empty with `unknown` | |
| `price` | Reject if empty or 0 or less | |
| `event_time`, `product_id`, `user_id` | Reject if empty or not convertible | |
| `event_type` | Reject if not view, cart, remove_from_cart or purchase | |
| Duplicates | Same `event_time`, `event_type`, `product_id`, `user_id`, `user_session` | |

---

## Part B - Connect Databricks to Storage Safely (60 minutes)

### Task 3: Give yourself permission to write secrets
Your Key Vault uses role-based access, so even you need a role before you can add a secret:

```powershell
az login
$rg = "rg-bigdata-dev"
$sa = "stbigdatadev001"
$kv = "kv-bigdata-dev-001"
$me = az ad signed-in-user show --query id -o tsv
$kvid = az keyvault show -n $kv --query id -o tsv
az role assignment create --assignee-object-id $me --assignee-principal-type User --role "Key Vault Secrets Officer" --scope $kvid
```

Wait 2 to 3 minutes for the role to take effect.

### Task 4: Store the storage key in Key Vault
This command reads the key and sends it straight into Key Vault. It never prints on screen:

```powershell
az keyvault secret set --vault-name $kv --name storage-account-key --value (az storage account keys list -g $rg -n $sa --query "[0].value" -o tsv) --query id -o tsv
```

It should print the ID of the new secret. Check in the portal: Key Vault, then Secrets, and you should see `storage-account-key`.

### Task 5: Let Databricks read from Key Vault
Databricks reads the vault using an Azure application called `AzureDatabricks`. Give it the role:

```powershell
$dbx = az ad sp show --id 2ff814a6-3304-4ab8-85cb-cd0e6f879c1d --query id -o tsv
az role assignment create --assignee-object-id $dbx --assignee-principal-type ServicePrincipal --role "Key Vault Secrets User" --scope $kvid
```

### Task 6: Create the secret scope
1. Get the two values you need:

```powershell
az keyvault show -n $kv --query properties.vaultUri -o tsv
az keyvault show -n $kv --query id -o tsv
```

2. Open your Databricks workspace from the portal (**Launch Workspace**).
3. Take the address in your browser, which looks like `https://adb-1234567890.1.azuredatabricks.net/`, and add `#secrets/createScope` to the end. Press Enter.
4. Fill in the form:
   - Scope name: `kv-scope`
   - Manage Principal: **All workspace users**
   - DNS Name: the `vaultUri` from step 1
   - Resource ID: the ID from step 1
5. Click **Create**.

Tip: the browser URL for this page is case sensitive. Write `createScope` with a capital S.

---

## Part C - Cluster and First Look at the Data (60 minutes)

### Task 7: Create a cluster
1. In Databricks, open **Compute**, then **Create compute**.
2. Name: `cluster-bigdata-dev`
3. Choose **Single node**. Keep the default runtime version.
4. Node type: the smallest one with 4 cores, for example `Standard_DS3_v2`. If your quota does not allow it, choose the smallest available.
5. Set **Terminate after 20 minutes of inactivity**.
6. Click **Create** and wait until the status is green (about 5 minutes).

Check the auto-terminate setting again before you move on. It is the most important setting on this page.

### Task 8: Create the notebook and connect
1. **Workspace**, then **Create**, then **Notebook**. Name it `silver_ecommerce`, language Python, cluster `cluster-bigdata-dev`.
2. **Cell 1 - settings:**

```python
dbutils.widgets.text("storage_account", "stbigdatadev001")
dbutils.widgets.text("secret_scope", "kv-scope")

sa = dbutils.widgets.get("storage_account")
scope = dbutils.widgets.get("secret_scope")

spark.conf.set("spark.sql.session.timeZone", "UTC")
spark.conf.set(
    f"fs.azure.account.key.{sa}.dfs.core.windows.net",
    dbutils.secrets.get(scope, "storage-account-key"),
)

bronze_path = f"abfss://bronze@{sa}.dfs.core.windows.net/ecommerce/*/*/*.csv"
silver_path = f"abfss://silver@{sa}.dfs.core.windows.net/ecommerce_events"
rejected_path = f"abfss://silver@{sa}.dfs.core.windows.net/ecommerce_rejected"
```

The widgets let Data Factory pass in different values on Day 5.

3. **Cell 2 - check access:**

```python
display(dbutils.fs.ls(f"abfss://bronze@{sa}.dfs.core.windows.net/ecommerce/"))
```

You should see `year=2019/`. If you get a "secret does not exist" or permission error, wait a few minutes after Task 5 and run Cell 1 again. If it still fails, in the portal open the Key Vault, then Access control (IAM), and check that the `AzureDatabricks` application has Key Vault Secrets User.

4. Print the secret to see what protection does:

```python
print(dbutils.secrets.get(scope, "storage-account-key"))
```

It shows `[REDACTED]`. Delete this cell afterwards.

### Task 9: Look at the raw data
Read only a small sample first. Do not count 110 million rows yet.

```python
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType

cols = ["event_time", "event_type", "product_id", "category_id", "category_code",
        "brand", "price", "user_id", "user_session"]
raw_schema = StructType([StructField(c, StringType(), True) for c in cols])

raw = spark.read.option("header", "true").schema(raw_schema).csv(bronze_path)
raw.printSchema()
display(raw.limit(10))
```

Write in your notes what you see. Which columns have empty values in these 10 rows?

---

## Part D - Build Silver (120 minutes)

Each task below is one new notebook cell. Run them in order.

### Task 10: Fix the types
```python
typed = raw.select(
    F.to_timestamp("event_time", "yyyy-MM-dd HH:mm:ss 'UTC'").alias("event_time"),
    F.col("event_type"),
    F.col("product_id").cast("long"),
    F.col("category_id").cast("long"),
    F.col("category_code"),
    F.col("brand"),
    F.col("price").cast("decimal(10,2)"),
    F.col("user_id").cast("long"),
    F.col("user_session"),
)
typed.printSchema()
display(typed.limit(5))
```

Check that `event_time` shows real timestamps and not nulls. If it is null in every row, the format pattern is wrong. Compare it with the text you saw in Task 9.

### Task 11: Flag the bad records
Each row gets a reason if it breaks a rule, or stays empty if it is fine:

```python
valid_types = ["view", "cart", "remove_from_cart", "purchase"]

flagged = typed.withColumn(
    "reject_reason",
    F.when(F.col("event_time").isNull(), "bad_event_time")
     .when(~F.col("event_type").isin(valid_types) | F.col("event_type").isNull(), "bad_event_type")
     .when(F.col("product_id").isNull(), "missing_product_id")
     .when(F.col("user_id").isNull(), "missing_user_id")
     .when(F.col("price").isNull() | (F.col("price") <= 0), "bad_price"),
)

rejected = flagged.filter(F.col("reject_reason").isNotNull())
good = flagged.filter(F.col("reject_reason").isNull()).drop("reject_reason")
```

Nothing runs yet, because Spark is lazy. That is expected.

### Task 12: Clean the good rows
```python
clean = (
    good
    .fillna({"brand": "unknown", "category_code": "unknown"})
    .withColumn("main_category", F.split("category_code", r"\.").getItem(0))
    .withColumn("event_date", F.to_date("event_time"))
    .dropDuplicates(["event_time", "event_type", "product_id", "user_id", "user_session"])
)
```

### Task 13: Write the rejected rows
```python
(rejected
    .write.format("delta")
    .mode("overwrite")
    .save(rejected_path))
```

This reads all 14 GB, so expect it to take several minutes. Use the time to start your report file.

### Task 14: Write silver
```python
(clean
    .repartition("event_date")
    .write.format("delta")
    .mode("overwrite")
    .partitionBy("event_date")
    .save(silver_path))
```

The `repartition("event_date")` line is what avoids the small-files problem. This is the longest step of the day. Do not stop the cell. Check progress in the Spark UI link under the cell if you are curious.

### Task 15: Look at what you built
```python
display(dbutils.fs.ls(silver_path))

events = spark.read.format("delta").load(silver_path)
events.printSchema()
display(events.limit(10))
```

Write in your notes how many `event_date=` folders you see and why that number makes sense.

Look at the table history, which comes from the Delta transaction log:

```python
display(spark.sql(f"DESCRIBE HISTORY delta.`{silver_path}`"))
```

---

## Part E - Data Quality Checks (60 minutes)

### Task 16: Reconcile the counts
```python
raw_count = raw.count()
rejected_df = spark.read.format("delta").load(rejected_path)
rejected_count = rejected_df.count()
silver_count = events.count()
dupes_removed = raw_count - rejected_count - silver_count

print(f"raw rows:           {raw_count:,}")
print(f"rejected rows:      {rejected_count:,}")
print(f"silver rows:        {silver_count:,}")
print(f"duplicates removed: {dupes_removed:,}")
```

The duplicates number must not be negative. If it is, rows are counted twice somewhere and you must find out why before continuing.

### Task 17: Why were rows rejected?
```python
display(rejected_df.groupBy("reject_reason").count().orderBy(F.desc("count")))
```

Write down each reason and its count.

### Task 18: Check the silver table itself
Run these four checks and note each result.

Nulls in every column (the first five and `price` should be zero):

```python
display(events.select([F.sum(F.col(c).isNull().cast("int")).alias(c) for c in events.columns]))
```

Valid values and price range:

```python
display(events.groupBy("event_type").count())
display(events.agg(F.min("price"), F.max("price"), F.min("event_date"), F.max("event_date")))
```

Any duplicates left (the answer must be 0):

```python
key = ["event_time", "event_type", "product_id", "user_id", "user_session"]
print(events.groupBy(key).count().filter("count > 1").count())
```

If the date range goes outside October and November 2019, find out which rows cause it and decide whether they belong in silver. Write your decision in your notes.

### Task 19: Write the data quality report
Create `data-quality-report.md` with:

```
# Data Quality Report - Silver ecommerce_events

## Row counts
| Stage | Rows |
|---|---|
| Raw (bronze) | |
| Rejected | |
| Duplicates removed | |
| Silver | |

## Rejected rows by reason
(table)

## Checks on silver
| Check | Result |
|---|---|
| Nulls in key columns | |
| Event types | |
| Price range | |
| Date range | |
| Duplicate keys | |

## Notes
(anything unexpected and what you decided)
```

Fill it with your real numbers.

---

## Part F - Wrap-up (30 minutes)

### Task 20: Save the notebook
In Databricks, use **File**, then **Export**, then **IPython Notebook (.ipynb)**. Save it as `notebooks/silver_ecommerce.ipynb` in your project folder. Check that no key or token appears in the file. You will put it on GitHub in Week 4.

### Task 21: Update your notes
Add to `project-notes.md`:
- The final cleaning rules table from Task 2, with the reasons filled in
- The partition column and why you chose it
- Why silver is Delta and not CSV
- One thing that surprised you in the data

### Task 22: Shut down
- Stop the cluster now with **Terminate** instead of waiting for auto-terminate.
- Confirm in Compute that the cluster state is Terminated.
- Delete nothing else in Azure. You need the tables tomorrow.

---

## End-of-Day Checklist

- [ ] The storage key is in Key Vault, and no key appears in any notebook or file
- [ ] `kv-scope` exists and `dbutils.secrets.get` works in the notebook
- [ ] The cluster has auto-terminate set and is now terminated
- [ ] `silver/ecommerce_events` is a Delta table partitioned by `event_date`
- [ ] `silver/ecommerce_rejected` exists and has a `reject_reason` column
- [ ] The counts reconcile: raw = rejected + silver + duplicates removed
- [ ] No nulls in key columns, no duplicate keys, prices above 0
- [ ] `data-quality-report.md` has real numbers
- [ ] The notebook is exported to `notebooks/silver_ecommerce.ipynb`
- [ ] My cleaning rules and reasons are in `project-notes.md`

---

## Tomorrow

Day 4 builds the gold layer. You will read the clean silver table and build business tables that answer the questions you wrote on Day 1: daily totals, the view to cart to purchase funnel, top categories and brands. You will write them as Delta and run OPTIMIZE.