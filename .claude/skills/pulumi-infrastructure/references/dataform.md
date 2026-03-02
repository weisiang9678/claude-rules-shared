# Dataform Repositories, Release Configs, and Workflow Configs

## Prerequisites

SSH authentication requires **two separate keys** — do not confuse them:

| Key | Purpose | Where it goes |
|-----|---------|---------------|
| **Deploy Key** | A key pair YOU generate. Lets Dataform authenticate TO GitHub. | Private → Secret Manager, Public → GitHub Deploy Keys |
| **Host Key** | GitHub's SERVER key. Lets Dataform verify it's connecting to the real GitHub. | Obtained via `ssh-keyscan -t rsa github.com` |

**Setup checklist** (one-time per GCP project):

1. Enable APIs: `dataform.googleapis.com`, `bigquery.googleapis.com`, `secretmanager.googleapis.com`
2. Generate an RSA deploy key — **Dataform does not support Ed25519**:
   ```bash
   ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ./dataform_deploy_key -N ""
   ```
3. Add the public key to GitHub as a **read-only** Deploy Key
4. Store the private key in Secret Manager:
   ```bash
   gcloud secrets create DATAFORM-GITHUB-SSH-KEY --replication-policy=automatic
   gcloud secrets versions add DATAFORM-GITHUB-SSH-KEY --data-file=./dataform_deploy_key
   rm ./dataform_deploy_key ./dataform_deploy_key.pub
   ```
5. Get GitHub's SSH host key (`ssh-keyscan -t rsa github.com`) and store it as a GitHub Actions variable `SSH_HOST_PUBLIC_KEY`
6. Set `input:github_ssh_key_secret_name` in the Pulumi stack file

## Stack Configuration

Top-level keys consumed by the Dataform section:

```yaml
input:github_repo_url: git@github.com:Org/repo.git
input:github_ssh_key_secret_name: DATAFORM-GITHUB-SSH-KEY
```

Repository, release config, and workflow config structure:

```yaml
input:dataform:
  repositories:
    - name: dataform-poc                      # (required)
      display_name: Dataform Sandbox          # (required)
      branch: sandbox                         # (required) default git branch
      time_zone: Asia/Kuala_Lumpur            # (required) TZ database name
      env: sandbox                            # (required) → vars.env in SQLX
      default_schema: dataform_output         # (required) target BigQuery dataset
      deletion_policy: FORCE                  # (optional, default: DELETE)
      locations:                              # (required)
        - asia-southeast1

      release_configs:
        - name: release-sandbox               # (required) referenced by workflow_configs
          branch: sandbox                     # (required) git branch to compile
          cron: "0 * * * *"                   # (required) UTC cron schedule

      workflow_configs:
        - name: daily-run                     # (required)
          release_config: release-sandbox     # (required) must match a release_configs name
          cron: "0 */2 * * *"                 # (required) UTC cron schedule
          full_refresh: false                 # (optional, default: false)
          transitive_dependencies_included: true   # (optional, default: true)
          transitive_dependents_included: true     # (optional, default: true)
```

### Repository Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique repository identifier |
| `display_name` | Yes | Label shown in GCP Dataform UI |
| `branch` | Yes | Default git branch for this repository |
| `time_zone` | Yes | TZ database name for all cron schedules in this repo |
| `env` | Yes | Exposed as `dataform.projectConfig.vars.env` inside SQLX files |
| `default_schema` | Yes | BigQuery dataset for compiled outputs. Must match a dataset in `input:bigquery` |
| `deletion_policy` | No | `DELETE` (default) — fails if nested resources exist. `FORCE` — cascade-deletes all release/workflow configs |
| `locations` | Yes | GCP locations to deploy into. Each location gets its own independent copy |

### Release Config Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier. Referenced by `workflow_configs` |
| `branch` | Yes | Git branch to compile |
| `cron` | Yes | UTC cron schedule for recompilation |

### Workflow Config Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier |
| `release_config` | Yes | Must match a `name` from `release_configs` above |
| `cron` | Yes | UTC cron execution schedule |
| `full_refresh` | No | `false` (default) — incremental. `true` — full rebuild of incremental tables |
| `transitive_dependencies_included` | No | `true` (default) — also run upstream dependencies |
| `transitive_dependents_included` | No | `true` (default) — also run downstream dependents |

## Python Implementation

The nested loop pattern: `for repo_cfg` → `for region` → release configs → workflow configs.

### Repository

