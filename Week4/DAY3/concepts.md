# Week 4, Day 3 - Secrets and GitHub
### Concepts

---

## Today in One Minute

Today you make sure nothing secret sits in your code, give Data Factory only the access it needs, and put your work on GitHub. This is what makes a project safe to share.

---

## 1. What Counts as a Secret

Storage account keys, SAS tokens, connection strings, passwords, API tokens (like your Kaggle token). If someone could use it to get into your resources, it is a secret.

If a secret reaches GitHub, treat it as stolen, even in a private repo. Rotate it.

---

## 2. Key Vault

Key Vault is a safe place for secrets. Code asks for the secret by name instead of containing it.

```
Notebook   -> dbutils.secrets.get(scope, name)  -> Key Vault
Data Factory linked service -> reads the secret from Key Vault
```

You set up a Key Vault secret scope in Week 2. Today you check that nothing was missed.

---

## 3. Managed Identity

Even better than a secret is no secret at all. Data Factory has its own identity in Azure. You give that identity a role on the storage account, and it connects without any key.

---

## 4. Least Privilege

Give each identity only the access it needs, at the smallest scope.

| Too much | Better |
|---|---|
| Owner on the subscription | Storage Blob Data Contributor on the storage account |

Data Factory only needs to read and write data in storage, so that one role is enough.

---

## 5. Git and GitHub

Git keeps the history of your files. GitHub stores them online. For this project it gives you a backup, a record of changes, and a portfolio piece.

Before every push, check that no secrets or data files are included. Your `.gitignore` from Week 3 Day 1 helps with this.

---

## Key Terms

| Term | Meaning |
|---|---|
| Secret | Anything that grants access |
| Key Vault | Azure's store for secrets |
| Managed identity | An identity Azure gives to a service, with no key to manage |
| Role assignment | Giving an identity a role on a resource |
| Least privilege | Only the access that is needed |