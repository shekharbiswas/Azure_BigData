# Week 4, Day 3 - Secrets and GitHub
### Tasks

---

**Goal for today:** no secrets in code, Data Factory connects with a managed identity, and your project is on GitHub.

**Time:** about 6 hours.

**Output:** a GitHub repo with your pipeline code and no secrets in it.

Keep adding to `week4-notes.md`. Use the same variables as before:

```powershell
$rg = "rg-bigdata-dev"
$sa = "stbigdatadev001"
```

---

## Part A - Warm-up 

### Task 1: Check your understanding
Read `concepts.md`, then answer in your notes, in one line each:
1. Name three things that count as secrets in this project.
2. What is the benefit of a managed identity over a key?
3. What does least privilege mean?

---

## Part B - Audit for Secrets 

### Task 2: Look for secrets
Check each of these and write what you find in your notes:
- Every Databricks notebook: search for `key`, `token`, `password`, `sas` and `connection`.
- Every Data Factory linked service: does it use a pasted key, Key Vault, or a managed identity?
- Your project folder: any `.env` file or file with a token in it.

If you find nothing, say so in your notes and continue.

---

## Part C - Move Secrets to Key Vault 

### Task 3: Store and use secrets
For each secret you found, store it in Key Vault. Example for the storage key:

```powershell
$key = az storage account keys list -g $rg -n $sa --query "[0].value" -o tsv
az keyvault secret set --vault-name kv-bigdata-dev-001 --name storage-key --value $key
```

If you get a Forbidden error, give yourself the role first:

```powershell
$me = az ad signed-in-user show --query id -o tsv
$kv = az keyvault show -n kv-bigdata-dev-001 --query id -o tsv
az role assignment create --assignee $me --role "Key Vault Secrets Officer" --scope $kv
```

In notebooks, read the secret with `dbutils.secrets.get`, as you did in Week 2. Remove the pasted value from the notebook.

---

## Part D - Managed Identity for Data Factory 

### Task 4: Give Data Factory a role on storage

```powershell
$adf = az datafactory show -g $rg -n adf-bigdata-dev --query identity.principalId -o tsv
$sid = az storage account show -g $rg -n $sa --query id -o tsv
az role assignment create --assignee-object-id $adf --assignee-principal-type ServicePrincipal --role "Storage Blob Data Contributor" --scope $sid
```

Role assignments can take a few minutes to start working.

### Task 5: Switch the linked service
In Data Factory, open Manage, then Linked services, then your storage linked service. Change the authentication type to **System-assigned managed identity**, click Test connection, and save. Publish, then run the pipeline once for one date to confirm it still works.

---

## Part E - GitHub (1 hour)

### Task 6: Create the repo
Create a private GitHub repo called `azure-pipeline`. In your project folder, create this layout:

```
azure-pipeline/
  adf/            Data Factory definitions
  notebooks/      silver and gold notebooks
  sql/            queries (Day 4)
  docs/           architecture.png
  README.md       (Day 5)
```

### Task 7: Connect Data Factory to GitHub
In Data Factory, open Manage, then Git configuration, then Configure. Choose GitHub, select your repo, set the collaboration branch to `main`, and set the root folder to `/adf`. Import your existing resources into the repo when asked.

### Task 8: Add the notebooks
In Databricks, open each notebook and use File, then Export, then Source file. Save them in the `notebooks/` folder.

### Task 9: Check before you push
In your project folder, run:

```powershell
git grep -i -E "accountkey|sharedaccesssignature|password|token"
```

Open anything it finds and make sure it is not a real secret. Confirm `.gitignore` still excludes `data/`, `*.csv`, `*.zip` and `.env`. Then commit and push.

---

## End-of-Day Checklist

- [ ] I checked notebooks, linked services and my folder for secrets
- [ ] Any secret I found is in Key Vault and removed from code
- [ ] Data Factory has Storage Blob Data Contributor on the storage account only
- [ ] The storage linked service uses the managed identity and the pipeline still runs
- [ ] The repo has the Data Factory definitions and notebooks
- [ ] The grep check found no real secrets
- [ ] No data files are in the repo

---

## Tomorrow

Day 4 serves your gold tables with SQL, measures one performance change, and estimates the cost of a run.