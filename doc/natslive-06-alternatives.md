# Natslive: Comparison with Alternatives

## What is Natslive?

**Natslive** is a lightweight, cloud-agnostic, event-routing layer built on top of [NATS](https://nats.io). It functions like a decentralized, programmable version of AWS EventBridge or Knative Eventing—but optimized for minimalism, performance, and flexibility. Its core goal is to provide dynamic, rule-based routing of events across various protocols and infrastructure types (cloud, edge, hybrid, on-premises) using NATS as the backbone.

## Key Differentiators

### Dynamic Routing
Publish routing rules (filters, destinations, etc.) dynamically via NATS messages—no restarts or redeployments needed.

### Advanced Filtering Options
- **NREL**: A high-performance, Go-compiled expression language for complex routing logic (temporal filters, numeric comparisons, regular expressions, etc.)
- **WREL**: A WebAssembly-based alternative that supports user-defined filtering logic in any WASM-compatible language (JavaScript, Rust, etc.) and executes securely in sandboxed environments.

### Multi-Protocol Support
Connect various systems (Kafka, MQTT, Redis, HTTP, WebSockets, AWS, gRPC, etc.) via source/sink plugins, with native protocol-bridging capabilities.

### Minimal & Modular Architecture
Run Natslive in a minimal 12MB binary for edge devices or scale it into full enterprise builds with extended transports and filtering capabilities.

### Zero-HTTP Core
Operate without requiring HTTP—every core operation happens inside NATS, which is ideal for low-latency or disconnected environments.

## Target Users

- **DevOps teams** looking for a self-hosted, flexible event router
- **IoT/Edge deployments** where lightweight and offline-friendly binaries matter
- **Hybrid cloud organizations** avoiding vendor lock-in but needing protocol bridging
- **Platform teams** building internal eventing platforms that need secure multi-tenant filtering (especially with WREL)

## Alternatives Comparison

### Cloud Provider Solutions

#### AWS EventBridge
- **Strengths**: Fully managed, tightly integrated with AWS services, low operational overhead
- **Limitations**: Vendor lock-in, limited filtering capabilities (pattern-only), no edge support
- **Best for**: Organizations fully committed to AWS ecosystem

#### Google Eventarc
- **Strengths**: Native GCP integration, serverless operation
- **Limitations**: GCP-specific, limited protocol support
- **Best for**: GCP-centric architectures 

#### Azure Event Grid
- **Strengths**: Tight Azure integration, managed service
- **Limitations**: Azure-bound, limited extensibility
- **Best for**: Microsoft cloud environments

### Kubernetes-Native Solutions

#### Knative Eventing
- **Strengths**: Kubernetes-native, strong cloud event support, good integration with K8s ecosystem
- **Limitations**: Requires Kubernetes (and often Istio), heavyweight, not edge-friendly
- **Best for**: Organizations standardized on Kubernetes for all workloads

### General-Purpose Alternatives

#### Temporal/Cadence
- **Strengths**: Rich workflow capabilities, state management, history
- **Limitations**: Heavyweight, primarily focused on workflows (not event routing), limited protocol bridging
- **Best for**: Complex stateful workflows with history requirements

#### Apache NiFi
- **Strengths**: Visual programming interface, comprehensive transformation capabilities
- **Limitations**: Very large footprint, complex setup, not suitable for edge or embedded use
- **Best for**: Data flow orchestration with visual design requirements

## Feature Comparison Matrix

| Feature                            | Natslive | EventBridge | Knative | NiFi | Temporal |
|-----------------------------------|----------|-------------|---------|------|----------|
| Lightweight binary                | ✅       | ❌          | ❌      | ❌   | ❌       |
| Cloud/vendor agnostic             | ✅       | ❌          | ✅      | ✅   | ✅       |
| Dynamic runtime rule config       | ✅       | Limited     | ✅      | ✅   | Partial  |
| Complex filtering (DSL or WASM)   | ✅       | Pattern only| ✅      | ✅   | ❌       |
| Edge/IoT ready                    | ✅       | ❌          | ❌      | ❌   | ❌       |
| Custom transports + WASM support  | ✅       | ❌          | ❌      | ❌   | ❌       |
| No platform dependencies          | ✅       | ❌          | ❌      | ✅   | ✅       |
| Multi-tenant rule isolation       | ✅ (WREL)| ❌          | Limited | ❌   | ❌       |
| Protocol-native transport         | ✅       | HTTP-based  | HTTP-based | ✅ | Limited |
| Operation without orchestrator    | ✅       | N/A (SaaS)  | ❌      | ✅   | ✅       |

## Selection Guide by Use Case

### For Edge Computing / IoT
**Best option: Natslive**
- Lightweight binary footprint
- Can operate without internet connectivity
- Native MQTT and other IoT protocol support
- WASM for updatable processing logic without firmware changes

### For Hybrid Cloud / Multi-Cloud
**Best option: Natslive**
- Protocol bridging between environments
- No cloud vendor lock-in
- Consistent operation model across deployments
- Flexible routing between on-premises and cloud resources

### For Kubernetes-Native Environments
**Best options: Knative Eventing or Natslive**
- If already using Kubernetes ecosystem extensively: **Knative**
- If needing lighter weight, more protocol options, or edge deployment: **Natslive**

### For All-In on Single Cloud Provider
**Best options: Cloud-native services or Natslive**
- If standardizing on one provider and accepting lock-in: **EventBridge/Eventarc/Event Grid**
- If needing more routing flexibility or possible future migration: **Natslive**

### For Complex Workflow Orchestration
**Best options: Temporal/Cadence or NiFi**
- If workflow state management is critical: **Temporal/Cadence**
- If visual flow design is important: **NiFi**
- If primarily needing message routing with flexible filtering: **Natslive**

## Conclusion

Natslive occupies a unique position in the event routing ecosystem by combining lightweight architecture, protocol flexibility, and advanced filtering capabilities. Its particular strengths in edge deployments, protocol bridging, and WebAssembly-based rule execution make it especially valuable for organizations with heterogeneous infrastructure, hybrid deployments, or resource-constrained environments.

While cloud-provided solutions offer better integration with their specific platforms and Knative provides advantages in pure Kubernetes environments, Natslive excels when independence from specific infrastructure, lightweight operation, or cross-protocol routing are priorities.