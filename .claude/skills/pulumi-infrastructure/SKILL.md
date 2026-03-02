---
description: "Use when provisioning GCP infrastructure with Pulumi, including service accounts, BigQuery tables, Cloud Storage, and Cloud Scheduler."
---

# Pulumi Infrastructure

Provision GCP infrastructure using Pulumi IaC patterns.

## Directory Structure

```
infrastructure/pulumi/
├── __main__.py           # Main Pulumi program
├── helpers/
│   ├── __init__.py       # Module exports
│   └── naming.py         # Resource naming utilities
├── bigquery/             # BigQuery table schema JSON files
│   └── {dataset}/
│       └── {table}.json
├── monitoring/           # Cloud Monitoring dashboard JSON
│   └── dashboard.json
├── Pulumi.yaml           # Pulumi project definition (see below)
└── Pulumi.{stack}.yaml   # Stack configurations (sandbox, production)
```

### Pulumi.yaml Configuration

```yaml
name: data-integration
runtime:
  name: python
  options:
    toolchain: uv  # Uses uv for Python dependency management
```

The `toolchain: uv` setting tells Pulumi to use uv for installing Python dependencies when running `pulumi install`.

## Procedure: Adding New Resources

### Step 1: Update Stack Configuration

Edit `Pulumi.{stack}.yaml` to add resource config under `input:projects`:

```yaml
input:projects:
  my-project:
    service_account:
      account_id: cloud-run-my-project
      description: Service account for My Project Cloud Run Jobs
      display_name: Cloud Run Job My Project
    service_account_iam:
      account_id: cloud-run-my-project
      roles:
        - roles/bigquery.admin
        - roles/run.invoker
        - roles/logging.logWriter
    artifact_registry:
      repository_id: data-integration-my-project
      format: DOCKER
      location: asia-southeast1
```

### Step 2: Add Resource Creation in __main__.py

Follow the resource loop pattern - create resources by type:

```python
for project_name, project_config in projects.items():
    # Service Account
    service_account = gcp.serviceaccount.Account(
        resource_name=make_resource_name("serviceaccount", project_config["service_account"]["account_id"]),
        **project_config["service_account"],
    )

    # IAM bindings
    for role in project_config["service_account_iam"]["roles"]:
        gcp.projects.IAMMember(
            resource_name=make_resource_name("iammember", project_config["service_account"]["account_id"], role),
            member=service_account.member,
            project=gcp_project,
            role=role,
        )
```

### Step 3: Deploy

Trigger the infrastructure-provision workflow or run locally:

```bash
cd infrastructure/pulumi
pulumi up --stack sandbox
```

## Config Namespacing

| Prefix | Purpose | Examples |
|--------|---------|----------|
| `gcp:` | GCP provider settings | `gcp:project` |
| `input:` | Input values for resources | `input:projects` |
| `label:` | Labels applied to resources | `label:component` |

## Stack File Principles

### Single Source of Truth

- **Environment-specific values** → stack file (`Pulumi.{stack}.yaml`)
- **Derived values (Pulumi Outputs)** → code (`__main__.py`)

Never hardcode environment-specific values in Python. The stack file is the single source of truth for what differs between sandbox and production.

### Explicit Asset-Type Keys

Each managed asset type gets its own dedicated `input:` key. The structure depends on how many projects the repo manages:

**Single project** — flat `input:{asset_type}` keys:

```yaml
input:bigquery:
  datasets: [...]
input:dataform:
  repositories: [...]
input:service_account:
  account_id: dataform-executor
```

Users scan top-level keys to see everything the stack manages.

**Multiple projects** — nest under `input:projects:{project_name}:{asset_type}`:

```yaml
input:projects:
  my-project:
    bigquery:
      datasets: [...]
    service_account:
      account_id: cloud-run-my-project
```

Each project's assets are grouped together.

**Never** nest unrelated asset types under the same key.

### Self-Documenting Pattern

Stack files should be readable by non-infrastructure engineers:

- Section headers with `# ════` dividers separating major resource groups
- Field-level documentation: `(required)`, `(optional, default: X)`
- Copy-paste examples showing how to add new entries
- Nesting mirrors GCP resource hierarchy (e.g., repository → release_config → workflow_config)

## YAML Anchors for Reusability

Define once, use many times:

```yaml
config:
  # Define anchor
  input:artifact_registry_cleanup_policies: &ar_cleanup
    - action: DELETE
      id: delete-prerelease
      condition:
        older_than: 2592000s

  # Reference with alias
  input:projects:
    project-a:
      artifact_registry:
        cleanup_policies: *ar_cleanup
    project-b:
      artifact_registry:
        cleanup_policies: *ar_cleanup
```

