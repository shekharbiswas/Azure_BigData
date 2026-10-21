# Week 1, Day 3 - Azure Storage and ADLS Gen2

**Estimated time:** ~6 hours in total (about 1 hour of reading, the rest hands-on with `tasks.md`)

---

## Why This Day Matters

Everything in this module reads from and writes to one place: the data lake. Today you build it. By the end of the day you will have a storage account with bronze, silver and gold containers and your first dataset uploaded. Most settings you choose today are hard to change later, so it is worth getting them right the first time.

---

## 1. Storage Accounts

A storage account is the top-level Azure item that holds your data. Inside it you create containers, and inside containers you keep files.

```
Storage account   stbigdatadev001
  └── Container   bronze
        └── Folders and files
```

Azure storage accounts can hold different kinds of data (files, tables, queues). In this module we only use the file part, called **Blob storage**.

---

## 2. Blob Storage vs ADLS Gen2

Both store files. The difference is how they treat folders.

| | Blob storage | ADLS Gen2 |
|---|---|---|
| Folders | Fake. A "folder" is only part of a file name | Real folders |
| Rename or delete a folder | Copies or deletes every file one by one | One quick operation |
| Permissions | Set on the container | Can be set on folders and files |
| Best for | Simple file storage | Big data and analytics |

**Example:** you have a folder with 100,000 files and you want to rename it. In plain Blob storage, Azure has to rename 100,000 files. In ADLS Gen2, the folder is renamed in one step.

### The hierarchical namespace

ADLS Gen2 is not a separate product. It is a normal storage account with one setting turned on: **Enable hierarchical namespace**. That setting gives you real folders.

**Important:** choose this when you create the account. A storage account created without it is not ADLS Gen2, and converting it later is a separate, more involved process. Databricks and Data Factory in the coming weeks work best with it on.

---

## 3. Settings You Choose When Creating the Account

| Setting | What to choose in this module | Why |
|---|---|---|
| Subscription and resource group | Your subscription, `rg-bigdata-dev` | Keeps everything for the project together |
| Storage account name | `stbigdatadev001` style | 3-24 characters, lowercase letters and numbers only, unique across all of Azure |
| Region | East US | Use East US for every resource in this module, including virtual machines |
| Performance | Standard | Good enough and cheaper than Premium |
| Redundancy | LRS (see section 6) | Cheapest, fine for learning |
| Hierarchical namespace | On | This is what makes it ADLS Gen2 |
| Default access tier | Hot | Data you use often (see section 5) |

The name and the region cannot be changed after creation. If you pick wrongly, you create a new account.

---

## 4. Containers and Folder Design

A container is the top-level folder of the account. Use one container for each layer of the medallion layout from Day 2:

| Container | What goes in it |
|---|---|
| `bronze` | Raw data, exactly as it arrived. Never edited |
| `silver` | Cleaned and validated data |
| `gold` | Aggregated tables ready for reports and models |

Container names use lowercase letters, numbers and hyphens only.

### Folder design inside a container

Choose a pattern and use it everywhere. A simple one:

```
bronze/<source>/<dataset>/ingest_date=YYYY-MM-DD/<file>
```

Example:

```
bronze/taxi/yellow_trips/ingest_date=2026-10-14/yellow_tripdata_2026-01.parquet
```

Why this works:
- You can see where a file came from and when it arrived
- You can load or delete one day without touching the rest
- Later tools can read `ingest_date` as a column

Avoid spaces, capital letters and special characters in folder and file names. They cause small problems in many tools.

---

## 5. Access Tiers

The tier decides how much you pay for storing data and how much you pay to read it.

| Tier | For | Storage cost | Cost to read |
|---|---|---|---|
| Hot | Data used often | Highest | Lowest |
| Cool | Data used rarely | Lower | Higher |
| Cold | Data used very rarely | Lower still | Higher still |
| Archive | Data kept for compliance or backup | Lowest | Highest, and slow |

Things to know:
- Archive data is offline. Before you can read it, you have to bring it back, which can take hours.
- Cool, Cold and Archive have minimum storage periods. If you delete or move data earlier, you may still be charged for the full period.
- Prices and the minimum periods change, so check the current Azure pricing page before deciding for a real project.

**Example:** bronze data from last year that nobody reads can move to Cool or Cold. The gold tables behind a dashboard stay in Hot.

For this module, keep everything in Hot. The volumes are small.

---

## 6. Redundancy Options

Redundancy means how many copies of your data Azure keeps, and where.

| Option | Meaning | Use |
|---|---|---|
| LRS (locally redundant) | 3 copies in one data center | Cheapest. Learning and test data |
| ZRS (zone redundant) | 3 copies across different zones in one region | Protects against a data center failure |
| GRS (geo redundant) | LRS plus a copy in a second region | Protects against a whole region failing |
| GZRS | ZRS plus a copy in a second region | The highest protection |

The more protection you choose, the more you pay. For this module, LRS is enough. For production data that the business depends on, ZRS or GRS is the normal choice.

---

## 7. Access to the Data

Creating a storage account does not automatically let you read the files in it with your login. Being the owner of the subscription controls the resource, but reading and writing the data is a separate permission (the data role **Storage Blob Data Contributor**).

If the portal says you do not have permission to list the data, that is the reason. Day 4 explains roles properly. Today, if you hit this, follow the note in the tasks file.

The storage account also has access keys that give full access to everything. Treat them like passwords: never paste them into code, notes or GitHub.

---

## 8. Cost Reminder

- A storage account costs a small amount per GB, even when nobody uses it
- Large files, many tiers and extra redundancy all increase the cost
- Your budget alert from Day 1 is the safety net
- When the project ends, delete the resource group

---

## Key Takeaways for Today

1. A storage account holds containers, and containers hold folders and files
2. ADLS Gen2 is a storage account with the hierarchical namespace turned on, and it must be chosen at creation
3. One container per medallion layer: `bronze`, `silver`, `gold`
4. Pick one folder pattern and keep it everywhere
5. Tiers trade storage cost against read cost, and redundancy trades cost against protection
6. Owning the subscription is not the same as being allowed to read the data

Move to `tasks.md` now to build your data lake.