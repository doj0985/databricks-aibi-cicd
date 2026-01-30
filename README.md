# biops

CI/CD framework for Databricks AI BI dashboards using Databricks Asset Bundles.

## Overview

This project manages dashboard deployments across environments using a GitOps workflow:

```
dev (branch) → staging (branch) → prod (branch)
     ↓              ↓                 ↓
  Validate      Deploy to         Deploy to
   only         Staging           Production
```

## Prerequisites

1. Install the Databricks CLI: https://docs.databricks.com/dev-tools/cli/databricks-cli.html

2. Authenticate to your Databricks workspace:
   ```bash
   databricks configure
   ```

## Workflow

### 1. Make changes in the Databricks UI

Edit your dashboard in the dev workspace using the Databricks UI.

### 2. Pull changes locally

```bash
databricks bundle generate dashboard --existing-id <dashboard-id>
```

This generates:
- `resources/<dashboard-name>.dashboard.yml` — resource configuration
- `src/<dashboard-name>.lvdash.json` — dashboard definition

### 3. Update warehouse_id (Important!)

The generated resource file will have a hardcoded `warehouse_id`. You must replace it with a variable:

```yaml
# Before (generated)
warehouse_id: 27a3fd0190890b59

# After (correct)
warehouse_id: ${var.warehouse_id}
```

> **Note:** The CI validation will fail if hardcoded warehouse_ids are detected.

### 4. Commit and push to dev branch

```bash
git add .
git commit -m "Update dashboard"
git push origin dev
```

### 5. Create PR to staging

- Create a PR from `dev` → `staging`
- CI runs validation checks
- After merge, the dashboard deploys to staging automatically

### 6. UAT in staging

Test the dashboard in the staging environment.

### 7. Create PR to prod

- Create a PR from `staging` → `prod`
- After merge, the dashboard deploys to production automatically

## Project Structure

```
biops/
├── .github/workflows/
│   ├── validate.yml        # Runs on PRs (validates bundle)
│   ├── deploy-staging.yml  # Runs on push to staging
│   └── deploy-prod.yml     # Runs on push to prod
├── databricks.yml          # Bundle configuration with targets
├── resources/              # Dashboard resource definitions
│   └── *.dashboard.yml
└── src/                    # Dashboard JSON files
    └── *.lvdash.json
```

## Targets

| Target | Purpose | Triggered by |
|--------|---------|--------------|
| dev | Local validation and generate commands | Manual |
| staging | Staging environment | Push to `staging` branch |
| prod | Production environment | Push to `prod` branch |

## Variables

Warehouse IDs are managed via variables in `databricks.yml`:

```yaml
variables:
  warehouse_id:
    description: SQL warehouse ID to use for the dashboard

targets:
  staging:
    variables:
      warehouse_id: <staging-warehouse-id>
  prod:
    variables:
      warehouse_id: <prod-warehouse-id>
```

If you have dashboards that use different warehouses, define additional variables:

```yaml
variables:
  warehouse_id:
    description: Default warehouse
  warehouse_id_analytics:
    description: Analytics team warehouse
```

## GitHub Secrets

Configure these secrets in your GitHub repository (Settings → Secrets → Actions):

| Secret | Description |
|--------|-------------|
| `DATABRICKS_HOST` | Workspace URL (e.g., `https://your-workspace.cloud.databricks.com`) |
| `DATABRICKS_TOKEN` | Personal access token or service principal token |

## Useful Commands

```bash
# Validate bundle locally
databricks bundle validate

# Pull a dashboard from the workspace
databricks bundle generate dashboard --existing-id <dashboard-id>

# Deploy to a specific target (manual)
databricks bundle deploy --target staging
databricks bundle deploy --target prod
```

## Documentation

- [Databricks Asset Bundles](https://docs.databricks.com/dev-tools/bundles/index.html)
- [Databricks CLI](https://docs.databricks.com/dev-tools/cli/databricks-cli.html)
