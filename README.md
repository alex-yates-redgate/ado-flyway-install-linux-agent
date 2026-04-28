# Flyway Azure DevOps Pipelines

A sample set of Azure DevOps pipelines for database deployments using Flyway, supporting a trunk-based development workflow with PR validation, automated Dev deployments, and manual promotions to PreProd and Prod.

## Workflow Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEVELOPMENT WORKFLOW                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Developer creates feature branch                                        │
│                    │                                                        │
│                    ▼                                                        │
│  2. Developer works locally, opens PR to main                               │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────┐                                         │
│  │  pr-validation.yml             │  ◄── Automatic PR check                 │
│  │  • flyway check -code/changes  │                                         │
│  │  • Publishes HTML report       │                                         │
│  └────────────────────────────────┘                                         │
│                    │                                                        │
│                    ▼                                                        │
│  3. PR reviewed, approved, merged to main                                   │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────┐                                         │
│  │  main-cicd.yml                 │  ◄── Automatic on merge                 │
│  │  • flyway migrate to Dev       │                                         │
│  │  • Integration/regression tests│                                         │
│  └────────────────────────────────┘                                         │
│                    │                                                        │
│                    ▼                                                        │
│  4. Testing completes successfully                                          │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────┐                                         │
│  │  migrate.yml → PreProd         │  ◄── Manual trigger (click to deploy)   │
│  │  • Optional: FLYWAY_CHERRYPICK │                                         │
│  │      and FLYWAY_TARGET params  │                                         │
│  │  • Approval gate               │                                         │
│  └────────────────────────────────┘                                         │
│                    │                                                        │
│                    ▼                                                        │
│  ┌────────────────────────────────┐                                         │
│  │  migrate.yml → Prod            │  ◄── Manual trigger (click to deploy)   │
│  │  • Optional: FLYWAY_CHERRYPICK │                                         │
│  │      and FLYWAY_TARGET params  │                                         │
│  │  • Approval gate               │                                         │
│  └────────────────────────────────┘                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hotfix Scenario

Need to deploy an urgent fix to Prod while other migrations are still being tested?

1. Run `migrate.yml` manually
2. Select `Prod` in the `ENVIRONMENT` dropdown
3. Use one of these options:
   - **Cherry-pick specific migrations:** Enter migration(s) in `FLYWAY_CHERRYPICK` (e.g., `V1.0.7`) — only those migrations deploy
   - **Deploy up to a version:** Enter a version in `FLYWAY_TARGET` (e.g., `1.0.7`) — all migrations up to and including that version deploy
4. Approve when prompted
5. Only the specified migration(s) deploy — no Dev re-run, no test re-run

## Repository Structure

```
.
├── README.md
├── LICENSE
└── pipelines/
    ├── pr-validation.yml            # PR check: flyway check + report
    ├── main-cicd.yml                # Auto: Dev deploy + tests (on merge to main)
    ├── migrate.yml                  # Manual: Deploy to Dev/PreProd/Prod
    ├── undo.yml                     # Manual: Undo on Dev/PreProd/Prod
    └── templates/
        ├── install-flyway.yml       # Installs and licences Flyway CLI
        ├── flyway-check.yml         # Runs flyway check, publishes report
        ├── flyway-migrate.yml       # Runs flyway migrate
        ├── flyway-undo.yml          # Runs flyway undo
        └── run-tests.yml            # Placeholder for integration/regression tests
```

## Variable Naming Convention

All variables use `SCREAMING_SNAKE_CASE` for consistency:

| Variable | Category | Description |
|----------|----------|-------------|
| `FLYWAY_EMAIL` | Flyway config | Redgate account email (from variable group) |
| `FLYWAY_PAT` | Flyway config | Redgate personal access token (from variable group) |
| `FLYWAY_VERSION` | Flyway config | CLI version to install (from variable group) |
| `FLYWAY_EDITION` | Flyway config | Community, Teams, or Enterprise (from variable group) |
| `FLYWAY_TARGET` | Flyway param | Runtime: Flyway `-target` parameter |
| `FLYWAY_CHERRYPICK` | Flyway param | Runtime: Flyway `-cherryPick` parameter |
| `ENVIRONMENT` | Pipeline param | Target environment — maps to both ADO and Flyway environments |

- **`FLYWAY_*`** prefix indicates Flyway-specific configuration or parameters
- **`ENVIRONMENT`** is the deployment target, used for both Azure DevOps approval gates and Flyway environment configuration

## Prerequisites

### 1. Redgate Account

Sign up at <https://www.red-gate.com/> and create a personal access token (PAT).

### 2. Azure DevOps Variable Group

Go to *Pipelines › Library* and create a group called **`flyway-general`**:

| Variable | Description | Secret? |
|----------|-------------|---------|
| `FLYWAY_EMAIL` | Your Redgate account email address | No |
| `FLYWAY_PAT` | Your Redgate personal access token | **Yes** |
| `FLYWAY_VERSION` | Version to install: `latest` or e.g. `11.9.2` | No |
| `FLYWAY_EDITION` | `Community`, `Teams`, or `Enterprise` | No |

### 3. Azure DevOps Environments

Create environments in *Project Settings › Environments*:

| Environment | Approval Gate? | Purpose |
|-------------|----------------|---------|
| `Dev` | Optional | Development database |
| `PreProd` | **Yes** | Pre-production / staging |
| `Prod` | **Yes** | Production |

To add approval gates:
1. Click the environment name
2. Go to *Approvals and checks*
3. Add *Approvals* and select the required approvers

### Environment Mapping (Important)

