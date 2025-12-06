# Distributed Tracing

## What It Is

**Distributed Tracing** is a method for tracking requests as they flow through distributed systems, providing visibility into the path, timing, and dependencies of each request.

---

## The Analogy 📦

Think of **package tracking**:
- Package gets a tracking number
- Each facility scans and logs
- You see the complete journey
- Know exactly where delays occurred

Distributed tracing does this for requests across services.

---

## Why It Exists

### The Problem: Debugging Microservices
```
User reports: "Checkout is slow"

In monolith:
  Look at one log file, find the slow function

In microservices:
  Request touches 10 services
  Each has its own logs
  Which service is slow?
  What's the call sequence?
```

### What Tracing Solves:
- **Request flow visibility** - See entire path
- **Latency analysis** - Find bottlenecks
- **Dependency mapping** - Understand service relationships
- **Root cause analysis** - Pinpoint failures

---

## Core Concepts

### Trace
```
A trace represents the entire journey of a request

Trace ID: abc123
├── Service A (50ms)
│   ├── Service B (30ms)
│   │   └── Database (20ms)
│   └── Service C (15ms)
└── Total: 50ms
```

### Span
```
A span represents a single operation within a trace

Span {
  trace_id: "abc123"
  span_id: "span456"
  parent_span_id: "span123"
  operation_name: "HTTP GET /users"
  service_name: "user-service"
  start_time: 1640000000.000
  duration: 30ms
  tags: {
    "http.method": "GET",
    "http.status_code": 200
  }
  logs: [
    {timestamp: ..., message: "Cache miss"}
  ]
}
```

### Context Propagation
```
┌─────────────────────────────────────────────────────────────────┐
│                   Context Propagation                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Service A                    Service B                         │
│   ┌─────────────────┐         ┌─────────────────┐               │
│   │ Create trace    │         │ Extract context │               │
│   │ trace_id: abc   │ ──────▶ │ trace_id: abc   │               │
│   │ span_id: 001    │ Headers │ parent: 001     │               │
│   └─────────────────┘         │ span_id: 002    │               │
│                               └─────────────────┘               │
│                                                                  │
│   HTTP Headers:                                                  │
│   X-Trace-Id: abc123                                            │
│   X-Span-Id: span001                                            │
│   X-Parent-Span-Id: root                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                  Distributed Tracing Flow                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Request arrives at API Gateway                             │
│      → Generate trace_id, create root span                      │
│                                                                  │
│   2. Gateway calls User Service                                 │
│      → Pass trace context in headers                            │
│      → User Service creates child span                          │
│                                                                  │
│   3. User Service calls Database                                │
│      → Create span for DB query                                 │
│                                                                  │
│   4. Each service sends spans to collector                      │
│      → Async, non-blocking                                      │
│                                                                  │
│   5. Collector aggregates and stores                            │
│      → Jaeger, Zipkin, etc.                                     │
│                                                                  │
│   6. Query and visualize                                        │
│      → See complete trace timeline                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation

### OpenTelemetry (Standard)
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

# Setup
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

jaeger_exporter = JaegerExporter(
    agent_host_name="localhost",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Create spans
with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order_id", "12345")
    
    with tracer.start_as_current_span("validate_payment"):
        # Payment validation logic
        pass
    
    with tracer.start_as_current_span("update_inventory"):
        # Inventory update logic
        pass
```

### Auto-Instrumentation
```python
# Many frameworks have auto-instrumentation
# Flask example
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

# Now all Flask routes and HTTP requests are traced automatically
```

---

## Tracing Tools

| Tool | Type | Features |
|------|------|----------|
| **Jaeger** | Open source | Uber-developed, CNCF |
| **Zipkin** | Open source | Twitter-developed |
| **Datadog APM** | Commercial | Full observability |
| **AWS X-Ray** | Cloud | AWS integration |
| **Honeycomb** | Commercial | High cardinality |

---

## Sampling Strategies

```
Problem: Tracing every request is expensive

Solutions:

1. Head-based sampling
   - Decide at trace start
   - Sample 1% of requests
   - Simple but may miss errors

2. Tail-based sampling
   - Decide after trace complete
   - Keep all errors, slow requests
   - More complex, requires buffering

3. Adaptive sampling
   - Adjust rate based on traffic
   - Higher rate for rare paths
```

---

## Best Practices

```
1. Use consistent trace context format (W3C Trace Context)
2. Add meaningful span names and tags
3. Include business context (user_id, order_id)
4. Sample appropriately for volume
5. Set up alerts on trace anomalies
6. Correlate with logs and metrics
```

---

## Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Logs          Metrics         Traces                          │
│   ────          ───────         ──────                          │
│   What          How much        Where                           │
│   happened?     /how fast?      did it go?                      │
│                                                                  │
│   Detailed      Aggregated      Request                         │
│   events        numbers         journey                         │
│                                                                  │
│   Debug         Alert           Debug                           │
│   specific      on trends       distributed                     │
│   issues                        issues                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interview Tips

When discussing Distributed Tracing:
1. Explain trace, span, and context propagation
2. Know OpenTelemetry basics
3. Discuss sampling strategies
4. Mention correlation with logs/metrics
5. Give debugging scenario examples

