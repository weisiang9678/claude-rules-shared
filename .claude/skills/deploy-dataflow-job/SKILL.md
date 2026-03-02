# Deploy Dataflow Job

Deploy Apache Beam streaming pipelines to Google Cloud Dataflow using Flex Templates.

## Trigger

Use this skill when the user asks to:
- Deploy a Dataflow job
- Create an Apache Beam streaming pipeline
- Set up Flex Templates for Dataflow
- Configure blue-green deployment for Dataflow

## Project Structure

```
projects/{job_name}/
├── pyproject.toml      # Dependencies and brick references
├── Dockerfile          # Flex Template container
├── metadata.json       # Template parameters
├── setup.py            # Generated setup for Beam
└── generate_setup.py   # Script to generate setup.py
```

## Procedure: Creating New Dataflow Job

### Step 1: Create pyproject.toml

Create `projects/{job_name}/pyproject.toml`:

```toml
[build-system]
requires = ["hatchling", "hatch-polylith-bricks"]
build-backend = "hatchling.build"

[project]
name = "{job-name}"
version = "1.0.0"
description = "Description of the Dataflow job."
requires-python = "~=3.13.0"
authors = [{ name = "Data Engineering Team", email = "de-team@pulsifi.me" }]
dependencies = [
    "apache-beam[docs,gcp,test]==2.69.0",
    "grpcio>=1.64.3",
    "jsonschema>=4.24.0",
    "numpy>=1.26.4",
    "setuptools>=80.9.0",
]

[tool.hatch.build.hooks.polylith-bricks]
enabled = true

[tool.polylith.bricks]
"../../bases/{namespace}/{base_name}" = "{namespace}/{base_name}"
"../../components/{namespace}/{component1}" = "{namespace}/{component1}"
"../../components/{namespace}/{component2}" = "{namespace}/{component2}"

[tool.uv]
default-groups = "all"
```

### Step 2: Create Dockerfile

Create `projects/{job_name}/Dockerfile`:

```dockerfile
FROM gcr.io/dataflow-templates-base/python313-template-launcher-base:20251030-rc00

ARG WORKDIR=/dataflow/template
RUN mkdir -p ${WORKDIR}
WORKDIR ${WORKDIR}

ENV FLEX_TEMPLATE_PYTHON_PY_FILE="${WORKDIR}/core.py"
ENV FLEX_TEMPLATE_PYTHON_SETUP_FILE="${WORKDIR}/setup.py"
ENV PIP_ROOT_USER_ACTION=ignore

RUN mkdir {namespace}
RUN touch {namespace}/__init__.py

COPY components/{namespace}/{component1} {namespace}/{component1}
COPY components/{namespace}/{component2} {namespace}/{component2}
COPY bases/{namespace}/{base_name} {namespace}/{base_name}
COPY projects/{job_name}/requirements.txt requirements.txt
COPY projects/{job_name}/setup.py setup.py
RUN mv {namespace}/{base_name}/core.py core.py


# Install apache-beam and other dependencies to launch the pipeline
RUN apt-get update
RUN pip install pip -U
RUN pip install -r requirements.txt --no-cache-dir
```

**Key differences from Cloud Run:**
- Uses `gcr.io/dataflow-templates-base/python313-template-launcher-base` base image
- Sets `FLEX_TEMPLATE_PYTHON_PY_FILE` for pipeline entry point
- Sets `FLEX_TEMPLATE_PYTHON_SETUP_FILE` for Beam worker dependencies
- Uses `pip` instead of `uv` (Dataflow worker requirement)

### Step 3: Create metadata.json

Create `projects/{job_name}/metadata.json`:

```json
{
    "name": "{job-name}-dataflow-job-template",
    "description": "Description of your Dataflow job.",
    "parameters": [
        {
            "name": "project_id",
            "helpText": "The GCP Project ID",
            "label": "Project ID",
            "isOptional": false
        },
        {
            "name": "input_subscription",
            "helpText": "The GCP Pub/Sub subscription to read from, e.g. projects/<PROJECT_ID>/subscriptions/<SUBSCRIPTION_NAME>",
            "label": "Input Subscription",
            "isOptional": false
        },
        {
            "name": "validation_failure_topic",
            "helpText": "The GCP Pub/Sub topic to store failed validation message, e.g. projects/<PROJECT_ID>/topics/<TOPIC_NAME>",
            "label": "Failure Message Topic",
            "isOptional": false
        },
        {
            "name": "version",
            "helpText": "The semantic release version.",
            "label": "Semantic release version",
            "isOptional": false
        },
        {
            "name": "epic",
            "helpText": "The Jira epic",
            "label": "Jira epic",
            "isOptional": false
        },
        {
            "name": "component",
            "helpText": "The Jira component",
            "label": "Jira component",
            "isOptional": false
        }
    ]
}
```

### Step 4: Create generate_setup.py

Create `projects/{job_name}/generate_setup.py`:

