# Week 1, Day 5 - Tasks

**Total time:** ~6 hours. Work through these in order, each block builds on the last.

The code samples are starting points. Adjust the names to your own account, container and dataset, and read each error message carefully. They usually tell you what is wrong.

All Azure resources stay in **East US**.

---

## Project Folder

Create a folder `week1-lake/` with this layout:

```
week1-lake/
├── data/            (local copy of your dataset)
├── out/             (files created by the format test)
├── src/
│   ├── lake_io.py
│   ├── compare_formats.py
│   └── week1_final.py
├── format-results.md
└── README.md
```

Do not put any key, secret or connection string in these files. The storage account name is fine.

---

## Block 1 (45 min) - Set Up Python and Check the Connection

1. Create and activate a virtual environment, then install the packages:
   ```bash
   cd week1-lake
   python -m venv .venv
   source .venv/bin/activate        # Windows: .venv\Scripts\activate
   pip install azure-identity azure-storage-file-datalake pandas pyarrow
   ```
2. Check you are still signed in with the CLI:
   ```bash
   az account show -o table
   ```
   If it fails, run `az login`.
3. Download the dataset you uploaded on Day 3 into `data/`. You can use the portal (open the file, then Download), or Python after Block 3.
4. Test the sign-in from Python. This prints only the expiry time, not the token:
   ```python
   from azure.identity import DefaultAzureCredential
   from datetime import datetime, timezone

   token = DefaultAzureCredential().get_token("https://storage.azure.com/.default")
   print("Token OK, expires:", datetime.fromtimestamp(token.expires_on, tz=timezone.utc))
   ```
   Never print or save the token itself.

**Checkpoint:** The script prints an expiry time, so your Python code can sign in using your CLI login.

---

## Block 2 (60 min) - Format Comparison on Your Machine

Create `src/compare_formats.py`. Use the local copy of your dataset (it may be CSV or Parquet, adjust the first line).

```python
import os
import time
import pandas as pd

SRC = "data/sample.csv"            # your dataset
os.makedirs("out", exist_ok=True)

df = pd.read_csv(SRC)
print("Rows:", len(df), "Columns:", len(df.columns))

def timed(fn, repeat=3):
    times = []
    for _ in range(repeat):
        start = time.perf_counter()
        fn()
        times.append(time.perf_counter() - start)
    return sum(times) / len(times)

writers = {
    "csv":     lambda: df.to_csv("out/data.csv", index=False),
    "jsonl":   lambda: df.to_json("out/data.jsonl", orient="records", lines=True),
    "parquet": lambda: df.to_parquet("out/data.parquet", index=False),
}
readers = {
    "csv":     lambda: pd.read_csv("out/data.csv"),
    "jsonl":   lambda: pd.read_json("out/data.jsonl", lines=True),
    "parquet": lambda: pd.read_parquet("out/data.parquet"),
}

first_col = df.columns[0]
one_col = {
    "csv":     lambda: pd.read_csv("out/data.csv", usecols=[first_col]),
    "jsonl":   lambda: pd.read_json("out/data.jsonl", lines=True)[[first_col]],
    "parquet": lambda: pd.read_parquet("out/data.parquet", columns=[first_col]),
}

print(f"{'format':8} {'size MB':>9} {'write s':>9} {'read all s':>11} {'read 1 col s':>13}")
for name in writers:
    write_t = timed(writers[name])
    size_mb = os.path.getsize(f"out/data.{name if name != 'jsonl' else 'jsonl'}") / 1e6
    read_t = timed(readers[name])
    col_t = timed(one_col[name])
    print(f"{name:8} {size_mb:9.2f} {write_t:9.2f} {read_t:11.2f} {col_t:13.2f}")
```

Steps:
1. Run it and copy the table of results into `format-results.md`.
2. Add your machine details and the number of rows.
3. Write three sentences: which format is smallest, which is fastest to read, and what happened when only one column was read.
4. If the data is small and the differences are tiny, say so, and explain what you would expect with 100 times more data.

**Checkpoint:** `format-results.md` has real numbers for size and time for three formats, with your own conclusion.

---

## Block 3 (75 min) - Write the Lake Helper Module

Create `src/lake_io.py`. Small reusable functions mean you will not repeat the connection code in every script.

```python
import io
import pandas as pd
from azure.identity import DefaultAzureCredential
from azure.storage.filedatalake import DataLakeServiceClient

ACCOUNT_NAME = "<your-storage-account>"      # not a secret


def get_service(account_name=ACCOUNT_NAME):
    url = f"https://{account_name}.dfs.core.windows.net"
    return DataLakeServiceClient(account_url=url, credential=DefaultAzureCredential())


def list_paths(service, container, prefix=""):
    fs = service.get_file_system_client(container)
    return [p.name for p in fs.get_paths(path=prefix or None) if not p.is_directory]


def upload_file(service, container, remote_path, local_path):
    fs = service.get_file_system_client(container)
    with open(local_path, "rb") as f:
        fs.get_file_client(remote_path).upload_data(f, overwrite=True)


def download_bytes(service, container, remote_path):
    fs = service.get_file_system_client(container)
    return fs.get_file_client(remote_path).download_file().readall()


def read_dataframe(service, container, remote_path, columns=None):
    data = download_bytes(service, container, remote_path)
    if remote_path.endswith(".parquet"):
        return pd.read_parquet(io.BytesIO(data), columns=columns)
    if remote_path.endswith(".jsonl"):
        return pd.read_json(io.BytesIO(data), lines=True)
    return pd.read_csv(io.BytesIO(data))


def write_parquet(service, container, remote_path, df):
    buffer = io.BytesIO()
    df.to_parquet(buffer, index=False)
    buffer.seek(0)
    fs = service.get_file_system_client(container)
    fs.get_file_client(remote_path).upload_data(buffer, overwrite=True)
```

