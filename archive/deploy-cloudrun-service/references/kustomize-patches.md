# Common Kustomize Patches for Cloud Run Services

Values in `{curly_braces}` are placeholders — replace with actual values. Values ending in `_PLACEHOLDER` are literal strings replaced by CI/CD at deploy time.

## Autoscaling

Services typically set both min and max scale:

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/metadata/annotations/autoscaling.knative.dev~1minScale
        value: "1"
      - op: add
        path: /spec/template/metadata/annotations/autoscaling.knative.dev~1maxScale
        value: "5"
    target:
      kind: Service
      name: {service_name}
```

| Setting | Purpose | Common Values |
|---------|---------|---------------|
| `minScale` | Keep instances warm (avoid cold starts) | `"0"` (sandbox), `"1"` (production) |
| `maxScale` | Limit scaling | `"2"` to `"10"` depending on load |

## Service Account

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/serviceAccountName
        value: {service_account_email}
    target:
      kind: Service
      name: {service_name}
```

## Environment Variables

Standard set for Cloud Run Services:

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
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
      kind: Service
      name: {service_name}
```

Project-specific env vars (e.g., `BUCKET_NAME`, `BUCKET_URL`) are added as additional `op: add` entries in the same patch.

## Secrets

### Secrets Annotation

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/metadata/annotations/run.googleapis.com~1secrets
        value: "{secret_env_name}:projects/{gcp_project_id}/secrets/{secret_name}"
    target:
      kind: Service
      name: {service_name}
```

### Secret as Environment Variable

```yaml
patches:
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: {secret_env_name}
          valueFrom:
            secretKeyRef:
              name: {secret_env_name}
              key: latest
    target:
      kind: Service
      name: {service_name}
```

## Resource Limits

```yaml
patches:
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/cpu
        value: "1"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 1Gi
    target:
      kind: Service
      name: {service_name}
```

## Startup and Liveness Probes

Probes are typically defined in the base manifest. Override via patches if needed:

```yaml
patches:
  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/startupProbe/initialDelaySeconds
        value: 120
      - op: replace
        path: /spec/template/spec/containers/0/startupProbe/failureThreshold
        value: 10
    target:
      kind: Service
      name: {service_name}
```

Recommended probe settings:

| Probe | Setting | Guideline |
|-------|---------|-----------|
| **startupProbe** | `initialDelaySeconds` | Allow enough time for app to initialize |
| | `failureThreshold` | At least 3–5 to avoid premature restarts |
| | `periodSeconds` | 10–30s between checks |
| **livenessProbe** | `initialDelaySeconds` | Higher than startup total window |
| | `periodSeconds` | 30–60s for steady-state checks |

## JSON Patch Escaping

| Character | Escape |
|-----------|--------|
| `/` | `~1` |
| `~` | `~0` |

Example: `autoscaling.knative.dev/minScale` → `autoscaling.knative.dev~1minScale`

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
cd infrastructure/cloudrun_service/{service_name}/overlays/production
kustomize build .
```
