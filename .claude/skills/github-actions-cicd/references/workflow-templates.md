# Workflow Templates

## Infrastructure Provision

```yaml
name: Infrastructure Provision
run-name: Provision ${{ github.event_name == 'workflow_dispatch' && 'production' || 'sandbox' }} infrastructure

on:
  pull_request:
    types: [labeled]
  workflow_dispatch:

jobs:
  provision-infrastructure:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event_name == 'pull_request' && github.event.label.name == 'provision infrastructure')
    runs-on: ubuntu-latest
    environment: ${{ github.event_name == 'workflow_dispatch' && 'production' || 'sandbox' }}
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - name: "Determine stack and version"
        id: determine-stack-version
        run: |
          if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
            STACK="production"
            VERSION=$(echo "${{ github.ref }}" | sed 's|refs/tags/v||')
          else
            STACK="sandbox"
            TIMESTAMP=$(date -u +%Y%m%d-%H%M%S)
            BRANCH=${GITHUB_REF_NAME//[\/\-]/_}
            VERSION=${BRANCH}-${TIMESTAMP}
          fi
          echo "Deploying to stack: ${STACK} with version: ${VERSION}"
          echo "STACK=${STACK}" >> $GITHUB_OUTPUT
          echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

      - uses: actions/setup-python@v5
        with:
          python-version-file: "pyproject.toml"

      - uses: astral-sh/setup-uv@v5
        with:
          version: "0.7.8"

      - name: "Authenticate to Google Cloud"
        uses: google-github-actions/auth@v3
        with:
          token_format: "access_token"
          workload_identity_provider: "projects/${{ secrets.GCP_PROJECT_NUMBER }}/locations/global/workloadIdentityPools/..."
          service_account: "github-actions@${{ vars.GCP_PROJECT_ID }}.iam.gserviceaccount.com"

      - name: "Extract Pulumi version from lock file"
        id: extract-pulumi-version
        run: |
          PULUMI_VERSION=$(uv tree --package pulumi --depth 0 --frozen | sed 's/.* v//')
          echo "Using Pulumi version: ${PULUMI_VERSION}"
          echo "PULUMI_VERSION=${PULUMI_VERSION}" >> $GITHUB_OUTPUT

      - uses: pulumi/actions@v6
        with:
          pulumi-version: ${{ steps.extract-pulumi-version.outputs.PULUMI_VERSION }}

      - working-directory: infrastructure/pulumi
        run: pulumi install

      - uses: pulumi/actions@v6
        env:
          PULUMI_CONFIG_PASSPHRASE: ${{ secrets.PULUMI_CONFIG_PASSPHRASE }}
        with:
          command: up
          cloud-url: gs://${{ vars.PULUMI_BUCKET }}
          work-dir: infrastructure/pulumi
          stack-name: ${{ steps.determine-stack-version.outputs.STACK }}
          upsert: true
          refresh: true
```

## Deploy Cloud Run Function

```yaml
name: Deploy
run-name: Deploy to ${{ github.event.inputs.environment || 'sandbox' }}

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options:
          - sandbox
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: "Configure version tag"
        id: configure-version-tag
        run: |
          VERSION=${{ github.sha }}
          echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

      - uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: "..."
          service_account: "..."

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v5
        with:
          context: .
          file: ./projects/{function_name}/Dockerfile
          push: true
          tags: ${{ vars.IMAGE_URL }}:${{ steps.configure-version-tag.outputs.VERSION }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: "Deploy Cloud Run Function"
        run: |
          cd infrastructure/cloudrun_function/{function_name}/overlays/${{ github.event.inputs.environment }}
          kustomize edit set image IMAGE_URL=${{ vars.IMAGE_URL }}:${{ steps.configure-version-tag.outputs.VERSION }}
          kustomize build . | sed "s|VERSION_PLACEHOLDER|${{ steps.configure-version-tag.outputs.VERSION }}|g" > /tmp/service.yaml
          gcloud run services replace /tmp/service.yaml --region=${{ vars.REGION }}
```

## Deploy Cloud Run Service

```yaml
name: Deploy
run-name: Deploy to ${{ github.event.inputs.environment || 'sandbox' }}

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options:
          - sandbox
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: "Configure version tag"
        id: configure-version-tag
        run: |
          VERSION=${{ github.sha }}
          echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

      - uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: "..."
          service_account: "..."

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v5
        with:
          context: .
          file: ./projects/{service_name}/Dockerfile
          push: true
          tags: ${{ vars.IMAGE_URL }}:${{ steps.configure-version-tag.outputs.VERSION }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: "Deploy Cloud Run Service"
        run: |
          cd infrastructure/cloudrun_service/{service_name}/overlays/${{ github.event.inputs.environment }}
          kustomize edit set image IMAGE_URL=${{ vars.IMAGE_URL }}:${{ steps.configure-version-tag.outputs.VERSION }}
          kustomize build . | sed "s|VERSION_PLACEHOLDER|${{ steps.configure-version-tag.outputs.VERSION }}|g" > /tmp/service.yaml
          gcloud run services replace /tmp/service.yaml --region=${{ vars.REGION }}
```

## Deploy Cloud Run Job

```yaml
name: Deploy
run-name: Deploy to ${{ github.event.inputs.environment || 'sandbox' }}

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options:
          - sandbox
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: "Configure version tag"
        id: configure-version-tag
        run: |
          VERSION=${{ github.sha }}
          echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT

      - uses: google-github-actions/auth@v3
        with:
          workload_identity_provider: "..."
          service_account: "..."

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v5
        with:
          context: .
          file: ./projects/{job_name}/Dockerfile
          push: true
          tags: ${{ vars.IMAGE_URL }}:${{ steps.configure-version-tag.outputs.VERSION }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: "Deploy Cloud Run Job"
        run: |
          cd infrastructure/cloudrun_job/{job_name}/overlays/${{ github.event.inputs.environment }}
          kustomize edit set image IMAGE_URL=${{ vars.IMAGE_URL }}:${{ steps.configure-version-tag.outputs.VERSION }}
          kustomize build . | sed "s|VERSION_PLACEHOLDER|${{ steps.configure-version-tag.outputs.VERSION }}|g" > /tmp/job.yaml
          gcloud run jobs replace /tmp/job.yaml --region=${{ vars.REGION }}
```
