# WREL: WASM Rule Expression Language

## Overview

WREL (WASM Rule Expression Language) represents a transformative evolution in Natslive's architecture, decoupling the rule evaluation engine from Natslive's core runtime through WebAssembly. This architectural shift fundamentally expands Natslive from a lightweight event router into a runtime-extensible, secure, multi-tenant event processing engine.

## Core Concept

**Current NREL Model:**
- A Go-native DSL compiled at rule definition time into an in-memory AST/evaluator
- Tightly coupled to Natslive's build system
- Requires recompilation for new features or language changes

**WREL Model:**
- Rules compiled ahead-of-time (or on demand) into WebAssembly modules
- Natslive executes rules in isolated WASM sandboxes
- Complete decoupling of filtering logic from the router core

## Key Benefits

### 1. Runtime Extensibility Without Plugins

WREL provides plugin-level extensibility without the baggage of Go's plugin model:

- Deploy new filtering logic without rebuilding Natslive
- Test new expression language features or optimization strategies by swapping WASM modules
- Add customer-specific logic (geo-fencing, pricing models, compliance rules) via WASM deployment
- Implement "hot updates" for filtering logic while the router continues running

### 2. Enhanced Security via Isolation

WASM sandboxing provides strong security guarantees:

- Rules execute in tightly constrained environments with no file/network access
- Enforced execution timeouts, memory limits, and instruction quotas
- A malformed or malicious rule cannot crash the router
- Critical for multi-tenant environments where rule authors may not be fully trusted

### 3. Language Agnosticism

WREL opens up rule authoring beyond a single DSL:

- Support for multiple languages that compile to WASM (JavaScript, Rust, TinyGo, etc.)
- Teams can write rules in languages they already know
- Potential for language-specific SDKs and tooling
- Example: Write rules in JavaScript but run them safely in a low-memory event router

### 4. Optimized Rule Distribution

Rules become distributable, cacheable assets:

- Compile rules once (on CI or rule ingestion)
- Store compiled WASM modules in JetStream, Redis, or other storage
- Natslive fetches and executes only the matching modules
- Enables efficient rule distribution in distributed deployments

## Implementation Considerations

### Hybrid Evaluation Model

A pragmatic approach would implement a tiered evaluation strategy:

| Rule Type | Evaluation Strategy | Use Case |
|-----------|---------------------|----------|
| Simple pattern/equality | Native Go filter engine | High-performance basic routing |
| Complex logic / expression | NREL via Go compiler | Advanced built-in filtering |
| Custom / user-provided | WASM module | Extensible, user-defined logic |

This provides:
- Optimal performance for common cases
- Familiar expressiveness for standard advanced filtering
- Extensibility and safety for custom logic

### Technical Challenges

#### 1. Performance Considerations

WASM introduces some overhead:
- VM initialization costs
- Boundary crossing between Go and WASM
- Memory management and garbage collection
- Serialization/deserialization of events

Mitigation strategies:
- Persistent WASM runtimes (Wasmer, Wazero)
- Batch processing for high-volume routes
- Caching of frequently used modules
- Binary-optimized ABI for efficient data transfer

#### 2. Interface Design

The WASM modules and host need a well-defined interface:
- Standard input/output formats
- Error handling conventions
- Resource allocation and management
- Metadata and introspection capabilities

A typical interface might look like:
```go
// Host-side interface (Go)
type WRELModule interface {
    Evaluate(event []byte) (bool, error)
    GetMetadata() ModuleMetadata
    Close() error
}

// WASM module exports (in target language)
export function evaluate(eventPtr, eventLen) -> bool;
export function getMetadata() -> number; // Pointer to metadata string
```

#### 3. Developer Experience

Tools needed for seamless WREL adoption:
- CLI for compiling rules to WASM
- Testing framework for rule validation
- Registry for storing and versioning modules
- Monitoring and observability hooks
- Documentation and examples in multiple languages

## Deployment Architecture

### Rule Lifecycle

1. **Authoring**
   - Write rules in NREL, JavaScript, Rust, or other supported languages
   - Validate against schema or sample events
   - Compile to WASM module

2. **Distribution**
   - Upload to rule registry
   - Version and tag modules
   - Associate with routes and subjects

3. **Execution**
   - Natslive loads appropriate modules
   - Events flow through matched subjects
   - Rules evaluate in sandboxed WASM environment
   - Matching events forward to destinations

### Runtime Components

