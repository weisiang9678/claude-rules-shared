---
description: "Use when deploying Cloud Run Jobs, Services, or Functions, or creating Dockerfiles and Kustomize manifests for Cloud Run."
---

# Deploy Cloud Run

Deploy containerized workloads to Cloud Run using Docker and Kustomize. Covers Jobs, Services, and Functions.

## Choosing the Right Type

| Aspect | Job | Service | Function |
|--------|-----|---------|----------|
| Use case | Batch processing, scheduled tasks | HTTP APIs, web apps, long-running services | Event-driven, single-purpose HTTP handlers |
| API | `run.googleapis.com/v1` / `Kind: Job` | `serving.knative.dev/v1` / `Kind: Service` | `serving.knative.dev/v1` / `Kind: Service` |
| Infra directory | `infrastructure/cloudrun_job/` | `infrastructure/cloudrun_service/` | `infrastructure/cloudrun_function/` |
| Ports | None | Required (8080) | Required (8080) |
| Probes | None | httpGet startup + liveness | tcpSocket startup |
| Autoscaling | N/A | minScale + maxScale | maxScale only |
| Concurrency | N/A | Default | 1 (`containerConcurrency`) |
| Framework | Direct `python main.py` | App-specific (Flask, Streamlit, etc.) | `functions-framework` |
| gcloud command | `gcloud run jobs replace` | `gcloud run services replace` | `gcloud run services replace` |
| Manifest filename | `job.yaml` | `service.yaml` | `service.yaml` |

## Project Structure

```
projects/{name}/
├── pyproject.toml      # Dependencies and brick references
├── Dockerfile          # Container definition
└── main.py             # Entry point (optional, Service only)

infrastructure/cloudrun_{type}/{name}/
├── base/
│   ├── {job|service}.yaml    # Base manifest
│   └── kustomization.yaml
└── overlays/                  # Only create environments the project needs
    ├── sandbox/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Procedure: Creating New Cloud Run Workload

### Step 1: Create Dockerfile

See [shared-dockerfile.md](references/shared-dockerfile.md) for the common template and key principles.

For type-specific CMD, PORT, and framework differences, see:
- **Job:** [job-specifics.md](references/job-specifics.md)
- **Service:** [service-specifics.md](references/service-specifics.md) — also covers system dependencies and static assets
- **Function:** [function-specifics.md](references/function-specifics.md) — requires `functions-framework`

### Step 2: Create Base Manifest

Create the base manifest in `infrastructure/cloudrun_{type}/{name}/base/`:

- **Job:** `job.yaml` — see [job-specifics.md](references/job-specifics.md) for full template
- **Service:** `service.yaml` — see [service-specifics.md](references/service-specifics.md) for full template
- **Function:** `service.yaml` — see [function-specifics.md](references/function-specifics.md) for full template

### Step 3: Create Base Kustomization

Create `infrastructure/cloudrun_{type}/{name}/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - {job|service}.yaml
```

### Step 4: Create Environment Overlays

Create overlays for each required environment. Common options are sandbox, staging, and production — not all projects need all three.

See [shared-kustomize-patches.md](references/shared-kustomize-patches.md) for common patches (service account, env vars, secrets, resource limits).

For type-specific overlay patterns, see the relevant specifics file:
- **Job:** [job-specifics.md](references/job-specifics.md) — deeper JSON patch paths
- **Service:** [service-specifics.md](references/service-specifics.md) — autoscaling, probe overrides
- **Function:** [function-specifics.md](references/function-specifics.md) — FUNCTION_TARGET, maxScale

## Procedure: Deployment

### Step 1: Build and Push Image

```bash
docker build -f projects/{name}/Dockerfile -t {image_url}:{version} .
docker push {image_url}:{version}
```

### Step 2: Update Image in Kustomize

```bash
cd infrastructure/cloudrun_{type}/{name}/overlays/{environment}
kustomize edit set image IMAGE_URL={image_url}:{version}
```

### Step 3: Build and Deploy Manifest

```bash
kustomize build . | sed "s|VERSION_PLACEHOLDER|{version}|g" > /tmp/{job|service}.yaml
gcloud run {jobs|services} replace /tmp/{job|service}.yaml --region=asia-southeast1
```

## Reference Files

- [shared-dockerfile.md](references/shared-dockerfile.md) — Common Dockerfile template and principles
- [shared-kustomize-patches.md](references/shared-kustomize-patches.md) — Common Kustomize patches
- [job-specifics.md](references/job-specifics.md) — Job manifest, paths, and conventions
- [service-specifics.md](references/service-specifics.md) — Service manifest, probes, autoscaling, CMD variations
- [function-specifics.md](references/function-specifics.md) — Function manifest, functions-framework, annotations
