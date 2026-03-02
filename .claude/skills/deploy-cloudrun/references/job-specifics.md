# Job-Specific Configuration

Cloud Run Job-specific manifest, paths, and conventions. For shared patterns, see [shared-dockerfile.md](shared-dockerfile.md) and [shared-kustomize-patches.md](shared-kustomize-patches.md).

## Key Characteristics

- **API:** `apiVersion: run.googleapis.com/v1`, `kind: Job`
- **Infra directory:** `infrastructure/cloudrun_job/{job_name}/`
- **Manifest filename:** `job.yaml`
- **No HTTP port** — no PORT/EXPOSE in Dockerfile
- **No probes** — no startup or liveness probes
- **No autoscaling** — runs to completion
- **CMD:** `["python", "main.py"]`
- **gcloud command:** `gcloud run jobs replace`

## JSON Patch Paths

Jobs have an extra `template` nesting level compared to Services/Functions:

```
/spec/template/spec/template/spec/containers/0/env/-
/spec/template/spec/template/spec/serviceAccountName
/spec/template/spec/template/spec/containers/0/resources/limits/...
```

## Base Manifest Template

`infrastructure/cloudrun_job/{job_name}/base/job.yaml`:

```yaml
apiVersion: run.googleapis.com/v1
kind: Job
metadata:
  name: {job_name}
  labels:
    component: data-pipeline
  annotations:
    run.googleapis.com/launch-stage: BETA
spec:
  template:
    metadata:
      labels:
        component: data-pipeline
      annotations:
        run.googleapis.com/execution-environment: gen2
    spec:
      taskCount: 1
      template:
        spec:
          maxRetries: 3
          timeoutSeconds: 600
          containers:
          - name: worker
            image: IMAGE_URL
            env:
            - name: COMPONENT
              value: data-pipeline
            - name: LOG_EXECUTION_ID
              value: "true"
            resources:
              limits:
                cpu: "1"
                memory: 512Mi
```

## Overlay Template

`infrastructure/cloudrun_job/{job_name}/overlays/{environment}/kustomization.yaml`:

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
      - op: add
        path: /spec/template/spec/template/spec/serviceAccountName
        value: {service_account_email}
    target:
      kind: Job
      name: {job_name}

  # Environment variables (standard)
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

  # Secrets annotation (optional - include if job needs secret access)
  # - patch: |-
  #     - op: add
  #       path: /spec/template/metadata/annotations/run.googleapis.com~1secrets
  #       value: "{secret_env_name}:projects/{gcp_project_id}/secrets/{secret_name}"
  #   target:
  #     kind: Job
  #     name: {job_name}

  # Secret environment variables (optional - include if job needs secrets)
  # - patch: |-
  #     - op: add
  #       path: /spec/template/spec/template/spec/containers/0/env/-
  #       value:
  #         name: {secret_env_name}
  #         valueFrom:
  #           secretKeyRef:
  #             name: {secret_env_name}
  #             key: latest
  #   target:
  #     kind: Job
  #     name: {job_name}
```

## Deployment

```bash
kustomize build . | sed "s|VERSION_PLACEHOLDER|{version}|g" > /tmp/job.yaml
gcloud run jobs replace /tmp/job.yaml --region=asia-southeast1
```
