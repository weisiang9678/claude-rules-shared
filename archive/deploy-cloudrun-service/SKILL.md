# Deploy Cloud Run Service

Deploy containerized HTTP services to Cloud Run using Docker and Kustomize.

## Trigger

Use this skill when the user asks to:
- Deploy a Cloud Run Service (web server, API)
- Create an always-running HTTP service
- Set up Kustomize for Cloud Run Service
- Configure health checks for Cloud Run

## Project Structure

```
projects/{service_name}/
├── pyproject.toml      # Dependencies and brick references
├── Dockerfile          # Container definition
└── main.py             # Entry point (optional)

infrastructure/cloudrun_service/{service_name}/
├── base/
│   ├── service.yaml       # Base service definition
│   └── kustomization.yaml
└── overlays/              # Only create environments the project needs
    ├── sandbox/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Procedure: Creating New Cloud Run Service

### Step 1: Create Dockerfile

Similar to Cloud Run Job, but the application must expose an HTTP port:

```dockerfile
FROM python:3.13-slim@sha256:{digest}

# Install uv from official image
COPY --from=ghcr.io/astral-sh/uv:0.7.8 /uv /uvx /usr/local/bin/

WORKDIR /app

# Copy project files for dependency resolution
COPY projects/{service_name}/pyproject.toml ./pyproject.toml
COPY uv.lock ./uv.lock

# Install dependencies
RUN uv sync --frozen --no-default-groups --no-install-project

# Copy Polylith bricks
COPY components/{namespace}/logging/ ./{namespace}/logging/
COPY components/{namespace}/settings/ ./{namespace}/settings/
COPY bases/{namespace}/{base_name}/ ./{namespace}/{base_name}/
RUN mv {namespace}/{base_name}/core.py main.py

EXPOSE 8080

# Enable venv in PATH
ENV PATH="/app/.venv/bin:$PATH"

CMD ["python", "main.py"]
```

### Step 2: Create Base Service Manifest

Create `infrastructure/cloudrun_service/{service_name}/base/service.yaml`:

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

### Step 3: Create Kustomization Files

Base `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service.yaml
```

### Step 4: Create Environment Overlays

Create overlays for each required environment. Common options are sandbox, staging, and production — not all projects need all three.

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
  # Service account
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

## Procedure: Deployment

```bash
# Build and push image
docker build -f projects/{service_name}/Dockerfile -t {image_url}:{version} .
docker push {image_url}:{version}

# Update and deploy
cd infrastructure/cloudrun_service/{service_name}/overlays/production
kustomize edit set image IMAGE_URL={image_url}:{version}
kustomize build . | sed "s|VERSION_PLACEHOLDER|{version}|g" > /tmp/service.yaml
gcloud run services replace /tmp/service.yaml --region=asia-southeast1
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

## Reference Files

- [dockerfile-template.md](references/dockerfile-template.md) - Full Dockerfile template
- [kustomize-patches.md](references/kustomize-patches.md) - Common Kustomize patches
