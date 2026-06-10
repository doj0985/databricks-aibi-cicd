# biops

CI/CD framework for Databricks AI/BI dashboards using Databricks Asset Bundles.

Dashboards are authored visually in the Databricks UI, pulled down as code, and
promoted through environments via a GitOps branch flow. Validation runs on every
PR; staging deploys automatically; production deploys via a service principal
behind a manual approval gate.

## Overview

```
dev (branch) → staging (branch) → prod (branch)
     ↓              ↓                 ↓
  Validate      Deploy to         Deploy to
   only          Staging          Production
                                 (SP + approval)
```

This repo is wired to a **Databricks Free Edition** workspace, which is reachable
from GitHub-hosted runners (no IP access lists). For a real multi-workspace setup,
give `staging` and `prod` their own `host` and `warehouse_id` in `databricks.yml`.

## Prerequisites

1. Install the Databricks CLI: https://docs.databricks.com/dev-tools/cli/databricks-cli.html
2. A Databricks workspace the GitHub runner can reach (see [Networking](#networking)).
3. A service principal in that workspace with an OAuth secret (see [Service principal](#setting-up-a-service-principal)).

## Workflow

### 1. Make changes in the Databricks UI

Edit your dashboard in the dev workspace.

### 2. Pull changes locally

```bash
databricks bundle generate dashboard --existing-id <dashboard-id> --bind
```

This generates `resources/<name>.dashboard.yml` and `src/<name>.lvdash.json`. The
`--bind` flag links the local resource to the existing dashboard so future deploys
update it in place instead of creating a copy.

### 3. Replace the hardcoded warehouse_id

The generated resource file hardcodes a `warehouse_id`. Replace it with a variable:

```yaml
warehouse_id: ${var.warehouse_id}   # not 27a3fd0190890b59
```

> CI fails the build if a hardcoded `warehouse_id` is found in `resources/*.dashboard.yml`.

### 4. Commit and push to dev

```bash
git add . && git commit -m "Update dashboard" && git push origin dev
```

Pushing to `dev` runs `validate` against the `dev` target.

### 5. Promote to staging

- Open a PR `dev → staging`. CI validates the **staging** target.
- Merge (requires approval). The dashboard auto-deploys to staging.

### 6. UAT in staging, then promote to prod

- Open a PR `staging → prod`. CI validates the **prod** target.
- Merge (requires approval). The prod deploy **pauses for approval** (the
  `production` environment gate), then deploys via the service principal.

## Project Structure

```
biops/
├── .github/workflows/
│   ├── validate.yml        # PRs into staging/prod + push to dev (target-aware)
│   ├── deploy-staging.yml  # Push to staging
│   └── deploy-prod.yml     # Push to prod (behind approval gate)
├── databricks.yml          # Bundle config with dev/staging/prod targets
├── resources/              # Dashboard resource definitions (*.dashboard.yml)
└── src/                    # Dashboard JSON (*.lvdash.json)
```

## Targets

| Target | Mode | Deploys via | Triggered by |
|--------|------|-------------|--------------|
| dev | local | n/a (validate/generate only) | Manual / push to `dev` |
| staging | production | service principal (`run_as`) | Merge to `staging` |
| prod | production | service principal (`run_as`) | Merge to `prod` + approval |

## Variables

```yaml
variables:
  warehouse_id:
    description: SQL warehouse ID for the dashboard
  service_principal_id:
    description: Application (client) ID of the CI/CD service principal
    default: <sp-application-id>
```

Per-target values (e.g. a distinct `warehouse_id` per environment) are set under
each target's `variables:` block.

## GitHub Secrets

Configure under Settings → Secrets and variables → Actions:

| Secret | Description |
|--------|-------------|
| `DATABRICKS_HOST` | Workspace URL |
| `DATABRICKS_CLIENT_ID` | Service principal's **application (client) ID** |
| `DATABRICKS_CLIENT_SECRET` | Service principal's OAuth secret |

### Setting up a Service Principal

1. Create a service principal in the workspace (Settings → Identity and access → Service principals, or `databricks service-principals create --display-name ...`).
2. Generate an OAuth secret for it (`databricks service-principal-secrets-proxy create <sp-id>`).
3. Grant it access to deploy and manage the dashboards.
4. Put the application ID and secret into the GitHub secrets above.

> **Gotcha:** In `databricks.yml`, `service_principal_name` (in `permissions` and
> `run_as`) must be the SP's **application ID**, not its display name — otherwise
> deploys fail with `Principal ... does not exist`. Also keep `permissions`
> **per-target**, not top-level: a top-level block applies to every target and
> breaks if a principal doesn't exist in one of the workspaces.

### The production approval gate

`deploy-prod.yml` sets `environment: production`. For the gate to actually pause,
create a `production` Environment (Settings → Environments) with a **required
reviewer**. Without it, prod deploys run immediately with no approval.

## Networking

CI deploys from GitHub-hosted runners, whose IPs rotate across a large Azure range.
If the target workspace enforces **IP access lists**, runner calls are blocked with
`Source IP ... is blocked by Databricks IP ACL (403)`. Options: use a workspace
without IP ACLs (e.g. Free Edition), a self-hosted runner with a static allowlisted
IP, or GitHub larger runners with static IPs.

## Rollback

Every change is a commit on an environment branch, so rolling back is a git
operation — never edit the dashboard directly in the workspace.

```bash
git checkout -b revert-bad-change
git revert <bad-commit-sha>     # add -m 1 when reverting a merge commit
git push origin revert-bad-change
```

Open a PR into the affected environment branch; merging it redeploys the previous
definition. To redeploy a known-good version manually:

```bash
git checkout <good-commit-sha>
databricks bundle deploy --target prod
```

## Useful Commands

```bash
databricks bundle validate --target staging          # validate a target
databricks bundle generate dashboard --existing-id <id> --bind
databricks bundle deploy --target staging            # manual deploy
databricks bundle summary --target prod              # show deployed resources
```

## Documentation

- [Databricks Asset Bundles](https://docs.databricks.com/dev-tools/bundles/index.html)
- [Databricks CLI](https://docs.databricks.com/dev-tools/cli/databricks-cli.html)
