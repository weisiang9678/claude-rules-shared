# Common Kustomize Patches for Cloud Run Jobs

Values in `{curly_braces}` are placeholders — replace with actual values. Values ending in `_PLACEHOLDER` are literal strings replaced by CI/CD at deploy time.

## Service Account

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/template/spec/serviceAccountName
        value: {service_account_email}
    target:
      kind: Job
      name: {job_name}
```

## Environment Variables

Standard set for Cloud Run Jobs:

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: ENVIRONMENT
          value: production
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: VERSION
          value: VERSION_PLACEHOLDER
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: GCP_PROJECT_ID
          value: {gcp_project_id}
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: GCP_REGION_ABBREV
          value: GCP_REGION_ABBREV_PLACEHOLDER
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: GCP_REGION_SUFFIX
          value: GCP_REGION_SUFFIX_PLACEHOLDER
    target:
      kind: Job
      name: {job_name}
```

Project-specific env vars are added as additional `op: add` entries in the same patch.

## Secrets

### Secrets Annotation

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/metadata/annotations/run.googleapis.com~1secrets
        value: "{secret_env_name}:projects/{gcp_project_id}/secrets/{secret_name}"
    target:
      kind: Job
      name: {job_name}
```

### Secret as Environment Variable

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: {secret_env_name}
          valueFrom:
            secretKeyRef:
              name: {secret_env_name}
              key: latest
    target:
      kind: Job
      name: {job_name}
```

## Resource Limits

```yaml
patches:
  - patch: |-
      - op: replace
        path: /spec/template/spec/template/spec/containers/0/resources/limits/cpu
        value: "1"
      - op: replace
        path: /spec/template/spec/template/spec/containers/0/resources/limits/memory
        value: 1Gi
    target:
      kind: Job
      name: {job_name}
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
cd infrastructure/cloudrun_job/{job_name}/overlays/production
kustomize build .
```