Then, in a short test script or a Python session:
1. Create the service with `get_service()` and list everything in `bronze`. Confirm you see your Day 3 file.
2. Upload the three files from `out/` (CSV, JSON Lines, Parquet) into a test folder:
   `bronze/_sandbox/format_test/`
3. List the folder again and confirm all three files are there, and check the sizes match what you measured locally.

If you get `AuthorizationPermissionMismatch`, your data role from Day 4 is missing or has not taken effect yet. Check it with `az role assignment list`, and wait a few minutes.

**Checkpoint:** Your helper module works, and three files from the format test are in `bronze/_sandbox/format_test/`.

---

## Block 4 (75 min) - Read Back from the Lake

Write a script (or notebook cell) that uses `lake_io.py`:

1. Read each of the three files back from the lake into a DataFrame.
2. Check the row count of each one against the local original, and confirm they all match.
3. Read only two columns from the Parquet file, with `columns=[...]`. Compare the time with reading the whole file.
4. Time how long it takes to read each format **from the lake** (this includes the download). Add the numbers to `format-results.md` next to your local numbers.
5. Check the data types after reading. Print `df.dtypes` for the CSV version and the Parquet version. Write down at least one column where the types differ and why (for example, dates read from CSV as text).

**Checkpoint:** Row counts match across all formats, and you have noted a real type difference between the CSV and Parquet versions.

---

## Block 5 (60 min) - Bronze to Silver: Your First Mini-Pipeline

Write a function that moves your dataset from bronze to silver with light cleaning.

1. Read the raw file from `bronze` using `read_dataframe`.
2. Apply simple, logged cleaning steps. For example:
   - Lowercase and trim the column names
   - Remove fully duplicated rows
   - Convert the date columns to real datetimes, and the number columns to numeric types
   - Drop rows where an important key column is missing
3. Print how many rows each step removed, and the totals before and after.
4. Write the result to `silver` as Parquet, using a folder pattern that mirrors bronze:
   `silver/<source>/<dataset>/data.parquet`
5. Compare the size of the bronze file with the silver file, and write the numbers in `format-results.md`.
6. Open the silver file from Python again, and check that the column types are what you set.

Do not modify anything in `bronze`.

**Checkpoint:** The silver Parquet file exists, the row counts add up, and bronze is unchanged.

---

## Block 6 (45 min) - One Command, Clean Up, and Notes

1. Create `src/week1_final.py` that runs the bronze-to-silver step from start to finish with one command:
   ```bash
   python src/week1_final.py
   ```
   It should print the steps, the row counts, and a final summary line. Running it a second time must work without errors, and it should give the same result.
2. Delete the sandbox test files from the lake:
   ```bash
   az storage fs directory delete --file-system bronze --name _sandbox --account-name <storage-account> --auth-mode login --yes
   ```
   If the command differs in your CLI version, delete the folder in the portal.
3. Check costs and running items:
   - Open **Cost analysis** and note the spend so far
   - Confirm that no virtual machine or other compute resource is running. If you created a virtual machine in East US for any reason, stop and delete it, because a stopped-but-allocated VM may still cost money
   - Confirm your budget alert from Day 1 is still active
4. Write `README.md` for `week1-lake/` with: what the project does, how to set it up (`az login`, install packages), how to run it, and which role the user needs on the storage account.

**Checkpoint:** One command runs the full bronze-to-silver step, the sandbox is gone, and the README explains the setup.

---

## End of Day Checklist

- [ ] Python packages installed in a virtual environment
- [ ] Python signs in with `DefaultAzureCredential` and no secret in the code
- [ ] CSV, JSON Lines and Parquet compared with real numbers for size and time
- [ ] `lake_io.py` can list, upload, download and read files in the lake
- [ ] Row counts match across formats after reading back from the lake
- [ ] Silver Parquet file written from cleaned bronze data, bronze untouched
- [ ] `python src/week1_final.py` runs from start to finish and can be repeated
- [ ] Sandbox files deleted, and no compute left running
- [ ] `format-results.md` and `README.md` written
- [ ] Can explain: why columnar beats row-based for analytics, why bronze stays in its original format, and why `overwrite=True` makes a script safe to rerun

---

## End of Week 1

You now have:
- An Azure account with a budget alert and a resource group in East US
- A secure ADLS Gen2 data lake with bronze, silver and gold containers
- Access set up with RBAC, and a Key Vault for secrets
- A Python script that reads from and writes to your own lake

Week 2 builds on this with Azure Databricks (PySpark and Delta Lake) and Azure Data Factory. The lake you built this week is where every Week 2 exercise starts, so keep it in place.