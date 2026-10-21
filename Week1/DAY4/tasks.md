# Week 1, Day 4 - Tasks

**Total time:** ~6 hours. Work through these in order, each block builds on the last.

Replace the placeholders in `<angle brackets>` with your own names. Examples use bash. In PowerShell, set variables with `$name = ...` instead of `name=...`. Portal screens and CLI options change from time to time, so use `--help` if a command looks different.

All resources stay in **East US** (`eastus` in the CLI).

---

## Setup: Keep a Scratch File

Create `commands-day4.md` and paste every command you run, with a one-line note about what it did. Do **not** paste secret values into this file.

---

## Block 1 (45 min) - Install the CLI and Sign In

1. Check whether the CLI is installed:
   ```bash
   az --version
   ```
   If not, install it from the official Azure CLI installation page for your operating system.
2. Sign in:
   ```bash
   az login
   ```
3. Check the active account and subscription:
   ```bash
   az account show -o table
   ```
4. Save the subscription and tenant IDs in variables. These are identifiers, not secrets, but do not share them publicly.
   ```bash
   SUB_ID=$(az account show --query id -o tsv)
   TENANT_ID=$(az account show --query tenantId -o tsv)
   echo $SUB_ID
   ```
5. Confirm your resource group is visible and is in East US:
   ```bash
   az group show --name rg-bigdata-dev --query "{name:name, location:location}" -o table
   ```

**Checkpoint:** `az account show` shows the right subscription, and `rg-bigdata-dev` is listed with location `eastus`.

---

## Block 2 (60 min) - Look at Who Has Access

1. Get your own user ID and the storage account ID:
   ```bash
   MY_ID=$(az ad signed-in-user show --query id -o tsv)
   STORAGE_ID=$(az storage account show --name <storage-account> --resource-group rg-bigdata-dev --query id -o tsv)
   ```
2. List your role assignments on the resource group:
   ```bash
   az role assignment list --assignee $MY_ID --resource-group rg-bigdata-dev -o table
   ```
3. List every role assignment on the storage account:
   ```bash
   az role assignment list --scope $STORAGE_ID -o table
   ```
4. In the portal, open the storage account, go to **Access control (IAM)**, then **Role assignments**. Find your own row and note the role and the scope.
5. In the portal, open **Roles** and read the permissions of `Storage Blob Data Contributor` (look at **Data actions**). Then do the same for `Owner` and notice that it has no data actions.
6. In `commands-day4.md`, write two sentences: what roles do you have, and at which scope?

**Checkpoint:** You can name each of your roles and say whether it is a control plane or a data plane role.

---

## Block 3 (75 min) - See the Difference: With and Without a Data Role

### Part A: Try without the data role
1. If you gave yourself `Storage Blob Data Contributor` on Day 3, remove it so you can see the error:
   ```bash
   az role assignment delete --assignee $MY_ID --role "Storage Blob Data Contributor" --scope $STORAGE_ID
   ```
2. Wait one or two minutes, then try to list the containers using your sign-in:
   ```bash
   az storage fs list --account-name <storage-account> --auth-mode login -o table
   ```
3. If it fails, note the error message. A 403 with `AuthorizationPermissionMismatch` is the expected result. If it still works, wait a few minutes and try again, because the old assignment may not have expired yet.

### Part B: Grant the data role
1. Give yourself the role, scoped to the storage account:
   ```bash
   az role assignment create \
     --assignee-object-id $MY_ID \
     --assignee-principal-type User \
     --role "Storage Blob Data Contributor" \
     --scope $STORAGE_ID
   ```
2. Wait a few minutes, then repeat the listing command from Part A. It should now work.
3. List the files in `bronze`:
   ```bash
   az storage fs file list --file-system bronze --account-name <storage-account> --auth-mode login -o table
   ```
   You should see the dataset you uploaded on Day 3.

**Checkpoint:** You have seen the failure without the role and the success with it, and you can explain why in one sentence.

---

## Block 4 (75 min) - Create a Key Vault and Store a Secret

1. Create the vault in East US with Azure RBAC as the permission model. The name must be 3-24 characters and unique, so add digits.
   ```bash
   az keyvault create \
     --name <kv-bigdata-dev-001> \
     --resource-group rg-bigdata-dev \
     --location eastus \
     --enable-rbac-authorization true \
     --tags project=bigdata owner=<your-name>
   ```
2. Try to add a secret **before** giving yourself a role, and note the error:
   ```bash
   az keyvault secret set --vault-name <kv-name> --name demo-secret --value "first-value"
   ```
