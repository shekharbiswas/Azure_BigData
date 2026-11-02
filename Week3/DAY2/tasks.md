# Week 3, Day 2 - Bronze
### Tasks

---

**Goal for today:** a Data Factory pipeline that copies both Kaggle files from `source` into date-partitioned folders in `bronze`, with exact size matches and a run in the history.

**Time:** about 6 hours.

**Output:** a published ADF pipeline named `pl_ingest_bronze`, raw files in `bronze/`, and notes added to `project-notes.md`.

Keep adding to `project-notes.md`. Use the names below exactly, because Days 3 to 5 refer to them.

---

## Part A - Warm-up (20 minutes)

### Task 1: Check your understanding
Read `concepts.md`, then answer in your notes, in one or two lines each:
1. What is the difference between a linked service and a dataset?
2. Why do we use a Binary dataset for the bronze copy?
3. Why is managed identity better than a storage account key?
4. What does "idempotent" mean for a pipeline?

---

## Part B - Plan the Bronze Layout (30 minutes)

### Task 2: Decide the folder layout
Write in your notes:
- The exact bronze path for each of the two files (use the layout from `concepts.md`)
- One reason you chose to partition by `year` and `month`
- One reason you do not convert the CSV files to another format in bronze

### Task 3: Update your architecture sketch
Open `architecture.png` from Day 1. Add the bronze folder layout next to the bronze box. Save it again.

---

## Part C - Give Data Factory Access (45 minutes)

### Task 4: Assign the role
Set your names again, as in Day 1:

```powershell
az login
$rg = "rg-bigdata-dev"
$sa = "stbigdatadev001"
$adf = "adf-bigdata-dev"
```

Find the identity of your Data Factory and the ID of your storage account:

```powershell
$principal = az datafactory show -g $rg -n $adf --query identity.principalId -o tsv
$scope = az storage account show -g $rg -n $sa --query id -o tsv
```

Give the Data Factory identity the role:

```powershell
az role assignment create --assignee-object-id $principal --assignee-principal-type ServicePrincipal --role "Storage Blob Data Contributor" --scope $scope
```

If `$principal` is empty, your Data Factory has no managed identity yet. In the portal, open the Data Factory, then Identity, and check that System assigned is On. Then run the lines again.

Wait 3 to 5 minutes. Use the time for Part B if it is not done.

### Task 5: Open Data Factory Studio
In the portal, open your Data Factory and click **Launch studio**. Spend two minutes finding the four icons on the left: Home, Author (pencil), Monitor (gauge) and Manage (toolbox).

---

## Part D - Linked Service and Dataset (60 minutes)

### Task 6: Create the linked service
1. Go to **Manage**, then **Linked services**, then **New**.
2. Choose **Azure Data Lake Storage Gen2**.
3. Name: `ls_adls_bigdata`
4. Authentication type: **System Assigned Managed Identity**
5. Choose your storage account from the Azure subscription list.
6. Click **Test connection**. It must say Successful.
7. Click **Create**.

If the test fails with a 403 or permission error, the role has not spread yet. Wait a few minutes and test again.

### Task 7: Create one parameterized dataset
1. Go to **Author**, then the **+** next to Factory Resources, then **Dataset**.
2. Choose **Azure Data Lake Storage Gen2**, then **Binary**.
3. Name: `ds_adls_binary`
4. Linked service: `ls_adls_bigdata`. Leave the file path empty for now and click OK.
5. Open the **Parameters** tab and add three parameters, all of type String:
   - `container`
   - `directory`
   - `fileName`
6. Open the **Connection** tab. For the file path, choose **Add dynamic content** in each box:
   - Container: `@dataset().container`
   - Directory: `@dataset().directory`
   - File: `@dataset().fileName`

This single dataset now works for any file in any container.

---

## Part E - Build the Pipeline (90 minutes)

### Task 8: Create the pipeline and its parameter
1. Under Pipelines, click **New pipeline**.
2. Name it `pl_ingest_bronze`.
3. Click on the empty canvas, then open the **Parameters** tab.
4. Add a parameter named `files`, type **Array**, with this default value:

```json
[
  {"fileName": "2019-Oct.csv", "year": "2019", "month": "10"},
  {"fileName": "2019-Nov.csv", "year": "2019", "month": "11"}
]
```

If you only uploaded October yesterday, keep only the first line.

