# Week 1, Day 2 — Big Data Concepts and Data Lake Architecture

**Estimated time:** 6 hours (concepts + design exercises, paired with `tasks.md`)

---

## Why This Day Matters

Every tool you will use later, Spark, Delta Lake, Data Factory, exists because of the ideas in this file. If you understand why data is split up, why raw data is kept untouched, and why the lake has layers, the tools become much easier to learn. Today has no code. The work is understanding and design, and the lake layout you design today is the one you build on Day 3.

---

## 1. What Makes Data "Big"?

Big data is not just "a lot of rows". It is data where a single machine and ordinary tools stop being practical. Three properties are used to describe it:

| The three Vs | Meaning | Example |
|---|---|---|
| Volume | How much data | Billions of transactions, terabytes of logs |
| Velocity | How fast it arrives | Card payments, app clicks, sensor readings |
| Variety | How many shapes it has | Tables, JSON, logs, images, text |

A dataset can be big on only one of these. A small number of huge image files is a variety and volume problem. A steady stream of tiny messages is a velocity problem.

**Common trap:** thinking five million rows is big data. It usually fits in pandas on a laptop. The practice dataset in this module is modest on purpose. The techniques you learn are the same ones used at a thousand times the size, and that is what matters.

---

## 2. Why One Machine Is Not Enough

A single machine has three limits: how fast it can read from disk, how much fits in memory, and how long you are willing to wait.

A simple example. Suppose you need to scan 1 TB of data and one disk reads about 200 MB per second:

```
1 TB = 1,000,000 MB
1,000,000 MB / 200 MB per second = 5,000 seconds = about 83 minutes
```

Now split the same data across 20 machines, each reading its own piece at the same time:

```
83 minutes / 20 = about 4 minutes
```

The cloud makes this practical: you rent the 20 machines for a few minutes, then release them.

**Two ways to grow:**
- **Scale up:** buy a bigger machine. It works until you hit the biggest machine available, and it gets expensive fast.
- **Scale out:** use many ordinary machines together. This is what big data tools do.

---

## 3. Distributed Processing

Distributed processing means splitting data into pieces, processing the pieces at the same time on different machines, and combining the results.

Think of counting votes in a large election. Each booth counts its own ballots, and then the totals are added up. No single person counts everything.

```
Big dataset
   |
 split into pieces (partitions)
   |
 machine 1   machine 2   machine 3   ...    each works on its own piece
   |
 combine the partial results
   |
 Final result
```

**One idea to keep for Spark in Week 2:** some operations need data from different pieces to be brought together, for example grouping all rows of the same city. Moving data between machines is slow compared with working locally. Good big data design tries to move as little data as possible.

---

## 4. Partitioning

Partitioning means organizing data into separate pieces by a key, most often a date. On a data lake, each partition is simply a folder:

```
trips/
  year=2026/month=03/day=01/part-0001.parquet
  year=2026/month=03/day=02/part-0001.parquet
```

Two benefits:
1. **Parallelism:** different machines can work on different folders at once.
2. **Skipping data:** a query for 2 March only reads that folder and ignores the rest. This is called partition pruning.

**Common traps:**
- Partitioning by a column with huge numbers of different values (for example customer ID) creates millions of tiny folders and slows everything down.
- Too many tiny files is also slow. A good file is often tens to hundreds of MB.
- Pick the partition key that matches how people filter the data. For event data, that is almost always the date.

---

## 5. Batch vs Streaming

| | Batch | Streaming |
|---|---|---|
| How data is processed | In chunks, on a schedule (hourly, daily) | Continuously, as each event arrives |
| Delay before results | Minutes to hours | Seconds or less |
| Complexity | Lower | Higher |
| Typical use | Daily sales reports, reloading a warehouse | Fraud alerts, live dashboards |
| Azure examples | Data Factory + Databricks | Event Hubs + Databricks streaming |

Most business reporting only needs batch, and batch is simpler and cheaper. This module builds batch pipelines. Streaming is mentioned so you can recognize when a problem needs it.

---

## 6. Data Lake vs Data Warehouse

| | Data lake | Data warehouse |
|---|---|---|
| What it stores | Any data: raw files, JSON, logs, tables | Structured, cleaned tables |
| Structure applied | When you read the data (schema on read) | Before the data is loaded (schema on write) |
| Cost of storage | Low | Higher |
| Main users | Data engineers, data scientists | Analysts, business users |
| Strength | Flexible, cheap, keeps everything | Fast, reliable SQL reporting |
| Weakness | Can become a messy "data swamp" without structure | Less flexible, costs more at large scale |

Modern designs often combine the two, storing tables as files in a lake but giving them warehouse-like reliability. Delta Lake, which you will meet in Week 2, does exactly that.

---

## 7. The Medallion Architecture

The medallion layout organizes a lake into three layers. Data moves through them, getting cleaner and more useful at each step.

| Layer | What it holds | Rules | Used by |
|---|---|---|---|
| Bronze | Raw data exactly as received | Never modify. Add only the load date and source. | Engineers (for replay and audit) |
| Silver | Cleaned, typed, deduplicated data | Fix types, remove duplicates, drop or flag bad records | Engineers, data scientists |
| Gold | Aggregated, business-ready tables | Built to answer specific business questions | Analysts, dashboards |

```
Source files -> bronze (raw) -> silver (clean) -> gold (business tables)
```

**Why keep bronze untouched?** If you find a bug in your cleaning rules next month, you can fix the rule and rebuild silver and gold from the original raw data. If you had overwritten the raw files, that data would be lost.

**Why separate silver and gold?** Silver is general-purpose clean data that many gold tables can be built from. Gold is shaped for one purpose, such as daily revenue per city.

---

## Key Takeaways for Today

1. Big data means one machine and ordinary tools stop being practical. It is about volume, velocity and variety, not only row counts.
2. Distributed processing splits data into pieces, works on them in parallel, and combines the results.
3. Partition by the column people filter on, usually the date, and avoid tiny files and too many partitions.
4. Batch is simpler and enough for most reporting. Streaming is for when seconds matter.
5. Lakes are flexible and cheap, warehouses are structured and fast. Modern designs combine them.
6. Bronze is raw and untouched, silver is clean, gold is business-ready.

Move to `tasks.md` now to practice these ideas and design your own lake layout.