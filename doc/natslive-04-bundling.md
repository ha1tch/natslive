# Bundling Strategy for Natslive

## Go's Compilation Model and Natslive Architecture

### Core vs. Extended Feature Sets

Before discussing Go's compilation advantages, it's important to establish Natslive's approach to feature bundling. Natslive adopts a tiered architecture that separates core routing capabilities from extended features:

1. **Core Features**: Essential routing functionality present in all builds:
   - NATS-based message transport
   - Subject-based routing
   - Basic HTTP integration
   - Simple configuration management

2. **Extended Features**: Advanced capabilities that can be included or excluded:
   - Additional transport protocols (Kafka, MQTT, etc.)
   - Natslive Rule Expression Language (NREL)
   - Advanced persistence options
   - Metrics and telemetry integrations

This separation creates a foundation for the bundling strategy, allowing users to select the feature set appropriate for their use case.

### Leveraging Go's Static Compilation Advantage

Unlike runtime-interpreted or JVM-based languages that rely on dynamic class loading for extensibility, Go's compilation model produces single, statically-linked binaries. This creates a fundamentally different approach to extensibility than the traditional "plugin" model seen in ecosystems like Java's OSGi or Node.js's npm modules.

For Natslive, this means embracing Go's strengths rather than fighting against its design philosophy. By leveraging compile-time inclusion of transport implementations, Natslive can deliver:

1. **Simplified Deployment**: A single binary with no external dependencies, ideal for containerized and edge environments.
2. **Reduced Operational Complexity**: No plugin directories to manage, versioning conflicts to resolve, or dynamic loading errors to troubleshoot.
3. **Improved Performance**: Elimination of reflection overhead and dynamic dispatch costs.
4. **Enhanced Security**: Smaller attack surface without the need for runtime code loading capabilities.

## Binary Bundle Strategy

Instead of a plugin architecture, Natslive adopts a **bundle strategy** with several distribution models:

### Pre-compiled Standard Bundles

Official Natslive releases will include several pre-compiled bundles targeting common use cases:

| Bundle Name | Target Environment | NREL Support | Included Transports | Binary Size | Memory Footprint |
|-------------|-------------------|-------------|---------------------|-------------|------------------|
| `natslive-minimal` | Edge, IoT | No | NATS, HTTP | ~12MB | ~15MB RAM |
| `natslive-minimal-nrel` | Edge, IoT | Yes | NATS, HTTP | ~15MB | ~22MB RAM |
| `natslive-cloud` | Cloud Environments | No | NATS, HTTP, AWS (SQS/SNS/EventBridge), GCP (Pub/Sub), Azure (Event Hubs/Service Bus) | ~20MB | ~32MB RAM |
| `natslive-cloud-nrel` | Cloud Environments | Yes | NATS, HTTP, AWS (SQS/SNS/EventBridge), GCP (Pub/Sub), Azure (Event Hubs/Service Bus) | ~25MB | ~42MB RAM |
| `natslive-enterprise` | Data Centers, Hybrid Cloud | Yes | NATS, HTTP, Kafka, MQTT, AMQP, Redis | ~35MB | ~50MB RAM |
| `natslive-complete` | Development, All-purpose | Yes | All supported transports | ~50MB | ~70MB RAM |

Each bundle is a fully self-contained binary requiring no external dependencies beyond configuration.

### Build-time Configuration

For teams requiring custom feature combinations, Natslive provides build-time configuration through Go build tags:

```bash
# Build with just NATS and Kafka support, no NREL
go build -tags "nats kafka" -o natslive-custom ./cmd/natslive

# Build with cloud provider transports and NREL
go build -tags "nats http aws gcp nrel" -o natslive-cloud-custom ./cmd/natslive

# Build with specific IoT focus, minimal footprint (no NREL)
go build -tags "nats mqtt" -o natslive-iot ./cmd/natslive

# Build with all transports but no NREL (for extremely memory-constrained devices)
go build -tags "nats http kafka mqtt aws gcp azure redis !nrel" -o natslive-lite ./cmd/natslive
```

This approach allows organizations to create purpose-built binaries containing exactly the transport implementations they need, without the overhead of unused transports.

### Docker Image Variants

The bundle strategy extends to container deployments with specific Docker image variants that clearly indicate NREL status:

```Dockerfile
# Example Dockerfile for minimal variant without NREL
FROM golang:1.20 as builder
WORKDIR /go/src/app
COPY . .
RUN go build -tags "nats http !nrel" -o /go/bin/natslive ./cmd/natslive

FROM gcr.io/distroless/base-debian11
COPY --from=builder /go/bin/natslive /
ENTRYPOINT ["/natslive"]
```

