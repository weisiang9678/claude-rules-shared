# Pub/Sub Topics, Subscriptions, and IAM

## Overview

Pub/Sub resources are provisioned in a specific order to handle dependencies:
1. Schemas (optional)
2. Topics
3. Topic IAM bindings
4. Pull subscriptions
5. Pull subscription IAM bindings
6. Push subscriptions (with automatic dead letter IAM)

---

## Pub/Sub Topics

### Basic Topic

```yaml
input:projects:
  app-cdc-stream:
    pubsub_topics:
      - name: app-cdc-stream
        message_retention_duration: 604800s
      - name: app-cdc-dlq
        message_retention_duration: 604800s
```

### Topic with Schema

```yaml
pubsub_topics:
  - name: app-metadata-stream
    message_retention_duration: 604800s
    schema:
      definition_path: ./config/schema.proto
      name: app_metadata_stream_schema
      type: PROTOCOL_BUFFER
```

### Python Implementation

```python
# Schema creation (if defined)
if schema_config := topic_config.get("schema"):
    schema_path = os.path.join(pulumi_dir, schema_config["definition_path"])
    with open(schema_path, "r") as f:
        definition = f.read()

    created_pb_schema = gcp.pubsub.Schema(
        resource_name=make_resource_name("pubsubschema", schema_config["name"]),
        name=schema_config["name"],
        definition=definition,
        type=schema_config["type"],
    )

# Topic creation
schema_settings = None
if schema_config := topic_config.get("schema"):
    schema_settings = {
        "schema": pb_schema.id,
        "encoding": "JSON",
    }

created_topic = gcp.pubsub.Topic(
    resource_name=make_resource_name("topic", topic_name),
    name=topic_name,
    message_retention_duration=topic_config["message_retention_duration"],
    schema_settings=schema_settings,
    labels=labels,
    project=gcp_project,
    opts=pulumi.ResourceOptions(depends_on=depends_on_topics if depends_on_topics else None),
)
```

---

## Topic IAM Bindings

### Stack Configuration

```yaml
pubsub_topics_iam:
  - topic: app-cdc-stream
    role: "roles/pubsub.publisher"
    service_accounts:
      - de-interceptor-postgres@{project}.iam.gserviceaccount.com
  - topic: app-cdc-dlq
    role: "roles/pubsub.publisher"
    service_accounts:
      - service-{GCP_PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com
      - dataflow-worker-app-cdc-stream@{project}.iam.gserviceaccount.com
```

### Python Implementation

```python
for iam_binding in topics_iam_config:
    topic_name = iam_binding["topic"]
    role = iam_binding["role"]
    topic_resource = project_topics.get(topic_name)

    for svc_acct in iam_binding["service_accounts"]:
        svc_acct_formatted = svc_acct.format(GCP_PROJECT_NUMBER=GCP_PROJECT_NUMBER)

        gcp.pubsub.TopicIAMMember(
            resource_name=make_resource_name("topiciammember", topic_name, svc_acct_formatted),
            topic=topic_name,
            role=role,
            member=f"serviceAccount:{svc_acct_formatted}",
            opts=pulumi.ResourceOptions(
                depends_on=[topic_resource] if topic_resource else None
            ),
        )
```

---

## Pull Subscriptions

### Multi-Region with Template Placeholders

```yaml
pull_subscriptions:
  # Multi-region with template placeholders
  - name: app-cdc-stream-sub-{REGION_ABBREV}
    topic: app-cdc-stream
    ack_deadline_seconds: 10
    retain_acked_messages: false
    expiration_policy:
      ttl: ""
    filter: 'attributes.aws_region_abbr="{REGION_ABBREV}" AND NOT hasPrefix(attributes.table, "mv_")'
    locations:
      - asia-southeast1  # Expands to REGION_ABBREV=sg
      - europe-west3     # Expands to REGION_ABBREV=de

  # Global (no templating)
  - name: app-cdc-dlq-sub
    topic: app-cdc-dlq
    ack_deadline_seconds: 10
    retain_acked_messages: true
    expiration_policy:
      ttl: ""
    locations:
      - global
```

### Python Implementation

