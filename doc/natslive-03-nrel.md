# Natslive Rule Expression Language (NREL)

## Beyond Simple Matching: The Need for Expressive Rules

The initial design of Natslive included basic JSON path matching for event filtering:

```json
{
  "route_id": "route-user-eu",
  "match": "events.user.*",
  "filter": {
    "payload.region": "eu"
  },
  "forward_to": [...]
}
```

While this approach works for simple cases, real-world event routing often requires more sophisticated filtering capabilities:

- Compound conditions with AND/OR/NOT logic
- Numeric comparisons and range checks
- String pattern matching with wildcards and regex
- Temporal conditions based on timestamps
- Set operations (membership, intersection)
- Mathematical expressions and transformations

To address these needs without sacrificing performance or simplicity, Natslive introduces a purpose-built Rule Expression Language.

## NREL: A Compiled Expression DSL

Natslive Rule Expression Language (NREL) is a lightweight, compiled expression language specifically designed for event filtering and routing. It combines:

1. A simple, human-readable syntax inspired by SQL WHERE clauses
2. Efficient compilation to an optimized evaluation tree at rule definition time
3. High-performance evaluation at event routing time

### Example Expressions

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

## Integration with Routing Rules

NREL expressions are integrated directly into routing rule definitions:

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

The `filter_expr` field replaces the simpler `filter` object when more complex conditions are needed.

## Compilation and Evaluation Process

NREL expressions follow a two-phase lifecycle:

### 1. Compilation Phase (At Rule Definition Time)

When a routing rule is defined or updated:

1. The NREL expression is parsed into an abstract syntax tree
2. Type checking and semantic validation are performed
3. The AST is optimized (constant folding, short-circuit optimization)
4. The optimized expression is compiled to an efficient evaluation tree
5. The compiled expression is stored with the routing rule

This compilation happens once per rule definition, ensuring runtime evaluation is as efficient as possible.

### 2. Evaluation Phase (At Event Routing Time)

When an event is being processed:

1. Subject-based matching narrows down potential rules
2. For matching rules, the pre-compiled expression evaluator is invoked
3. The expression is evaluated against the event payload
4. If the expression evaluates to true, the event is forwarded

## Core Language Features

### Data Types

NREL supports the following data types:

- **String**: Text values with UTF-8 encoding
- **Number**: 64-bit floating point values
- **Boolean**: true/false values
- **Null**: Absence of a value
- **Array**: Ordered collection of values
- **Object**: Unordered collection of key-value pairs
- **Timestamp**: ISO-8601 datetime values
- **Duration**: Time periods (e.g., "1h30m")

### Operators

#### Comparison Operators
- `==`, `!=`: Equality and inequality
- `>`, `>=`, `<`, `<=`: Numeric comparison
- `LIKE`: String pattern matching with wildcards
- `MATCHES`: Regular expression matching
- `IN`: Set membership
- `BETWEEN`: Range checking

#### Logical Operators
- `AND`, `OR`: Logical conjunction and disjunction
- `NOT`: Logical negation

#### Existence Operators
- `EXISTS`: Check if a field exists
- `IS NULL`, `IS NOT NULL`: Check for null values

### Functions

NREL includes a library of built-in functions:

#### String Functions
- `length(str)`: Return string length
- `concat(str1, str2, ...)`: Concatenate strings
- `substring(str, start, end)`: Extract substring
- `upper(str)`, `lower(str)`: Case conversion

#### Numeric Functions
- `abs(num)`: Absolute value
- `round(num, precision)`: Round to precision
- `min(a, b)`, `max(a, b)`: Minimum/maximum

#### Temporal Functions
- `now()`: Current timestamp
- `date(ts)`: Extract date from timestamp
- `duration(str)`: Parse duration string (e.g., "1h30m")
- `timeSince(ts)`: Duration since timestamp

#### Array Functions
- `count(array)`: Array length
- `contains(array, value)`: Check for value
- `any(array, expr)`: True if any element matches
- `all(array, expr)`: True if all elements match

