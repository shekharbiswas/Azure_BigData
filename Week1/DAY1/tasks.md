# Week 1, Day 1 — Tasks

**Total time:** ~6 hours. A light day. Work through these in order, each block builds on the last.

---

## Block 1 (60 min) — Create Your Azure Account

1. Go to the Azure sign-up page and create a free account. If you are a student, check whether Azure for Students is available to you.
2. Complete the identity and payment-card verification. The card is only used for verification, so do not share it with anyone and do not store it in your notes.
3. Sign in to the Azure portal (portal.azure.com).
4. Open **Subscriptions** from the search bar and confirm you can see one active subscription. Note its name.
5. Read the free-offer terms shown for your account: how long the credit lasts and what happens when it ends.

**Checkpoint:** You can sign in to the portal and see an active subscription.

---

## Block 2 (60 min) — Explore the Portal

Spend time clicking around, without creating anything yet.

1. Use the search bar to open: Resource groups, Cost Management + Billing, Microsoft Entra ID, and Activity log.
2. In Microsoft Entra ID, find your tenant name and your own user.
3. Open **Create a resource** and browse the categories. Find where Storage accounts, Data Factory and Azure Databricks appear (do not create them).
4. Pin Resource groups and Cost Management to your portal favorites.

**Checkpoint:** You can find any of the four pages above using the search bar.

---

## Block 3 (60 min) — Create Your First Resource Group

1. Choose the region you will use for the whole module (for example Central India or South India) and write it down.
2. Create a resource group named `rg-bigdata-dev` in that region.
3. Add two tags: `project = bigdata` and `owner = ` your name.
4. Open the group after it is created and confirm it is empty.
5. Look at the group's **Access control (IAM)** page and find your own role. Do not change anything.

**Checkpoint:** `rg-bigdata-dev` exists in your chosen region with both tags.

---

## Block 4 (90 min) — Set a Budget Alert

This is the most important task of the day.

1. Open **Cost Management + Billing** and go to **Budgets**.
2. Create a budget for your subscription with a small monthly amount you are comfortable with.
3. Add alerts at 50%, 80% and 100% of the budget, sent to your email.
4. Save it, then confirm it appears in the budget list.
5. Look at the **Cost analysis** page and note what it currently shows (probably zero).

**Checkpoint:** A budget with three email alerts exists on your subscription.

---

## Block 5 (60 min) — Naming Convention Sheet

Create a short text file called `azure-names.md` in your project folder and fill it in:

1. Your chosen region.
2. Your naming pattern: `<type>-<project>-<env>`.
3. The planned names for the resources you will create this week, filling in each line:
   - Resource group: `rg-bigdata-dev`
   - Storage account: (must be 3-24 characters, lowercase letters and numbers only, and unique across Azure, so add a few digits of your own)
   - Key Vault: kv-bigdata-dev-4821
   - Data Factory (for Week 2):
4. A line stating your cost rules: budget alert set, one resource group per project, delete at project end.

**Checkpoint:** The file exists and every name follows the pattern.

---

## Block 6 (60 min) — Mini Task: "My Azure Map"

In your own words, in a file called `azure-map.md`:

1. Draw the hierarchy (tenant, subscription, resource group, resource) as a text diagram, and fill in the real names from your account for each level.
2. Write 3 sentences answering: why does every project get its own resource group?
3. Write 3 sentences answering: which two kinds of resource can keep costing money while you are not using them, and what will you do about each?

**Checkpoint:** You can explain the hierarchy and the cost habits without looking at the concepts file.

---

## End of Day Checklist

- [ ] Azure account created and a subscription is visible
- [ ] Region chosen and written down
- [ ] `rg-bigdata-dev` created with two tags
- [ ] Budget with email alerts set on the subscription
- [ ] `azure-names.md` and `azure-map.md` written
- [ ] Can explain: tenant vs subscription vs resource group vs resource
- [ ] Can explain why deleting a resource group is the cleanest way to end a project

If any checkbox is unchecked, finish it before Day 2. Tomorrow moves from Azure itself to the data side: what big data is and how a data lake is designed.
