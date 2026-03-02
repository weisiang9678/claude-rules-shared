# Shared Kustomize Patches

Common patches for all Cloud Run workload types. Values in `{curly_braces}` are placeholders — replace with actual values. Values ending in `_PLACEHOLDER` are literal strings replaced by CI/CD at deploy time.

> **Path depth varies by type.** Jobs use deeper paths (`/spec/template/spec/template/spec/...`) due to an extra `template` nesting level. Services and Functions use `/spec/template/spec/...`. See [Path Differences](#path-differences) and the type-specific files for exact paths and complete overlay templates:
> - [job-specifics.md](job-specifics.md) — deeper paths, `kind: Job`
> - [service-specifics.md](service-specifics.md) — autoscaling, probe overrides
> - [function-specifics.md](function-specifics.md) — `FUNCTION_TARGET`, maxScale only

## Service Account

```yaml
patches:
  - patch: |-
      - op: add  # Use "replace" if serviceAccountName is already in base manifest
        path: /spec/template/spec/serviceAccountName    # Service/Function path
        # path: /spec/template/spec/template/spec/serviceAccountName  # Job path
        value: {service_account_email}
    target:
      kind: {Job|Service}
      name: {name}
```

## Environment Variables

Standard set for all Cloud Run workloads:

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-    # Service/Function path
        # path: /spec/template/spec/template/spec/containers/0/env/-  # Job path
        value:
          name: ENVIRONMENT
          value: production
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: VERSION
          value: VERSION_PLACEHOLDER
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: GCP_PROJECT_ID
          value: {gcp_project_id}
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: GCP_REGION_ABBREV
          value: GCP_REGION_ABBREV_PLACEHOLDER
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: GCP_REGION_SUFFIX
          value: GCP_REGION_SUFFIX_PLACEHOLDER
    target:
      kind: {Job|Service}
      name: {name}
```

> **Note:** If VERSION is already in the base manifest (e.g. Service), omit it from the overlay patch. Project-specific env vars are added as additional `op: add` entries in the same patch.

## Secrets

### Secrets Annotation

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/metadata/annotations/run.googleapis.com~1secrets
        value: "{secret_env_name}:projects/{gcp_project_id}/secrets/{secret_name}"
    target:
      kind: {Job|Service}
      name: {name}
```

### Secret as Environment Variable

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-    # Service/Function path
        # path: /spec/template/spec/template/spec/containers/0/env/-  # Job path
        value:
          name: {secret_env_name}
          valueFrom:
            secretKeyRef:
              name: {secret_env_name}
              key: latest
    target:
      kind: {Job|Service}
      name: {name}
```

## Resource Limits

```yaml
patches:
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/cpu      # Service/Function path
        # path: /spec/template/spec/template/spec/containers/0/resources/limits/cpu  # Job path
        value: "1"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        # path: /spec/template/spec/template/spec/containers/0/resources/limits/memory  # Job path
        value: 1Gi
    target:
      kind: {Job|Service}
      name: {name}
```

## JSON Patch Escaping

| Character | Escape |
|-----------|--------|
| `/` | `~1` |
| `~` | `~0` |

Example: `run.googleapis.com/secrets` → `run.googleapis.com~1secrets`

## Path Differences

Cloud Run Functions and Services use Knative Service; Jobs have an extra `template` nesting level:

| Path | Function / Service | Job |
|------|-------------------|-----|
| Container env | `/spec/template/spec/containers/0/env/-` | `/spec/template/spec/template/spec/containers/0/env/-` |
| Service account | `/spec/template/spec/serviceAccountName` | `/spec/template/spec/template/spec/serviceAccountName` |
| Annotations | `/spec/template/metadata/annotations/` | `/spec/template/metadata/annotations/` |

## Debugging

Preview final manifest:
```bash
cd infrastructure/cloudrun_{type}/{name}/overlays/{environment}
kustomize build .
```
