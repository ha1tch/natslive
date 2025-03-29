# Natslive: Real-World Use Cases

This document showcases concrete use cases for Natslive, demonstrating how its unique architecture and capabilities solve real-world problems that would be difficult to address with alternative solutions.

## Use Case 1: Edge Device to Cloud Integration

### Scenario: Small Device → Remote Microservices via HTTPS/Kafka

A common challenge in IoT architectures is efficiently connecting resource-constrained edge devices to cloud-based services without overwhelming the devices with heavy client libraries or complex networking code.

#### Components
* **Tiny edge device** (Raspberry Pi, ESP32, or embedded Linux)
* Lightweight agent publishing events to NATS (minimal overhead)
* **Natslive router** deployed on-premises or in the cloud
* Remote cloud services consuming events via HTTP or Kafka

#### Solution Architecture

```
┌────────────────┐      ┌───────────────┐      ┌─────────────────────────┐
│                │      │               │      │                         │
│  Edge Device   │ NATS │   Natslive    │ HTTP │  Cloud API Service      │
│  (Sensor)      │─────▶│   Router      │─────▶│                         │
│                │      │               │      │                         │
└────────────────┘      │               │      └─────────────────────────┘
                        │               │
                        │               │      ┌─────────────────────────┐
                        │               │ Kafka│                         │
                        │               │─────▶│  Data Processing Stream │
                        │               │      │                         │
                        └───────────────┘      └─────────────────────────┘
```

#### Implementation

1. **Device Publishes Event via NATS**:
   ```bash
   # Example (Go or CLI client):
   nats pub events.sensor.temperature '{"id": "sensor-123", "value": 74.2, "unit": "F", "timestamp": "2023-10-15T08:42:15Z"}'
   ```

2. **Natslive Router Configuration**:
   ```yaml
   routes:
     - route_id: high-temp-alert
       match: "events.sensor.temperature"
       filter_expr: "payload.value > 70"
       forward_to:
         - type: http
           url: "https://api.mycloudservice.com/alerts"
           headers:
             Content-Type: "application/json"
             Authorization: "Bearer {{secret.api_key}}"
         - type: kafka
           brokers: ["kafka.example.com:9092"]
           topic: "sensor-alerts"
           key_template: "{{payload.id}}"
   ```

3. **Natslive Handles**:
   * Protocol translation (NATS → HTTP/Kafka)
   * Authentication for each destination
   * Content transformation if needed
   * Retries and error handling
   * Filtering logic

#### Benefits

* **Edge devices remain simple** - only need NATS client (lightweight)
* **No cloud SDKs needed on edge** - reducing dependencies, binary size, and attack surface
* **Dynamic routing without device updates** - change destinations or filtering rules without touching edge code
* **Protocol translation handled centrally** - edge devices don't need to understand HTTP, Kafka, etc.
* **Enhanced security** - credentials for external services stay in Natslive, not on edge devices

This scenario is particularly powerful for large-scale IoT deployments where managing and updating edge devices is challenging and expensive.

## Use Case 2: Multi-cloud Event Routing

### Scenario: Cross-cloud Event Propagation with Filtering

Organizations increasingly adopt multi-cloud strategies but struggle with event-driven communication across cloud boundaries. Each cloud provider has its own eventing system (EventBridge, Pub/Sub, Event Grid) with limited interoperability.

#### Components
* Applications running in AWS, GCP, and on-premises
* Natslive routers in each environment
* NATS as the backbone connecting all environments
* Different consuming services across clouds

#### Solution Architecture

