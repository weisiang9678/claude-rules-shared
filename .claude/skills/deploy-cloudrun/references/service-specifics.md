# Service-Specific Configuration

Cloud Run Service-specific manifest, probes, autoscaling, and conventions. For shared patterns, see [shared-dockerfile.md](shared-dockerfile.md) and [shared-kustomize-patches.md](shared-kustomize-patches.md).

## Key Characteristics

- **API:** `apiVersion: serving.knative.dev/v1`, `kind: Service`
- **Infra directory:** `infrastructure/cloudrun_service/{service_name}/`
- **Manifest filename:** `service.yaml`
- **HTTP port required** — EXPOSE 8080 in Dockerfile
- **Startup probe:** `httpGet` with detailed configuration
- **Liveness probe:** `httpGet` for steady-state health checks
- **Autoscaling:** `minScale` + `maxScale`
- **CMD:** Varies (python, streamlit, gunicorn)
- **gcloud command:** `gcloud run services replace`

## Base Manifest Template

`infrastructure/cloudrun_service/{service_name}/base/service.yaml`:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: {service_name}
  labels:
    component: my-component
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/cpu-throttling: "false"
        run.googleapis.com/startup-cpu-boost: "true"
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "2"
    spec:
      serviceAccountName: {service_account_email}
      timeoutSeconds: 300
      containers:
      - name: {service_name}
        image: IMAGE_URL
        ports:
        - containerPort: 8080
          name: http1
        env:
        - name: VERSION
          value: VERSION_PLACEHOLDER
        startupProbe:
          httpGet:
            path: /health_check
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health_check
            port: 8080
        resources:
          limits:
            cpu: "1"
            memory: 2Gi
  traffic:
  - latestRevision: true
    percent: 100
```

## Overlay Template

`infrastructure/cloudrun_service/{service_name}/overlays/{environment}/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: {gcp_project_id}

resources:
  - ../../base

images:
  - name: IMAGE_URL
    newName: IMAGE_URL_PLACEHOLDER

patches:
  # Service account (replace — already in base)
  - patch: |-
      - op: replace
        path: /spec/template/spec/serviceAccountName
        value: {service_account_email}
    target:
      kind: Service
      name: {service_name}

  # Environment variables (VERSION is in base; add ENVIRONMENT and remaining standard vars)
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: ENVIRONMENT
          value: production
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

## Startup Probe Configuration (Critical)

Aggressive settings cause deployment failures:

```yaml
startupProbe:
  httpGet:
    path: /health_check
    port: 8080
  initialDelaySeconds: 5    # Wait before first probe
  timeoutSeconds: 5         # Allow 5s for response
  periodSeconds: 10         # Probe every 10s
  successThreshold: 1       # 1 success to pass
  failureThreshold: 3       # Allow 3 failures (~35s window)
```

**Avoid:**
- `failureThreshold: 1` - Fails on first probe failure
- `initialDelaySeconds: 0` - Probes before app starts
- `timeoutSeconds: 1` - Too short for slow init

## Probe Override Patches

Override probe settings per environment when needed:

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
| | `failureThreshold` | At least 3-5 to avoid premature restarts |
| | `periodSeconds` | 10-30s between checks |
| **livenessProbe** | `initialDelaySeconds` | Higher than startup total window |
| | `periodSeconds` | 30-60s for steady-state checks |

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

## CMD Variations

Services may use different entry points depending on the framework:

```dockerfile
# Direct Python execution
CMD ["python", "main.py"]

# Streamlit
CMD ["streamlit", "run", "main.py", "--server.port=8080", "--server.address=0.0.0.0"]

# Flask / Gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "main:app"]
```

## System Dependencies

If the service needs system packages, add `apt-get` before uv installation:

```dockerfile
FROM python:3.13-slim@sha256:{digest}

RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends ca-certificates curl

# Install uv from official image
COPY --from=ghcr.io/astral-sh/uv:0.7.8 /uv /uvx /usr/local/bin/
# ... rest of Dockerfile
```

## Static Assets

Some services need additional config files or templates copied into the image:

```dockerfile
# Copy static assets (if needed)
COPY .streamlit/ ./.streamlit/
```

## Deployment

```bash
kustomize build . | sed "s|VERSION_PLACEHOLDER|{version}|g" > /tmp/service.yaml
gcloud run services replace /tmp/service.yaml --region=asia-southeast1
```