Official Docker images follow a naming convention that explicitly indicates NREL capability:
- `natslive/natslive-minimal:v1.0.0` (no NREL)
- `natslive/natslive-minimal-nrel:v1.0.0` (with NREL)
- `natslive/natslive-cloud:v1.0.0` (no NREL)
- `natslive/natslive-cloud-nrel:v1.0.0` (with NREL)
- `natslive/natslive-enterprise:v1.0.0` (with NREL by default)
- `natslive/natslive-complete:v1.0.0` (with NREL by default)

This naming convention ensures clarity for operators regarding which images include the more powerful but resource-intensive NREL capabilities.

## Internal Architecture for Transport Bundling

While abandoning runtime plugins, Natslive maintains a clean internal architecture for transport implementations:

### Transport Interface

All transports implement standard interfaces, enabling consistent internal handling:

```go
// Core transport interfaces that all implementations must satisfy
type EventSource interface {
    Start(ctx context.Context) (<-chan *Event, error)
    Stop() error
    Status() SourceStatus
}

type EventSink interface {
    DeliverEvent(ctx context.Context, event *Event) error
    Status() SinkStatus
}
```

### Feature Registries

During initialization, compiled-in features register themselves with appropriate registries:

```go
// At package init time, each transport registers its factories
func init() {
    // Transport registration
    transport.RegisterSourceType("kafka", NewKafkaSource)
    transport.RegisterSinkType("kafka", NewKafkaSink)
}
```

This approach preserves the clean separation between the router core and transport implementations without runtime plugin loading.

### Configuration-Driven Instantiation

Even though all supported transports in a bundle are compiled into the binary, only those specified in configuration are instantiated:

```yaml
# Only these transports will be initialized
sources:
  - name: user-events
    type: nats  # Will instantiate NATS source
    subjects: ["events.user.>"]
  
  - name: payment-events
    type: kafka  # Will instantiate Kafka source
    topics: ["payments"]
    consumer_group: "natslive-router"
```

This ensures memory resources are only allocated to transports actively in use.

## Benefits Over Plugin-based Architecture

The bundle approach offers several advantages compared to traditional plugin systems:

### Operational Benefits

1. **Simplified Deployment**: One self-contained binary instead of a core plus multiple plugin files.
2. **Version Coherence**: All transports are versioned together, eliminating plugin compatibility issues.
3. **Reduced Failure Modes**: No runtime loading errors or plugin misconfigurations.
4. **Streamlined Updates**: Single-step upgrade process for all functionality.

### Performance Benefits

1. **Compile-time Optimization**: The Go compiler can optimize across transport boundaries.
2. **No Reflection Overhead**: Direct static dispatch to transport implementations.
3. **Shared Memory Efficiency**: Common data structures can be shared without serialization/deserialization.
4. **Reduced Startup Time**: No plugin discovery and loading phase.

### Security Benefits

1. **Reduced Attack Surface**: No need for file system access to load plugins.
2. **Supply Chain Protection**: All code is reviewed and compiled together.
3. **Static Analysis**: Security tools can analyze the complete codebase as a unit.

## NREL vs. No-NREL Builds

### Why Offer NREL-less Builds?

Offering builds without the Natslive Rule Expression Language (NREL) provides several key advantages:

1. **Minimal Memory Footprint**: Critical for edge devices and highly resource-constrained environments.
2. **Reduced Binary Size**: Important for IoT deployments where bandwidth for updates is limited.
3. **Lower Complexity**: For simple routing cases where basic pattern matching is sufficient.
4. **Faster Startup Time**: Without the NREL parser and compiler, initialization is faster.
5. **Improved Security Posture**: Removing the expression evaluation engine can reduce attack surface.

### When to Choose Each Option

| Scenario | Recommended Build |
|----------|------------------|
| Edge IoT with limited resources | Minimal without NREL |
| Simple routing patterns | Basic builds without NREL |
| Complex filtering needs | NREL-enabled builds |
| Enterprise data processing | Full builds with NREL |

### Transparent Fallback Mechanism

For configurations that specify NREL expressions but run on NREL-less builds, Natslive provides a transparent fallback mechanism:

1. When parsing route configurations, NREL-less builds detect `filter_expr` fields.
2. A basic pattern matching extraction attempts to convert simple expressions to pattern matches.
3. For expressions that cannot be converted, clear warnings are logged.
4. Administrators can use the validation API to check if their rules are compatible with NREL-less builds.

