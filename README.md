# ado-flyway-install-linux-agent

A sample Azure DevOps build pipeline that installs and licences the Flyway CLI on a Linux agent.

## Repository structure

```
.
├── azure-pipelines.yml          # Calling pipeline — reads credentials from a library variable group
└── templates/
    └── install-flyway.yml       # Reusable template — installs, licences, and verifies Flyway
```

The repository is intentionally kept flat so it can be used as a starting point for any Linux Flyway pipeline (Azure DevOps hosted or self-hosted runners). Add further templates under `templates/` as your project grows.

## Prerequisites

1. **Redgate account** — sign up at <https://www.red-gate.com/> and create a personal access token (PAT).

2. **Azure DevOps variable group** — go to *Pipelines › Library* and create a group called **`flyway-credentials`** containing the following variables:

   | Variable | Description | Secret? |
   |---|---|---|
   | `FLYWAY_EMAIL` | Your Redgate account email address | No |
   | `FLYWAY_PAT` | Your Redgate personal access token | **Yes** |
   | `FLYWAY_VERSION` | Version to install: `latest` or a specific version e.g. `11.9.2` | No |
   | `FLYWAY_EDITION` | `Community`, `Teams`, or `Enterprise` | No |

3. **Pipeline permissions** — authorise the variable group for use by the pipeline the first time it runs.

## How it works

`azure-pipelines.yml` calls `templates/install-flyway.yml`, which runs three labelled steps:

| Step | Always runs? | What it does |
|---|---|---|
| **Check Flyway installation** | ✅ Yes | Resolves the desired version (queries `maven-metadata.xml` when `FLYWAY_VERSION=latest`), then runs `flyway --version` to check whether the correct version and edition are already present. Sets a flag to skip the next two steps if everything is already correct. |
| **Install Flyway** | Only if needed | Downloads the tarball from `download.red-gate.com`, extracts it, and symlinks the binary to `/usr/local/bin/flyway`. |
| **Verify Flyway installation** | Only if needed | Re-runs `flyway --version` and fails the pipeline if the expected version/edition string is not present. |

Licensing is applied at runtime via the `FLYWAY_EMAIL` and `FLYWAY_TOKEN` environment variables, which Flyway picks up automatically.

## Using the template in your own pipeline

Copy `templates/install-flyway.yml` into your repository and call it from your pipeline:

```yaml
steps:
  - template: templates/install-flyway.yml
    parameters:
      FLYWAY_EMAIL: $(FLYWAY_EMAIL)
      FLYWAY_PAT: $(FLYWAY_PAT)
      FLYWAY_VERSION: $(FLYWAY_VERSION)   # or hard-code: '11.9.2'
      FLYWAY_EDITION: $(FLYWAY_EDITION)   # or hard-code: 'Community'
```

Then add your own Flyway steps (migrate, validate, info, etc.) after the template call.
