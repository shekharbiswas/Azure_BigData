# Week 1, Day 5 - File Formats and Python with ADLS

**Estimated time:** ~6 hours in total (about 1.5 hours of reading, the rest hands-on with `tasks.md`)

---

## Why This Day Matters

You have a data lake and a secure way into it. Today you answer two questions that decide how fast and how cheap every later pipeline will be:

1. **In which file format should the data be stored?**
2. **How does code read and write files in the lake?**

The first question affects storage cost, query speed and how much memory your jobs need. The second one is the doorway from your own scripts into Azure. At the end of the day you will have a Python script that reads raw data from `bronze`, cleans it a little, and writes Parquet into `silver`. That is the first real mini-pipeline of the module.

---

## 1. Why the File Format Matters

A file format is the way data is arranged inside a file. Two files can hold exactly the same table and still be very different:

- **Size:** a smaller file costs less to store and moves faster across the network
- **Speed:** some formats can be read in a fraction of the time
- **Memory:** some formats let you read only the part you need
- **Safety:** some formats remember data types, others make you guess

With a small file none of this matters. With 50 GB it decides whether a job takes 2 minutes or 2 hours, and what your Azure bill looks like.

---

## 2. Row-Based vs Column-Based Storage

Take a small table:

| id | city | amount |
|---|---|---|
| 1 | Pune | 120 |
| 2 | Delhi | 340 |
| 3 | Pune | 90 |

**Row-based** formats (CSV, JSON) store one whole row after another:

```
1,Pune,120
2,Delhi,340
3,Pune,90
```

**Column-based** (columnar) formats (Parquet) store one whole column after another:

```
id:      1, 2, 3
city:    Pune, Delhi, Pune
amount:  120, 340, 90
```

### Why columns win for analytics

Most analytic questions touch only a few columns. For example: "What is the total amount per city?" needs only `city` and `amount`.

- With a row-based file, the reader must go through every row and skip over the columns it does not need. It still reads all the bytes.
- With a columnar file, the reader goes directly to the two columns it needs and ignores the rest. This is called **column pruning**.

If your table has 40 columns and you need 3, a columnar file can read about 3/40 of the data.

Columnar storage also compresses better, because values in one column look alike (many repeated city names, similar numbers). Similar values squeeze into much less space.

**When row-based is still good:** when you always need the whole record at once, such as a transaction system writing one order at a time. That is why databases for applications are often row-based, and data lakes for analytics prefer columnar.

---

## 3. The Three Formats

### CSV (comma-separated values)

- Plain text, one row per line
- Opens in any editor or spreadsheet
- **No types:** everything is text. A column of dates is just text, and `007` can silently become `7`
- **No schema** stored in the file, so every reader has to guess
- Breaks easily: a comma or a line break inside a value needs careful quoting
- Large, because numbers and dates are written out as text

Good for: exchanging small files with people, and raw data that arrives as CSV.

### JSON and JSON Lines

- Plain text with named fields: `{"id": 1, "city": "Pune", "amount": 120}`
- Handles **nested data** (a list inside a record), which CSV cannot
- Repeats the field names in every record, so it is bulky
- **JSON Lines** (`.jsonl`) puts one record per line. That is the better form for big data, because a reader can process the file line by line without loading everything

Good for: data from APIs and logs, and anything nested.

### Parquet

- Binary (not readable in a text editor)
- **Columnar**, with the schema stored inside the file (column names and types)
- **Compressed by default** (Snappy is the common default), usually much smaller than CSV
- Supports column pruning, and keeps simple statistics (such as min and max values) for blocks of rows, so a reader can skip blocks that cannot match a filter
- The standard format for data lakes, Spark and Databricks

Good for: nearly everything you store in silver and gold.

### Side by side

| | CSV | JSON Lines | Parquet |
|---|---|---|---|
| Human-readable | Yes | Yes | No |
| Stores data types | No | Partly | Yes |
| Schema in the file | No | No | Yes |
| Nested data | No | Yes | Yes |
| Typical size | Large | Largest | Smallest |
| Read only some columns | No, reads all text | No | Yes |
| Best use | Exchange, small data | APIs, logs | Analytics, data lakes |

The size and speed gaps depend on your data. Today you will measure them yourself, because your own numbers are more convincing than anyone's table.

---

## 4. Compression

Compression makes a file smaller by encoding repeated patterns more efficiently. The trade is some CPU time for less storage and less transfer.

| Codec | Character |
|---|---|
| Snappy | Fast, moderate size reduction. The usual Parquet default |
| Gzip | Slower, smaller files |
| Zstd | Good balance of speed and size, increasingly popular |

For this module, keep the Parquet default (Snappy). Remember that CSV can also be compressed (for example `.csv.gz`), but a compressed CSV still has to be read in full and cannot be split easily across machines.

---

## 5. Which Format in Which Layer?

Connect this to the medallion layout from Day 2:

