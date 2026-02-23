## Notes — Fetching Azure Key Vault Secrets & App Registration (GitHub Actions → Azure)

---

# 1. Overall Concept (Big Picture)

**Goal:**
Allow **GitHub Actions pipeline** to securely access **Azure resources (Key Vault)** **without storing secrets** in GitHub.

**Solution Used:**
➡️ **Azure AD App Registration + OpenID Connect (OIDC)**

Flow:

```
GitHub Workflow
      ↓
OIDC Token (temporary identity)
      ↓
Azure AD App Registration (Service Principal)
      ↓
Azure Role Assignment
      ↓
Azure Key Vault
      ↓
Fetch Secrets securely
```

No client secret required.

---

# 2. Why App Registration is Needed

Azure must know:

* WHO is accessing resources
* FROM WHERE access is coming
* WHAT permissions are allowed

App Registration acts as:

✅ Identity for GitHub Actions
✅ Trust bridge between GitHub and Azure
✅ Security boundary

It creates a **Service Principal** automatically.

---

# 3. Authentication Types (Important)

## Old Method (Not Recommended)

```
Client ID
Client Secret
Tenant ID
```

Problems:

* Secrets expire
* Manual rotation needed
* Security risk

---

## Modern Method — OIDC (Recommended)

GitHub generates **temporary identity token**.

Azure verifies:

* repository
* branch
* environment
* workflow source

No stored secrets.

---

# 4. Entity Type in App Registration (Most Confusing Part)

During Federated Credential creation you select:

| Entity Type  | Meaning                  | When Used            |
| ------------ | ------------------------ | -------------------- |
| Branch       | Specific branch workflow | Most common          |
| Environment  | Deployment environments  | Production pipelines |
| Pull Request | PR workflows             | Validation jobs      |
| Tag          | Release pipelines        | Version releases     |

---

## Why Azure Cares About Entity Type

Azure validates **WHO exactly is requesting access**.

Example:

If configured for:

```
repo:myorg/app
branch:main
```

Then:

✅ workflow from `main` → allowed
❌ workflow from `dev` → denied

This prevents unauthorized pipelines.

---

## Example

### Federated Credential Condition

```
Repository : org/project
Entity Type: Branch
Branch     : main
```

Only:

```
.github/workflows/*.yml
running from main branch
```

can login to Azure.

---

# 5. Best Practice — Entity Type Selection

### Recommended Setup

| Stage         | Entity Type              |
| ------------- | ------------------------ |
| CI Validation | Pull Request             |
| Dev Deploy    | Branch (dev)             |
| Prod Deploy   | Environment (production) |
| Release       | Tag                      |

Production should ideally use **Environment** because GitHub supports approvals.

---

# 6. Azure Setup Steps (Summary Notes)

### Step 1 — Create App Registration

Azure Portal → Entra ID → App Registrations → New Registration

Outputs:

* Client ID
* Tenant ID

---

### Step 2 — Add Federated Credential

```
App Registration
 → Certificates & Secrets
 → Federated Credentials
 → Add Credential
```

Choose:

```
Issuer: GitHub
Repository: owner/repo
Entity Type: Branch
Branch: main
```

---

### Step 3 — Assign Azure Role

Go to resource (Key Vault):

```
Access Control (IAM)
 → Add Role Assignment
```

Typical roles:

| Role                   | Purpose             |
| ---------------------- | ------------------- |
| Key Vault Secrets User | Read secrets        |
| Key Vault Reader       | View metadata       |
| Contributor            | Avoid unless needed |

Principle = App Registration name.

---

# 7. Key Vault Permission Model (Important)

Two models exist:

## 1️⃣ Access Policies (Old)

Manual permission configuration.

## 2️⃣ RBAC (Recommended)

Uses Azure roles.

Use RBAC whenever possible.

---

# 8. GitHub Workflow Authentication

### Required Permission

```yaml
permissions:
  id-token: write
  contents: read
```

`id-token: write` allows GitHub to request OIDC token.

---

### Azure Login Step

```yaml
- name: Azure Login
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

No client secret needed.

---

# 9. Fetch Secrets from Key Vault

### Method 1 — Azure CLI

```yaml
- name: Get Secret
  run: |
    SECRET=$(az keyvault secret show \
      --name db-password \
      --vault-name my-kv \
      --query value -o tsv)

    echo "DB_PASSWORD=$SECRET" >> $GITHUB_ENV
```

---

### Method 2 — Azure Key Vault Action

```yaml
- uses: azure/get-keyvault-secrets@v1
  with:
    keyvault: my-kv
    secrets: db-password
```

---

# 10. Complete Flow (Interview Explanation)

**How it works internally:**

1. GitHub workflow starts.
2. GitHub requests OIDC token.
3. Token contains:

   * repo name
   * branch
   * workflow identity
4. Azure validates token against federated credential.
5. Azure issues temporary access token.
6. Workflow accesses Key Vault using RBAC permissions.

---

# 11. Common Issues & Fixes (Earlier Problems)

## Issue 1 — Login Failed

Cause:

* Missing `id-token: write`

Fix:

```yaml
permissions:
  id-token: write
```

---

## Issue 2 — Unauthorized to Key Vault

Cause:

* Role not assigned.

Fix:
Assign:

```
Key Vault Secrets User
```

---

## Issue 3 — Works locally but fails in pipeline

Cause:

* Entity type mismatch.

Example:
Configured → `main`
Running → `dev`

---

## Issue 4 — Federated Credential Not Matching

Check:

```
repo name
branch name
case sensitivity
```

---

# 12. Security Advantages

OIDC provides:

✅ No stored secrets
✅ Short-lived tokens
✅ Branch-level restriction
✅ Environment approvals
✅ Zero credential rotation

---

# 13. Mental Model (Easy Memory Trick)

Think:

```
App Registration = Identity Card
Federated Credential = Trust Rule
Role Assignment = Permission
Key Vault = Resource
GitHub Workflow = Person using ID
```

---

# 14. Interview One-Line Answer

> GitHub Actions authenticates to Azure using OIDC federation via an App Registration, where Azure validates repository identity and grants temporary RBAC-based access to fetch secrets from Azure Key Vault securely without storing credentials.

---