```
┌───────────────────────┐      ┌───────────────┐      ┌───────────────────────┐
│                       │      │               │      │                       │
│  AWS Cloud            │      │   NATS        │      │  GCP Cloud            │
│  ┌─────────────┐      │      │   Cluster     │      │  ┌─────────────┐      │
│  │ Lambda      │ SQS  │      │               │      │  │ Cloud       │      │
│  │ Function    │─────────────│───────────────│──────────│ Function    │      │
│  └─────────────┘      │      │               │      │  └─────────────┘      │
│                       │      │               │      │                       │
│  ┌─────────────┐      │      │  ┌─────────┐  │      │  ┌─────────────┐      │
│  │ Natslive    │ NATS │      │  │Natslive │  │ NATS │  │ Natslive    │      │
│  │ Router      │─────────────│──│Router   │──│──────────│ Router      │      │
│  └─────────────┘      │      │  └─────────┘  │      │  └─────────────┘      │
│                       │      │               │      │                       │
└───────────────────────┘      └───────────────┘      └───────────────────────┘
                                      │
                                      │
                               ┌──────▼──────┐
                               │             │
                               │ On-premises │
                               │ Systems     │
                               │             │
                               └─────────────┘
```

#### Implementation

1. **AWS Lambda Publishes Event**:
   ```javascript
   // AWS Lambda using NATS client
   const NATS = require('nats');
   
   exports.handler = async function(event) {
     const nc = await NATS.connect({ servers: process.env.NATS_URL });
     await nc.publish('events.order.created', JSON.stringify({
       orderId: event.orderId,
       customerId: event.customerId,
       totalAmount: event.totalAmount,
       region: 'us-east-1'
     }));
     await nc.close();
     return { success: true };
   };
   ```

2. **Natslive Router Configuration (GCP)** with WREL filtering:
   ```yaml
   sources:
     - name: nats-events
       type: nats
       subjects: ["events.>"]

   routes:
     - route_id: high-value-orders
       match: "events.order.created"
       # Using WREL for complex filtering logic
       wrel_module: "order-analytics-filter"
       forward_to:
         - type: gcp-pubsub
           project: "my-gcp-project"
           topic: "high-value-orders"
   ```

3. **WREL Module for Complex Filtering**:
   ```javascript
   // WREL JavaScript module compiled to WASM
   export function evaluate(event) {
     const payload = JSON.parse(event);
     
     // Complex business logic
     if (payload.totalAmount > 1000) {
       if (payload.region.startsWith('us-') && !blacklistedCustomers.includes(payload.customerId)) {
         return true;
       }
     }
     return false;
   }
   
   const blacklistedCustomers = ['CUST-1234', 'CUST-5678'];
   ```

#### Benefits

* **Cloud-agnostic event backbone** - events flow freely across cloud boundaries
* **Unified filtering logic** - consistent business rules regardless of cloud provider
* **Reduced vendor lock-in** - easier to migrate workloads between clouds
* **Simplified cloud service integration** - Natslive handles authentication and protocol details
* **Cost optimization** - filter events before they cross cloud boundaries to reduce data transfer costs

This approach gives organizations the freedom to choose the best services from each cloud provider while maintaining a cohesive event-driven architecture.

## Use Case 3: Multi-tenant SaaS Platform

### Scenario: Secure Customer-specific Event Processing

SaaS platforms often need to allow customers to customize how events are processed and routed within their tenancy, but without compromising platform stability or security.

#### Components
* Multi-tenant SaaS application generating events
* Natslive with WREL for tenant-specific filtering and routing
* Custom processing logic per tenant
* Variety of integration endpoints for different customers

#### Solution Architecture

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  SaaS Platform                                      │
│                                                     │
│  ┌─────────────┐     ┌───────────────────────────┐  │
│  │             │     │                           │  │
│  │ Application │────▶│ Natslive Router          │  │
│  │ Services    │     │                           │  │
│  │             │     │  ┌─────────────────────┐  │  │
│  └─────────────┘     │  │ Tenant A WREL       │  │  │
│                      │  └─────────────────────┘  │  │
│                      │                           │──────┐
│                      │  ┌─────────────────────┐  │  │   │
│                      │  │ Tenant B WREL       │  │  │   │
│                      │  └─────────────────────┘  │  │   │
│                      │                           │  │   │
│                      │  ┌─────────────────────┐  │  │   │
│                      │  │ Tenant C WREL       │  │  │   │
│                      │  └─────────────────────┘  │  │   │
│                      │                           │  │   │
│                      └───────────────────────────┘  │   │
│                                                     │   │
└─────────────────────────────────────────────────────┘   │
                                                          │
    ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
    │                   │    │                   │    │                   │
    │  Tenant A         │    │  Tenant B         │    │  Tenant C         │
    │  Integration      │    │  Integration      │    │  Integration      │
    │                   │    │                   │    │                   │
    └───────────────────┘    └───────────────────┘    └───────────────────┘
