# Week 1, Day 4 - Access, Security and the Azure CLI

**Estimated time:** ~6 hours in total (about 1.5 hours of reading, the rest hands-on with `tasks.md`)

---

## Why This Day Matters

Yesterday you built a data lake. Today you decide who and what is allowed to use it. Most real-world cloud problems are not about missing features. They are about a script that gets "403 Forbidden", a password pasted into GitHub, or a person who has more access than they need. If you understand identities, roles and secrets now, every later week (Databricks, Data Factory, the final pipeline) becomes much easier to set up and debug.

By the end of today you will:
- Sign in to Azure from the command line
- Give yourself the right access to the lake, and see what happens without it
- Create a Key Vault and keep a secret in it
- Know when to use a user, a service principal or a managed identity

---

## 1. Authentication and Authorization

These two words are often mixed up. They answer two different questions.

| Term | Question it answers | Example |
|---|---|---|
| Authentication | Who are you? | You sign in with your email and password |
| Authorization | What are you allowed to do? | You may read files in `bronze`, but not delete the storage account |

Azure handles both. **Microsoft Entra ID** does authentication (it knows who you are). **RBAC** does authorization (it knows what you can do).

A useful way to remember: authentication is showing your ID at the door. Authorization is the list that says which rooms you may enter.

---

## 2. Who Can Be Given Access? (Identities in Microsoft Entra ID)

Access is always given to an identity. Entra ID has four kinds that matter for us:

| Identity | What it is | Typical use |
|---|---|---|
| User | A person with a sign-in | You, doing work in the portal or the CLI |
| Group | A collection of users | "Data team", so you grant access once for everyone |
| Service principal | An identity for an app or script, with its own credentials | A script running outside Azure, such as a GitHub pipeline |
| Managed identity | A service principal that Azure creates and looks after for you | A Data Factory or Databricks workspace reading the lake |

**Tip:** give access to groups rather than to individual people. When someone joins or leaves the team, you change the group membership and not dozens of role assignments.

---

## 3. RBAC: Role-Based Access Control

RBAC is the system that decides what an identity can do. Every permission in Azure is a **role assignment**, and every role assignment has three parts:

```
WHO         WHAT (role)                       WHERE (scope)
you     +   Storage Blob Data Contributor  +  storage account stbigdatadev001
```

- **Who:** a user, group, service principal or managed identity
- **What:** a role, which is a named bundle of permissions
- **Where:** the scope the role applies to

### Scope and inheritance

Scopes are nested, the same way Azure is organized:

```
Subscription
  └── Resource group
        └── Resource (storage account)
              └── Container
```

A role given at a higher level flows down to everything below it. If you give someone a role on the resource group, they have it on every resource inside it. If you give it on one container, it only applies to that container.

**Least privilege** means giving the smallest role at the smallest scope that still lets the person or app do the job. A script that only reads bronze should get a read-only role on the bronze container, not a role on the whole subscription.

### Control plane and data plane

This is the idea that surprises most beginners, and it is the reason you may have seen a permission message on Day 3.

| Plane | What it controls | Example actions |
|---|---|---|
| Control plane | The resource itself | Create or delete the storage account, change its tier, set tags |
| Data plane | The data inside the resource | Read, write or delete files in a container |

Being the **Owner** of a storage account lets you manage the account. It does **not** let you read the files in it. For that you need a data role.

**Example:** your friend owns a bank branch building. The owner can change the locks and repaint the walls. That does not mean they can open the customers' safe-deposit boxes. Those need a separate key.

### Roles you will meet in this module

| Role | Plane | What it allows |
|---|---|---|
| Owner | Control | Everything on the resource, including giving access to others |
| Contributor | Control | Create and manage resources, but not give access to others |
| Reader | Control | See that resources exist |
| Storage Blob Data Reader | Data | Read files and list folders |
| Storage Blob Data Contributor | Data | Read, write and delete files |
| Storage Blob Data Owner | Data | Everything on the data, including folder-level permission settings |
| Key Vault Secrets User | Data | Read the value of secrets |
| Key Vault Secrets Officer | Data | Create, update, list and delete secrets |