```python
for subscription_config in subscriptions_config:
    template_name = subscription_config["name"]
    topic_name = subscription_config["topic"]
    topic_resource = project_topics.get(topic_name)
    locations = subscription_config["locations"]

    for location in locations:
        # Global location means no templating
        if location == "global":
            subscription_name = template_name
            subscription_filter = subscription_config.get("filter")
        else:
            region_abbrev = GCPLocationAbbreviation.get_abbr(location).value
            subscription_name = template_name.format(REGION_ABBREV=region_abbrev)
            template_filter = subscription_config.get("filter")
            subscription_filter = (
                template_filter.format(REGION_ABBREV=region_abbrev)
                if template_filter
                else None
            )

        created_subscription = gcp.pubsub.Subscription(
            resource_name=make_resource_name("subscription", subscription_name),
            name=subscription_name,
            project=gcp_project,
            topic=topic_name,
            ack_deadline_seconds=subscription_config["ack_deadline_seconds"],
            retain_acked_messages=subscription_config["retain_acked_messages"],
            expiration_policy=subscription_config["expiration_policy"],
            filter=subscription_filter,
            labels=labels,
            opts=pulumi.ResourceOptions(
                depends_on=[topic_resource] if topic_resource else None
            ),
        )
```

---

## Pull Subscription IAM

### Stack Configuration

```yaml
pull_subscriptions_iam:
  - subscription: app-cdc-stream-sub-{REGION_ABBREV}
    role: "roles/pubsub.subscriber"
    service_accounts:
      - dataflow-worker-app-cdc-stream@{project}.iam.gserviceaccount.com
    locations:
      - asia-southeast1
      - europe-west3
  - subscription: app-cdc-dlq-sub
    role: "roles/pubsub.subscriber"
    service_accounts:
      - de-team@{project}.iam.gserviceaccount.com
    locations:
      - global
```

### Python Implementation

```python
for iam_binding in subscriptions_iam_config:
    template_subscription = iam_binding["subscription"]
    role = iam_binding["role"]
    locations = iam_binding["locations"]

    for location in locations:
        # Global location means no templating
        if location == "global":
            subscription_name = template_subscription
        else:
            region_abbrev = GCPLocationAbbreviation.get_abbr(location).value
            subscription_name = template_subscription.format(REGION_ABBREV=region_abbrev)

        subscription_resource = project_subscriptions.get(subscription_name)

        for svc_acct in iam_binding["service_accounts"]:
            svc_acct_formatted = svc_acct.format(GCP_PROJECT_NUMBER=GCP_PROJECT_NUMBER)
            gcp.pubsub.SubscriptionIAMMember(
                resource_name=make_resource_name(
                    "subscriptioniammember", subscription_name, svc_acct_formatted
                ),
                subscription=subscription_name,
                role=role,
                member=f"serviceAccount:{svc_acct_formatted}",
                opts=pulumi.ResourceOptions(
                    depends_on=[subscription_resource] if subscription_resource else None
                ),
            )
```

---

## Push Subscriptions

### Stack Configuration

```yaml
push_subscriptions:
  - name: app-description-stream-sub-{REGION_ABBREV}
    topic: app-metadata-stream
    ack_deadline_seconds: 120
    retain_acked_messages: false
    message_retention_duration: "604800s"
    enable_message_ordering: false
    filter: 'attributes.region="{REGION}" AND attributes.metadata_type="table_description"'
    expiration_policy:
      ttl: ""
    locations:
      - asia-southeast1
    push_config:
      push_endpoint: https://propagate-description-{GCP_PROJECT_NUMBER}.{REGION}.run.app
      oidc_token:
        service_account_email: cloud-func-app-metadata-stream@{project}.iam.gserviceaccount.com
    dead_letter_policy:
      dead_letter_topic: app-metadata-dlq
      max_delivery_attempts: 5
```

### Python Implementation

