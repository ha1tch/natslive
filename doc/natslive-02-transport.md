# Transport Layer In-Depth

## Core Transport Philosophy

Natslive's transport architecture is built on the principle of **protocol-native delivery** - events should flow through their natural protocols whenever possible, minimizing translation overhead and preserving protocol-specific features. This approach enables Natslive to serve as a lightweight "routing fabric" connecting diverse messaging ecosystems.

## Transport Categories

Natslive's transport system is organized into three functional categories:

### 1. Source Transports

Source transports are responsible for ingesting events into the Natslive routing system.

| Transport | Status | Use Cases | Features |
|-----------|--------|-----------|----------|
| **NATS** | Core | Microservices, internal event flows | Subject wildcards, queue groups, request-reply patterns |
| **HTTP/Webhooks** | Standard | External API integrations, webhook receivers | HTTP/S endpoints, header authentication, payload validation |
| **Kafka** | Plugin | Enterprise event streams, high-volume analytics | Consumer groups, offset management, schema registry integration |
| **MQTT** | Plugin | IoT device telemetry, sensor networks | QoS levels, retained messages, LWT handling |
| **AWS SQS** | Plugin | AWS integration, managed queue processing | Dead-letter queues, FIFO support, visibility timeout management |
| **gRPC** | Experimental | Service mesh integration, strong typing | Stream and unary requests, bidirectional capabilities |

### 2. Internal Transport

The internal transport is responsible for moving events between Natslive components and handling rule evaluation.

* **NATS**: All internal communication between Natslive components uses NATS, leveraging its lightweight and scalable design.
* **JetStream**: Used for guaranteed delivery and persistence when required.

### 3. Sink Transports

Sink transports deliver matched events to their final destinations.

| Transport | Status | Delivery Patterns | Features |
|-----------|--------|-------------------|----------|
| **NATS** | Core | Pub/Sub, Request/Reply | Direct republish, subject templating, metadata preservation |
| **HTTP/Webhooks** | Standard | POST, custom methods | Retry policies, batch delivery, authentication |
| **Kafka** | Plugin | Topic production | Partitioning strategies, header mapping, schema enforcement |
| **AWS SNS/SQS/EventBridge** | Plugin | Managed AWS targets | Native AWS authentication, cross-region delivery |
| **Redis** | Plugin | Pub/Sub, Streams | Channel mapping, RESP protocol optimizations |
| **WebSockets** | Experimental | Real-time clients | Connection pools, client session management |

## Transport Protocol Bridging

Natslive provides native protocol bridging to translate between different transport protocols while preserving semantic integrity. This involves:

### Message Format Translation

* **Envelope Mapping**: Maps protocol-specific message envelopes to a common internal format
* **Header Preservation**: Transforms transport-specific headers and metadata between protocols
* **Content Negotiation**: Handles content-type conversion (JSON, Avro, Protobuf, etc.)

### Protocol-Specific Features

* **Delivery Guarantees**: Maps QoS levels across protocols (e.g., MQTT QoS 2 → Kafka acks=all)
* **Ordering Semantics**: Preserves or adapts message ordering guarantees
* **Idempotency Controls**: Manages duplicate detection when crossing protocol boundaries

## Transport Configuration

Each transport is configured through Natslive's unified configuration system, which can be provided via:

1. **Static Configuration**: YAML/JSON files loaded at startup
2. **Dynamic Configuration**: Runtime updates via NATS control subjects
3. **Kubernetes CRDs**: When running in Kubernetes environments

### Example Transport Configuration

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

## Transport Security

Natslive implements comprehensive security controls across all transport types:

* **Authentication**: Protocol-native auth mechanisms with credential management
* **Authorization**: Subject/topic/endpoint-level permissions
* **Encryption**: TLS/SSL for all supported transports
* **Credential Management**: Integration with secret stores (Kubernetes Secrets, Vault, etc.)

## Scaling Transport Layers

Natslive's transport architecture is designed for horizontal scaling:

* **Stateless Operation**: Transport handlers maintain minimal state for maximum scalability
* **Connection Pooling**: Manages outbound connections efficiently across instances
* **Load Distribution**: Uses NATS queue groups to distribute incoming message load
* **Backpressure Handling**: Implements adaptive rate limiting and backpressure propagation

## Deployment Topologies

### Edge-to-Cloud

For IoT and edge scenarios, Natslive can be deployed in a hierarchical pattern:

```
[Edge Devices] → [MQTT Broker] → [Local Natslive] → [NATS] → [Cloud Natslive] → [Cloud Services]
```

This topology allows filtering and aggregation at the edge while forwarding selected events to cloud environments.

### Multi-Region Global Routing

For global applications, Natslive can leverage NATS's super-cluster capabilities:

```
[Region 1 Services] → [Region 1 Natslive] → [NATS Global Mesh] → [Region N Natslive] → [Region N Services]
```

This enables efficient cross-region event routing with minimal latency.

### Hybrid Cloud Integration

For hybrid scenarios, Natslive bridges on-premise and cloud environments:

```
[On-Premise Apps] → [Natslive] → [NATS Gateway] → [Cloud Natslive] → [AWS EventBridge] → [AWS Services]
```

## Transport Observability

Each transport integration includes comprehensive observability:

* **Metrics**: Transport-specific throughput, latency, and error metrics
* **Logs**: Structured logging of transport operations
* **Traces**: OpenTelemetry integration for cross-transport tracing
* **Health Checks**: Protocol-native health probes for monitoring systems

## Transport Plugin Development

Natslive provides a Transport Plugin SDK to extend support for additional protocols:

```go
// Example Transport Plugin Interface
type TransportSink interface {
    Initialize(config TransportConfig) error
    Deliver(ctx context.Context, event *Event) error
    Close() error
}

type TransportSource interface {
    Initialize(config TransportConfig) error
    Start(ctx context.Context, eventCh chan<- *Event) error
    Stop() error
}
```

The plugin system provides common facilities for:
* Connection management
* Authentication
* Retry logic
* Serialization/deserialization
* Monitoring

## Transport Selection Guidelines

When implementing a Natslive-based architecture, consider these factors for transport selection:

1. **Existing Infrastructure**: Start with transports already in your ecosystem
2. **Delivery Requirements**: Match protocols to QoS and reliability needs
3. **Scalability Needs**: Consider throughput and partitioning capabilities
4. **Cloud Strategy**: Align with your hybrid/multi-cloud approach
5. **Client Ecosystem**: Choose transports with strong client library support

## Future Transport Roadmap

The Natslive transport roadmap includes:

* **AMQP Support**: For RabbitMQ and other AMQP broker integration
* **WebRTC Data Channels**: For peer-to-peer event distribution
* **Apache Pulsar**: For unified streaming and messaging
* **Transport Adapters for Cloud PaaS**: Native integrations for Google Pub/Sub, Azure Event Hub, etc.
* **On-demand Transport Loading**: Dynamic loading of transport plugins based on configuration