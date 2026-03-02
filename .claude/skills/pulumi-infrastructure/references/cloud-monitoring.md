# Cloud Monitoring

Configure observability with log-based metrics, alert policies, and dashboards.

## Stack Configuration

Monitoring is configured at the workspace level (not per-project) under `input:monitoring`:

```yaml
config:
  # Reusable metric descriptor (YAML anchor)
  input:log_metric_distribution_descriptor: &log_metric_distribution_descriptor
    metric_kind: DELTA
    value_type: DISTRIBUTION
    unit: "1"

  input:log_metric_distribution_buckets: &log_metric_distribution_buckets
    linear_buckets:
      num_finite_buckets: 50
      width: 1000
      offset: 0

  input:monitoring:
    notification_channel_id: projects/{project}/notificationChannels/{channel_id}

    log_based_metrics:
      - name: my_metric_name
        description: Description of what this metric tracks
        filter: |
          resource.type="cloud_run_job"
          jsonPayload.event="Some event"
        value_extractor: EXTRACT(jsonPayload.payload.some_field)
        metric_descriptor: *log_metric_distribution_descriptor
        bucket_options: *log_metric_distribution_buckets

    alert_policies:
      - name: my-alert-policy
        display_name: Human Readable Alert Name
        severity: CRITICAL  # or WARNING, ERROR
        combiner: OR
        documentation: |
          Markdown documentation shown in alert notifications.
        conditions:
          - display_name: Condition Name
            condition_threshold:
              filter: metric.type="..." resource.type="..."
              comparison: COMPARISON_GT
              threshold_value: 0
              duration: 0s
              aggregations:
                - alignment_period: 60s
                  per_series_aligner: ALIGN_COUNT
        alert_strategy:
          auto_close: 604800s  # 7 days

    dashboard:
      display_name: "My Dashboard Name"
      dashboard_json_file: monitoring/dashboard.json
```

## Python Implementation

### Log-Based Metrics

```python
monitoring_config = input_config.get_object("monitoring")
if not monitoring_config:
    return  # No monitoring configured

log_based_metrics = monitoring_config.get("log_based_metrics")
if log_based_metrics:
    for metric_config in log_based_metrics:
        config = metric_config.copy()
        name = config.pop("name")

        # Strip whitespace from multiline YAML filters
        config["filter"] = config["filter"].strip()

        gcp.logging.Metric(
            resource_name=make_resource_name("logmetric", name),
            name=name,
            **config,
        )
```

### Alert Policies

```python
notification_channel_id = monitoring_config["notification_channel_id"]

alert_policies = monitoring_config.get("alert_policies")
if alert_policies:
    for alert_config in alert_policies:
        config = alert_config.copy()
        name = config.pop("name")

        # Transform documentation string to Args format
        if "documentation" in config:
            config["documentation"] = gcp.monitoring.AlertPolicyDocumentationArgs(
                content=config["documentation"],
                mime_type="text/markdown",
            )

        # Add notification channel
        config["notification_channels"] = [notification_channel_id]

        gcp.monitoring.AlertPolicy(
            resource_name=make_resource_name("alertpolicy", name),
            **config,
        )
```

### Dashboard

```python
dashboard_config = monitoring_config.get("dashboard")
if dashboard_config:
    dashboard_json_file = dashboard_config["dashboard_json_file"]
    dashboard_path = Path(__file__).resolve().parent / dashboard_json_file

    if not dashboard_path.exists():
        raise FileNotFoundError(f"Dashboard JSON file not found: {dashboard_json_file}")

    dashboard_json = dashboard_path.read_text()

    # Optionally override display name from config
    dashboard_data = json.loads(dashboard_json)
    if "display_name" in dashboard_config:
        dashboard_data["displayName"] = dashboard_config["display_name"]

    gcp.monitoring.Dashboard(
        resource_name=make_resource_name("dashboard", "my-dashboard"),
        dashboard_json=json.dumps(dashboard_data),
    )
```

## Log-Based Metric Types

| Type | Use Case | Config |
|------|----------|--------|
| Counter | Count occurrences | No `value_extractor`, no `metric_descriptor` |
| Distribution | Extract numeric values | `value_extractor` + `metric_descriptor` + `bucket_options` |

### Counter Metric Example

```yaml
- name: job_executions_count
  description: Count of job executions
  filter: |
    resource.type="cloud_run_job"
    jsonPayload.event="Job completed"
```

### Distribution Metric Example

```yaml
- name: rows_loaded_distribution
  description: Distribution of rows loaded per execution
  filter: |
    resource.type="cloud_run_job"
    jsonPayload.event="Data loaded"
  value_extractor: EXTRACT(jsonPayload.payload.row_count)
  metric_descriptor:
    metric_kind: DELTA
    value_type: DISTRIBUTION
    unit: "1"
  bucket_options:
    linear_buckets:
      num_finite_buckets: 50
      width: 1000
      offset: 0
```

## Alert Policy Condition Types

| Type | Use Case | Config Key |
|------|----------|------------|
| Metric Threshold | Numeric metric exceeds threshold | `condition_threshold` |
| Log Match | Log entries match filter | `condition_matched_log` |
| Absence | Metric stops reporting | `condition_absent` |

### Metric Threshold Example

```yaml
conditions:
  - display_name: Job Failed
    condition_threshold:
      filter: metric.type="run.googleapis.com/job/completed_execution_count" metric.labels.result="failed"
      comparison: COMPARISON_GT
      threshold_value: 0
      duration: 0s
      aggregations:
        - alignment_period: 60s
          per_series_aligner: ALIGN_COUNT
```

### Log Match Example

```yaml
conditions:
  - display_name: Error Logs Detected
    condition_matched_log:
      filter: |
        resource.type="cloud_run_job"
        severity="ERROR"
```

## Alert Strategy Options

```yaml
alert_strategy:
  auto_close: 604800s  # Auto-close after 7 days (in seconds)
  notification_rate_limit:
    period: 300s  # Max one notification per 5 minutes
```

## Dashboard JSON

Create dashboard JSON at `monitoring/dashboard.json`. Export from Cloud Console or build programmatically.

Key structure:

```json
{
  "displayName": "Dashboard Name",
  "mosaicLayout": {
    "columns": 48,
    "tiles": [
      {
        "width": 24,
        "height": 16,
        "widget": {
          "title": "Widget Title",
          "xyChart": { ... }
        }
      }
    ]
  }
}
```

## Common Log Filters

| Resource | Filter |
|----------|--------|
| Cloud Run Job | `resource.type="cloud_run_job"` |
| Cloud Run Service | `resource.type="cloud_run_revision"` |
| Cloud Function | `resource.type="cloud_function"` |
| GKE Container | `resource.type="k8s_container"` |

## Notification Channels

Get channel ID from Cloud Console or CLI:

```bash
gcloud beta monitoring channels list --project={project}
```

Common channel types: Email, Slack, PagerDuty, Pub/Sub.
