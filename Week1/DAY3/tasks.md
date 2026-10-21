# Week 1, Day 3 - Tasks

**Total time:** ~6 hours. Work through these in order, each block builds on the last.

Portal screens and menu names change from time to time. If a label looks different from what is written here, look for the closest match.

---

## Block 1 (45 min) - Plan Before You Create

1. Open your `azure-names.md` file from Day 1 and your lake design from Day 2.
2. Confirm the storage account name. It must be 3-24 characters, lowercase letters and numbers only, and unique across Azure (for example `stbigdatadev001`, with digits of your own).
3. Confirm the region is **East US**, the same as `rg-bigdata-dev`. Use East US for every resource in this module, including virtual machines.
4. Write down the folder pattern you will use inside the containers, for example:
   `<source>/<dataset>/ingest_date=YYYY-MM-DD/<file>`
5. Choose the first dataset to upload: the one from your Day 2 design. If you do not have one yet, use any small CSV file of a few MB.

**Checkpoint:** You have a name, a region, a folder pattern and a dataset chosen before you open the portal.

---

## Block 2 (75 min) - Create the Storage Account

1. In the portal, search for **Storage accounts** and select **Create**.
2. Fill in the **Basics** tab:
   - Resource group: `rg-bigdata-dev`
   - Storage account name: your planned name
   - Region: East US
   - Performance: Standard
   - Redundancy: LRS
3. On the **Advanced** tab, turn on **Enable hierarchical namespace**. Leave the default access tier on Hot.
4. Add the tags `project = bigdata` and `owner = ` your name.
5. Select **Review + create**, check the summary, then **Create**.
6. When deployment finishes, open the account. On the overview page, confirm that hierarchical namespace shows as enabled.

**Checkpoint:** The storage account exists in `rg-bigdata-dev`, and hierarchical namespace is enabled.

---

## Block 3 (60 min) - Build the Containers and Folders

1. In the storage account, open **Containers** (under Data storage).
2. Create three containers: `bronze`, `silver`, `gold`.
3. Open `bronze` and create the folder structure for your first dataset using your pattern, for example:
   `taxi/yellow_trips/ingest_date=2026-10-14/`
4. Create the matching top-level folders in `silver` and `gold` (one per source and dataset, without the date folder).

**Permission note:** if the portal says you are not authorized to list the data, you have the owner role on the resource but not the data role. In the container view, check the authentication method. Ask your instructor, or give yourself the role **Storage Blob Data Contributor** on the storage account (Access control (IAM), then Add role assignment). Day 4 explains roles in detail. Do not copy the storage account keys anywhere.

**Checkpoint:** Three containers exist, and your folder pattern is visible inside them.

---

## Block 4 (60 min) - Upload Your First Dataset

1. Open the dated folder inside `bronze` and upload your first dataset file.
2. Open the file's properties and note: the size, the access tier, and the full path.
3. Check the file name: lowercase, no spaces, no special characters.
4. Try to rename the dated folder to a different date, then rename it back. Notice that it happens as one step. This is the benefit of the hierarchical namespace.

**Checkpoint:** Your dataset file is in `bronze` under the correct folder path, and you have recorded its size and path.

---

## Block 5 (60 min) - Tiers and Redundancy

This block is about reading settings, not changing costs.

1. Open the storage account's **Configuration** page. Note the current replication (redundancy) and the default access tier.
2. Open the file you uploaded and look at its blob tier. Find where you could change it. Do not change it to Archive.
3. Open the Azure pricing page for storage (search for "Azure Blob Storage pricing"), select East US, and find the price per GB for Hot and Cool. Write both down with today's date.
4. Look up the current Azure pricing for LRS and GRS and note how they differ.
5. Answer in `lake-notes.md` (next block): which tier and which redundancy would you choose for each of bronze, silver and gold in a real project, and why?

**Checkpoint:** You know where the tier and redundancy settings are, and you have two real prices written down.

---

## Block 6 (60 min) - Mini Task: "My Data Lake"

Create a file called `lake-notes.md` and fill it in:

1. The storage account name, region, redundancy, and whether hierarchical namespace is on.
2. A text diagram of your lake with real names:

   ```
   stbigdatadev001
     bronze/
       ...
     silver/
       ...
     gold/
       ...
   ```

3. Your folder pattern, with one real example path from your upload.
4. Three sentences: what is the difference between plain Blob storage and ADLS Gen2, and why do we want ADLS Gen2 for big data?
5. Three sentences: why is bronze kept untouched?
6. Your answer to the tier and redundancy question from Block 5.

**Checkpoint:** You can explain your lake without looking at the concepts file.

---

## End of Day Checklist

- [ ] Storage account created in East US with hierarchical namespace enabled
- [ ] Containers `bronze`, `silver`, `gold` exist
- [ ] Folder pattern chosen and visible in the containers
- [ ] First dataset uploaded to `bronze`
- [ ] Tiers and redundancy settings found, with two prices noted
- [ ] `lake-notes.md` written
- [ ] Storage keys not copied into any file
- [ ] Can explain: Blob vs ADLS Gen2, and why bronze is never edited

If any checkbox is unchecked, finish it before Day 4. Tomorrow covers who can access this lake and how secrets are kept safe.