```
┌───────────────────────────────────────────────────────────────┐
│                      Natslive Router                          │
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────┐   │
│  │ NATS        │    │ Subject     │    │ Transport        │   │
│  │ Connection  │───▶│ Matcher     │───▶│ Forwarder       │   │
│  └─────────────┘    └──────┬──────┘    └──────────────────┘   │
│                            │                                   │
│                     ┌──────▼──────┐                            │
│                     │ Rule        │                            │
│                     │ Evaluator   │                            │
│                     └──────┬──────┘                            │
│                            │                                   │
│  ┌─────────────┐    ┌──────▼──────┐    ┌──────────────────┐   │
│  │ Native      │    │ WASM        │    │ Rule Registry    │   │
│  │ Filters     │◀───┤ Runtime     │───▶│ Client           │   │
│  └─────────────┘    └─────────────┘    └──────────────────┘   │
│                                                               │
└───────────────────────────────────────────────────────────────┘
            │                                    ▲
            │                                    │
            ▼                                    │
┌───────────────────────┐            ┌───────────────────────┐
│ Rule Registry         │            │ Event Sources/Sinks   │
│                       │            │                       │
│ ┌─────────────────┐   │            │ ┌─────────────────┐   │
│ │ WASM Modules    │   │            │ │ NATS            │   │
│ └─────────────────┘   │            │ └─────────────────┘   │
│                       │            │                       │
│ ┌─────────────────┐   │            │ ┌─────────────────┐   │
│ │ Metadata        │   │            │ │ HTTP            │   │
│ └─────────────────┘   │            │ └─────────────────┘   │
│                       │            │                       │
│ ┌─────────────────┐   │            │ ┌─────────────────┐   │
│ │ Versioning      │   │            │ │ Kafka, etc.     │   │
│ └─────────────────┘   │            │ └─────────────────┘   │
└───────────────────────┘            └───────────────────────┘
```

## Use Cases Unlocked by WREL

### 1. Multi-tenant SaaS Platforms

Enable customers to define custom filtering logic without compromising platform stability:
- Each tenant gets their own isolated rule environment
- Custom business logic per customer
- Safe execution regardless of rule complexity
- Tenant-specific routing policies

### 2. Edge Computing with Adaptive Logic

Deploy intelligent filtering at the edge with the ability to update logic remotely:
- Push new business rules to edge devices without firmware updates
- Differentiate processing by device type, location or capabilities
- Process sensitive data locally with custom filters
- Adapt to changing network conditions or data patterns

### 3. Internal Developer Platforms

Offer event routing as a service within your organization:
- Teams write their own routing logic in familiar languages
- Platform team maintains the infrastructure and safety guarantees
- Self-service deployment of event processing rules
- Unified event backbone with team-specific processing

### 4. Regulatory and Compliance Filtering

Implement and update data filtering for compliance without router changes:
- Region-specific data handling (GDPR, CCPA, etc.)
- Financial transaction filtering (AML, KYC)
- Healthcare data processing (HIPAA)
- Dynamic updates as regulations change

## Bundle Strategy with WREL

The addition of WREL impacts Natslive's bundle strategy:

| Bundle Name | WREL Support | Target Environment | Binary Size | Memory Footprint |
|-------------|-------------|---------------------|-------------|------------------|
| `natslive-minimal` | No | Ultra-constrained edge | ~12MB | ~15MB RAM |
| `natslive-wrel` | Yes | Standard edge/IoT | ~18MB | ~30MB RAM |
| `natslive-cloud-wrel` | Yes | Cloud environments | ~30MB | ~50MB RAM |
| `natslive-enterprise` | Yes | Data centers/hybrid | ~40MB | ~60MB RAM |

## From NREL to WREL: Migration Path

For existing users of NREL, a smooth transition path should include:

1. **Transpilation Tools**
   - Convert existing NREL rules to WASM-compatible format
   - Automated testing to ensure behavioral equivalence
   - Performance comparison utilities

2. **Dual-mode Operation**
   - Support both NREL and WREL simultaneously during transition
   - Feature flag to control evaluation strategy
   - Gradual migration of rule sets

3. **Adoption Timeline**
   - Phase 1: WREL SDK and tooling release
   - Phase 2: WREL support in natslive-wrel bundles
   - Phase 3: WREL as the recommended approach for complex rules
   - Phase 4: Long-term maintenance mode for NREL

## Conclusion

WREL represents a transformative evolution for Natslive, expanding it from "a highly efficient but static router" to "a dynamic, programmable, language-neutral event engine with a safe runtime boundary and infinite composability."

This architectural shift addresses previous limitations around extensibility, multi-tenancy, and runtime flexibility while preserving Natslive's core strength as a lightweight, efficient event router. WREL positions Natslive for a broader range of use cases, from edge computing to enterprise event processing, making it not just a powerful tool for today's needs but a future-proof foundation for tomorrow's event-driven architectures.

By embracing WebAssembly as the boundary between router and rule logic, Natslive can remain minimal at its core while offering unlimited extensibility in a secure, controlled manner – truly embodying the Unix philosophy of "do one thing well" while enabling complex systems to be built from simple components.