```go
// Example fallback mechanism (in NREL-less builds)
func handleFilterExpression(expr string) (*PatternFilter, error) {
    // Try to convert simple equality expressions to pattern matches
    patterns, compatible := extractPatterns(expr)
    if !compatible {
        log.Warn("Expression too complex for NREL-less build: %s", expr)
        return nil, errors.New("complex expressions require NREL-enabled build")
    }
    return NewPatternFilter(patterns), nil
}
```

## Implementation Timeline

The Natslive bundle strategy will be implemented in phases:

1. **Phase 1**: Core architecture with NATS and HTTP transports (Q2 2023)
2. **Phase 2**: Add NREL support with build tags (Q3 2023)
3. **Phase 3**: Add Kafka, MQTT, and AWS transports with build tags (Q3 2023)
4. **Phase 4**: Complete enterprise transports and bundle standardization (Q4 2023)
5. **Phase 5**: Custom build tooling for organization-specific bundles (Q1 2024)

## Extending with Custom Transports

Organizations requiring custom transport implementations have two options:

### 1. Fork and Customize

Since all transports are compiled in, custom implementations require forking the Natslive codebase and adding new transport packages:

```
natslive/
  ├── transports/
  │   ├── nats/
  │   ├── http/
  │   ├── kafka/
  │   └── mycompany/  # Custom transport implementation
  └── cmd/
      └── natslive/
          └── main.go  # Import custom transport package
```

### 2. Sidecar Pattern

For proprietary transports that cannot be open-sourced, a sidecar pattern is recommended:

```
┌─────────────────┐     ┌───────────────────────┐
│                 │     │                       │
│  Natslive       │◄────┤  Transport Sidecar    │
│  (Core)         │     │  (Custom Protocol)    │
│                 │     │                       │
└─────────────────┘     └───────────────────────┘
```

The sidecar handles the proprietary protocol and forwards events to Natslive via its native NATS interface.

## Bundle Maintenance Strategy

Maintaining multiple bundles requires disciplined development practices:

1. **Continuous Integration Matrix**: Each commit is tested against all bundle configurations.
2. **Transport Isolation**: Transport implementations avoid dependencies on other transports.
3. **Code Coverage Requirements**: All transports maintain high test coverage regardless of bundle inclusion.
4. **Feature Flags**: New experimental transports can be hidden behind build flags until stable.

## Contrasting with Alternative Approaches

### Why Not Dynamic Go Plugins?

Go does offer a plugin package, but it comes with significant limitations:

1. **Build Environment Consistency**: Plugins must be built with exactly the same Go version and dependencies.
2. **Platform Specificity**: Plugins are not portable across operating systems or architectures.
3. **Limited HMR**: No true hot module reloading capabilities.
4. **Debugging Complexity**: Significantly complicates debugging and error tracing.

### Why Not Scripting Language Extensions?

Embedding a scripting language (Lua, JavaScript) for transport extensions was considered but rejected due to:

1. **Performance Overhead**: Interpretation and boundary crossing costs.
2. **Security Concerns**: Additional attack vectors through the scripting engine.
3. **Operational Complexity**: Managing scripts alongside the binary.
4. **Development Experience**: Requiring contributors to work in multiple languages.

## Compilation Conditional Features

The Go build system allows for conditional compilation based on build tags, which Natslive leverages extensively for both transport selection and feature enablement. Go uses build tags rather than C-style preprocessor directives:

```go
// File: nrel_filters.go

//go:build nrel
// +build nrel

package filter

// This file is only included in builds with the "nrel" build tag

func init() {
    // Register NREL-specific filter types
    RegisterFilterType("expression", NewExpressionFilter)
}

// NREL-specific filter implementation
type ExpressionFilter struct {
    compiledExpr *nrel.CompiledExpression
    sourceExpr   string
}

func NewExpressionFilter(config map[string]interface{}) (EventFilter, error) {
    // Create and compile an NREL expression
}
```

This approach allows feature-specific code to be completely excluded from builds where it's not needed, minimizing binary size and memory footprint.

## Conclusion

The bundle-based approach to Natslive implementations embraces Go's strengths while providing the flexibility needed for diverse deployment scenarios. By avoiding runtime plugins in favor of compile-time inclusion and selectively enabling advanced features like NREL, Natslive delivers a more reliable, performant, and adaptable event routing solution.

Offering both NREL and NREL-less builds ensures Natslive can serve the full spectrum of use cases - from resource-constrained IoT edge devices to sophisticated enterprise event processing. This approach gives operators the power to choose exactly the right balance of features and resource utilization for their specific needs.

This strategy aligns perfectly with Go's philosophy of simplicity, efficiency, and maintainability. For Natslive users, this means a more robust product with fewer operational headaches, better performance characteristics, and the flexibility to deploy in virtually any environment.