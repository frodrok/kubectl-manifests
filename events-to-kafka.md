There are three main approaches, and they can be combined. Since YouTrack workflows can make HTTP calls but can't speak Kafka's binary protocol, you need a bridge layer. The cleanest option with your existing Strimzi setup is the **Strimzi HTTP Bridge**.

## Architecture overview

```
Approach A (recommended):
  YouTrack workflow (onChange) → HTTP POST → Strimzi Kafka Bridge → Kafka topic

Approach B:
  YouTrack workflow (onChange) → HTTP POST → Custom bridge microservice → Kafka

Approach C:
  External poller → YouTrack REST API (activities endpoint) → Kafka
```

**Approach A is the path of least resistance** — Strimzi ships a Kafka HTTP Bridge component that accepts plain JSON over HTTP and produces to Kafka topics. YouTrack workflows already have an `http` module that can POST JSON. No custom code to deploy or maintain.

Here's how to wire it up:

### 1. Add the Strimzi Kafka Bridge to your cluster

```yaml
# Add to 10-strimzi-kafka.yaml or create a new file
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaBridge
metadata:
  name: youtrack-bridge
  namespace: youtrack-dev
spec:
  replicas: 1
  bootstrapServers: youtrack-kafka-kafka-bootstrap:9092
  http:
    port: 8080
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
      cpu: 500m
```

This gives you a service at `youtrack-bridge-bridge-service.youtrack-dev.svc.cluster.local:8080` that accepts HTTP and produces to Kafka.

### 2. Create a YouTrack workflow that POSTs to the bridge

In YouTrack, go to **Administration → Workflows → New workflow** (e.g. `kafka-integration`), then add an **On-change** module:

```javascript
const entities = require('@jetbrains/youtrack-scripting-api/entities');
const http = require('@jetbrains/youtrack-scripting-api/http');

const KAFKA_BRIDGE_URL = 'http://youtrack-bridge-bridge-service.youtrack-dev.svc.cluster.local:8080';
const TOPIC = 'youtrack-events';

exports.rule = entities.Issue.onChange({
  title: 'Publish issue changes to Kafka',
  guard: (ctx) => {
    const issue = ctx.issue;
    // Fire on create, resolve, reopen, assignment change, or state change
    return issue.becomesReported ||
           issue.becomesResolved ||
           issue.becomesUnresolved ||
           issue.fields.isChanged(ctx.State) ||
           issue.fields.isChanged(ctx.Assignee);
  },
  action: (ctx) => {
    const issue = ctx.issue;

    // Determine what changed
    let eventType = 'UPDATED';
    if (issue.becomesReported) eventType = 'CREATED';
    else if (issue.becomesResolved) eventType = 'RESOLVED';
    else if (issue.becomesUnresolved) eventType = 'REOPENED';

    const payload = {
      records: [{
        key: issue.id,
        value: {
          eventType: eventType,
          issueId: issue.id,
          summary: issue.summary,
          description: issue.description ? issue.description.substring(0, 500) : null,
          project: issue.project.name,
          reporter: issue.reporter ? issue.reporter.login : null,
          assignee: issue.fields.Assignee ? issue.fields.Assignee.login : null,
          state: issue.fields.State ? issue.fields.State.name : null,
          priority: issue.fields.Priority ? issue.fields.Priority.name : null,
          url: issue.url,
          timestamp: Date.now()
        }
      }]
    };

    const connection = new http.Connection(KAFKA_BRIDGE_URL, null, 5000);
    connection.addHeader('Content-Type', 'application/vnd.kafka.json.v2+json');

    const response = connection.postSync(
      '/topics/' + TOPIC,
      null,
      JSON.stringify(payload)
    );

    if (!response.isSuccess) {
      console.error('Failed to publish to Kafka: ' + response.code + ' ' + response.response);
    }
  },
  requirements: {
    State: { type: entities.State.fieldType },
    Assignee: { type: entities.User.fieldType },
    Priority: { type: entities.EnumField.fieldType }
  }
});
```

Then attach the workflow to each project you want to emit events from.

### 3. Add more event modules as needed

You can add additional modules to the same workflow for **comments**, **work items**, and **scheduled checks**:

**Comment events** (add as a separate On-change module):

```javascript
const entities = require('@jetbrains/youtrack-scripting-api/entities');
const http = require('@jetbrains/youtrack-scripting-api/http');

const KAFKA_BRIDGE_URL = 'http://youtrack-bridge-bridge-service.youtrack-dev.svc.cluster.local:8080';
const TOPIC = 'youtrack-events';

exports.rule = entities.Issue.onChange({
  title: 'Publish comment events to Kafka',
  guard: (ctx) => {
    return ctx.issue.comments.added.isNotEmpty();
  },
  action: (ctx) => {
    const issue = ctx.issue;
    issue.comments.added.forEach((comment) => {
      const payload = {
        records: [{
          key: issue.id,
          value: {
            eventType: 'COMMENT_ADDED',
            issueId: issue.id,
            summary: issue.summary,
            commentAuthor: comment.author ? comment.author.login : null,
            commentText: comment.text ? comment.text.substring(0, 500) : null,
            url: issue.url,
            timestamp: Date.now()
          }
        }]
      };

      const connection = new http.Connection(KAFKA_BRIDGE_URL, null, 5000);
      connection.addHeader('Content-Type', 'application/vnd.kafka.json.v2+json');
      connection.postSync('/topics/' + TOPIC, null, JSON.stringify(payload));
    });
  },
  requirements: {}
});
```

### Quick summary

| Piece | What it does |
|---|---|
| **Strimzi KafkaBridge** | Accepts `POST /topics/{topic}` with JSON, produces to Kafka |
| **YouTrack onChange workflow** | Fires on issue events, POSTs JSON to the bridge |
| **KafkaTopic** | `youtrack-events` — holds all the events |
| **Consumers** | Anything downstream reads from the topic (analytics, sync, notifications) |

The bridge URL format for producing is `POST /topics/{topic}` with content type `application/vnd.kafka.json.v2+json` and a body of `{"records": [{"key": "...", "value": {...}}]}`. That's the Strimzi/Confluent HTTP Bridge standard.
