# Deploy Cloud Run Functions

Deploy containerized Cloud Functions to Cloud Run using Docker, functions-framework, and Kustomize.

## Trigger

Use this skill when the user asks to:
- Deploy a Cloud Run Function
- Set up a new Cloud Function project
- Create Kustomize manifests for Cloud Run Function
- Create Dockerfile for Cloud Run Function

## Project Structure

```
projects/{project_name}/
├── pyproject.toml      # Dependencies and brick references
└── Dockerfile          # Container definition (functions-framework)

infrastructure/cloudrun_function/{function_name}/
├── base/
│   ├── service.yaml       # Base service definition (Knative)
│   └── kustomization.yaml
└── overlays/              # Only create environments the project needs
    ├── sandbox/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Procedure: Creating New Cloud Run Function

### Step 1: Create Dockerfile

Create `projects/{project_name}/Dockerfile`:

```dockerfile
FROM python:3.13-slim@sha256:{digest}

# Install uv from official image
COPY --from=ghcr.io/astral-sh/uv:0.7.8 /uv /uvx /usr/local/bin/

WORKDIR /app

# Copy project files for dependency resolution
COPY projects/{project_name}/pyproject.toml ./pyproject.toml
COPY uv.lock ./uv.lock

# Install dependencies
RUN uv sync --frozen --no-default-groups --no-install-project

# Copy Polylith bricks
COPY components/{namespace}/gcp/ ./{namespace}/gcp/
COPY components/{namespace}/settings/ ./{namespace}/settings/
COPY bases/{namespace}/{base_name}/ ./{namespace}/{base_name}/
RUN mv {namespace}/{base_name}/core.py main.py

ENV PORT=8080
EXPOSE 8080

# Enable venv in PATH so functions-framework can be found
ENV PATH="/app/.venv/bin:$PATH"

# Run with functions-framework
CMD exec functions-framework --target=${FUNCTION_TARGET} --port=${PORT}
```

**Key differences from Cloud Run Service/Job Dockerfile:**
- Requires `functions-framework` in project dependencies
- CMD uses `functions-framework` instead of `python main.py`
- Entry point is dynamic via `FUNCTION_TARGET` env var

### Step 2: Create Base Service Manifest

Create `infrastructure/cloudrun_function/{function_name}/base/service.yaml`:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: {function_name}
  labels:
    component: pipeline
    epic: 3p-data-pipeline
    owner: data-engineering
    workload-type: cloud-function
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      labels:
        component: pipeline
        epic: 3p-data-pipeline
        owner: data-engineering
        workload-type: cloud-function
      annotations:
        cloudfunctions.googleapis.com/trigger-type: HTTP_TRIGGER
        run.googleapis.com/client-name: cloudfunctions
        run.googleapis.com/startup-cpu-boost: "true"
    spec:
      containerConcurrency: 1
      timeoutSeconds: 60
      containers:
      - name: worker
        image: IMAGE_URL
        ports:
        - name: http1
          containerPort: 8080
        env:
        - name: COMPONENT
          value: pipeline
        - name: EPIC
          value: 3p-data-pipeline
        - name: LOG_EXECUTION_ID
          value: "true"
        resources:
          limits:
            cpu: "0.5"
            memory: 512Mi
        startupProbe:
          timeoutSeconds: 60
          periodSeconds: 120
          failureThreshold: 3
          successThreshold: 1
          tcpSocket:
            port: 8080
  traffic:
  - percent: 100
    latestRevision: true
```

### Step 3: Create Base Kustomization

Create `infrastructure/cloudrun_function/{function_name}/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service.yaml
```

### Step 4: Create Environment Overlays

Create overlays for each required environment. Common options are sandbox, staging, and production — not all projects need all three.

`infrastructure/cloudrun_function/{function_name}/overlays/{environment}/kustomization.yaml`:

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
  # Autoscaling
  - patch: |-
      - op: add
        path: /spec/template/metadata/annotations/autoscaling.knative.dev~1maxScale
        value: "200"
    target:
      kind: Service
      name: {function_name}

  # Service account
  - patch: |-
      - op: add
        path: /spec/template/spec/serviceAccountName
        value: {service_account_email}
    target:
      kind: Service
      name: {function_name}

  # Environment variables
  - patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: FUNCTION_TARGET
          value: {python_function_name}
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
      name: {function_name}
```

## Procedure: Deployment

### Step 1: Build and Push Image

```bash
docker build -f projects/{project_name}/Dockerfile -t {image_url}:{version} .
docker push {image_url}:{version}
```

### Step 2: Update Image in Kustomize

```bash
cd infrastructure/cloudrun_function/{function_name}/overlays/{environment}
kustomize edit set image IMAGE_URL={image_url}:{version}
```

### Step 3: Build and Deploy Manifest

```bash
kustomize build . | sed "s|VERSION_PLACEHOLDER|{version}|g" > /tmp/service.yaml
gcloud run services replace /tmp/service.yaml --region=asia-southeast1
```

## Key Differences from Service and Job

| Aspect | Cloud Run Function | Service / Job |
|--------|-------------------|---------------|
| Infra Directory | `cloudrun_function/` | `cloudrun_service/` or `cloudrun_job/` |
| Framework | `functions-framework` | Direct `python main.py` |
| Entry Point | `FUNCTION_TARGET` env var | Hardcoded in CMD |
| Startup Probe | `tcpSocket` | `httpGet` or None |
| containerConcurrency | `1` | Default or N/A |
| Labels | `workload-type: cloud-function` | Custom |
| Annotations | `cloudfunctions.googleapis.com/*` | `run.googleapis.com/*` |

## Reference Files

- [dockerfile-template.md](references/dockerfile-template.md) - Full Dockerfile template
- [kustomize-patches.md](references/kustomize-patches.md) - Common Kustomize patches