```

#### Implementation

1. **SaaS Application Publishes Event**:
   ```go
   // Go code within SaaS platform
   nc, _ := nats.Connect(natsURL)
   
   // Publish event with tenant context
   nc.Publish("events.user.login", []byte(`{
     "tenantId": "tenant-456",
     "userId": "user-123",
     "timestamp": "2023-10-15T14:32:10Z",
     "ipAddress": "192.168.1.1",
     "userAgent": "Mozilla/5.0..."
   }`))
   ```

2. **Tenant-specific Routing Configuration**:
   ```yaml
   routes:
     - route_id: tenant-456-security-alerts
       match: "events.user.login"
       # Filter events for this specific tenant
       filter: 
         "payload.tenantId": "tenant-456"
       # Use tenant-specific WREL module
       wrel_module: "tenant-456-security-rules"
       forward_to:
         - type: http
           url: "https://hooks.tenant-456.example.com/security"
           headers:
             X-API-Key: "{{secret.tenant_456_api_key}}"
         - type: slack
           webhook: "{{secret.tenant_456_slack_webhook}}"
           channel: "#security-alerts"
   ```

3. **Tenant-specific WREL Security Rules**:
   ```rust
   // Rust code compiled to WASM
   #[no_mangle]
   pub extern "C" fn evaluate(event_ptr: *const u8, event_len: usize) -> bool {
       let event_data = unsafe { 
           std::slice::from_raw_parts(event_ptr, event_len) 
       };
       
       // Parse the JSON
       if let Ok(event) = serde_json::from_slice::<serde_json::Value>(event_data) {
           // Tenant-specific security rules
           if let Some(ip) = event.get("ipAddress").and_then(|v| v.as_str()) {
               // Check if IP is from suspicious ranges
               if is_suspicious_ip(ip) {
                   return true;
               }
           }
           
           // Check for suspicious login patterns
           if let Some(timestamp) = event.get("timestamp").and_then(|v| v.as_str()) {
               if is_unusual_time(timestamp) {
                   return true;
               }
           }
       }
       
       false
   }
   
   fn is_suspicious_ip(ip: &str) -> bool {
       // Tenant-specific IP blocking logic
       ip.starts_with("185.") || BLOCKED_RANGES.iter().any(|range| ip_in_range(ip, range))
   }
   
   fn is_unusual_time(timestamp: &str) -> bool {
       // Tenant-specific time-based rules
       // ...
       false
   }
   
   static BLOCKED_RANGES: &[&str] = &["192.168.0.0/16", "10.0.0.0/8"];
   ```

#### Benefits

* **Security through isolation** - each tenant's rules run in separate WASM sandboxes
* **Custom logic without code changes** - tenants can deploy specialized processing rules
* **Reduced blast radius** - a tenant's bad rule can't affect other tenants
* **Unified platform** - consistent event flow with tenant-specific customization
* **Simplified compliance** - tenant-specific data handling rules for regulatory requirements

This model allows SaaS platforms to offer customization and integration flexibility that would otherwise require extensive development or risky plugin architectures.

## Use Case 4: Legacy System Modernization

### Scenario: Incremental Migration with Protocol Bridging

Organizations with legacy systems often struggle with modernization, especially when those systems use outdated protocols or proprietary messaging formats.

#### Components
* Legacy systems using proprietary protocols or older message formats
* Modern microservices using contemporary protocols
* Natslive providing protocol translation and message transformation
* Incremental migration path without "big bang" rewrites

#### Solution Architecture

```
┌─────────────────┐      ┌───────────────────────────────┐      ┌─────────────────┐
│                 │      │                               │      │                 │
│  Legacy System  │ MQTT │  Natslive Router             │ NATS │  Modern         │
│  (Proprietary   │─────▶│                               │─────▶│  Microservices  │
│   Protocol)     │      │  ┌─────────────────────────┐  │      │                 │
│                 │      │  │ Message Transformation  │  │      │                 │
└─────────────────┘      │  │ WREL                    │  │      └─────────────────┘
                         │  └─────────────────────────┘  │
