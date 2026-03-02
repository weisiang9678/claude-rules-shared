# Cloud Storage Buckets

## Stack Configuration

```yaml
input:projects:
  my-project:
    cloud_storage:
      name: my-project-data
      location: asia-southeast1
      storage_class: STANDARD
      uniform_bucket_level_access: true
      public_access_prevention: enforced
```

## Python Implementation

Use `get()` to handle optional cloud storage config:

```python
for project_name, project_config in projects.items():
    cloud_storage_config = project_config.get("cloud_storage")
    if not cloud_storage_config:
        continue

    bucket = gcp.storage.Bucket(
        resource_name=make_resource_name("bucket", cloud_storage_config["name"]),
        **cloud_storage_config,
        labels=labels,
    )
    cloud_storages[project_name] = bucket
```

**Why `get()` with early continue:**
- Cloud Storage is optional per project
- Skip projects that don't need buckets
- Dict spread works for remaining config

## Common Configuration Options

| Option | Description | Common Values |
|--------|-------------|---------------|
| `name` | Globally unique bucket name | `{project}-{purpose}` |
| `location` | GCP region or multi-region | `asia-southeast1`, `US`, `EU` |
| `storage_class` | Storage tier | `STANDARD`, `NEARLINE`, `COLDLINE`, `ARCHIVE` |
| `uniform_bucket_level_access` | Disable ACLs, use IAM only | `true` (recommended) |
| `public_access_prevention` | Block public access | `enforced` (recommended) |

## Security Best Practices

Always set these for production buckets:

```yaml
uniform_bucket_level_access: true   # IAM-only access control
public_access_prevention: enforced  # Block all public access
```

## Lifecycle Rules (Optional)

```yaml
cloud_storage:
  name: my-bucket
  location: asia-southeast1
  lifecycle_rules:
    - action:
        type: Delete
      condition:
        age: 30  # Delete objects older than 30 days
    - action:
        type: SetStorageClass
        storage_class: NEARLINE
      condition:
        age: 7  # Move to NEARLINE after 7 days
```
