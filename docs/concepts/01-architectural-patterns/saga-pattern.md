# Saga Pattern

## What It Is

The **Saga Pattern** is a way to manage distributed transactions across multiple microservices. Instead of one big transaction, it breaks the operation into a sequence of local transactions, each with a compensating action if something fails.

---

## The Analogy 🎬

Imagine **booking a vacation**:
1. Book flight ✈️
2. Book hotel 🏨
3. Book car rental 🚗

With **traditional transactions**: Everything succeeds or fails together (rarely possible across different companies)

With **Saga**: Book each separately. If car rental fails, you call to cancel the hotel, then cancel the flight. Each step has its own "undo" action.

---

## Why It Exists

### The Problem with Distributed Transactions:
- **2PC (Two-Phase Commit) doesn't scale** - Locks resources across services
- **Microservices are autonomous** - Each has its own database
- **Network failures** - Can't guarantee all services respond
- **Long-running operations** - Can't hold locks for hours

### What Saga Solves:
- Maintain data consistency across services WITHOUT distributed locks
- Handle partial failures gracefully
- Support long-running business processes
- Each service remains autonomous

---

## How It Works

### Two Implementation Approaches:

#### 1. Choreography (Event-Driven)
Each service publishes events, and other services react independently.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Choreography-Based Saga                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐    OrderCreated    ┌───────────┐    PaymentDone    │
│  │  Order  │ ─────────────────▶ │  Payment  │ ────────────────▶ │
│  │ Service │                    │  Service  │                   │
│  └─────────┘                    └───────────┘                   │
│       ▲                              │                          │
│       │                              ▼                          │
│       │      PaymentFailed     ┌───────────┐                    │
│       └─────────────────────── │ Inventory │                    │
│         (Compensate)           │  Service  │                    │
│                                └───────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. Orchestration (Central Coordinator)
A central orchestrator tells each service what to do and when.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Orchestration-Based Saga                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ┌────────────────┐                         │
│                      │  Saga          │                         │
│                      │  Orchestrator  │                         │
│                      └───────┬────────┘                         │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│        ┌─────────┐    ┌───────────┐    ┌───────────┐           │
│        │  Order  │    │  Payment  │    │ Inventory │           │
│        │ Service │    │  Service  │    │  Service  │           │
│        └─────────┘    └───────────┘    └───────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| **Coupling** | Loose | Tighter (to orchestrator) |
| **Complexity** | Spread across services | Centralized |
| **Visibility** | Harder to trace | Easy to monitor |
| **Failure handling** | Each service decides | Orchestrator handles |
| **Best for** | Simple flows | Complex, multi-step processes |

---

## When to Use It

### ✅ Good Fit:
- **E-commerce checkout** - Order → Payment → Inventory → Shipping
- **Travel booking** - Flight + Hotel + Car (different systems)
- **Money transfers** - Debit A → Credit B
- **User registration** - Create account → Setup profile → Send email

### ❌ Avoid When:
- Single database can handle the transaction
- Immediate consistency is critical (use 2PC or single DB)
- Very simple operations (over-engineering)

---

## How to Use It Effectively

### Compensating Actions:
Every step needs a rollback plan:

| Action | Compensation |
|--------|--------------|
| Create Order | Cancel Order |
| Reserve Inventory | Release Inventory |
| Charge Payment | Refund Payment |
| Send Shipping | Cancel Shipment |

### Best Practices:

1. **Make operations idempotent** - Same request, same result
2. **Store saga state** - Know where you are if orchestrator crashes
3. **Use correlation IDs** - Track all steps of a saga
4. **Design for failure** - Assume every step CAN fail
5. **Compensations may fail too** - Have retry mechanisms

---

## Real-World Examples

| Company | Use Case |
|---------|----------|
| **Uber** | Ride booking (match driver → payment → trip) |
| **Amazon** | Order fulfillment across warehouses |
| **Airbnb** | Booking + payment + host notification |
| **Netflix** | Content licensing workflows |

---

## Interview Tips

When discussing Saga in interviews:
1. Compare with 2PC and explain why Saga is preferred
2. Discuss both choreography and orchestration approaches
3. Emphasize compensating transactions
4. Mention idempotency requirements