```python
for repo_cfg in dataform_cfg["repositories"]:
    for region in repo_cfg["locations"]:
        dataform_repo = gcp.dataform.Repository(
            make_resource_name("dataform", "repository", repo_cfg["name"], region),
            name=repo_cfg["name"],
            project=gcp_project,
            region=region,
            display_name=repo_cfg["display_name"],
            deletion_policy=repo_cfg.get("deletion_policy", "DELETE"),
            service_account=dataform_executor_sa.email,
            git_remote_settings={
                "url": github_repo_url,
                "default_branch": repo_cfg["branch"],
                "ssh_authentication_config": {
                    "user_private_key_secret_version": github_ssh_key_secret_version,
                    "host_public_key": github_ssh_host_public_key,
                },
            },
            workspace_compilation_overrides={
                "default_database": gcp_project,
            },
            opts=pulumi.ResourceOptions(depends_on=[dataform_executor_sa]),
        )
```

### Release Config

```python
        release_configs: dict[str, gcp.dataform.RepositoryReleaseConfig] = {}

        for rc in repo_cfg["release_configs"]:
            release_config = gcp.dataform.RepositoryReleaseConfig(
                make_resource_name("dataform", "release", rc["name"], region),
                project=gcp_project,
                region=region,
                repository=dataform_repo.name,
                name=rc["name"],
                git_commitish=rc["branch"],
                cron_schedule=rc["cron"],
                time_zone=repo_cfg["time_zone"],
                code_compilation_config={
                    "default_database": gcp_project,
                    "default_location": region,
                    "default_schema": repo_cfg["default_schema"],
                    "vars": {"env": repo_cfg["env"]},
                },
                opts=pulumi.ResourceOptions(depends_on=[dataform_repo]),
            )
            release_configs[rc["name"]] = release_config
```

### Workflow Config (with cross-reference validation)

```python
        for wc in repo_cfg["workflow_configs"]:
            rc_name = wc["release_config"]
            if rc_name not in release_configs:
                raise ValueError(
                    f"Workflow '{wc['name']}' references unknown release config "
                    f"'{rc_name}'. Declared release configs: {list(release_configs)}"
                )
            release_config = release_configs[rc_name]
            gcp.dataform.RepositoryWorkflowConfig(
                make_resource_name("dataform", "workflow", wc["name"], region),
                project=gcp_project,
                region=region,
                repository=dataform_repo.name,
                name=wc["name"],
                release_config=release_config.id,
                cron_schedule=wc["cron"],
                time_zone=repo_cfg["time_zone"],
                invocation_config={
                    "transitive_dependencies_included": wc.get(
                        "transitive_dependencies_included", True
                    ),
                    "transitive_dependents_included": wc.get(
                        "transitive_dependents_included", True
                    ),
                    "fully_refresh_incremental_tables_enabled": wc.get(
                        "full_refresh", False
                    ),
                    "service_account": dataform_executor_sa.email,
                },
                opts=pulumi.ResourceOptions(depends_on=[release_config]),
            )
```

## Cross-Reference Validation

Workflow configs reference release configs by name. Validate that the referenced name exists before using it:

```python
rc_name = wc["release_config"]
if rc_name not in release_configs:
    raise ValueError(
        f"Workflow '{wc['name']}' references unknown release config "
        f"'{rc_name}'. Declared release configs: {list(release_configs)}"
    )
```

This catches YAML typos at `pulumi preview` time instead of producing cryptic runtime errors.

## Deletion Safety

| Setting | Sandbox | Production |
|---------|---------|------------|
| Repository `deletion_policy` | `FORCE` — cascade-deletes nested resources | `DELETE` — fails if nested resources exist |
| Dataset `delete_contents_on_destroy` | `false` — refuses to delete non-empty datasets | `false` — same protection |

Set `deletion_policy` in the stack file, not hardcoded in Python. The code reads it with a safe default:

```python
deletion_policy=repo_cfg.get("deletion_policy", "DELETE"),
```

## Multi-Region Deployment

The `locations` list enables deploying the same repository to multiple GCP regions. Each location gets independent copies of all resources:

```yaml
locations:
  - asia-southeast1
  - us-central1
```

The region is included in every resource name to avoid collisions:

```python
make_resource_name("dataform", "repository", repo_cfg["name"], region)
# → "dataform_repository_dataform_poc_asia_southeast1"
```

**Warning:** Changing `name` or `locations` values renames the Pulumi resource, which triggers a **destroy + recreate**. This is destructive — the old resource is deleted and a new one created. Always run `pulumi preview` first to check for unexpected replacements.
