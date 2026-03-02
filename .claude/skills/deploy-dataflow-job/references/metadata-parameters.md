# Flex Template Metadata Parameters

Common parameters for Dataflow Flex Template `metadata.json` files.

## Required Parameters

### GCP Project

```json
{
    "name": "project_id",
    "helpText": "The GCP Project ID",
    "label": "Project ID",
    "isOptional": false
}
```

### Pub/Sub Input

```json
{
    "name": "input_subscription",
    "helpText": "The GCP Pub/Sub subscription to read from, e.g. projects/<PROJECT_ID>/subscriptions/<SUBSCRIPTION_NAME>",
    "label": "Input Subscription",
    "isOptional": false
}
```

### Dead Letter Queue

```json
{
    "name": "validation_failure_topic",
    "helpText": "The GCP Pub/Sub topic to store failed validation message, e.g. projects/<PROJECT_ID>/topics/<TOPIC_NAME>",
    "label": "Failure Message Topic",
    "isOptional": false
}
```

### Version Tracking

```json
{
    "name": "version",
    "helpText": "The semantic release version.",
    "label": "Semantic release version",
    "isOptional": false
}
```

### Jira Integration

```json
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
```

## Optional Parameters

### Window Duration

```json
{
    "name": "merge_or_delete_window_duration",
    "helpText": "Minute to perform merge and delete statements for a BigQuery table, minute. Defaulted to 5 minutes.",
    "label": "Window Duration",
    "isOptional": true
}
```

### Partition Expiration

```json
{
    "name": "partition_expiration",
    "helpText": "Duration to expire of BigQuery table partition, millisecond. Defaulted to 7 days.",
    "label": "Partition Expiration",
    "isOptional": true
}
```

### Worker Region

```json
{
    "name": "worker_region",
    "helpText": "The GCP region where workers will be deployed, e.g. asia-southeast1",
    "label": "Worker Region",
    "isOptional": true
}
```

## Complete Example

```json
{
    "name": "app-cdc-stream-dataflow-job-template",
    "description": "Pulsifi Application PostgreSQL CDC streaming pipeline.",
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
            "name": "merge_or_delete_window_duration",
            "helpText": "Minute to perform merge and delete statements for a BigQuery table, minute. Defaulted to 5 minutes.",
            "label": "Window Duration",
            "isOptional": true
        },
        {
            "name": "partition_expiration",
            "helpText": "Duration to expire of BigQuery table partition, millisecond. Defaulted to 7 days.",
            "label": "Partition Expiration",
            "isOptional": true
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

## Parameter Usage in gcloud

Parameters are passed via `--parameters` flag:

```bash
gcloud dataflow flex-template run "{job_name}" \
    --template-file-gcs-location="gs://{bucket}/flex-template.json" \
    --parameters="project_id={gcp_project_id},input_subscription=projects/{gcp_project_id}/subscriptions/{subscription},validation_failure_topic=projects/{gcp_project_id}/topics/{topic},merge_or_delete_window_duration=10,partition_expiration=604800000,version={version},epic={epic},component={component}"
```

## User Labels

Use `--additional-user-labels` for monitoring and cost tracking:

```bash
--additional-user-labels="epic={epic},component={component},owner=data-engineering,version={formatted_version}"
```

**Note:** Label values must be lowercase and can only contain letters, numbers, underscores, and hyphens. Version strings are formatted to comply:

```bash
FORMATTED_VERSION=$(echo "$VERSION" | tr '_.' '-' | cut -c1-40)
```
