# Week 3, Day 4 - Gold and Chaining
### Concepts

---

## Today in One Minute

Silver is clean but still has one row per click, about 110 million of them. Nobody answers a business question by scrolling through that. Today you do two things:

1. Build **gold**: small tables that each answer one of your business questions.
2. Connect Data Factory to Databricks and build the **chained pipeline** (copy, then silver notebook, then gold notebook), so tomorrow you only need to schedule it and test it end to end.

```
silver/ecommerce_events   (110 million rows)
        |
        |   gold notebook
        v
gold/daily_summary
gold/category_funnel
gold/revenue_by_category_brand
gold/hourly_traffic
gold/product_performance
gold/ref_category_group   (small reference table)
```

---

## 1. What Gold Is For

Gold is the layer people actually use: analysts, dashboards, managers. A gold table is small, easy to read, and built to answer a specific question. If a gold table does not answer a question someone cares about, it should not exist.

| | Silver | Gold |
|---|---|---|
| Rows | One per event, about 110 million | One per day, category, hour or product, thousands |
| Audience | Data engineers | Analysts and business users |
| Column names | Close to the source | Plain business words |
| Built from | Bronze | Silver only |
| Can be rebuilt? | Yes, from bronze | Yes, from silver |

---

## 2. Grain: What Does One Row Mean?

The **grain** of a table is what one row represents. Decide it before you write any code, and write it in one sentence.

| Gold table | Grain (one row is...) | Answers |
|---|---|---|
| `daily_summary` | One day | Events, purchases and revenue per day. View to purchase rate. |
| `category_funnel` | One category | The view to cart to purchase funnel |
| `revenue_by_category_brand` | One month, category and brand | Which categories and brands earn the most |
| `hourly_traffic` | One hour of the day (0 to 23) | When traffic is highest |
| `product_performance` | One product | Which products are viewed a lot but rarely bought |

If you cannot say what one row means, the table is not ready to be built. A wrong grain is the most common gold mistake, and it leads to numbers that double count.

---

## 3. Aggregation Patterns

Almost every gold table is a `groupBy` followed by `agg`. Two patterns cover most of today.

**Plain aggregates:** `count`, `sum`, `avg`, `countDistinct`.

**Conditional counts.** The `event_type` column holds four kinds of event. To count only views in a group, turn each row into 1 or 0 and add up:

```python
F.sum((F.col("event_type") == "view").cast("int")).alias("views")
```

This lets one pass over the data produce views, carts and purchases side by side in one row.

**Revenue is not the sum of all prices.** Every click carries a price, but money is only earned on a `purchase`. Revenue means: add up `price` only on purchase rows. Using all rows would multiply revenue by about 100.

---

## 4. Rates and Division by Zero

A rate divides one count by another, for example `purchases / views`. Two habits:

- **Guard against zero.** If `views` is 0 the division fails or gives a null. Write it so that a zero gives an empty value on purpose.
- **Be clear what the rate means.** We count events, not people. A rate of 2 percent means "2 purchase events per 100 view events". It does not mean 2 percent of visitors bought. Say this in your notes so nobody reads it wrongly.

---

## 5. Reference Data and Joins

**Reference data** is a small table that gives meaning to codes in a big table. Our silver table has 13 or so `main_category` values. A manager would rather see "Tech" and "Home" than twenty raw category names. So we keep a small table that maps each category to a group, and **join** it in.

When one side of a join is tiny, use a **broadcast join**. Spark sends a copy of the small table to every worker, so the big table is never shuffled. In code: `F.broadcast(ref)`.

Use a **left join** from the big table. That way a category missing from the reference table is kept (with a null group that you fill with `Other`) and not silently dropped. Dropping rows during a join is a classic way to lose revenue without noticing.

---

## 6. Writing Gold as Delta

- **Do not partition small tables.** A table of 61 rows split into 61 folders is slower, not faster. Partitioning only helps when a table is large.
- **Overwrite is fine.** Gold is rebuilt from silver each run. Running it twice gives the same result (idempotent).
- **Name columns for readers.** `revenue`, `view_to_purchase_rate`, `unique_users`. Not `sum(price)`.
- **Round for display.** Rates to 2 decimals. Keep money in decimal.

---

## 7. OPTIMIZE: Fixing Small Files

A `groupBy` makes Spark shuffle the data into 200 pieces by default. When you write a result with only 61 rows, you can end up with dozens of tiny files. Reading many tiny files is slow.

`OPTIMIZE` merges small files into bigger ones:

```sql
OPTIMIZE delta.`<path>`
```

You can check the effect with `DESCRIBE DETAIL`, which shows `numFiles` before and after.

`OPTIMIZE ... ZORDER BY (column)` goes one step further: it sorts data inside the files by that column, so queries that filter on it skip most files. Use it on columns you filter on often. Today: `product_id` in `product_performance`.

Two related things to know, but not to do today:
- Delta keeps old file versions for time travel. `VACUUM` deletes the old ones after a retention period (7 days by default). Never shorten this period on a table others are using.
- OPTIMIZE does not change your data. It only rearranges files.

---

## 8. Check Gold Against Silver

A gold table is only useful if the numbers are right. Cross-check it:

- The sum of `total_events` per day must equal the row count of silver.
- The revenue in `daily_summary` must equal the revenue in `revenue_by_category_brand`, and both must equal the sum of purchase prices in silver.
- No category should disappear in the join.

If two gold tables give different revenue, one of them is wrong. Find out which before you present anything.

---

## 9. Chaining ADF and Databricks

So far you ran notebooks by hand. In production, Data Factory runs them.

| Piece | What it is |
|---|---|
| **Databricks linked service** | The saved connection from ADF to your workspace and cluster |
| **Notebook activity** | An ADF step that runs one Databricks notebook |
| **Base parameters** | Values ADF passes to the notebook. They arrive as the notebook's **widgets**, which is why the notebooks have `storage_account` and `secret_scope` widgets. |
| **Dependency (green arrow)** | "Run the next step only if this one succeeded" |
| **Execute Pipeline activity** | Runs another pipeline as a step. We use it to reuse `pl_ingest_bronze`. |

How ADF logs in to Databricks:

| Method | Problem |
|---|---|
| Personal access token | A secret that must be stored and rotated |
| **Managed identity** | None. You give the ADF identity the Contributor role on the workspace. |

We use managed identity again, the same idea as Day 2.

**Cluster choice.** We use your existing cluster. ADF starts it if it is terminated, which takes about 5 minutes, and it stops by itself after 20 idle minutes. Real projects often use a **job cluster** instead, a temporary cluster created for one run and deleted afterwards. That is cheaper and is something to consider in Week 4.

**Testing one step at a time.** Running 110 million rows through every step each time you test a change would waste hours. ADF lets you **deactivate** an activity and mark it as succeeded. It is skipped, but later steps still run. Today you use this to test the gold step alone. Tomorrow you switch everything on.

---

## Key Terms

| Term | Meaning |
|---|---|
| Grain | What one row of a table represents |
| Conditional aggregation | Counting or summing only rows that match a condition |
| Conversion rate | One count divided by another, such as purchases per views |
| Reference data | A small table that gives meaning to codes |
| Broadcast join | Sending a small table to every worker so the big one is not shuffled |
| Left join | Keeps every row of the left table, matched or not |
| OPTIMIZE | Merges small files into larger ones |
| Z-ORDER | Sorts data inside files by a column to speed up filters |
| VACUUM | Deletes old file versions after the retention period |
| Base parameters | Values ADF passes to a notebook as widgets |
| Execute Pipeline | An activity that runs another pipeline |