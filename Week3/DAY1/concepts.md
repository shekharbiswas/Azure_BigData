# Week 3, Day 1 - Plan and Provision
### Concepts

---

## Today in One Minute

Today is a light start. You will not write any pipeline code. You will decide what you are building, set up the Azure environment, and move a large dataset into your data lake. By the end of the day, about 14 GB of real data should be sitting in Azure, ready for Day 2.

---

## 1. What We Are Building

A pipeline is a set of steps that moves data from where it starts to where people can use it, the same way every time.

```
Kaggle dataset (CSV files)
  -> source/   the files as we got them (stand-in for an outside system)
  -> bronze/   raw copy made by Data Factory (Day 2)
  -> silver/   cleaned Delta tables (Day 3)
  -> gold/     business tables (Day 4)
```

On Day 5 one Data Factory pipeline runs all of it with a single trigger.

---

## 2. Why This Dataset Counts as Big Data

You will use the Kaggle dataset **eCommerce behavior data from multi-category store** (`mkechinov/ecommerce-behavior-data-from-multi-category-store`). It holds every click on an online store for two months.

| Fact | Value |
|---|---|
| Files | `2019-Oct.csv`, `2019-Nov.csv` |
| Size | About 14 GB in total |
| Rows | About 110 million (Oct about 42 million, Nov about 67 million) |

A simple test: can you open it in Excel or load it with `pandas.read_csv` on a normal laptop? Excel stops at about 1 million rows. Pandas would need far more memory than most laptops have. That is why we use Azure and Spark for this project. If a laptop handles your data easily, it is not big data.

### The columns

| Column | Meaning |
|---|---|
| `event_time` | When the event happened (UTC) |
| `event_type` | `view`, `cart`, `remove_from_cart` or `purchase` |
| `product_id` | Product identifier |
| `category_id` | Category identifier |
| `category_code` | Category path such as `electronics.smartphone` (often empty) |
| `brand` | Brand name (often empty) |
| `price` | Product price |
| `user_id` | Customer identifier |
| `user_session` | Session identifier |

Notice that `category_code` and `brand` have gaps. Day 3 deals with that.

---

## 3. Source vs Bronze

In real projects the data starts in someone else's system, such as an API, a database or an SFTP server. Data Factory then copies it into your lake.

We do not have such a system, so we make one: a container called `source`. You upload the Kaggle files there today. On Day 2, Data Factory copies them from `source` to `bronze` in date-partitioned folders. This way you practice the real copy step instead of dropping files straight into bronze.

| Container | Role | Who writes to it |
|---|---|---|
| `source` | Pretend outside system | You, once, today |
| `bronze` | Raw copy, never edited | Data Factory |
| `silver` | Cleaned Delta tables | Databricks |
| `gold` | Business tables | Databricks |

---

## 4. Moving Large Files Into Azure

Small files are easy. Large files need a little planning.

**Time is set by your internet speed.** Rough guide for 14 GB:

| Upload speed | Time |
|---|---|
| 20 MB/s | about 12 minutes |
| 5 MB/s | about 50 minutes |
| 1 MB/s | about 4 hours |

Check your upload speed first, so the wait does not surprise you.

**What to do and avoid:**
- Use the command line, not the portal's upload button. The browser can time out on multi-GB files.
- Azure splits big files into blocks and uploads several blocks at once. The `--max-connections` option sets how many at the same time.
- A file only appears in the container when the whole upload finishes. If the upload breaks, nothing half-written is left behind. Run the same command again.
- Upload one file at a time, so a failure only costs you one file.
- Keep the terminal open and the laptop awake until it finishes.
- Always check afterwards: the size in Azure must equal the size on your disk.

**Disk space.** The download is a zip file that is unpacked. Keep about 30 GB free until you delete the zip.

If your internet is too slow, upload only `2019-Oct.csv` (about 5 GB). It is still big data.

---

## 5. Good Business Questions

Gold tables exist to answer questions. Write the questions first, then build toward them. A good question is specific and can be answered with a number or a small table.

Weak: "Analyze customer behavior."
Good: "What percentage of product views end in a purchase, per day?"

Examples for this dataset:
- How many events and how many purchases happen per day?
- What is the funnel from view to cart to purchase?
- Which categories and brands earn the most revenue?
- At what hours of the day is traffic highest?
- Which products are viewed a lot but rarely bought?

You will choose four or five of these (or your own) in the tasks.

---

## 6. Cost Reminder

| Item | Costs when idle? |
|---|---|
| Storage account with 14 GB | A little, a few rupees per month |
| Data Factory | Only when pipelines run |
| Databricks workspace | No, but running clusters cost a lot |
| Key Vault | Almost nothing |

Always set auto-terminate on clusters, and keep the budget alert active.

---

## Key Terms

| Term | Meaning |
|---|---|
| Pipeline | Repeatable steps that move and prepare data |
| Source | Where the data starts, outside your lake |
| Bronze, silver, gold | Raw, cleaned, business-ready layers |
| Block upload | Large files sent in pieces, many at once |
| Business question | A specific question a gold table answers |