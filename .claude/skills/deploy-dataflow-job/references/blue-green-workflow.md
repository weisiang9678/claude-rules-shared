# Blue-Green Deployment Workflow

GitHub Actions workflow pattern for zero-downtime Dataflow deployments.

## Workflow Structure

```yaml
deploy-dataflow:
  runs-on: ubuntu-latest
  environment: ${{ github.event.inputs.environment || 'sandbox' }}
  permissions:
    id-token: write
    contents: read
  steps:
    # ... setup steps ...

    - name: "Build flex-template for Dataflow job"
      id: build-flex-template
      working-directory: ./projects/{job_name}
      env:
        IMAGE_PATH: ${{ steps.set-image-url.outputs.IMAGE_VERSION }}
        FLEX_TEMPLATE_PATH: gs://${{ vars.PULUMI_BUCKET }}/flex-template/dataflow-flex-template.json
      run: |
        gcloud dataflow flex-template build ${FLEX_TEMPLATE_PATH} \
            --image=${IMAGE_PATH} \
            --sdk-language="PYTHON" \
            --metadata-file="metadata.json"

        echo "FLEX_TEMPLATE_PATH=${FLEX_TEMPLATE_PATH}" >> $GITHUB_OUTPUT

    - name: "Deploy new Dataflow job (GREEN)"
      id: deploy-green
      env:
        VERSION: ${{ steps.configure-version-tag.outputs.VERSION }}
        GCP_REGION_ABBR: sg
        GCP_REGION: asia-southeast1
        FLEX_TEMPLATE_PATH: ${{ steps.build-flex-template.outputs.FLEX_TEMPLATE_PATH }}
        JOB_NAME_PREFIX: {job_name}
      run: |
        FORMATTED_VERSION=$(echo "$VERSION" | tr '_.' '-' | cut -c1-40)
        NEW_JOB_NAME="${JOB_NAME_PREFIX}-${FORMATTED_VERSION}-${GCP_REGION_ABBR}"
        echo "Deploying new Dataflow job (GREEN): ${NEW_JOB_NAME}"

        gcloud dataflow flex-template run "${NEW_JOB_NAME}" \
          --template-file-gcs-location="${FLEX_TEMPLATE_PATH}" \
          --region="${GCP_REGION}" \
          --project="${{ vars.GCP_PROJECT_ID }}" \
          --service-account-email="{service_account}@${{ vars.GCP_PROJECT_ID }}.iam.gserviceaccount.com" \
          --max-workers=4 \
          --enable-streaming-engine \
          --additional-experiments="enable_streaming_engine,enable_windmill_service,beam_fn_api,use_unified_worker,use_runner_v2,use_portable_job_submission,use_multiple_sdk_containers" \
          --parameters="project_id=${{ vars.GCP_PROJECT_ID }},input_subscription=projects/${{ vars.GCP_PROJECT_ID }}/subscriptions/{subscription}-${GCP_REGION_ABBR},validation_failure_topic=projects/${{ vars.GCP_PROJECT_ID }}/topics/{dlq_topic},version=${VERSION},epic={epic},component={component}" \
          --additional-user-labels="epic={epic},component={component},owner=data-engineering,version=${FORMATTED_VERSION}"

        echo "NEW_JOB_NAME=${NEW_JOB_NAME}" >> $GITHUB_OUTPUT

    - name: "Wait for new Dataflow job to be Running"
      id: wait-for-green
      env:
        GCP_REGION: asia-southeast1
        NEW_JOB_NAME: ${{ steps.deploy-green.outputs.NEW_JOB_NAME }}
      run: |
        echo "Waiting for new job '${NEW_JOB_NAME}' to reach Running state..."

        MAX_ATTEMPTS=30
        ATTEMPT=0

        while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
          ATTEMPT=$((ATTEMPT + 1))

          JOB_INFO=$(gcloud dataflow jobs list \
            --region="${GCP_REGION}" \
            --project="${{ vars.GCP_PROJECT_ID }}" \
            --filter="name=${NEW_JOB_NAME}" \
            --format="value(id,state)" \
            --limit=1 \
            2>/dev/null || true)

          if [ -z "${JOB_INFO}" ]; then
            echo "Job not found yet, waiting... (attempt ${ATTEMPT}/${MAX_ATTEMPTS})"
            sleep 30
            continue
          fi

          JOB_ID=$(echo "${JOB_INFO}" | cut -f1)
          JOB_STATE=$(echo "${JOB_INFO}" | cut -f2)

          echo "Job ${NEW_JOB_NAME} (${JOB_ID}) is in state: ${JOB_STATE}"

          if [ "${JOB_STATE}" = "Running" ]; then
            echo "New job is now Running!"
            echo "NEW_JOB_ID=${JOB_ID}" >> $GITHUB_OUTPUT
            echo "GREEN_HEALTHY=true" >> $GITHUB_OUTPUT
            exit 0
          elif [ "${JOB_STATE}" = "Failed" ] || [ "${JOB_STATE}" = "Cancelled" ]; then
            echo "::error::New job failed to start with state: ${JOB_STATE}"
            echo "GREEN_HEALTHY=false" >> $GITHUB_OUTPUT
            exit 1
          fi

          echo "Job still starting, waiting... (attempt ${ATTEMPT}/${MAX_ATTEMPTS})"
          sleep 30
        done

        echo "::error::Timeout waiting for new job to reach Running state"
        echo "GREEN_HEALTHY=false" >> $GITHUB_OUTPUT
        exit 1

    - name: "Drain old Dataflow jobs (BLUE)"
      if: steps.wait-for-green.outputs.GREEN_HEALTHY == 'true'
      env:
        GCP_REGION: asia-southeast1
        JOB_NAME_PREFIX: {job_name}
        NEW_JOB_ID: ${{ steps.wait-for-green.outputs.NEW_JOB_ID }}
      run: |
        echo "New job is healthy. Draining old jobs (BLUE)..."
        echo "Excluding new job ID: ${NEW_JOB_ID}"

        RUNNING_JOBS=$(gcloud dataflow jobs list \
          --region="${GCP_REGION}" \
          --project="${{ vars.GCP_PROJECT_ID }}" \
          --filter="name~^${JOB_NAME_PREFIX} AND state=Running" \
          --format="value(id,name)" \
          2>/dev/null || true)

        if [ -z "${RUNNING_JOBS}" ]; then
          echo "No other running Dataflow jobs found to drain."
          exit 0
        fi

        DRAINED_COUNT=0

        while IFS=$'\t' read -r JOB_ID JOB_NAME; do
          if [ -n "${JOB_ID}" ] && [ "${JOB_ID}" != "${NEW_JOB_ID}" ]; then
            echo "Draining old job (BLUE): ${JOB_NAME} (${JOB_ID})"
            gcloud dataflow jobs drain "${JOB_ID}" \
              --region="${GCP_REGION}" \
              --project="${{ vars.GCP_PROJECT_ID }}" || true
            DRAINED_COUNT=$((DRAINED_COUNT + 1))
          fi
        done <<< "${RUNNING_JOBS}"

        echo "Drain commands sent for ${DRAINED_COUNT} job(s)."
```