| Layer | Format | Reason |
|---|---|---|
| Bronze | Keep the data **as it arrived** (CSV, JSON or the source's Parquet) | Bronze is the untouched copy. Converting it would change the original |
| Silver | Parquet | Cleaned data with fixed types, read many times |
| Gold | Parquet (in Week 2, Delta tables, which are Parquet files plus a transaction log) | Fast reads for reports and models |

So the typical flow is: raw CSV arrives in bronze, your job reads it, cleans it, and writes Parquet to silver.

---

## 6. File Size and the Small-Files Problem

A data lake works best with files that are not too small and not too huge.

- **Thousands of tiny files** (a few KB each) are slow. Each file needs a separate request, and the overhead adds up.
- **One giant file** is hard to process in parallel.
- A common rule of thumb is files of roughly 100 MB to 1 GB for large datasets. For the small datasets in this module, size does not matter much, but the idea does.

Design habit: write fewer, larger files, and group them in folders by date.

---

## 7. Python and ADLS: The Azure SDK

The Azure SDK for Python lets your code talk to the lake. Two packages do the job:

| Package | Purpose |
|---|---|
| `azure-identity` | Signs in (gets credentials) |
| `azure-storage-file-datalake` | Works with files and folders in ADLS Gen2 |

You also use `pandas` to work with tables and `pyarrow` so pandas can read and write Parquet.

```bash
pip install azure-identity azure-storage-file-datalake pandas pyarrow
```

Use a virtual environment so these packages stay separate from your other projects.

### The pieces

Think of the SDK as a set of nested objects that mirror the lake:

```
DataLakeServiceClient      -> the storage account
  └── FileSystemClient     -> one container (bronze, silver, gold)
        └── DataLakeFileClient -> one file
```

In ADLS Gen2 language, a **container is called a file system**.

### Connecting

```python
from azure.identity import DefaultAzureCredential
from azure.storage.filedatalake import DataLakeServiceClient

account_name = "stbigdatadev001"
account_url = f"https://{account_name}.dfs.core.windows.net"

service = DataLakeServiceClient(account_url=account_url, credential=DefaultAzureCredential())
```

Notes:
- The address uses **`dfs`**, not `blob`. The `dfs` endpoint is the one for ADLS Gen2 file operations.
- `DefaultAzureCredential` finds your sign-in automatically. On your laptop it uses the `az login` from yesterday. In Azure, it uses a managed identity. There is no password in the code.
- The storage account name is not a secret, so it is fine in code or a settings file. Keys and secrets are not.

### Listing, uploading and downloading

```python
fs = service.get_file_system_client("bronze")          # a container

# List paths under a folder
for p in fs.get_paths(path="taxi/yellow_trips"):
    print(p.name)

# Upload a local file
file_client = fs.get_file_client("taxi/yellow_trips/ingest_date=2026-10-14/sample.csv")
with open("sample.csv", "rb") as f:
    file_client.upload_data(f, overwrite=True)

# Download a file into memory
data = file_client.download_file().readall()           # bytes
```

### Reading a file into pandas

Once you have the bytes, wrap them in an in-memory buffer and hand them to pandas:

```python
import io
import pandas as pd

df = pd.read_csv(io.BytesIO(data))
# or, for Parquet:
df = pd.read_parquet(io.BytesIO(data), columns=["city", "amount"])
```

### Writing a DataFrame to the lake as Parquet

```python
buffer = io.BytesIO()
df.to_parquet(buffer, index=False)
buffer.seek(0)

fs_silver = service.get_file_system_client("silver")
fs_silver.get_file_client("taxi/yellow_trips/data.parquet").upload_data(buffer, overwrite=True)
```

`overwrite=True` makes the script safe to run twice. Running it again replaces the file instead of failing. A job that gives the same result when repeated is called **idempotent**, and it is a habit that will pay off in every pipeline.

### Note on size

Reading a whole file into memory is fine for the files in this module. For very large files, tools like Spark (Week 2) read the data in parts and across several machines instead.

---

## 8. Measuring a Format Comparison Properly

When you compare formats today, keep the test fair:

- Use the **same data** for each format
- Measure file **size** in bytes (`os.path.getsize`)
- Measure **time** with `time.perf_counter()` around only the write or read call
- Repeat each measurement two or three times and look at the average, because the first run is often slower
- Compare **reading all columns** and **reading one column**
- Write down your machine type and file size, because results depend on both

A small file may show little difference. If so, say that honestly, and note what would likely happen at 100 times the size.

---

## Common Problems and What They Mean

| Message or symptom | Likely cause | Fix |
|---|---|---|
| 403 `AuthorizationPermissionMismatch` | No data role on the storage account | Assign Storage Blob Data Contributor (Day 4) and wait a few minutes |
| `DefaultAzureCredential failed to retrieve a token` | Not signed in | Run `az login` again |
| `ResourceNotFound` or path not found | Wrong container name or folder path | List the container and copy the exact path |
| Connection error to the account | Used the `blob` address, or the wrong account name | Use `https://<account>.dfs.core.windows.net` |
| `ImportError` for pyarrow or azure packages | Wrong environment | Activate the virtual environment and reinstall |
| Parquet write fails on a mixed-type column | A column holds both numbers and text | Convert the column to one type before writing |

---

## Key Takeaways for Today

1. File format changes size, speed and cost, and the difference grows with the data
2. Columnar storage reads only the columns you need, and compresses well
3. CSV has no types, JSON Lines handles nested data, and Parquet is the standard for the lake
4. Bronze keeps data as it arrived, and silver and gold use Parquet
5. Aim for fewer, larger files rather than many tiny ones
6. `azure-identity` plus `azure-storage-file-datalake` connect Python to the lake, and `DefaultAzureCredential` means no password in code
7. Use `overwrite=True` so scripts can be run again safely

Move to `tasks.md` now to build your first mini-pipeline.