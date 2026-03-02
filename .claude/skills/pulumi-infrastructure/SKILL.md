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

## Common Resources

See reference files for patterns:
- [service-accounts.md](references/service-accounts.md) - Service accounts with IAM
- [bigquery-tables.md](references/bigquery-tables.md) - BigQuery datasets and tables
- [cloud-storage.md](references/cloud-storage.md) - Cloud Storage buckets
- [cloud-scheduler.md](references/cloud-scheduler.md) - Scheduler to Cloud Run Jobs
- [cloud-monitoring.md](references/cloud-monitoring.md) - Log metrics, alerts, dashboards

## Important Notes

- Infrastructure changes are rare (weeks/months)
- Always test in sandbox first
- Production requires manual workflow_dispatch trigger
- Changes don't require application redeployment
- `pulumi install` with `toolchain: uv` handles Python dependencies automatically — no manual `uv sync` needed in CI workflows (workspace has `default-groups = "all"`)
