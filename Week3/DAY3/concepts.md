# Week 3, Day 3 - Silver
### Concepts

---

## Today in One Minute

Bronze holds the raw files exactly as they arrived. They are safe, but not pleasant to use: everything is text, some cells are empty, some rows are repeated or wrong. Today you write your first Databricks notebook to turn bronze into **silver**: clean, typed, trustworthy tables stored in Delta format.

```
bronze/ecommerce/year=2019/month=NN/*.csv
        |
        |   Databricks notebook (PySpark)
        v
silver/ecommerce_events/     clean Delta table, partitioned by event_date
silver/ecommerce_rejected/   bad rows, kept with the reason they were rejected
```

---

## 1. What Silver Is For

Silver is the layer everyone can trust. If someone asks "is this data correct?", silver is where the answer is yes. Gold tables are built only from silver, never directly from bronze.

| | Bronze | Silver |
|---|---|---|
| Format | Original CSV | Delta |
| Types | Everything is text | Real types: timestamp, number, decimal |
| Empty values | Left as they are | Handled by a clear rule |
| Duplicates | Possible | Removed |
| Bad rows | Present | Removed and kept aside |
| Can be rebuilt? | No, it is the original | Yes, from bronze, at any time |

The last row is the reason for the rule from the README: if silver is wrong, fix the notebook and rebuild from bronze. Silver is disposable. Bronze is not.

---

## 2. Schema: Tell Spark What to Expect

When Spark reads a CSV it can guess types (`inferSchema`). Do not do this on large data:

- Spark must read the whole file first just to guess, which doubles the work on 14 GB.
- The guess can be wrong, and can change between runs.

Instead we write the schema by hand. That is called **enforcing the schema**. You decide, once, what each column is.

A safe pattern for messy files:

1. **Read every column as text.** Nothing can fail at read time.
2. **Cast each column to its real type.** A value that cannot be converted becomes `null`.
3. **Treat unexpected nulls as bad records.** A row whose `price` became null was broken, and you can see exactly which ones.

Types we use for this dataset:

| Column | Type | Note |
|---|---|---|
| `event_time` | timestamp | Source text looks like `2019-10-01 00:00:00 UTC` |
| `event_type` | string | Only four allowed values |
| `product_id`, `category_id`, `user_id` | long | Large whole numbers. `category_id` needs 64 bits. |
| `price` | decimal(10,2) | Use decimal for money, never float. Floats give tiny errors such as 0.1 + 0.2 = 0.30000000000000004. |
| `category_code`, `brand`, `user_session` | string | |

---

## 3. Handling Nulls

There is no single right answer for empty values. There is only a **decision you write down**. Ask: what does empty mean here, and what will the next step need?

| Column | Empty means | Our rule |
|---|---|---|
| `category_code` | The store did not categorize the product | Fill with `unknown` |
| `brand` | The brand was not recorded | Fill with `unknown` |
| `event_time`, `product_id`, `user_id` | The row is broken, because we cannot say who did what, when | Reject the row |
| `price` | Cannot calculate revenue | Reject the row |

A useful extra column is `main_category`, the first part of `category_code`. For `electronics.smartphone` it is `electronics`. Gold tables will group by it.

Never fill a missing number with 0 just to make nulls go away. A price of 0 looks like real data and silently damages every average.

---

## 4. Duplicates

A duplicate is the same event recorded more than once. We define "same" with a **key**, the columns that identify one event:

`event_time` + `event_type` + `product_id` + `user_id` + `user_session`

`dropDuplicates` on that key keeps one row per event. Choosing the key is a decision: too few columns removes real events, too many leaves true duplicates behind. Write the key in your notes.

---

## 5. Bad Records: Quarantine, Do Not Delete

For rows that break a rule (negative price, missing user, unknown event type), there are two options:

- Drop them silently. Fast, but you can never answer "how many did we lose, and why?"
- **Quarantine** them: write them to a separate table with a `reject_reason` column.