## Deployment Summary Step

Always include a summary step for debugging:

```yaml
- name: "Deployment summary"
  if: always()
  env:
    NEW_JOB_NAME: ${{ steps.deploy-green.outputs.NEW_JOB_NAME }}
    NEW_JOB_ID: ${{ steps.wait-for-green.outputs.NEW_JOB_ID }}
    GREEN_HEALTHY: ${{ steps.wait-for-green.outputs.GREEN_HEALTHY }}
  run: |
    echo "============================================"
    echo "BLUE-GREEN DEPLOYMENT SUMMARY (${{ env.ENVIRONMENT }})"
    echo "============================================"
    echo "Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
    echo ""
    echo "NEW DEPLOYMENT (GREEN):"
    echo "  Job Name: ${NEW_JOB_NAME:-N/A}"
    echo "  Job ID:   ${NEW_JOB_ID:-N/A}"
    echo "  Healthy:  ${GREEN_HEALTHY:-N/A}"
    echo ""
    if [ "${GREEN_HEALTHY}" = "true" ]; then
      echo "STATUS: SUCCESS"
      echo "New job is running. Old jobs are draining in the background."
    else
      echo "STATUS: FAILED"
      echo "New job did not become healthy."
      echo "Old jobs were NOT drained to prevent service interruption."
    fi
    echo "============================================"
```

## Pre-deployment Steps

### Export Dependencies

```yaml
- name: "uv export dependencies and generate setup.py"
  working-directory: ./projects/{job_name}
  run: |
    uv sync --frozen
    uv export --no-default-groups --no-hashes --no-annotate --no-header --no-editable --no-emit-workspace --format=requirements.txt --output-file=./requirements.txt
    uv run generate_setup.py
```

### Version Tag Configuration

```yaml
- name: "Configure version tag"
  id: configure-version-tag
  run: |
    TIMESTAMP=$(date -u +%Y%m%d-%H%M%S)
    # Use git tag if available, otherwise use branch name
    if [[ "${{ github.ref_name }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+ ]]; then
      VERSION="${{ github.ref_name }}-${TIMESTAMP}"
    else
      BRANCH=${GITHUB_REF_NAME//[\/-]/_}
      VERSION=${BRANCH}-${TIMESTAMP}
    fi

    echo "VERSION=${VERSION}" >> $GITHUB_OUTPUT
```

### Image URL Configuration

```yaml
- name: "Set image url"
  id: set-image-url
  run: |
    IMAGE_VERSION="asia-southeast1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/{repository}/{job_name}:${{ steps.configure-version-tag.outputs.VERSION }}"
    IMAGE_LATEST="asia-southeast1-docker.pkg.dev/${{ vars.GCP_PROJECT_ID }}/{repository}/{job_name}:latest"

    echo "IMAGE_VERSION=$IMAGE_VERSION" >> $GITHUB_OUTPUT
    echo "IMAGE_LATEST=$IMAGE_LATEST" >> $GITHUB_OUTPUT
```

## Job Naming Convention

Format: `{job_name_prefix}-{formatted_version}-{region_abbr}`

- **job_name_prefix**: Base name (e.g., `app-cdc-stream`)
- **formatted_version**: Version with special chars replaced, max 40 chars
- **region_abbr**: Region abbreviation (e.g., `sg`, `de`)

```bash
FORMATTED_VERSION=$(echo "$VERSION" | tr '_.' '-' | cut -c1-40)
NEW_JOB_NAME="${JOB_NAME_PREFIX}-${FORMATTED_VERSION}-${GCP_REGION_ABBR}"
```