### Path Navigation

NREL uses dot notation for navigating nested structures:

- `payload.user.profile.preferences.theme`: Deep property access
- `payload.items[0].productId`: Array indexing
- `$event.metadata.headers["X-Correlation-Id"]`: Map/dictionary access with key

## Performance Considerations

NREL is designed for high-performance event processing:

1. **Compiled Evaluation**: Pre-compiled expressions eliminate parsing overhead at runtime
2. **Short-circuit Evaluation**: AND/OR expressions stop evaluating once result is determined
3. **Indexing Support**: Common patterns can leverage indexes for faster evaluation
4. **Memory Efficiency**: Minimal allocation during expression evaluation
5. **Benchmarked Performance**: Rule evaluation throughput of millions of events per second per core

## Extensibility

NREL can be extended with custom functions for domain-specific needs:

```json
{
  "route_id": "geo-proximity",
  "match": "events.location.*",
  "filter_expr": "geoDistance(payload.coordinates, [51.5074, -0.1278]) < 10.0",
  "forward_to": [...]
}
```

Custom functions can be registered at compile time in the appropriate Natslive bundle.

## Safety and Constraints

To ensure NREL expressions remain safe and performant:

1. **No Unbounded Loops**: NREL intentionally lacks loop constructs to prevent runaway execution
2. **No Side Effects**: Expressions cannot modify data or have external effects
3. **Execution Timeout**: Complex expressions have configurable evaluation timeouts
4. **Memory Limits**: Expression evaluation has bounded memory usage
5. **Sanitized Inputs**: All inputs are properly sanitized before evaluation

## Real-world Examples

### User Event Segmentation

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

### IoT Device Filtering

```
# Route alerts from temperature sensors exceeding thresholds with rapid change
payload.deviceType == "temperatureSensor" AND
payload.reading > 85.0 AND
payload.metadata.previousReading != NULL AND
(payload.reading - payload.metadata.previousReading) / 
  timeSince(payload.metadata.previousTimestamp).seconds() > 1.5
```

### Multi-region Deployment

```
# Route events to local processing only when they originate in same region
$event.metadata.sourceRegion == $context.currentRegion OR
(payload.priority == "critical" AND payload.needsCrossRegionProcessing == true)
```

## Implementation Strategy

The NREL implementation in Natslive follows these principles:

1. **Embeddable**: The parser and evaluator are part of the Natslive core, not external dependencies
2. **Efficient**: Written in Go for minimal overhead and maximum performance
3. **Testable**: Comprehensive test suite ensures correctness
4. **Documentable**: Clear error messages and validation help users write correct expressions
5. **Extensible**: Architecture allows for adding new functions and operators over time

The expression compiler and evaluator will be included in all Natslive bundles, ensuring consistent rule behavior across deployments.

## Future Extensions

Planned enhancements to NREL include:

1. **Schema-aware Validation**: Validate expressions against event schemas at rule definition time
2. **Expression Libraries**: Reuse common expression patterns across multiple rules
3. **Transformation Expressions**: Extend beyond filtering to include payload transformation
4. **Visual Rule Builder**: Web UI for creating and testing NREL expressions
5. **Performance Profiling**: Tools to analyze expression evaluation efficiency

## Migration Path

For users of the basic filtering mechanism, automatic conversion to NREL is provided:

```json
// Original filter
{
  "filter": {
    "payload.region": "eu",
    "payload.status": "active"
  }
}

// Automatically converted to NREL
{
  "filter_expr": "payload.region == 'eu' AND payload.status == 'active'"
}
```

This ensures backward compatibility while encouraging migration to the more powerful expression language.

## Conclusion

The Natslive Rule Expression Language transforms Natslive from a simple subject-based router to a sophisticated event processing system. By combining the flexibility of a purpose-built DSL with the performance of compiled evaluation, NREL enables complex routing scenarios without sacrificing the lightweight, efficient nature that makes Natslive valuable for edge and hybrid deployments.