We quarantine. The cost is small and it gives you two things: proof of what was removed, and a way to find problems at the source. It also allows a simple check at the end:

```
raw rows = silver rows + rejected rows + duplicates removed
```

If that does not add up, rows went missing somewhere.

---

## 6. Delta Lake

**Delta** is a way of storing tables on a data lake. Under the hood it is Parquet files plus a **transaction log** (the `_delta_log` folder) that records every change.

| Feature | Why it matters |
|---|---|
| Parquet storage | Columnar and compressed. Much smaller and faster than CSV. |
| ACID writes | A write either completes fully or not at all. Readers never see half a table. |
| Schema enforcement | Writing data with the wrong columns or types is refused. |
| Time travel | You can read the table as it was before a change. |
| Works with SQL | You can query it with SQL as well as PySpark |

Compare with the CSV you started from: about 14 GB of text becomes a few GB of Parquet, and a query reads only the columns it needs.

---

## 7. Partitioning Silver

Partitioning splits a table into folders by a column. Queries that filter on that column read only the matching folders.

We partition by **`event_date`**:

- Questions about this data nearly always involve dates ("per day", "last week").
- It gives about 61 partitions, each with roughly 1.8 million rows. That is a good size.

Rules of thumb:

| Choice | Problem |
|---|---|
| Partition by `user_id` | Millions of tiny folders. Very slow. |
| Partition by `event_type` | Only 4 folders, very uneven, and rarely the main filter |
| Partition by `event_date` | Balanced, matches the questions |

**The small-files trap.** After a shuffle (for example from removing duplicates), Spark has 200 tasks by default. If each task writes into each of 61 date folders, you can end up with thousands of tiny files, which makes later reads slow. The fix is one line: `repartition("event_date")` before writing, so all rows of one date go to the same task and each date folder gets about one file.

---

## 8. Databricks Basics for Today

- A **cluster** is a group of virtual machines that run your Spark code. You pay while it is running.
- A **notebook** is a document of code cells you run one at a time.
- We use a small **single-node** cluster. It is enough for this data and cheapest.
- **Always set auto-terminate** (we use 20 minutes). A forgotten cluster is the most common surprise bill.
- Spark is **lazy**. Transformations (`select`, `filter`, `dropDuplicates`) do nothing until an action (`write`, `count`, `show`) asks for a result. That is why a cell with many steps can finish instantly, then the write cell takes minutes.

---

## 9. Getting Databricks Into Storage Safely

Databricks needs permission to read bronze and write silver. For now we use the storage account key, but we never type it into a notebook. Instead:

```
Key Vault (holds the key)
   -> Databricks secret scope (a link to that vault)
      -> dbutils.secrets.get(...) in the notebook (returns the key, but never prints it)
```

If you try to print a secret in a notebook, Databricks shows `[REDACTED]`. In Week 4 you can replace this with something even stronger. For today the rule is: **no key in code, ever.**

---

## 10. Data Quality Checks

Cleaning rules are only useful if you check they worked. After writing silver you check:

- **Counts.** Raw, rejected, duplicates removed, silver. Do they add up?
- **Nulls.** No nulls left in columns that must be filled.
- **Valid values.** Only the four `event_type` values. All prices above 0.
- **Range.** Dates run from the first day of October to the last day of November, and not beyond.
- **Duplicates.** Zero rows share a key.

Write the results in a short report. A reader should be able to see in one minute whether silver can be trusted.

---

## Key Terms

| Term | Meaning |
|---|---|
| Enforce a schema | Set the columns and types yourself instead of guessing |
| Cast | Convert a value from one type to another |
| Quarantine | Keep rejected rows in a separate table, with the reason |
| Delta Lake | Parquet files plus a transaction log |
| Transaction log | The `_delta_log` folder that records every change |
| Partition | A folder that holds one slice of a table |
| Small-files problem | Too many tiny files, which slows reads |
| Lazy evaluation | Spark waits to run until a result is needed |
| Secret scope | A Databricks link to a place where secrets are stored |
| Reconciliation | Checking that counts before and after add up |