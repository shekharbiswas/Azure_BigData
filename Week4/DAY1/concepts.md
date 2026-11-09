# Week 4, Day 1 - Incremental Loading
### Concepts

---

## Today in One Minute

Your Week 3 pipeline works, but it processes everything every time. Today you change it to process one day of data at a time, and to give the same result if you run it twice. You are not building anything new. You are improving what you already have.

---

## 1. Full Load vs Incremental Load

| | Full load | Incremental load |
|---|---|---|
| What it reads | All the data, every run | Only the new slice (one day) |
| Run time | Grows as data grows | Stays about the same |
| Cost | Grows as data grows | Stays about the same |

Real data arrives every day. Reading 110 million rows to add one day of new rows is wasteful.

Our Kaggle data is just two files, so we pretend: each run takes a `load_date` (for example `2019-10-05`) and handles only the rows for that date.

---

## 2. The Date Parameter

A parameter is a value you pass into a pipeline when you start it.

```
ADF pipeline parameter: load_date = 2019-10-05
  -> passed to the Databricks notebook as a widget
  -> notebook filters the data to that date
```

On a schedule, the trigger passes in a date. For testing, you type it by hand.

---

## 3. Safe to Re-run (Idempotent)

A pipeline is safe to re-run when running it twice for the same date gives the same result as running it once.

This matters because runs fail and get repeated. If a re-run adds the same rows again, your numbers are wrong and nobody notices.

---

## 4. Delta MERGE

MERGE compares new rows to the table using a key.

```
new row matches an existing row  -> update it
new row has no match             -> insert it
```

Our data has no single id column, so the key is a combination: `event_time`, `event_type`, `product_id`, `user_id` and `user_session`.

Also include `event_date` in the match. If silver is partitioned by `event_date`, Delta then looks at one partition instead of the whole table, which is much faster.

---

## Key Terms

| Term | Meaning |
|---|---|
| Incremental load | Process only new data |
| Parameter | A value passed in when the pipeline starts |
| Idempotent | Same result no matter how many times you run it |
| MERGE | Update matching rows, insert the rest |
| Key | The columns that identify one row |