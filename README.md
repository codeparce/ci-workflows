# CI Workflows for Azure Static Web Apps

This repository contains reusable GitHub Actions workflows for building and deploying Node.js applications to Azure Static Web Apps using Terraform for infrastructure deployment.

## Workflow Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                        node-init.yml                             │
│                    (Main Orchestrator)                          │
│                                                                 │
│  Inputs:                                                        │
│    - node-version: Node.js version for the project             │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌─────────────────┐   ┌─────────────────────┐
│ node-build    │   │ azure-arquitecture│   │ azure-deploy        │
│               │   │                  │   │                     │
│ - Checkout    │   │ - Checkout       │   │ Needs:              │
│ - Setup Node  │   │ - Clone terraform│   │   - azure-terraform │
│ - Install deps│   │   scripts repo   │   │   - node            │
│ - npm build   │   │ - Setup Terraform│   │                     │
│ - Upload      │   │ - Deploy infra   │   │ Steps:              │
│   artifact    │   │ - Upload secrets │   │ - Download build    │
└───────────────┘   └─────────────────┘   │ - Download secrets  │
                                         │ - SWA deploy         │
                                         └─────────────────────┘
```

## Workflows

### 1. node-init.yml (Entry Point)

Main workflow that orchestrates the entire CI/CD pipeline. It's designed to be called from external repositories.

**Trigger:** `workflow_call`

**Inputs:**
| Name | Required | Description |
|------|----------|-------------|
| `node-version` | Yes | Node.js version to use for building |

**Calls:**
- `node-build.yml` - Builds the Node.js application
- `azure-arquitecture.yml` - Deploys Azure infrastructure via Terraform
- `azure-web-app.yml` - Deploys the application to Azure Static Web Apps

---

### 2. node-build.yml

Reusable workflow that builds a Node.js project.

**Trigger:** `workflow_call`

**Inputs:**
| Name | Required | Description |
|------|----------|-------------|
| `node-version` | Yes | Node.js version |

**Outputs:**
- `web-build` artifact containing the `dist/` directory

---

### 3. azure-arquitecture.yml

Reusable workflow that deploys Azure infrastructure using Terraform.

**Trigger:** `workflow_call`

**Actions:**
1. Checks out the repository code
2. Clones `codeparce/azure-terraform` repository (azure branch)
3. Sets up Terraform 1.6.6
4. Runs `main.sh` script in `2-static-web-app` directory
5. Uploads Azure Static Web App secrets as artifact

**Outputs:**
- `static-web-app-json` artifact containing deployment secrets

**Required Secrets:**
- `ARM_CLIENT_ID` - Azure service principal client ID
- `ARM_CLIENT_SECRET` - Azure service principal secret
- `ARM_TENANT_ID` - Azure tenant ID
- `ARM_SUBSCRIPTION_ID` - Azure subscription ID
- `PERSONAL_ACCES_TOKEN` - GitHub PAT for cloning terraform repo

---

### 4. azure-web-app.yml

Reusable workflow that deploys the built application to Azure Static Web Apps.

**Trigger:** `workflow_call`

**Actions:**
1. Checks out the repository code
2. Sets up Node.js 20
3. Downloads `web-build` artifact
4. Downloads `static-web-app-json` artifact
5. Installs SWA CLI
6. Deploys to Azure Static Web Apps

**Environment Selection:**
- If branch is `main` → deploys to `production` environment
- Otherwise → deploys to branch name environment

---

## Usage

To use these workflows in your repository, create a workflow file like:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: your-org/ci-workflows/.github/workflows/node-init.yml@main
    with:
      node-version: '20'
    secrets: inherit
```

## Requirements

### Azure Service Principal

Create an Azure service principal with the necessary permissions:

```bash
az ad sp create-for-rbac --name "github-actions-deploy"
```

### GitHub Secrets

Configure these secrets in your repository:
- `ARM_CLIENT_ID`
- `ARM_CLIENT_SECRET`
- `ARM_TENANT_ID`
- `ARM_SUBSCRIPTION_ID`
- `PERSONAL_ACCES_TOKEN` (with repo scope)

## License

MIT
