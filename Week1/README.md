# Week 1 — Azure and Big Data Foundations
### Module: Big Data with Azure

---

## Overview

This week builds the foundation for everything in the module: how Azure is organized, what "big data" actually means, and how data is stored, secured and accessed in the cloud. No prior Azure experience is assumed. By Friday, you'll have a secure data lake on Azure with bronze, silver and gold containers, and a Python script that reads and writes files in it.

**Prerequisites:** Comfortable with SQL and basic Python. An email address and a payment card for the Azure account sign-up (the card is only for identity verification).

**Tools used this week:** Azure portal, Azure CLI, VS Code, Python 3, `azure-identity`, `azure-storage-file-datalake`, `pandas`, `pyarrow`.

---

## Structure

Each day has two files:
- **`concepts.md`** — explains the day's ideas with short, clear examples
- **`tasks.md`** — hands-on exercises to practice what you just learned, plus an end-of-day checklist

```
week-1/
├── README.md          (this file)
├── day-1/
│   ├── concepts.md
│   └── tasks.md
├── day-2/
│   ├── concepts.md
│   └── tasks.md
├── day-3/
│   ├── concepts.md
│   └── tasks.md
├── day-4/
│   ├── concepts.md
│   └── tasks.md
└── day-5/
    ├── concepts.md
    └── tasks.md
```

---

## Azure Basics at a Glance

This section is a reference for the whole module. Come back to it whenever a term is unfamiliar.

### How Azure is organized

```
Tenant (Microsoft Entra ID)       your organization and its users
  └── Subscription                the billing boundary
        └── Resource group        a folder for one project's resources
              └── Resource        a storage account, a Data Factory, a cluster...
```

### Key terms

| Term | Meaning |
|---|---|
| Tenant | Your organization's identity directory (Microsoft Entra ID). Users, groups and apps live here. |
| Subscription | The account that gets billed. Resources are created inside a subscription. |
| Resource group | A container for related resources. Deleting it deletes everything inside. |
| Resource | Any single Azure item: a storage account, a database, a Data Factory. |
| Region | The physical location of the data centers (for example Central India). Pick one and keep everything in it. |
| Storage account | The top-level Azure storage item. Holds containers of files. |
| ADLS Gen2 | A storage account with the hierarchical namespace turned on, giving real folders. Built for big data. |
| Container | A top-level folder inside a storage account (we use `bronze`, `silver`, `gold`). |
| RBAC | Role-based access control. Decides who can do what on which resource. |
| Service principal | An identity for an app or script, not a person. |
| Managed identity | A service principal that Azure manages for you, with no password to store. |
| Key Vault | A safe place for secrets such as keys and connection strings. |

### Services used in this module

| Service | What it does | Where it appears |
|---|---|---|
| Azure Data Lake Storage Gen2 | Stores raw and processed data as files | Week 1 onward |
| Azure Key Vault | Stores secrets | Week 1 (Day 4), then all weeks |
| Azure Data Factory | Moves data and orchestrates pipelines | Week 2, Weeks 3-4 |
| Azure Databricks | Runs PySpark and Delta Lake processing | Week 2, Weeks 3-4 |
| Azure Monitor | Alerts and run monitoring | Week 4 |
| Databricks SQL or Synapse serverless | Queries the final tables | Week 4 |

### Ways to work with Azure

| Way | Best for |
|---|---|
| Azure portal (web) | Learning, exploring, one-off setup |
| Azure CLI (`az` commands) | Repeatable setup from a terminal |
| Python SDK | Reading and writing data from scripts |

Infrastructure-as-code tools (Bicep, Terraform) exist but are not covered in this module.

### Naming convention for the module

Use one pattern for everything: `<type>-<project>-<env>`.

| Resource | Example |
|---|---|
| Resource group | `rg-bigdata-dev` |
| Data Factory | `adf-bigdata-dev` |
| Key Vault | `kv-bigdata-dev-001` |
| Storage account | `stbigdatadev001` |

Storage account names are special: 3 to 24 characters, lowercase letters and numbers only, and unique across all of Azure.

### Cost and safety rules (all four weeks)

- Set a budget alert on day 1.
- Use one resource group per project, and delete it when the project ends.
- Keep all resources in the same region.
- Storage costs a little even when idle. Running clusters cost a lot. Always set auto-terminate on Databricks clusters.
- Never put keys, passwords or connection strings in code or in GitHub.

---

## Day-by-Day Breakdown

### Day 1 — Cloud and Azure Basics (light day)
What cloud computing is, how Azure is organized (tenant, subscription, resource group, resource), regions, and how costs work. Create the Azure account, explore the portal, create a resource group and set a budget alert. No code today.

### Day 2 — Big Data Concepts and Data Lake Architecture
The three Vs, why one machine is not enough, distributed processing and partitioning, batch vs streaming, data lake vs data warehouse, and the medallion layout (bronze, silver, gold). Design the lake layout for the project.

### Day 3 — Azure Storage and ADLS Gen2
Blob storage vs ADLS Gen2, the hierarchical namespace, containers and folder design, storage tiers and redundancy options. Create a storage account, build the bronze/silver/gold containers and upload the first dataset.

### Day 4 — Access, Security and the Azure CLI
RBAC and role assignments, service principals and managed identities, Key Vault and secrets, signing in with the Azure CLI. Grant yourself correct access to the lake and store a secret in Key Vault.

### Day 5 — File Formats and Python with ADLS
CSV, JSON and Parquet compared on size and speed, why columnar formats win at scale, and reading and writing files in ADLS from Python with the Azure SDK. Closes the week with a script that works against your own data lake.

---

## By End of Week 1, You Should Be Able To:

- [ ] Explain the Azure hierarchy (tenant, subscription, resource group, resource) and why regions matter
- [ ] Set up a budget alert and explain which resources cost money while idle
- [ ] Explain the medallion layout and what belongs in bronze, silver and gold
- [ ] Create an ADLS Gen2 account with bronze, silver and gold containers
- [ ] Grant access with RBAC and keep a secret in Key Vault
- [ ] Compare CSV, JSON and Parquet using real numbers from your own data
- [ ] Read and write files in ADLS from a Python script

---

## Looking Ahead

Week 2 puts this foundation to work: Azure Databricks with PySpark and Delta Lake for processing at scale, and Azure Data Factory for moving and orchestrating data. The data lake you build this week is where every Week 2 exercise starts.