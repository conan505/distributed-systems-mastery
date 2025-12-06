# Event-Carried State Transfer

## What It Is

**Event-Carried State Transfer** is a pattern where events contain all the data needed by consumers, eliminating the need for consumers to call back to the source service for additional information.

---

## The Analogy 📦

Think of **package delivery**:
- **Event notification only**: "Package arrived" → You must go to post office
- **Event-carried state**: Package delivered to your door with everything inside

The event carries the payload, not just a notification.

---

## Why It Exists

### The Problem: Chatty Services
```
Event Notification Pattern:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Order Service ──"OrderCreated(id=123)"──▶ Shipping Service    │
│                                                                  │
│   Shipping Service ──"GET /orders/123"──▶ Order Service         │
│   Shipping Service ──"GET /customers/456"──▶ Customer Service   │
│   Shipping Service ──"GET /products/789"──▶ Product Service     │
│                                                                  │
│   Problems:                                                      │
│   - 3 additional API calls                                      │
│   - Coupling to source services                                 │
│   - Source services must be available                           │
│   - Higher latency                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Event-Carried State Solution:
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Order Service publishes:                                       │
│   {                                                              │
│     "event": "OrderCreated",                                    │
│     "orderId": 123,                                             │
│     "customer": {                                               │
│       "id": 456,                                                │
│       "name": "John Doe",                                       │
│       "address": "123 Main St"                                  │
│     },                                                          │
│     "items": [                                                  │
│       {"productId": 789, "name": "Widget", "qty": 2}           │
│     ],                                                          │
│     "total": 99.99                                              │
│   }                                                              │
│                                                                  │
│   Shipping Service: Has everything needed, no callbacks!        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Benefits

### 1. Reduced Coupling
```
Without ECST:
- Consumer depends on producer's API
- Producer must be available
- API changes break consumers

With ECST:
- Consumer only depends on event schema
- Producer can be offline
- Schema evolution is manageable
```

### 2. Better Availability
```
Producer down?
- Without ECST: Consumer can't process events
- With ECST: Consumer has all data, continues working
```

### 3. Lower Latency
```
Without ECST: Event + N API calls
With ECST: Just the event (all data included)
```

### 4. Local Data Copies
```
Consumers can cache/store event data locally
- Faster queries
- Offline capability
- Reduced load on source
```

---

## Trade-offs

### Larger Events
```
Event notification: {"orderId": 123}  → 20 bytes
Event-carried: {full order details}   → 2 KB

More bandwidth, storage needed
```

### Data Staleness
```
Consumer has snapshot at event time
Source data may have changed since

Solutions:
- Accept eventual consistency
- Publish update events
- Include version/timestamp
```

### Schema Coupling
```
Consumers depend on event schema
Schema changes affect all consumers

Solutions:
- Schema registry
- Backward-compatible changes
- Event versioning
```

---

## When to Use

### ✅ Good Fit:
- **Decoupled microservices** - Reduce runtime dependencies
- **Event sourcing** - Events are the source of truth
- **Read-heavy consumers** - Cache data locally
- **Unreliable networks** - Reduce call failures

### ❌ Avoid When:
- Data changes frequently (stale quickly)
- Events would be very large
- Strong consistency required
- Simple notification sufficient

---

## Implementation Patterns

### 1. Full State in Event
```json
{
  "type": "CustomerUpdated",
  "customer": {
    "id": "123",
    "name": "John Doe",
    "email": "john@example.com",
    "address": {...},
    "preferences": {...}
  }
}
```

### 2. Delta Events
```json
{
  "type": "CustomerAddressChanged",
  "customerId": "123",
  "oldAddress": {...},
  "newAddress": {...}
}
```

### 3. Hybrid Approach
```json
{
  "type": "OrderCreated",
  "orderId": "123",
  "summary": {
    "customerName": "John",
    "total": 99.99
  },
  "detailsUrl": "/orders/123"  // For full details if needed
}
```

---

## Comparison with Alternatives

| Pattern | Data in Event | Callbacks | Coupling |
|---------|---------------|-----------|----------|
| **Event Notification** | ID only | Required | High |
| **Event-Carried State** | Full data | None | Low |
| **Hybrid** | Summary + URL | Optional | Medium |

---

## Real-World Examples

| System | Usage |
|--------|-------|
| **E-commerce** | Order events with full details |
| **Banking** | Transaction events with account state |
| **Shipping** | Shipment events with address, items |
| **Notifications** | User events with preferences |

---

## Interview Tips

When discussing Event-Carried State Transfer:
1. Contrast with event notification pattern
2. Explain the coupling reduction benefit
3. Discuss trade-offs (size, staleness)
4. Mention schema evolution challenges
5. Give concrete examples (order events)