Role names and their exact permissions are listed in the Azure documentation. When in doubt, open the role in the portal and read its permissions.

### Things that catch people out

- **Propagation delay:** a new role assignment can take a few minutes to start working. If it fails right after you grant it, wait and try again.
- **Wrong scope:** a role on a different storage account or resource group does nothing for this one.
- **Wrong subscription:** the CLI may be pointing at another subscription than the one you think. Always check which subscription is active.

---

## 4. Service Principals

A service principal is an identity for software. Imagine a Python script that runs every night on a server outside Azure. It cannot type a password into a sign-in page, so it needs its own identity.

To create one, Azure registers an **app** in Entra ID. That gives you:

| Value | What it is | Secret? |
|---|---|---|
| Tenant ID | Which directory the app belongs to | No |
| Client ID (app ID) | The app's name tag | No |
| Client secret | The app's password | Yes, treat it like a password |

The script signs in with those three values and gets the roles that were assigned to the service principal.

The weak point is the **client secret**. It can leak, it expires, and someone has to rotate it. That is why Azure offers managed identities for anything that runs inside Azure.

---

## 5. Managed Identities

A managed identity is a service principal where Azure keeps the credentials for you. You never see or store a password.

How it works in practice:
1. You turn on a managed identity for an Azure resource, for example a Data Factory
2. Azure creates the identity and handles its credentials
3. You give that identity a role, for example Storage Blob Data Contributor on the lake
4. The Data Factory can now read and write the lake, with no secret anywhere

There are two types:

| Type | Lifetime | Use |
|---|---|---|
| System-assigned | Tied to one resource and deleted with it | The simple default |
| User-assigned | Created on its own and shared by several resources | When many resources need the same access |

**Rule of thumb:**
- Code running in Azure (Data Factory, Databricks, a virtual machine): use a managed identity
- Code running outside Azure: use a service principal
- You, working by hand: use your own user sign-in

In Week 2 you will give Data Factory a managed identity so it can reach your lake.

---

## 6. Azure Key Vault

Key Vault is a protected store for sensitive values. It can hold three kinds of items:

| Item | Example |
|---|---|
| Secret | A password, an API key, a connection string |
| Key | An encryption key |
| Certificate | A TLS certificate |

In this module we only use **secrets**.

### Why not just put the password in the code?

Because code gets shared. A password in a script ends up in Git history, in screenshots and in chat messages, and once it is on GitHub, bots find it within minutes. With Key Vault:

- The secret lives in one place
- Access to it is controlled by RBAC and recorded in logs
- You can change the secret in one place without touching any code
- Old values are kept as versions

### How an app uses Key Vault

```
1. The app signs in with its identity (managed identity or your login)
2. The app asks Key Vault for "weather-api-key"
3. Key Vault checks the role assignment (Key Vault Secrets User)
4. Key Vault returns the value
5. The app uses it in memory and never writes it down
```

### Permission model

A Key Vault can authorize access in two ways: the older **access policies**, or **Azure RBAC**. We use Azure RBAC, because it works the same way as everything else you learned today. When you create the vault, choose the Azure RBAC permission model.

With RBAC, being the Owner of the vault is not enough to read or write secrets. You need a data role such as Key Vault Secrets Officer. This is the same control plane and data plane idea again.

### Good to know

- **Names:** a vault name is 3 to 24 characters, letters, numbers and hyphens, and must be unique across Azure.
- **Soft delete:** a deleted secret or vault is kept for a retention period (90 days by default) and can be recovered. A consequence: if you delete a vault, its name stays reserved until it is purged, so you cannot immediately create a new vault with the same name.
- **Versions:** each time you set a secret with the same name, a new version is created. Apps normally read the latest.
- **Cost:** small. You pay per operation, and storing a few secrets costs very little.