> **Note:** This pipeline assumes Azure DevOps environments and Flyway environments share the same names.
>
> When you select `ENVIRONMENT: PreProd`, the pipeline will:
> 1. Use the ADO `PreProd` environment (for approval gates and deployment tracking)
> 2. Pass `PreProd` to Flyway as the environment (for `flyway.toml` configuration)
>
> If your ADO and Flyway environment names differ, you'll need to modify the templates to map between them.

### 4. Branch Policy (PR Validation)

Configure `pipelines/pr-validation.yml` as a build validation policy on `main`:

1. Go to *Repos › Branches*
2. Click the `...` menu on `main` → *Branch policies*
3. Under *Build Validation*, add a policy pointing to the `pr-validation` pipeline

### 5. Create the Pipelines

Create pipelines in Azure DevOps pointing to:
- `pipelines/pr-validation.yml`
- `pipelines/main-cicd.yml`
- `pipelines/migrate.yml`
- `pipelines/undo.yml`

## Pipelines Reference

### pr-validation.yml

**Trigger:** PR to `main` (via branch policy)

Runs Flyway check commands and publishes an HTML report as a build artifact.

### main-cicd.yml

**Trigger:** Push to `main` (automatic on merge)

1. Deploys all pending migrations to Dev using `flyway migrate`
2. Runs integration and regression tests

### migrate.yml

**Trigger:** Manual only

Deploys to any environment with optional parameters:

| Parameter | Description |
|-----------|-------------|
| `ENVIRONMENT` | Target environment: `Dev`, `PreProd`, or `Prod`. Maps to both ADO and Flyway environments. |
| `FLYWAY_TARGET` | Optional: Migrate up to this version (Flyway `-target` parameter) |
| `FLYWAY_CHERRYPICK` | Optional: Comma-separated migrations to deploy (Flyway `-cherryPick` parameter) |

**Use cases:**
- Regular deployments to PreProd and Prod
- Cherry-pick hotfixes
- Re-deploy to Dev after an undo

### undo.yml

**Trigger:** Manual only

Rolls back migrations on any environment:

| Parameter | Description |
|-----------|-------------|
| `ENVIRONMENT` | Target environment: `Dev`, `PreProd`, or `Prod`. Maps to both ADO and Flyway environments. |
| `FLYWAY_TARGET` | Optional: Undo down to this version (Flyway `-target` parameter) |
| `FLYWAY_CHERRYPICK` | Optional: Comma-separated migrations to undo (Flyway `-cherryPick` parameter) |

## Step-by-Step Usage

### Normal Development Flow

1. **Create a feature branch** from `main`
2. **Add your migration scripts** (e.g., `V1.0.5__Add_customer_table.sql`)
3. **Open a PR** to `main`
4. **PR validation runs automatically** — review the Flyway check report
5. **Merge the PR** — Dev deployment and tests run automatically
6. **When testing passes**, run `migrate.yml` with `PreProd` selected
7. **After PreProd validation**, run `migrate.yml` with `Prod` selected

### Deploying a Hotfix

1. Run `migrate.yml` manually
2. Select the target in `ENVIRONMENT` (e.g., `Prod`)
3. Use one of these options:
   - `FLYWAY_CHERRYPICK`: Enter specific migration(s) to deploy (e.g., `V1.0.7`)
   - `FLYWAY_TARGET`: Enter a version to deploy up to (e.g., `1.0.7`)
4. Approve when prompted
5. Only the specified migration(s) are deployed

### Rolling Back a Migration

1. Run `undo.yml` manually
2. Select the target in `ENVIRONMENT`
3. Optionally specify:
   - `FLYWAY_TARGET`: Version to roll back to
   - `FLYWAY_CHERRYPICK`: Specific migration(s) to undo
4. Approve when prompted

### Re-deploying After an Undo

1. Run `migrate.yml` manually
2. Select `Dev` in `ENVIRONMENT` (or the appropriate environment)
3. Leave parameters empty to deploy all pending migrations, or use:
   - `FLYWAY_CHERRYPICK`: Deploy specific migration(s)
   - `FLYWAY_TARGET`: Deploy up to a specific version

## How the Install Template Works

`templates/install-flyway.yml` handles Flyway installation efficiently:

| Step | Runs? | What it does |
|------|-------|--------------|
| **Check installation** | Always | Resolves version (queries `maven-metadata.xml` if `latest`), checks if correct version/edition is already installed |
| **Install Flyway** | If needed | Downloads tarball, extracts to `/opt/flyway`, symlinks to `/usr/local/bin/flyway` |
| **Verify installation** | If needed | Confirms correct version/edition is now available |

Licensing is applied at runtime via `FLYWAY_EMAIL` and `FLYWAY_TOKEN` environment variables.

## Customisation

### Using On-Premises Agents

Replace the pool configuration in each pipeline:

```yaml
# From:
pool:
  vmImage: 'ubuntu-latest'

# To:
pool:
  name: 'YourAgentPoolName'
```

### Adding Database Connection Parameters

The Flyway templates will need database connection configuration. Add to your variable group or pass as parameters:

- `FLYWAY_URL` — JDBC connection string
- `FLYWAY_USER` — Database username
- `FLYWAY_PASSWORD` — Database password (secret)
- `FLYWAY_LOCATIONS` — Path to migration scripts

### Enabling the Actual Flyway Commands

The templates currently have the Flyway commands commented out with a warning. To enable:

1. Configure your database connections
2. Uncomment the `flyway` command in each template:
   - `templates/flyway-migrate.yml`
   - `templates/flyway-undo.yml`
   - `templates/flyway-check.yml`

## License

MIT License — see [LICENSE](LICENSE) for details.
