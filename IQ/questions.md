## 1. scheduling workflow in github actions

```
name: Scheduled Job

on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

jobs:
  run-script:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run script
        run: echo "Scheduled workflow running"
  ```

## 2. GitHub Actions — Different Types of Triggers (with Examples)

Triggers define **when a workflow starts**.
They are configured inside the `on:` section of a workflow YAML.

---

### 1. Push Trigger

Runs workflow when code is pushed to a branch.

```
on:
  push:
    branches:
      - main
```

**Use cases**

* CI builds
* Unit testing after commits
* Automatic deployments

---

### 2. Pull Request Trigger

Runs when a pull request is created or updated.

```yaml
on:
  pull_request:
    branches:
      - main
```

**Triggered on**

* PR opened
* Commit added to PR
* PR reopened

**Use cases**

* Code validation
* PR checks
* Security scans

---

### 3. Manual Trigger (`workflow_dispatch`)

Allows manual execution from GitHub UI.

```
on:
  workflow_dispatch:
```

### With Inputs

```
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Deploy environment"
        required: true
```

**Use cases**

* Manual deployments
* Emergency runs
* Testing pipelines

---

### 4. Scheduled Trigger (`schedule`)

Runs automatically using cron timing.

```
on:
  schedule:
    - cron: "0 3 * * *"
```

**Use cases**

* Nightly builds
* Backups
* Cleanup jobs

(Time is UTC.)

---

### 5. Workflow Run Trigger

Runs when another workflow completes.

```
on:
  workflow_run:
    workflows: ["Build Workflow"]
    types:
      - completed
```

**Use cases**

* Multi-stage pipelines
* CI → CD separation

---

### 6. Repository Dispatch (External Trigger)

Triggered via API call.

```
on:
  repository_dispatch:
    types: [deploy-event]
```

Triggered using GitHub API.

**Use cases**

* External system triggers
* Cross-repository automation

---

### 7. Release Trigger

Runs when a release is created or published.

```
on:
  release:
    types: [published]
```

**Use cases**

* Publish packages
* Production deployments

---

### 8. Issue / Issue Comment Trigger

Runs when issues are created or modified.

```
on:
  issues:
    types: [opened]
```

or

```
on:
  issue_comment:
    types: [created]
```

**Use cases**

* Automation bots
* Label management

---

### 9. Tag Trigger

Runs when tags are pushed.

```
on:
  push:
    tags:
      - "v*"
```

**Use cases**

* Versioned releases
* Artifact publishing

---

### 10. Fork Trigger

Triggered when repository is forked.

```
on:
  fork
```

**Use cases**

* Analytics
* Notifications

---

### 11. Deployment Trigger

Runs during GitHub deployments.

```
on:
  deployment
```

**Use cases**

* Environment automation
* Deployment tracking

---

### 12. Multiple Triggers in One Workflow

You can combine triggers.

```
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:
```

Workflow runs for any listed event.

---

### 13. Most Common Triggers (Real Projects)

| Trigger           | Purpose                |
| ----------------- | ---------------------- |
| push              | Continuous Integration |
| pull_request      | Code validation        |
| workflow_dispatch | Manual deployment      |
| schedule          | Nightly automation     |
| workflow_run      | Pipeline chaining      |
| release           | Production release     |

---

## 14. Interview Summary (Short Answer)

GitHub Actions workflows are triggered by repository events such as `push`, `pull_request`, `schedule`, `workflow_dispatch`, `release`, and `workflow_run`. These events initiate workflow execution through GitHub’s event-driven automation system.

## 3. gihut reusable workflows
In GitHub Actions, one workflow can call another workflow using **Reusable Workflows**.

This is done using:

```
workflow_call
```

---

### 1. Called Workflow (Reusable Workflow)

Create a workflow that can be invoked by others.

**File:**
`.github/workflows/deploy.yml`

```
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      AZURE_CREDENTIALS:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Show environment
        run: echo "Deploying to ${{ inputs.environment }}"

      - name: Azure Login
        uses: Azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
```

### Key part

```yaml
on:
  workflow_call:
```

This makes the workflow reusable.

---

### 2. Caller Workflow (Main Pipeline)

Now create another workflow that calls the above one.

**File:**
`.github/workflows/main.yml`

```
name: Main Workflow

on:
  push:
    branches: [main]

jobs:
  call-deploy:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: dev
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
```