```python
import tomllib
from pathlib import Path

SETUP = """
from setuptools import find_packages, setup

REQUIRED_PACKAGES = {REQUIRED_PACKAGES}

setup(packages=find_packages(), install_requires=REQUIRED_PACKAGES)
"""


def get_project_dependencies(
    pyproject_path: Path = Path("pyproject.toml"),
) -> list[str] | None:
    """
    Parses a pyproject.toml file and returns the list of dependencies
    from the [project.dependencies] section.

    Args:
        pyproject_path: Path to the pyproject.toml file.

    Returns:
        A list of dependency strings, or None if not found or an error occurs.
    """
    if not pyproject_path.exists():
        print(f"Error: {pyproject_path} not found.")
        return None

    try:
        with open(
            pyproject_path, "rb"
        ) as f:  # TOML files should be opened in binary mode
            data = tomllib.load(f)
    except tomllib.TOMLDecodeError as e:
        print(f"Error decoding TOML file {pyproject_path}: {e}")
        return None
    except Exception as e:
        print(f"An unexpected error occurred while reading {pyproject_path}: {e}")
        return None

    # Navigate through the TOML structure
    project_table = data.get("project")
    if not project_table:
        print("Error: [project] table not found in pyproject.toml.")
        return None

    dependencies_list = project_table.get("dependencies")
    if dependencies_list is None:  # It could be an empty list, which is valid
        print(
            "Warning: 'dependencies' array not found under [project] table or it's empty."
        )
        return []  # Return empty list if not found, as per PEP 621 it's optional

    if not isinstance(dependencies_list, list):
        print("Error: 'dependencies' under [project] is not a list.")
        return None

    # Ensure all items in the list are strings (as per PEP 621)
    if not all(isinstance(dep, str) for dep in dependencies_list):
        print("Error: Not all items in 'dependencies' list are strings.")
        return None

    return dependencies_list


with open("setup.py", "w") as unit:
    unit.write(SETUP.format(REQUIRED_PACKAGES=get_project_dependencies()))

print("setup.py created.")
```

### Step 5: Create setup.py template

Create `projects/{job_name}/setup.py` (initial template, will be regenerated):

```python
from setuptools import find_packages, setup

REQUIRED_PACKAGES = []

setup(packages=find_packages(), install_requires=REQUIRED_PACKAGES)
```

## Procedure: Deployment

### Step 1: Generate Dependencies

```bash
cd projects/{job_name}
uv sync --frozen
uv export --no-default-groups --no-hashes --no-annotate --no-header --no-editable --no-emit-workspace --format=requirements.txt --output-file=./requirements.txt
uv run generate_setup.py
```

### Step 2: Build and Push Docker Image

```bash
# From workspace root
docker build -f projects/{job_name}/Dockerfile -t {image_url}:{version} .
docker push {image_url}:{version}
```

### Step 3: Build Flex Template

```bash
gcloud dataflow flex-template build gs://{bucket}/flex-template/dataflow-flex-template.json \
    --image={image_url}:{version} \
    --sdk-language="PYTHON" \
    --metadata-file="projects/{job_name}/metadata.json"
```

### Step 4: Deploy Dataflow Job

```bash
gcloud dataflow flex-template run "{job_name}-{version}-{region_abbr}" \
    --template-file-gcs-location="gs://{bucket}/flex-template/dataflow-flex-template.json" \
    --region="{region}" \
    --project="{gcp_project_id}" \
    --service-account-email="{service_account}@{gcp_project_id}.iam.gserviceaccount.com" \
    --max-workers=4 \
    --enable-streaming-engine \
    --additional-experiments="enable_streaming_engine,enable_windmill_service,beam_fn_api,use_unified_worker,use_runner_v2,use_portable_job_submission,use_multiple_sdk_containers" \
    --parameters="project_id={gcp_project_id},input_subscription=projects/{gcp_project_id}/subscriptions/{subscription_name},validation_failure_topic=projects/{gcp_project_id}/topics/{dlq_topic},version={version},epic={epic},component={component}" \
    --additional-user-labels="epic={epic},component={component},owner=data-engineering,version={formatted_version}"
```

## Blue-Green Deployment

Dataflow streaming jobs require blue-green deployment for zero-downtime updates:

1. **Deploy new job (GREEN)** - Start new job with new version
2. **Wait for healthy** - Monitor until job reaches Running state
3. **Drain old jobs (BLUE)** - Send drain command to old jobs

### Drain vs Cancel

| Command | Behavior |
|---------|----------|
| `drain` | Processes remaining messages, then stops gracefully |
| `cancel` | Stops immediately, may lose in-flight messages |

**Always use `drain` for production deployments.**

### Drain Old Jobs

```bash
# List running jobs with prefix
gcloud dataflow jobs list \
    --region="{region}" \
    --project="{gcp_project_id}" \
    --filter="name~^{job_name_prefix} AND state=Running" \
    --format="value(id,name)"

# Drain specific job
gcloud dataflow jobs drain "{job_id}" \
    --region="{region}" \
    --project="{gcp_project_id}"
```

## Key Differences from Cloud Run

| Aspect | Dataflow | Cloud Run |
|--------|----------|-----------|
| Base Image | `gcr.io/dataflow-templates-base/python313-template-launcher-base` | `python:3.13-slim` |
| Package Manager | `pip` (required by Dataflow workers) | `uv` |
| Entry Point | `FLEX_TEMPLATE_PYTHON_PY_FILE` | `CMD` |
| Deployment | Flex Template + `gcloud dataflow` | `gcloud run` |
| Updates | Blue-green with drain | Rolling update |
| Scaling | Auto-managed by Dataflow | `minScale`/`maxScale` |

## Required IAM Roles

Service account for Dataflow worker needs:

| Role | Purpose |
|------|---------|
| `roles/dataflow.worker` | Run Dataflow jobs |
| `roles/bigquery.admin` | Write to BigQuery |
| `roles/pubsub.viewer` | View Pub/Sub resources |
| `roles/storage.objectUser` | Access GCS for temp files |
| `roles/artifactregistry.serviceAgent` | Pull container images |
| `roles/monitoring.admin` | Write custom metrics |
| `roles/datacatalog.categoryFineGrainedReader` | Access Data Catalog policies |

## Reference Files

- [blue-green-workflow.md](references/blue-green-workflow.md) - GitHub Actions workflow for blue-green deployment
- [metadata-parameters.md](references/metadata-parameters.md) - Common Flex Template parameters