3. Give yourself the Secrets Officer role on the vault:
   ```bash
   KV_ID=$(az keyvault show --name <kv-name> --query id -o tsv)
   az role assignment create \
     --assignee-object-id $MY_ID \
     --assignee-principal-type User \
     --role "Key Vault Secrets Officer" \
     --scope $KV_ID
   ```
4. Wait a few minutes, then set the secret again and read it back:
   ```bash
   az keyvault secret set --vault-name <kv-name> --name demo-secret --value "first-value"
   az keyvault secret show --vault-name <kv-name> --name demo-secret --query value -o tsv
   ```
5. Update the same secret with a new value, then list its versions:
   ```bash
   az keyvault secret set --vault-name <kv-name> --name demo-secret --value "second-value"
   az keyvault secret list-versions --vault-name <kv-name> --name demo-secret -o table
   ```
   You should see two versions.
6. Store something realistic: put the storage account key into the vault **without showing it on screen**:
   ```bash
   az keyvault secret set --vault-name <kv-name> --name storage-account-key \
     --value "$(az storage account keys list --account-name <storage-account> --resource-group rg-bigdata-dev --query '[0].value' -o tsv)"
   ```
   Do not run `echo` on it or paste it anywhere. This is only practice. In later weeks you will prefer identity-based access, which needs no key at all.

**Checkpoint:** The vault exists in East US, `demo-secret` has two versions, and `storage-account-key` is stored, with no secret value in your notes.

---

## Block 5 (60 min) - Least Privilege with a Service Principal (may need permission)

Some accounts, for example student or work tenants, do not allow creating app registrations. If you get a permission error in step 2, skip to the **fallback** at the end of this block.

1. Get the scope of the `bronze` container only:
   ```bash
   BRONZE_SCOPE="$STORAGE_ID/blobServices/default/containers/bronze"
   ```
2. Create a service principal with a read-only data role on that container, and capture its secret in a variable instead of printing it:
   ```bash
   SP_SECRET=$(az ad sp create-for-rbac \
     --name sp-bigdata-dev \
     --role "Storage Blob Data Reader" \
     --scopes "$BRONZE_SCOPE" \
     --query password -o tsv)
   SP_APP_ID=$(az ad sp list --display-name sp-bigdata-dev --query "[0].appId" -o tsv)
   ```
3. Store the secret in Key Vault, then clear the variable:
   ```bash
   az keyvault secret set --vault-name <kv-name> --name sp-bigdata-dev-secret --value "$SP_SECRET"
   unset SP_SECRET
   ```
4. Check what the service principal can do:
   ```bash
   az role assignment list --assignee $SP_APP_ID --all -o table
   ```
   Confirm that it has only one assignment, and that the scope ends with `/containers/bronze`.
5. Clean up so no unused credential stays alive:
   ```bash
   az ad app delete --id $SP_APP_ID
   az keyvault secret delete --vault-name <kv-name> --name sp-bigdata-dev-secret
   ```

**Fallback if app creation is blocked:** in `security-notes.md` (next block), write down the exact CLI command you would use to create a service principal that can only read the `gold` container, and explain which role and scope you chose and why.

**Checkpoint:** You created, inspected and removed a narrowly scoped identity (or wrote the fallback).

---

## Block 6 (45 min) - Mini Task: "My Access Map"

Create `security-notes.md` and fill it in:

1. A table of every role assignment you made today: who, role, scope.
2. For each one, one sentence on why that role and that scope.
3. Three sentences: why was `Owner` not enough to list the files in `bronze`?
4. Three sentences: when would you use a managed identity, and when a service principal?
5. A list of three habits you will follow to keep secrets out of code.
6. Your Key Vault name, its region (East US), and the names of the secrets in it (not their values).

**Checkpoint:** You can explain the control plane and the data plane without looking at the concepts file.

---

## End of Day Checklist

- [ ] Azure CLI installed, signed in, and pointing at the right subscription
- [ ] Saw the storage access error without a data role, and success with it
- [ ] `Storage Blob Data Contributor` assigned to you on the storage account
- [ ] Key Vault created in East US using the RBAC permission model
- [ ] `demo-secret` has two versions
- [ ] No secret values appear in your notes, code or terminal output you shared
- [ ] Service principal created, checked and deleted (or the fallback written)
- [ ] `commands-day4.md` and `security-notes.md` written
- [ ] Can explain: authentication vs authorization, control plane vs data plane, and least privilege

Leave the storage account, the Key Vault and your role assignments in place, because tomorrow's Python script uses your CLI sign-in to reach the lake.