```python
for subscription_config in push_subscriptions_config:
    template_name = subscription_config["name"]
    topic_name = subscription_config["topic"]
    topic_resource = project_topics.get(topic_name)
    locations = subscription_config["locations"]

    dlp_config = subscription_config["dead_letter_policy"]

    # Build depends_on list with topic and dead letter topic
    depends_on_resources: list[gcp.pubsub.Topic] = []
    if topic_resource:
        depends_on_resources.append(topic_resource)
    dead_letter_topic_resource = project_topics.get(dlp_config["dead_letter_topic"])
    if dead_letter_topic_resource:
        depends_on_resources.append(dead_letter_topic_resource)

    for location in locations:
        # Global location means no templating
        if location == "global":
            subscription_name = template_name
            subscription_filter = subscription_config.get("filter")
            push_endpoint = subscription_config["push_config"]["push_endpoint"].format(
                GCP_PROJECT_NUMBER=GCP_PROJECT_NUMBER
            )
        else:
            region_abbrev = GCPLocationAbbreviation.get_abbr(location).value
            subscription_name = template_name.format(REGION_ABBREV=region_abbrev)
            template_filter = subscription_config.get("filter")
            subscription_filter = (
                template_filter.format(REGION=location, REGION_ABBREV=region_abbrev)
                if template_filter
                else None
            )
            push_endpoint = subscription_config["push_config"]["push_endpoint"].format(
                GCP_PROJECT_NUMBER=GCP_PROJECT_NUMBER,
                REGION=location,
                REGION_ABBREV=region_abbrev,
            )

        formatted_push_config = {
            "push_endpoint": push_endpoint,
            "oidc_token": subscription_config["push_config"]["oidc_token"],
        }

        formatted_dlp = {
            "dead_letter_topic": f"projects/{gcp_project}/topics/{dlp_config['dead_letter_topic']}",
            "max_delivery_attempts": dlp_config["max_delivery_attempts"],
        }

        created_push_subscription = gcp.pubsub.Subscription(
            resource_name=make_resource_name("pushsubscription", subscription_name),
            name=subscription_name,
            project=gcp_project,
            topic=topic_name,
            ack_deadline_seconds=subscription_config["ack_deadline_seconds"],
            retain_acked_messages=subscription_config["retain_acked_messages"],
            message_retention_duration=subscription_config["message_retention_duration"],
            enable_message_ordering=subscription_config["enable_message_ordering"],
            filter=subscription_filter,
            expiration_policy=subscription_config["expiration_policy"],
            push_config=formatted_push_config,
            dead_letter_policy=formatted_dlp,
            labels=labels,
            opts=pulumi.ResourceOptions(
                depends_on=depends_on_resources if depends_on_resources else None
            ),
        )

        # Grant subscriber role to default GCP Pub/Sub service account for dead letter
        gcp.pubsub.SubscriptionIAMMember(
            resource_name=make_resource_name("dlqsubsiammember", subscription_name),
            member=f"serviceAccount:service-{GCP_PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com",
            role="roles/pubsub.subscriber",
            subscription=created_push_subscription.name,
        )
```

---

## Template Placeholders

| Placeholder | Source | Example Value |
|-------------|--------|---------------|
| `{REGION_ABBREV}` | Derived from `GCPLocationAbbreviation` enum | `sg`, `de` |
| `{REGION}` | Full region name from `locations` list | `asia-southeast1`, `europe-west3` |
| `{GCP_PROJECT_NUMBER}` | Environment variable | `123456789` |

### Location Abbreviation Mapping

Defined in `helpers/locations.py`:

```python
class GCPLocationAbbreviation(Enum):
    asia_southeast1 = "sg"
    europe_west3 = "de"

    @classmethod
    def get_abbr(cls, region):
        region_mapping = {
            "asia-southeast1": cls.asia_southeast1,
            "europe-west3": cls.europe_west3,
        }
        return region_mapping[region]
```

---

## Common Pub/Sub Roles

| Role | Purpose |
|------|---------|
| `roles/pubsub.publisher` | Publish messages to topics |
| `roles/pubsub.subscriber` | Subscribe and consume messages |
| `roles/pubsub.viewer` | View topics and subscriptions |

---

## YAML Anchors for Reuse

Use YAML anchors to avoid duplication in stack files:

```yaml
push_subscriptions:
  - name: app-description-stream-sub-{REGION_ABBREV}
    dead_letter_policy: &dead_letter_policy
      dead_letter_topic: app-metadata-dlq
      max_delivery_attempts: 5
    # ... other config

  - name: app-constraint-stream-sub-{REGION_ABBREV}
    dead_letter_policy: *dead_letter_policy  # Reuse anchor
    # ... other config
```

---

## Directory Structure

```
infrastructure/pulumi/
├── __main__.py
├── config/
│   └── schema.proto        # Protocol Buffer schema definitions
└── helpers/
    └── locations.py        # GCPLocationAbbreviation enum
```
