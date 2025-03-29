# Natslive: Lightweight Event Routing on NATS

## Overview
**Natslive** is a lightweight, cloud-agnostic, event-driven routing layer built on top of [NATS](https://nats.io). Inspired by AWS EventBridge and Knative Eventing, Natslive delivers dynamic, rule-based event routing with a minimalist architecture ideal for hybrid and local-first cloud environments.

## Key Features
- **Dynamic Routing**: Send routing rules via NATS messages.
- **Advanced Content Filtering**: Filter events with a powerful rule expression language or simple pattern matching.
- **NATS-native Transport**: No HTTP required; all messages stay within NATS.
- **Comprehensive Transport Support**: Connect to multiple messaging systems and protocols.
- **Resource-optimized Binaries**: Purpose-built bundles for different deployment environments.
- **Hybrid-Cloud Ready**: Deployable in local, on-prem, cloud, and mixed environments.
- **CloudEvents-compatible**: Optional support for CloudEvents for interop.

## Core Concepts

### Event Producers
Applications or services that publish events to NATS subjects such as:
```
events.user.created
events.order.completed
```

### Natslive Router
A NATS-connected service that:
- Listens for rule configuration messages.
- Subscribes to event subjects (e.g., `events.>`).
- Applies filtering logic to route events.
- Republishes matching events to target destinations.

### Routing Rules
Sent dynamically via NATS subjects (e.g., `natslive.config.add.route`).

#### Example Rule with Simple Filter (JSON)
```json
{
  "route_id": "route-user-eu",
  "match": "events.user.*",
  "filter": {
    "payload.region": "eu"
  },
  "forward_to": [
    { "type": "nats", "subject": "service.analytics" },
    { "type": "http", "url": "https://webhook.site/user-notify" }
  ]
}
```

#### Example Rule with Expression Filter (JSON)
```json
{
  "route_id": "premium-eu-customers",
  "match": "events.user.*",
  "filter_expr": "payload.region == 'eu' AND payload.subscription == 'premium' AND payload.lastLogin > now() - duration('7d')",
  "forward_to": [
    { "type": "nats", "subject": "service.vip-support" }
  ]
}
```

### Event Consumers
Any service that subscribes to the resulting NATS subject (e.g., `service.analytics`) or listens for HTTP POSTs via configured sinks.

## Transport Layer In-Depth

### Core Transport Philosophy

Natslive's transport architecture is built on the principle of **protocol-native delivery** - events should flow through their natural protocols whenever possible, minimizing translation overhead and preserving protocol-specific features.

### Transport Categories

Natslive's transport system is organized into three functional categories:

#### 1. Source Transports

Source transports ingest events into the Natslive routing system.

| Transport | Status | Use Cases | Features |
|-----------|--------|-----------|----------|
| **NATS** | Core | Microservices, internal event flows | Subject wildcards, queue groups, request-reply patterns |
| **HTTP/Webhooks** | Standard | External API integrations, webhook receivers | HTTP/S endpoints, header authentication, payload validation |
| **Kafka** | Standard | Enterprise event streams, high-volume analytics | Consumer groups, offset management, schema registry integration |
| **MQTT** | Standard | IoT device telemetry, sensor networks | QoS levels, retained messages, LWT handling |
| **AWS SQS** | Standard | AWS integration, managed queue processing | Dead-letter queues, FIFO support, visibility timeout management |
| **gRPC** | Experimental | Service mesh integration, strong typing | Stream and unary requests, bidirectional capabilities |

#### 2. Internal Transport

The internal transport moves events between Natslive components and handles rule evaluation.

* **NATS**: All internal communication between Natslive components uses NATS.
* **JetStream**: Used for guaranteed delivery and persistence when required.

#### 3. Sink Transports

Sink transports deliver matched events to their final destinations.

| Transport | Status | Delivery Patterns | Features |
|-----------|--------|-------------------|----------|
| **NATS** | Core | Pub/Sub, Request/Reply | Direct republish, subject templating, metadata preservation |
| **HTTP/Webhooks** | Standard | POST, custom methods | Retry policies, batch delivery, authentication |
| **Kafka** | Standard | Topic production | Partitioning strategies, header mapping, schema enforcement |
| **AWS SNS/SQS/EventBridge** | Standard | Managed AWS targets | Native AWS authentication, cross-region delivery |
| **Redis** | Standard | Pub/Sub, Streams | Channel mapping, RESP protocol optimizations |
| **WebSockets** | Experimental | Real-time clients | Connection pools, client session management |

### Transport Protocol Bridging

Natslive provides native protocol bridging to translate between different transport protocols while preserving semantic integrity:

- **Envelope Mapping**: Maps protocol-specific message envelopes to a common internal format
- **Header Preservation**: Transforms transport-specific headers and metadata between protocols
- **Content Negotiation**: Handles content-type conversion (JSON, Avro, Protobuf, etc.)
- **Delivery Guarantees**: Maps QoS levels across protocols (e.g., MQTT QoS 2 → Kafka acks=all)
- **Ordering Semantics**: Preserves or adapts message ordering guarantees
- **Idempotency Controls**: Manages duplicate detection when crossing protocol boundaries

### Transport Configuration Example

```yaml
sources:
  - name: internal-events
    type: nats
    subjects: ["events.>"]
    
  - name: external-webhooks
    type: http
    bind_address: "0.0.0.0:8080"
    endpoints:
      - path: "/hooks/github"
        forward_subject: "events.github"
        authentication:
          type: hmac-signature
          header: "X-Hub-Signature-256"
          secret_ref: "github-webhook-secret"

sinks:
  - name: analytics-kafka
    type: kafka
    brokers: ["kafka-1:9092", "kafka-2:9092"]
    topic_template: "analytics-{{payload.event_type}}"
    authentication:
      type: sasl-scram
      username_ref: "kafka-user"
      password_ref: "kafka-password"
    
  - name: notification-service
    type: http
    endpoint: "https://notifications.example.com/api/events"
    method: "POST"
    headers:
      Content-Type: "application/cloudevents+json"
      Authorization: "Bearer {{secret.notification-api-key}}"
    retry:
      max_attempts: 5
      backoff_base_ms: 100
      backoff_max_ms: 5000
```

### Transport Security

Natslive implements comprehensive security controls across all transport types:

* **Authentication**: Protocol-native auth mechanisms with credential management
* **Authorization**: Subject/topic/endpoint-level permissions
* **Encryption**: TLS/SSL for all supported transports
* **Credential Management**: Integration with secret stores (Kubernetes Secrets, Vault, etc.)

### Deployment Topologies

#### Edge-to-Cloud

For IoT and edge scenarios, Natslive can be deployed in a hierarchical pattern:

```
[Edge Devices] → [MQTT Broker] → [Local Natslive] → [NATS] → [Cloud Natslive] → [Cloud Services]
```

#### Multi-Region Global Routing

For global applications, Natslive can leverage NATS's super-cluster capabilities:

```
[Region 1 Services] → [Region 1 Natslive] → [NATS Global Mesh] → [Region N Natslive] → [Region N Services]
```

#### Hybrid Cloud Integration

For hybrid scenarios, Natslive bridges on-premise and cloud environments:

```
[On-Premise Apps] → [Natslive] → [NATS Gateway] → [Cloud Natslive] → [AWS EventBridge] → [AWS Services]
```

## Natslive Rule Expression Language (NREL)

For advanced filtering scenarios, Natslive provides a powerful but lightweight rule expression language.

### Why Simple Matching Isn't Always Enough

While basic JSON path matching works for simple cases:

```json
{
  "filter": {
    "payload.region": "eu"
  }
}
```

Real-world event routing often requires more sophisticated filtering capabilities:

- Compound conditions with AND/OR/NOT logic
- Numeric comparisons and range checks
- String pattern matching with wildcards and regex
- Temporal conditions based on timestamps
- Set operations (membership, intersection)
- Mathematical expressions and transformations

### NREL: A Compiled Expression DSL

Natslive Rule Expression Language (NREL) is a lightweight, compiled expression language specifically designed for event filtering and routing:

```
# Simple equality (equivalent to the basic JSON matcher)
payload.region == "eu"

# Compound condition with AND
payload.region == "eu" AND payload.userType == "premium"

# Numeric comparison with OR
payload.amount > 1000 OR payload.priority == "high"

# Pattern matching with LIKE
payload.email LIKE "*@example.com" 

# Regular expression matching
payload.userId MATCHES "^[A-Z]{2}[0-9]{6}$"

# Range checking
payload.timestamp BETWEEN "2023-01-01T00:00:00Z" AND "2023-12-31T23:59:59Z"

# Set membership
payload.category IN ["urgent", "security", "compliance"]

# Existence check
EXISTS payload.metadata.debugInfo

# Nested conditions with parentheses
(payload.region == "eu" OR payload.region == "asia") AND payload.amount > 100
```

### Real-world Example: User Event Segmentation

```
# Route premium users from European region with recent high-value purchases
payload.user.tier == "premium" AND 
payload.user.region IN ["eu-west-1", "eu-central-1"] AND
EXISTS payload.purchases AND
any(payload.purchases, purchase -> 
  purchase.value > 1000 AND 
  timeSince(purchase.timestamp) < duration("30d")
)
```

### Performance and Safety

NREL is designed for high-performance event processing:

1. **Compiled Evaluation**: Pre-compiled expressions eliminate parsing overhead at runtime
2. **Short-circuit Evaluation**: AND/OR expressions stop evaluating once result is determined
3. **Memory Efficiency**: Minimal allocation during expression evaluation
4. **Safety Constraints**: No unbounded loops or side effects, with execution timeouts
5. **Benchmarked Performance**: Rule evaluation throughput of millions of events per second per core

## Deployment Options

Natslive follows a bundle-based deployment strategy optimized for different environments:

### Pre-compiled Bundles

| Bundle Name | Target Environment | NREL Support | Included Transports | Binary Size | Memory Footprint |
|-------------|-------------------|-------------|---------------------|-------------|------------------|
| `natslive-minimal` | Edge, IoT | No | NATS, HTTP | ~12MB | ~15MB RAM |
| `natslive-minimal-nrel` | Edge, IoT | Yes | NATS, HTTP | ~15MB | ~22MB RAM |
| `natslive-cloud` | Cloud Environments | No | NATS, HTTP, AWS, GCP, Azure | ~20MB | ~32MB RAM |
| `natslive-cloud-nrel` | Cloud Environments | Yes | NATS, HTTP, AWS, GCP, Azure | ~25MB | ~42MB RAM |
| `natslive-enterprise` | Data Centers, Hybrid Cloud | Yes | NATS, HTTP, Kafka, MQTT, AMQP, Redis | ~35MB | ~50MB RAM |
| `natslive-complete` | Development, All-purpose | Yes | All supported transports | ~50MB | ~70MB RAM |

### Custom Build Options

For teams requiring custom feature combinations, Natslive provides build-time configuration:

```bash
# Build with cloud provider transports and NREL
go build -tags "nats http aws gcp nrel" -o natslive-cloud-custom ./cmd/natslive

# Build with IoT focus, minimal footprint (no NREL)
go build -tags "nats mqtt" -o natslive-iot ./cmd/natslive
```

### Docker Images

Official Docker images follow a naming convention that explicitly indicates capabilities:
- `natslive/natslive-minimal:v1.0.0` (no NREL)
- `natslive/natslive-minimal-nrel:v1.0.0` (with NREL)
- `natslive/natslive-cloud:v1.0.0` (no NREL)
- `natslive/natslive-cloud-nrel:v1.0.0` (with NREL)
- `natslive/natslive-enterprise:v1.0.0` (with NREL by default)
- `natslive/natslive-complete:v1.0.0` (with NREL by default)

## Mapping to Cloud Event Systems

### Mapping EventBridge to Natslive

AWS EventBridge provides centralized event routing, filtering, and delivery across AWS services. Natslive offers a similar model but is self-hosted, NATS-native, and cloud-agnostic.

| EventBridge Concept      | Natslive Equivalent                 |
|--------------------------|-------------------------------------|
| Event Bus                | NATS subject hierarchy              |
| PutEvents API            | NATS publish                        |
| Event Rules              | Dynamic routing rules via NATS      |
| Targets (Lambda, SQS)    | NATS subjects, HTTP/webhooks        |
| Event Filtering (Pattern)| JSON-based filter or NREL expression|

### Mapping Knative Eventing to Natslive

Knative Eventing is a Kubernetes-native eventing framework. Natslive offers similar routing capabilities but in a much lighter, infrastructure-agnostic package.

| Knative Concept             | Natslive Equivalent                 |
|----------------------------|-------------------------------------|
| Broker                     | NATS subject wildcard (e.g., `events.>`) |
| Trigger                    | Dynamic routing rule                |
| Sink (Service, Broker)     | NATS subject, HTTP sink             |
| CloudEvents                | Optional CloudEvents support        |
| Event delivery (HTTP POST) | Native NATS messaging or HTTP sink  |

While Knative requires a Kubernetes cluster and typically relies on Istio or Kourier, Natslive runs as a lightweight binary or container and doesn't require a full service mesh or orchestrator.

## Hybrid and Self-Hosted Evolution

Natslive is ideal for evolving systems from local-first or self-hosted architectures into hybrid or fully cloud-native infrastructures:

1. **Local-First Start**: All services communicate over local NATS with routing handled by Natslive.
2. **Hybrid Mode**: Bridge events to/from cloud systems using HTTP sinks, NATS clusters, or specialized adapters.
3. **Cloud Migration**: Gradually introduce cloud services as event targets without rearchitecting the core event system.

This flexibility allows teams to:
- Avoid vendor lock-in.
- Comply with data residency or compliance requirements.
- Integrate with existing cloud-native stacks such as Knative or AWS.
- Test and iterate locally before pushing to production clouds.

## Use Cases
- Microservice event routing
- IoT and edge event filtering
- Local-first SaaS deployments
- Incremental cloud migration
- Decoupling components in hybrid systems
- Event-driven workflows across clouds

## Future Extensions

### Rule Expiration / TTL

Implementing rule expiration or Time-To-Live (TTL) in Natslive provides a way to manage routing rules dynamically and automatically remove those that are temporary or short-lived.

**Example Rule with TTL:**
```json
{
  "route_id": "temporary-debug-route",
  "match": "events.debug.*",
  "ttl": 3600,
  "forward_to": [
    { "type": "nats", "subject": "debug.console" }
  ]
}
```

### Rule Persistence

Persisting routing rules is essential for ensuring continuity in case of service restarts, failures, or scaling events. Natslive supports several storage backends:

1. **JetStream (NATS-native)**: Routing rules stored in a JetStream Key-Value bucket
2. **Redis**: Rules stored as hashes or JSON blobs with optional TTL
3. **In-memory KV stores**: For fast and lightweight environments

### OpenTelemetry Integration

Integrating OpenTelemetry into Natslive provides a standardized way to gain visibility into message flows:

- Distributed tracing of events across producers, routers, and consumers
- Insight into rule matching time, processing latency, and forwarding outcomes
- Metrics for events received, routed, and error rates
- Export to Prometheus, Jaeger, Zipkin, and commercial observability platforms

### Multi-tenant Routing

Isolate event flows per domain, customer, or context for secure multi-tenant deployments.

### Additional Extensions
- Web-based dashboard for managing and visualizing routing rules
- CLI for rule management and diagnostics

## Summary

Natslive provides a powerful yet minimal layer for event-driven architectures built on NATS. With dynamic routing, comprehensive transport options, flexible filtering through NREL, and deployment bundles tailored to different environments, it enables developers to build hybrid, local-first systems that can scale seamlessly into the cloud.