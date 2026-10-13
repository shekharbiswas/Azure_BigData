# Week 1, Day 1 — Cloud and Azure Basics

**Estimated time:** ~3 hours (a light day: short reading + account setup, paired with `tasks.md`)

---

## Why This Day Matters

Every pipeline in this module runs on Azure. Before touching any data tool, you need an account, a clear picture of how Azure is organized, and a cost safety net. Today has no code. The goal is to get set up the right way so that nothing later surprises you, especially the bill.

---

## 1. What Is Cloud Computing?

Cloud computing means renting computing power and storage from a provider over the internet, instead of buying and running your own servers. You pay for what you use, and you can scale up or down in minutes.

Why this matters for big data: processing a very large dataset may need 20 machines for 10 minutes. On your own hardware you would have to buy all 20 and keep them forever. In the cloud you rent them for 10 minutes and release them.

### Three service levels

| Level | What you manage | Azure example |
|---|---|---|
| IaaS (infrastructure) | The operating system and everything above it | Virtual machines |
| PaaS (platform) | Only your data and code | Data Factory, Databricks, Storage |
| SaaS (software) | Nothing, you just use it | Microsoft 365, Power BI |

In this module we mostly use PaaS: Azure runs the machines, and we focus on the data.

---

## 2. What Is Azure?

Azure is Microsoft's cloud platform, with hundreds of services. You will only use a few. They fall into groups:

- **Storage**: Blob storage, Data Lake Storage Gen2
- **Analytics and data processing**: Databricks, Data Factory, Synapse
- **Security**: Key Vault, Microsoft Entra ID
- **Management**: Azure Monitor, Cost Management

Everything is created, billed and deleted the same way, which is why the next section matters.

---

## 3. How Azure Is Organized

```
Tenant (Microsoft Entra ID)
  └── Subscription
        └── Resource group
              └── Resource
```

- **Tenant:** your organization's identity directory. Your user account lives here.
- **Subscription:** the billing boundary. All costs land on a subscription.
- **Resource group:** a folder for one project's resources. You create one per project.
- **Resource:** one actual item, such as a storage account or a Data Factory.

**The most useful rule in Azure:** deleting a resource group deletes everything inside it. That is how you clean up a project in one step, and it is why every project gets its own group.

---

## 4. Regions

A region is the physical location of Azure's data centers, for example Central India or West Europe. Each resource lives in exactly one region.

Choose one region and use it for everything in the module:
- Data moving between regions can add cost and delay.
- Not every service or size is available in every region.
- The region decides where your data physically sits, which can matter for legal reasons.

**Common trap:** creating one resource in a different region by accident because the default was different. Check the region field every time you create something.

---

## 5. The Azure Portal

The portal is the web interface at portal.azure.com. The parts you will use most:

- **Search bar (top):** the fastest way to open any service or resource.
- **Resource groups:** lists your groups and everything inside them.
- **Create a resource:** the starting point for any new service.
- **Cost Management + Billing:** shows spend and lets you set budgets.
- **Activity log:** shows who did what, useful when something changes unexpectedly.

---

## 6. Cost — Know This on Day 1

Azure is pay-as-you-go. Some things cost money all the time, others only while running.

| Resource | Costs when idle? |
|---|---|
| Storage account with data | Yes, a small amount per GB |
| Running Databricks cluster | Yes, a lot, until you stop it |
| Data Factory pipeline | Only when it runs |
| Resource group (empty) | No |

**Habits to build now:**
1. Set a **budget alert** so you get an email before spend gets out of hand.
2. Delete the resource group when a project ends.
3. Stopping something is not always the same as deleting it. Check that billing actually stopped.

Free offers: new accounts usually get a free trial credit and some free monthly service tiers, and students may qualify for Azure for Students. The terms differ by country and change over time, so read the current offer on the sign-up page.

---

## 7. Naming and Tags

Consistent names make a growing set of resources easy to read. Use `<type>-<project>-<env>`:

```
rg-bigdata-dev        resource group
adf-bigdata-dev       data factory
kv-bigdata-dev-001    key vault
stbigdatadev001       storage account (lowercase letters and numbers only)
```

**Tags** are labels you attach to a resource, for example `project = bigdata` and `owner = your-name`. They make it easy to see what belongs to what in the cost reports.

---

## Key Takeaways for Today

1. Cloud means renting resources and paying for use. In this module we mostly use PaaS services.
2. Azure is organized as tenant, subscription, resource group, resource.
3. One resource group per project, and deleting the group removes everything in it.
4. Pick one region and use it everywhere.
5. Set a budget alert before creating anything else.

Move to `tasks.md` now to set up your account and put this into practice.