---

### How it works (Flow)

```
push → main.yml runs
        ↓
calls deploy.yml
        ↓
deploy job executes
```

---

## Important Rules

### 1. Must use `workflow_call`

Without this, workflow cannot be reused.

---

### 2. Use `uses:` at **job level**, not step level

Correct:

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/deploy.yml
```

Wrong:

```yaml
steps:
  - uses: ./.github/workflows/deploy.yml
```

---

### 3. Passing Inputs

Defined in called workflow:

```yaml
inputs:
  environment:
    type: string
```

Used as:

```yaml
${{ inputs.environment }}
```

---

### 4. Passing Secrets

Caller must explicitly pass secrets.

---

### Calling workflow from another repository

```
jobs:
  deploy:
    uses: org-name/repo-name/.github/workflows/deploy.yml@main
```

Example structure:

```
org/reusable-pipelines/.github/workflows/deploy.yml
```

---

### When to use reusable workflows

* Same deployment across environments
* Terraform pipelines
* Azure login + deployment logic reuse
* Standard CI templates across repos

---

### Reusable Workflow vs Composite Action

| Feature            | Reusable Workflow | Composite Action |
| ------------------ | ----------------- | ---------------- |
| Runs jobs          | ✅                 | ❌                |
| Multiple runners   | ✅                 | ❌                |
| Best for pipelines | ✅                 | ❌                |
| Step reuse         | ⚠️                | ✅                |

---

## 4. In **GitHub Actions YAML**, `inputs` and `with` are related but used in **different places and purposes**.

---

### 1. `inputs` — Define Parameters (Declaration)

`inputs` are used to **define parameters** that a workflow or action can receive.

They are like **function arguments definition**.

You declare them when:

* Creating a **reusable workflow**
* Creating a **custom action**
* Manual workflow trigger (`workflow_dispatch`)

---

### Example — Manual Workflow Inputs

```yaml id="eq4n3u"
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Deployment environment"
        required: true
        default: dev
```

Here:

* `environment` is an **input parameter**
* User provides value when running workflow manually.

Access it using:

```yaml id="a0d4ph"
${{ inputs.environment }}
```

---

### Example — Reusable Workflow Inputs

```yaml id="6v8i8l"
on:
  workflow_call:
    inputs:
      app-name:
        required: true
        type: string
```

This workflow now expects an input.

---

### 2. `with` — Pass Values (Usage)

`with` is used to **send values to an action or reusable workflow**.

It supplies values to predefined inputs.

---

### Example — Passing Inputs to an Action

```yaml id="79bgc1"
- uses: Azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

Here:

* `Azure/login` action already defines inputs.
* `with` provides values to them.

---

### 3. Relationship Between `inputs` and `with`

```text id="5ep9dp"
inputs  → defines parameters
with    → provides values to parameters
```

Like programming:

```text id="08p5t6"
function deploy(env)   ← inputs
deploy("prod")         ← with
```

---

### 4. Full Example (Reusable Workflow)

### Reusable workflow

```yaml id="pnq0an"
# deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
```

---

### Caller workflow

```yaml id="xtju5n"
jobs:
  call-deploy:
    uses: org/repo/.github/workflows/deploy.yml@main
    with:
      environment: prod
```

Flow:

1. `inputs` defined in reusable workflow
2. `with` sends value from caller

---

### 5. Where Each Is Used

| Keyword  | Used Where                 | Purpose               |
| -------- | -------------------------- | --------------------- |
| `inputs` | workflow/action definition | Declare parameters    |
| `with`   | step/job usage             | Pass parameter values |

---

### 6. Common Places You’ll See Them

### A. Manual Run Parameters

```yaml id="bs59i6"
on:
  workflow_dispatch:
    inputs:
```

### B. Calling Actions

```yaml id="0zddcc"
- uses: actions/checkout@v4
  with:
```

### C. Reusable Workflows

```yaml id="kg4p7h"
on:
  workflow_call:
    inputs:
```

---

### 7. Quick DevOps Interview Answer

**`inputs` define configurable parameters for workflows or actions, while `with` passes actual values to those inputs when executing an action or reusable workflow.**

---

### 8. Simple Memory Rule

```text id="v2r0w9"
inputs = declare variables
with   = assign values
```

---