┌─────────────────┐      │                               │      ┌─────────────────┐
│                 │ JMS  │  ┌─────────────────────────┐  │ HTTP │                 │
│  Legacy Java    │─────▶│  │ Protocol Bridge         │  │─────▶│  Cloud Services │
│  Application    │      │  └─────────────────────────┘  │      │                 │
│                 │      │                               │      │                 │
└─────────────────┘      └───────────────────────────────┘      └─────────────────┘
```

#### Implementation

1. **Configure Legacy Protocol Source**:
   ```yaml
   sources:
     - name: legacy-mqtt
       type: mqtt
       broker: "tcp://legacy-broker:1883"
       topics: ["inventory/#", "orders/#"]
       qos: 1
     
     - name: legacy-jms
       type: jms
       url: "tcp://legacy-app:61616"
       queue: "LEGACY.ORDER.QUEUE"
       username: "{{secret.jms_username}}"
       password: "{{secret.jms_password}}"
   ```

2. **Message Transformation with WREL**:
   ```javascript
   // JavaScript transformation logic compiled to WASM
   export function transform(message) {
     const legacy = JSON.parse(message);
     
     // Transform from legacy format to modern schema
     return JSON.stringify({
       orderId: legacy.ORDER_ID,
       customer: {
         id: legacy.CUST_NUM,
         name: legacy.CUST_NAME
       },
       items: legacy.ORDER_ITEMS.map(item => ({
         sku: item.ITEM_CODE,
         quantity: parseInt(item.QTY, 10),
         price: parseFloat(item.PRICE)
       })),
       metadata: {
         source: "legacy",
         originalFormat: "jms",
         migratedAt: new Date().toISOString()
       }
     });
   }
   ```

3. **Routing Configuration**:
   ```yaml
   routes:
     - route_id: legacy-order-transform
       match: "legacy.jms.orders"
       wrel_module: "order-transformer"
       wrel_function: "transform"  # Use transform instead of evaluate
       forward_to:
         - type: nats
           subject: "orders.created"
         - type: http
           url: "https://api.example.com/analytics"
           method: "POST"
   ```

#### Benefits

* **Incremental migration** - modernize systems piece by piece without full rewrites
* **Protocol bridging** - connect systems that wouldn't normally be able to communicate
* **Message transformation** - convert between legacy and modern formats
* **Reduced migration risk** - systems can run in parallel during transition
* **Lower technical debt** - avoid maintaining custom adapters for each legacy system

This approach allows organizations to gradually modernize their architecture without the risks and costs associated with complete system rewrites.

## Conclusion

These use cases demonstrate Natslive's versatility across different domains and scenarios. The common themes that make Natslive particularly valuable include:

1. **Protocol flexibility** - connecting systems that use different messaging protocols
2. **Lightweight footprint** - ideal for resource-constrained environments
3. **Dynamic rule updates** - change routing behavior without redeployment
4. **Secure isolation** - especially with WREL for multi-tenant scenarios
5. **Transformation capabilities** - adapt messages between different formats and schemas

Natslive excels in scenarios requiring both simplicity and flexibility - where heavyweight alternatives would introduce unnecessary complexity, and simplistic solutions would lack needed capabilities.

Whether bridging between clouds, connecting edge devices to enterprise services, enabling tenant-specific customization, or modernizing legacy systems, Natslive provides a consistent, efficient approach to event routing that adapts to your specific infrastructure needs.