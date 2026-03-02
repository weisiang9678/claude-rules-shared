# Function-Specific Configuration

Cloud Run Function-specific manifest, annotations, and conventions. For shared patterns, see [shared-dockerfile.md](shared-dockerfile.md) and [shared-kustomize-patches.md](shared-kustomize-patches.md).

## Key Characteristics

- **API:** `apiVersion: serving.knative.dev/v1`, `kind: Service` (same as Service)
- **Infra directory:** `infrastructure/cloudrun_function/{function_name}/`
- **Manifest filename:** `service.yaml`
- **HTTP port required** — EXPOSE 8080 in Dockerfile
- **Startup probe:** `tcpSocket` (not httpGet like Service)
- **Autoscaling:** `maxScale` only (no minScale)
- **Concurrency:** `containerConcurrency: 1`
- **Framework:** `functions-framework` (must be in project dependencies)
- **CMD:** `exec functions-framework --target=${FUNCTION_TARGET} --port=${PORT}`
- **Extra env var:** `FUNCTION_TARGET` — the Python function name to invoke
- **gcloud command:** `gcloud run services replace`

## Dockerfile Additions

Functions require `functions-framework` and use a different CMD:

```dockerfile
ENV PORT=8080
EXPOSE 8080

# Enable venv in PATH so functions-framework can be found
ENV PATH="/app/.venv/bin:$PATH"

# Run with functions-framework
CMD exec functions-framework --target=${FUNCTION_TARGET} --port=${PORT}
```

**Key differences from Service/Job Dockerfile:**
- Requires `functions-framework` in project dependencies
- CMD uses `functions-framework` instead of `python main.py`
- Entry point is dynamic via `FUNCTION_TARGET` env var

## Base Manifest Template

`infrastructure/cloudrun_function/{function_name}/base/service.yaml`:

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

## Overlay Template

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
  # Autoscaling (maxScale only)
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

  # Environment variables (includes FUNCTION_TARGET)
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

## Function-Specific Annotations and Labels

Functions use special annotations and labels to identify them as Cloud Functions:

| Annotation/Label | Value | Purpose |
|------------------|-------|---------|
| `cloudfunctions.googleapis.com/trigger-type` | `HTTP_TRIGGER` | Identifies function trigger type |
| `run.googleapis.com/client-name` | `cloudfunctions` | Identifies deployment origin |
| `workload-type` (label) | `cloud-function` | Distinguishes from regular services |

## Deployment

```bash
kustomize build . | sed "s|VERSION_PLACEHOLDER|{version}|g" > /tmp/service.yaml
gcloud run services replace /tmp/service.yaml --region=asia-southeast1
```