## Resource Naming

Use `make_resource_name()` for consistent Pulumi resource names:

```python
from helpers.naming import make_resource_name

# Creates: "bucket_my_data_bucket"
make_resource_name("bucket", "my-data-bucket")

# Creates: "iam_my_sa_run_invoker"
make_resource_name("iam", "my-sa", "roles/run.invoker")
```

## State Safety

### Resource Name Changes Are Destructive

Changing any part of a Pulumi resource name (the first argument to a resource constructor) triggers a **destroy + recreate**. The old resource is deleted and a new one is provisioned. This applies to all resources.

Common triggers:
- Renaming a resource's identifier in the stack file (e.g., changing `name: dataform-poc` to `name: dataform-prod`)
- Adding/removing a region from a `locations` list (changes the resource name suffix)

**Always run `pulumi preview` before `pulumi up`** to catch unexpected replacements.

### Deletion Safety Settings

Configure these in the stack file per environment — never hardcode in Python:

| Resource | Setting | Sandbox | Production |
|----------|---------|---------|------------|
| BigQuery Dataset | `delete_contents_on_destroy` | `false` | `false` |
| BigQuery Table | `deletion_protection` | `false` | `true` |
| Dataform Repository | `deletion_policy` | `FORCE` | `DELETE` |

Both environments protect non-empty datasets from accidental deletion. Sandbox relaxes table deletion protection and Dataform cascade-delete for faster iteration.

## Optional Resource Patterns

Use `.get()` to handle optional resources. The pattern differs based on context:

### Per-Project Resources (in a loop)

Use `if not: continue` to skip projects that don't need the resource:

```python
for project_name, project_config in projects.items():
    cloud_storage_config = project_config.get("cloud_storage")
    if not cloud_storage_config:
        continue  # Skip projects without this resource

    bucket = gcp.storage.Bucket(...)
```

### Workspace-Level Resources (not in a loop)

Use `if not: return` for early exit, then `if x:` for optional subsections:

```python
monitoring_config = input_config.get_object("monitoring")
if not monitoring_config:
    return  # No monitoring configured

log_based_metrics = monitoring_config.get("log_based_metrics")
if log_based_metrics:
    for metric in log_based_metrics:
        ...
```

**Why different patterns:**
- Per-project: `continue` skips to next project in the loop
- Workspace-level: `return` exits early; `if x:` guards optional sections

## Config Validation Patterns

### Cross-Reference Validation

When one config block references another by name (e.g., workflow configs reference release configs), validate the reference exists before using it:

```python
release_configs: dict[str, gcp.dataform.RepositoryReleaseConfig] = {}
for rc in repo_cfg["release_configs"]:
    release_config = gcp.dataform.RepositoryReleaseConfig(...)
    release_configs[rc["name"]] = release_config

for wc in repo_cfg["workflow_configs"]:
    rc_name = wc["release_config"]
    if rc_name not in release_configs:
        raise ValueError(
            f"Workflow '{wc['name']}' references unknown release config "
            f"'{rc_name}'. Declared release configs: {list(release_configs)}"
        )
```

This catches YAML typos at `pulumi preview` time instead of producing cryptic runtime errors.

### YAML-to-Code Alignment Checklist

When adding new stack config that the Python code consumes:

- **List vs dict**: if the YAML uses a list (`- name: foo`), iterate with `for item in cfg`; if it uses a dict (`foo: {bar: baz}`), iterate with `for key, val in cfg.items()`
- **Required keys**: use `config.require()` / `config.require_object()` — fail fast on missing config
- **Optional keys with defaults**: use `cfg.get("key", default)` in Python, document the default in YAML comments
- **Name references**: when one block references another by name, build a dict of created resources keyed by name, then validate lookups

## Common Resources

See reference files for patterns:
- [service-accounts.md](references/service-accounts.md) - Service accounts with IAM
- [bigquery-tables.md](references/bigquery-tables.md) - BigQuery datasets and tables
- [cloud-storage.md](references/cloud-storage.md) - Cloud Storage buckets
- [cloud-scheduler.md](references/cloud-scheduler.md) - Scheduler to Cloud Run Jobs
- [cloud-monitoring.md](references/cloud-monitoring.md) - Log metrics, alerts, dashboards
- [dataform.md](references/dataform.md) - Dataform repositories, release configs, workflow configs

## Important Notes

- Infrastructure changes are rare (weeks/months)
- Always test in sandbox first
- Production requires manual workflow_dispatch trigger
- Changes don't require application redeployment
- `pulumi install` with `toolchain: uv` handles Python dependencies automatically — no manual `uv sync` needed in CI workflows (workspace has `default-groups = "all"`)