---

## 7. The Azure CLI

The Azure CLI (`az`) lets you do from a terminal what you do in the portal. It matters because commands can be saved, repeated and shared, while clicks cannot.

### Signing in

```bash
az login                 # opens a browser to sign in
az account show -o table # which account and subscription is active?
az account list -o table # all subscriptions you can see
az account set --subscription "<name or id>"
```

After `az login`, other tools can reuse the same sign-in. You will see this on Day 5, when a Python script uses your CLI login to reach the lake.

### Command pattern

Commands follow the shape `az <service> <group> <action> --options`:

```bash
az group show --name rg-bigdata-dev
az storage account show --name stbigdatadev001
az keyvault secret list --vault-name kv-bigdata-dev-001
```

Use `--help` on any command, for example `az keyvault create --help`.

### Controlling the output

| Option | Result |
|---|---|
| `-o table` | A readable table |
| `-o json` | Full JSON (the default) |
| `-o tsv` | Plain values, good for storing in a variable |
| `--query "id"` | Pick one field (JMESPath syntax) |

Example: get a resource ID as plain text.

```bash
az storage account show --name stbigdatadev001 --query id -o tsv
```

### The `--auth-mode login` option

Many `az storage` commands default to using the account key. Add `--auth-mode login` to use your own sign-in and RBAC instead. This is the habit we want, since it does not involve keys at all.

### Shell differences

Examples here use bash style. In PowerShell, set variables with `$name = ...` and in Windows Command Prompt, quoting rules differ. If a command with quotes fails, check the quoting first.

---

## 8. Keeping Secrets Safe: Habits

1. Never put keys, passwords or connection strings in code, notebooks or GitHub.
2. Prefer sign-in with identity (your login, a managed identity) over keys.
3. If you must have a secret, keep it in Key Vault and read it at run time.
4. Do not print secrets to the terminal or paste them into notes. Terminal history can keep them.
5. Give the smallest role at the smallest scope.
6. If a secret may have leaked, rotate it right away, and do not just delete the message.

---

## 9. A Preview of Tomorrow: DefaultAzureCredential

On Day 5 your Python code will use `DefaultAzureCredential` from the `azure-identity` package. It tries several ways to sign in, in order, and uses the first one that works:

| Where the code runs | What it uses |
|---|---|
| Your laptop, after `az login` | Your Azure CLI sign-in |
| A service principal setup | Environment variables with the app's values |
| Azure (Data Factory, Databricks, VM) | The managed identity |

The same code works everywhere, with no password inside it. That is why signing in with the CLI today is not just an exercise.

---

## Common Problems and What They Mean

| Message or symptom | Likely cause | Fix |
|---|---|---|
| 403 `AuthorizationPermissionMismatch` on storage | You have no data role on the account | Assign Storage Blob Data Contributor or Reader |
| "Caller is not authorized" on Key Vault | You have no Key Vault data role | Assign Key Vault Secrets Officer or User |
| Role assigned, but still denied | Propagation delay, wrong scope or wrong subscription | Wait a few minutes, check scope and `az account show` |
| Command works in portal, fails in CLI | CLI is using another subscription | `az account set --subscription ...` |
| Vault name "already in use" after deleting | Soft delete keeps the name reserved | Pick a new name, or purge the old vault |

---

## Key Takeaways for Today

1. Authentication asks who you are, and authorization asks what you may do
2. A role assignment is who, plus what role, plus where (scope), and it flows downward
3. Owning a resource is control plane, and reading its data needs a separate data role
4. Use least privilege: the smallest role at the smallest scope
5. Managed identity inside Azure, service principal outside Azure, your own login by hand
6. Secrets belong in Key Vault, and never in code or GitHub
7. The CLI makes setup repeatable, and `--auth-mode login` avoids account keys

Move to `tasks.md` now to practice all of this on your own lake.