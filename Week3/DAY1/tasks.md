# Week 3, Day 1 - Plan and Provision
### Tasks

---

**Goal for today:** a plan on paper, a working empty Azure environment, and about 14 GB of Kaggle data uploaded to the `source` container.

**Time:** about 6 hours, plus upload time.

**Output:** architecture sketch, business questions, resources in Azure, dataset in `source/`.

Keep a notes file called `project-notes.md`. You will reuse it in the final write-up.

---

## Part A - Warm-up (15 minutes)

### Task 1: Check your understanding
Read `concepts.md`, then answer in your notes, in one or two lines each:
1. What is the difference between `source` and `bronze`?
2. Why do we never edit bronze?
3. Why can you not just load this dataset with `pandas.read_csv` on your laptop?

### Task 2: Check your upload speed
Run a speed test in your browser. Write down the upload speed in MB/s (divide Mbps by 8). Use the table in `concepts.md` to estimate how long 14 GB will take.

---

## Part B - Plan (45 minutes)

### Task 3: Look at the dataset
Open the dataset page on Kaggle and read the description:
`https://www.kaggle.com/datasets/mkechinov/ecommerce-behavior-data-from-multi-category-store`

In your notes, write down:
- The file names and sizes
- The nine columns
- Which columns you expect to be messy

### Task 4: Write business questions
Write four or five business questions for your gold tables. For each one, note which columns you will need.

Example:
| Question | Columns needed |
|---|---|
| What share of views end in a purchase, per day? | `event_time`, `event_type` |

### Task 5: Sketch the architecture
Draw the pipeline on paper or in a free tool such as draw.io. Include:
- Source, bronze, silver and gold
- Data Factory and Databricks, and which step each does
- Key Vault
- The trigger

Take a photo or export it as an image. Save it as `architecture.png`.

---

## Part C - Provision (60 minutes)

Use the naming pattern from Week 1. Storage account names must be unique across Azure, so change `001` if you get a "name already taken" error.

### Task 6: Create the resources

Sign in and set your names:

```powershell
az login
$rg = "rg-bigdata-dev"
$loc = "centralindia"
$sa = "stbigdatadev001"
```

Use the region you chose in Week 1 for `$loc`.

Create the resource group and the ADLS Gen2 storage account. The `--enable-hierarchical-namespace` option is what makes it ADLS Gen2:

```powershell
az group create -n $rg -l $loc
az storage account create -g $rg -n $sa -l $loc --sku Standard_LRS --kind StorageV2 --enable-hierarchical-namespace true --allow-blob-public-access false
```

Create the four containers:

```powershell
foreach ($c in "source","bronze","silver","gold") { az storage container create --account-name $sa --name $c --auth-mode key }
```

Create Data Factory, the Databricks workspace and Key Vault:

```powershell
az extension add --name datafactory
az extension add --name databricks
az datafactory create -g $rg -n adf-bigdata-dev -l $loc
az databricks workspace create -g $rg -n dbw-bigdata-dev -l $loc --sku standard
az keyvault create -g $rg -n kv-bigdata-dev-001 -l $loc
```

If a command says a resource provider is not registered, register it and try again:

```powershell
az provider register --namespace Microsoft.DataFactory
az provider register --namespace Microsoft.Databricks
az provider register --namespace Microsoft.KeyVault
```

### Task 7: Check in the portal
Open the resource group in the portal and confirm you see the storage account, Data Factory, Databricks workspace and Key Vault. Open the storage account and confirm the four containers exist. Take a screenshot.

### Task 8: Confirm the budget alert
Open Cost Management, then Budgets. Check that your budget alert from Week 1 is active. If not, create one now.

---

## Part D - Get the Data (30 minutes plus waiting)

### Task 9: Set up Kaggle access
Create a free Kaggle account if you do not have one. In your Kaggle account settings, open the API section and generate a new token.

Install the Kaggle command line tool:

```powershell
pip install kaggle
```

Set the token for this terminal session only. Do not save it in a file in your project:

```powershell
$env:KAGGLE_API_TOKEN = "paste-your-token-here"
```

Test it:

```powershell
kaggle datasets list --page-size 1
```

The token is a secret. Never put it in code, in notes or in GitHub.

### Task 10: Download the dataset
Make sure you have about 30 GB of free disk space, then run:

```powershell
mkdir data
kaggle datasets download -d mkechinov/ecommerce-behavior-data-from-multi-category-store -p data --unzip
```

This can take a while. Do not close the terminal. When it finishes:

```powershell
Get-ChildItem data | Select-Object Name, Length
```

Write down the file names and exact sizes in bytes.

Look at the first rows, without opening the whole file:

```powershell
Get-Content data\2019-Oct.csv -TotalCount 5
```

Write down what you see. Which columns have empty values?

---

## Part E - Upload the Big Files (waiting time)

### Task 11: Upload to the source container
Upload one file at a time. A progress bar is shown. Leave the terminal open until each one finishes:

```powershell
az storage blob upload --account-name $sa --container-name source --name ecommerce/2019-Oct.csv --file data\2019-Oct.csv --max-connections 8 --overwrite true --auth-mode key
az storage blob upload --account-name $sa --container-name source --name ecommerce/2019-Nov.csv --file data\2019-Nov.csv --max-connections 8 --overwrite true --auth-mode key
```

If an upload fails or you lose your connection, run the same command again for that file. Nothing half-written is left in the container.

If your upload speed is very slow, upload only the October file.

Use the waiting time to finish Task 4 and Task 5 if they are not done.

### Task 12: Verify the upload
List the files in Azure:

```powershell
az storage blob list --account-name $sa --container-name source --prefix ecommerce --auth-mode key --query "[].{name:name, bytes:properties.contentLength}" -o table
```

Compare the `bytes` column with the file sizes on your disk from Task 10. They must match exactly. If they do not, upload that file again.

---

## Part F - Wrap-up (15 minutes)

### Task 13: Clean up and save your work
- Delete nothing in Azure. You will need it tomorrow.
- Create a `.gitignore` file in your project folder with these lines, so data and secrets never reach GitHub:

```
data/
*.csv
*.zip
.env
```

- Put `project-notes.md` and `architecture.png` in your project folder.
- Optional: once the upload is verified, delete the local `data` folder to free disk space.

---

## End-of-Day Checklist

- [ ] I can explain the difference between `source` and `bronze`
- [ ] I wrote four or five business questions with the columns each needs
- [ ] I have an architecture sketch saved as `architecture.png`
- [ ] The resource group contains the storage account, Data Factory, Databricks workspace and Key Vault
- [ ] The storage account is ADLS Gen2 and has `source`, `bronze`, `silver` and `gold` containers
- [ ] My budget alert is active
- [ ] The Kaggle dataset is in `source/ecommerce/` and the sizes match my local files
- [ ] No tokens, keys or data files are in my project folder or in GitHub

---

## Tomorrow

Day 2 builds the bronze layer. You will create linked services and parameterized datasets in Data Factory, then copy the files from `source` to date-partitioned folders in `bronze`.