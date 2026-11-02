# Week 3, Day 2 - Bronze
### Concepts

---

## Today in One Minute

Yesterday you put about 14 GB of CSV files into the `source` container. Today you build your first Data Factory pipeline. It copies those files from `source` into `bronze`, into date-partitioned folders, without changing a single byte. You will not write code today. You will build with the Data Factory visual editor and a few short commands.

```
source/ecommerce/2019-Oct.csv
source/ecommerce/2019-Nov.csv
        |
        |   Data Factory copy activity
        v
bronze/ecommerce/year=2019/month=10/2019-Oct.csv
bronze/ecommerce/year=2019/month=11/2019-Nov.csv
```

---

## 1. What Data Factory Does (and Does Not Do)

Azure Data Factory (ADF) is an **orchestrator and mover**. It is good at two things:

- Moving data from one place to another (the copy activity)
- Running other things in the right order and at the right time (pipelines and triggers)

ADF is not where we clean or reshape data. That is Databricks' job on Days 3 and 4. Think of ADF as the delivery driver and the project manager: it picks up the boxes and tells the workers when to start. It does not open the boxes.

---

## 2. The Five Building Blocks

Everything in ADF is made from five pieces. Learn these and the rest follows.

| Piece | What it is | Everyday comparison |
|---|---|---|
| **Linked service** | A saved connection to a system (storage, database, API) | A contact in your phone: name plus how to reach them |
| **Dataset** | A pointer to specific data inside a linked service (container, folder, file) | The exact address of a shelf in a warehouse |
| **Activity** | One step of work, such as copy or run a notebook | One task on a to-do list |
| **Pipeline** | A group of activities run in order | The whole to-do list |
| **Trigger** | What starts a pipeline (a schedule, an event, or you) | The alarm clock |

You will build the first four today. The trigger comes on Day 5.

---

## 3. Parameters: Build Once, Reuse for Every File

Without parameters you would need one dataset for October and another for November. With 60 files that would be a mess.

A **parameter** is a blank that you fill in when the pipeline runs. You build one dataset with blanks for container, folder and file name, and one pipeline with a list of files to copy. The same pipeline then works for any file.

Values are filled in with **dynamic content**, a small expression language. Two you need today:

| Expression | Meaning |
|---|---|
| `@pipeline().parameters.files` | The value of the pipeline parameter called `files` |
| `@item().fileName` | Inside a loop: the `fileName` of the current item |
| `@dataset().directory` | Inside a dataset: its own parameter called `directory` |
| `@concat('a','b')` | Joins text together |

Parameters give you the answer to a question you will meet again in Week 4: "How do I load only the new data?" You change the parameter values, not the pipeline.

---

## 4. Copy Raw Means Copy Binary

Bronze must be an exact copy. If ADF reads a CSV as a table and writes it again, it may quietly change things: quotes, line endings, number formats, empty values. That breaks the rule "never edit bronze".

The fix is the **Binary** dataset type. ADF then treats the file as a block of bytes and moves it as it is. It does not look inside. Same bytes in, same bytes out.

You can prove it. The size in `bronze` must match the size in `source` exactly.

---

## 5. Date-Partitioned Folders

Bronze is organized by date:

```
bronze/ecommerce/year=2019/month=10/2019-Oct.csv
bronze/ecommerce/year=2019/month=11/2019-Nov.csv
```

Why do this?

- **Easy to find.** You know where October is without opening files.
- **Easy to reprocess.** If silver for November is wrong, you re-read only the November folder.
- **Ready for growth.** Next year's data goes into `year=2020` and nothing gets mixed.
- **Spark understands it.** The `key=value` folder style is called Hive-style partitioning. Spark can read `year` and `month` from the folder names and skip folders it does not need.

Our files are monthly, so we partition by `year` and `month`. If you had daily files you would add `day=`. Another common choice is partitioning by *ingestion date* (when you loaded the file) instead of *data date* (when the events happened). Data date is easier to reason about for this project, so we use it.

---

## 6. How ADF Gets Permission: Managed Identity

ADF must be allowed to read `source` and write `bronze`. There are two ways:

| Method | How it works | Problem |
|---|---|---|
| Storage account key | A long password that opens everything | Anyone who sees it has full access. It must be stored and rotated. |
| **Managed identity** | ADF gets its own identity in Azure. You give that identity a role on the storage account. | None. There is no password to leak. |

We use managed identity. You will give the ADF identity the role **Storage Blob Data Contributor** on your storage account. It means: read, write and delete data in blobs.

**Role changes are not instant.** After you assign a role, wait 2 to 5 minutes before testing. If "Test connection" fails with a permission error right after assigning, wait and try again before assuming you made a mistake.

---

## 7. Reruns Should Be Safe (Idempotent)

A good pipeline gives the same result if you run it once or three times. This is called **idempotency**.

Our copy activity writes each file to a fixed path and overwrites what is there. Running it twice leaves you with two files, not four. You will test this on purpose today. Silver and gold will follow the same rule.

---

## 8. Debug, Run, Publish

ADF has three ways to run, and the difference confuses many learners.

| Action | What happens |
|---|---|
| **Debug** | Runs your current draft once. Good for testing. It appears in the Monitor tab. |
| **Publish all** | Saves your draft as the real, live version. Triggers only use published work. |
| **Trigger now** | Runs the published version, not the draft |

If you close the browser without publishing, your draft may be lost. Click **Publish all** after each working step.

---

## 9. Cost and Speed

- ADF charges for copy activity time (measured in DIU-hours, "data integration units") and for activity runs. A 14 GB copy inside one storage account takes a few minutes and costs very little.
- Copy speed depends on the region. Keep everything in one region, as in the project rules. Copying across regions is slower and costs extra.
- Pipelines that are not running cost nothing.

---

## Key Terms

| Term | Meaning |
|---|---|
| Orchestrator | A tool that decides what runs, in what order, and when |
| Linked service | A saved connection to a system |
| Dataset | A pointer to specific data in a linked service |
| Activity | One step in a pipeline |
| Parameter | A blank filled in when the pipeline runs |
| Dynamic content | ADF's expression language for using parameters |
| Binary copy | Copying a file as raw bytes without reading it |
| Hive-style partitioning | Folders named `key=value`, such as `year=2019` |
| Managed identity | An identity Azure gives to a service, with no password |
| Idempotent | Running twice gives the same result as running once |