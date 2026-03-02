# BigQuery Datasets and Tables

## Stack Configuration

```yaml
input:projects:
  my-project:
    bigquery_dataset:
      dataset_id: bronze_my_project
      location: asia-southeast1
      delete_contents_on_destroy: true    # sandbox only; false in production
    bigquery_tables:
      - table_id: inventory_movement
        dataset_id: bronze_my_project
        schema_file: bigquery/bronze_my_project/inventory_movement.json
        deletion_protection: false         # sandbox only; true in production
      - table_id: sku_location
        dataset_id: bronze_my_project
        schema_file: bigquery/bronze_my_project/sku_location.json
        deletion_protection: false
```

## Python Implementation

### Dataset

```python
# Create dataset using dict spread
dataset_config = project_config["bigquery_dataset"]
dataset = gcp.bigquery.Dataset(
    resource_name=make_resource_name("dataset", dataset_config["dataset_id"]),
    **dataset_config,
)
```

### Tables with Schema Files

Use `copy()` and `pop()` to handle parameters that need processing before dict spread:

```python
from pathlib import Path

table_configs = project_config.get("bigquery_tables", [])

for table_config in table_configs:
    # Copy config to avoid mutating original
    table = table_config.copy()

    # Pop custom parameters that aren't Pulumi resource args
    schema_file = table.pop("schema_file")

    # Load schema from JSON file
    schema_path = Path(__file__).resolve().parent / schema_file
    if not schema_path.exists():
        raise FileNotFoundError(f"Schema file not found: {schema_file}")
    schema_json = schema_path.read_text()

    # Create table with dict spread + processed parameters
    gcp.bigquery.Table(
        resource_name=make_resource_name(
            "table", table["dataset_id"], table["table_id"]
        ),
        **table,  # Spreads table_id, dataset_id
        schema=schema_json,  # Processed parameter
        opts=pulumi.ResourceOptions(depends_on=[dataset]),
    )
```

**Why copy() + pop():**
- `copy()` - Avoid mutating original config dict
- `pop()` - Remove custom keys (like `schema_file`) that aren't Pulumi args
- Remaining keys can be spread directly to resource constructor

## Schema JSON Format

Create schema files at `infrastructure/pulumi/bigquery/{dataset}/{table}.json`:

```json
[
  {
    "name": "id",
    "type": "STRING",
    "mode": "REQUIRED",
    "description": "Unique identifier"
  },
  {
    "name": "created_at",
    "type": "TIMESTAMP",
    "mode": "REQUIRED",
    "description": "Record creation timestamp"
  },
  {
    "name": "data",
    "type": "JSON",
    "mode": "NULLABLE",
    "description": "JSON payload"
  }
]
```

## Deletion Safety Settings

| Resource | Setting | Sandbox | Production |
|----------|---------|---------|------------|
| Dataset | `delete_contents_on_destroy` | `true` — allows Pulumi to drop non-empty datasets | `false` — Pulumi refuses to delete datasets with tables |
| Table | `deletion_protection` | `false` — allows Pulumi to drop the table | `true` — Pulumi refuses to delete the table |

These settings are declared in the stack YAML and pass through to the resource constructor automatically via `**ds_cfg` / `**table` dict spread — no code change is needed when adding or modifying them.

## Directory Structure

```
infrastructure/pulumi/bigquery/
└── bronze_my_project/
    ├── inventory_movement.json
    └── sku_location.json
```
