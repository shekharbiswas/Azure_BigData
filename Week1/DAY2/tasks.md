# Week 1, Day 2 — Tasks

**Total time:** ~6 hours. A light day with no code. Work through these in order, each block builds on the last. Put all answers in a file called `day-2-notes.md`.

---

## Block 1 (45 min) — Explain It in Your Own Words

Without looking at the concepts file, write short answers (2-3 sentences each):

1. What are the three Vs? Give one real example for each.
2. Why is five million rows usually not a big data problem?
3. What is the difference between scaling up and scaling out?
4. What does a partition mean on a data lake?
5. In one sentence each: what goes in bronze, silver and gold?

Then open the concepts file and compare. Fix any answer that was unclear or wrong.

**Checkpoint:** Five answers written, and corrected where needed.

---

## Block 2 (1 hour) — The Scale Calculation

Use a calculator or spreadsheet. Assume one disk reads 200 MB per second.

1. How long does it take one machine to read 500 GB? Show the steps.
2. How long if the data is split evenly across 10 machines?
3. How long across 50 machines?
4. Compare your three numbers. Did 50 machines give 5 times the speed of 10 machines in real life? Write 2-3 sentences on why the real result is usually worse than the calculation (hint: combining results, moving data between machines, one slow machine holding everyone up).
5. A machine costs 10 rupees per hour (made-up number). Compare the cost of 1 machine for 83 minutes with 20 machines for 4 minutes. What do you notice?

**Checkpoint:** Three timings calculated, plus a written explanation of why more machines do not give perfect speedups.

---

## Block 3 (1 hour) — Batch or Streaming, Lake or Warehouse

For each scenario, decide **batch or streaming**, and **lake or warehouse** (or both), and give a one-line reason.

1. A retail chain wants a sales report every morning for the previous day.
2. A bank wants to flag a suspicious card payment within seconds.
3. A company stores millions of raw website log files for later analysis.
4. A finance team needs a monthly report with exact, audited numbers.
5. A factory has temperature sensors sending readings every second, and it wants an alert if a machine overheats.
6. A data science team wants raw data of many kinds to experiment on.
7. A manager wants a dashboard that refreshes once a day.

**Checkpoint:** Seven decisions with reasons. Some may reasonably be "both", explain your thinking.

---

## Block 4 (1 hour 15 min) — Walk the Medallion Layers

Here is a small raw file as it arrived in bronze:

| trip_id | pickup_time | distance_km | fare | city |
|---|---|---|---|---|
| 1 | 2026-03-01 08:15 | 5.2 | 210 | Delhi |
| 2 | 2026-03-01 08:15 | 5.2 | 210 | Delhi |
| 3 | 01/03/2026 09:40 | 12.0 | 480 | delhi |
| 4 | 2026-03-01 10:05 | -3.0 | 150 | Mumbai |
| 5 | 2026-03-01 11:20 | 7.5 | (blank) | Mumbai |
| 6 | 2026-03-02 07:50 | 3.1 | 95 | Pune |

1. List every problem you can find in this data. (There are at least five.)
2. For each problem, decide what the silver layer should do: fix it, remove it, or flag it. Explain your choice. Be careful with row 5: is it better to delete the row or keep it with a flag? Write both sides.
3. Write the clean version of the table as it would look in silver.
4. Design one gold table that answers: "How much revenue did each city make each day?" Write its columns and fill in the values using only the rows you kept.
5. Write 2-3 sentences: why do we keep the original messy file in bronze, instead of fixing it and overwriting it?

**Checkpoint:** A problem list, silver rules, a clean silver table, a gold table, and a written reason for keeping bronze.

---

## Block 5 (1 hour) — Choose the Project Dataset

You will use one dataset for Week 3 and Week 4 too, so choose carefully.

1. Find a public dataset with these properties:
   - Several million rows (or the ability to combine several months into that size)
   - A date or timestamp column
   - Numeric columns that can be summed or averaged
   - At least one category column (city, product type, zone)
   - Free to download
   The NYC Taxi trip records are a good default, published as monthly files by the NYC Taxi and Limousine Commission. Search for them, or choose another dataset that fits.
2. Download one month, or a small sample, to your computer.
3. Without writing code, open it (a text editor or spreadsheet, or the file's properties) and note: file format, file size, number of columns, and what each column means.
4. Write down 3 business questions the dataset could answer (for example "average fare per day", "busiest zones", "how revenue changes by weekday").
5. Estimate: if you needed two years of files, how large would that be in total?

**Checkpoint:** A chosen dataset with format, size, columns, three business questions and a total-size estimate.

---

## Block 6 (1 hour) — Design Your Lake Layout

Create a file called `lake-design.md` that describes the lake you will build tomorrow. Fill in:

1. **Containers:** the three container names (`bronze`, `silver`, `gold`).
2. **Folder pattern for bronze,** for example:
   ```
   bronze/<dataset-name>/year=YYYY/month=MM/day=DD/<file>
   ```
   Write the real path your first file would have.
3. **Partition key for silver and gold,** and one sentence on why you chose it.
4. **File format for each layer** (raw format in bronze, and your choice for silver and gold) with a one-line reason.
5. **What rules apply to each layer:** can files be edited? deleted? rebuilt?
6. **Names:** the storage account name from your `azure-names.md` from Day 1.
7. **Two risks** for your design, for example "too many small files" or "wrong date format in raw data", and how you would guard against each.

**Checkpoint:** `lake-design.md` is complete and uses your real dataset name and your real storage account name.

---

## End of Day Checklist

- [ ] `day-2-notes.md` has answers for Blocks 1 to 4
- [ ] Can explain the three Vs and why five million rows is usually not big data
- [ ] Can explain why more machines do not give perfect speedups
- [ ] Can say when batch is enough and when streaming is needed
- [ ] Can explain what bronze, silver and gold each contain and why bronze is never edited
- [ ] Project dataset chosen, with three business questions written
- [ ] `lake-design.md` written with real names and paths

If any checkbox is unchecked, finish it before Day 3. Tomorrow you turn the lake design into a real ADLS Gen2 storage account on Azure.