### Task 9: Add a ForEach loop
1. From the **Iteration & conditionals** group, drag **ForEach** onto the canvas.
2. Name it `foreach_file`.
3. On the **Settings** tab, in Items, add dynamic content: `@pipeline().parameters.files`
4. Leave Sequential unchecked. Files can be copied at the same time.

### Task 10: Add the copy activity inside the loop
1. Click the pencil icon inside `foreach_file` to open the loop.
2. Drag **Copy data** from the **Move & transform** group. Name it `copy_to_bronze`.
3. **Source** tab: dataset `ds_adls_binary`. Fill the three dataset parameters:

| Parameter | Value |
|---|---|
| `container` | `source` (typed as plain text) |
| `directory` | `ecommerce` (typed as plain text) |
| `fileName` | `@item().fileName` (dynamic content) |

4. **Sink** tab: dataset `ds_adls_binary`. Fill the parameters:

| Parameter | Value |
|---|---|
| `container` | `bronze` |
| `directory` | `@concat('ecommerce/year=', item().year, '/month=', item().month)` |
| `fileName` | `@item().fileName` |

5. Leave all other settings as they are.

### Task 11: Validate and run
1. Go back to the main canvas using the pipeline name at the top.
2. Click **Validate**. Fix any red errors.
3. Click **Debug**. Accept the default parameter values.
4. Open the **Output** tab at the bottom and watch the status change. A 14 GB copy takes a few minutes.

---

## Part F - Verify and Test (60 minutes)

### Task 12: Read the run history
1. Open **Monitor**, then **Pipeline runs**. Find your run. It should say Succeeded.
2. Click the run, then the glasses icon next to a copy activity.
3. Write down in your notes: data read, data written, files copied, duration, and throughput.

### Task 13: Check the files in Azure
List the bronze files:

```powershell
az storage blob list --account-name $sa --container-name bronze --prefix ecommerce --auth-mode key --query "[].{name:name, bytes:properties.contentLength}" -o table
```

Check that:
- The two paths look like `ecommerce/year=2019/month=10/2019-Oct.csv`
- The `bytes` values match the source files exactly

Compare with the source:

```powershell
az storage blob list --account-name $sa --container-name source --prefix ecommerce --auth-mode key --query "[].{name:name, bytes:properties.contentLength}" -o table
```

If the sizes differ, something went wrong. Look at the copy activity output for an error.

### Task 14: Prove the pipeline is idempotent
Run **Debug** a second time. When it finishes, list bronze again. You should still see exactly two files, with the same sizes. Write one line in your notes about why this is good.

### Task 15: Break it on purpose
Run **Debug** again, but this time change the `files` parameter in the run dialog. Use a file name that does not exist, for example `2019-Dec.csv` with year `2019` and month `12`.

Read the error message in the Output tab. Write down in your notes:
- Which activity failed
- What the error says in plain words
- Where in Azure you would look to confirm the cause

Then check that bronze did not get a new `month=12` folder with an empty file.

---

## Part G - Wrap-up (30 minutes)

### Task 16: Publish
Click **Publish all** and confirm. Check that the pipeline still shows in the list after you refresh the page.

### Task 17: Save your work
- In Data Factory, open the pipeline, then click the **{ } Code** button at the top right. Copy the JSON into a file called `adf/pl_ingest_bronze.json` in your project folder. You will put it on GitHub in Week 4.
- Add to `project-notes.md`:
  - The bronze folder layout and why you chose it
  - Why the copy is binary
  - The numbers from Task 12
  - What you learned from Task 15
- Delete nothing in Azure.

---

## End-of-Day Checklist

- [ ] I can explain linked service, dataset, activity, pipeline and trigger in one line each
- [ ] The Data Factory identity has the Storage Blob Data Contributor role on the storage account
- [ ] `ls_adls_bigdata` tests successfully
- [ ] `ds_adls_binary` has three parameters and is used for both source and sink
- [ ] `pl_ingest_bronze` has a `files` parameter and a ForEach with a copy activity
- [ ] Both files are in `bronze/ecommerce/year=2019/month=NN/` and their sizes match `source`
- [ ] A second run did not create duplicate files
- [ ] I ran one failing copy and read its error message
- [ ] I clicked Publish all and saved the pipeline JSON in my project folder
- [ ] No keys or tokens are in my notes or my project folder

---

## Tomorrow

Day 3 builds the silver layer. You will connect Databricks to your storage safely through Key Vault, read the raw files from bronze, fix types, handle empty values, remove duplicates and bad records, and write clean Delta tables.