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

## Implementation Deep Dive

### Orchestration Pattern (Production-Ready)

```python
from enum import Enum
from dataclasses import dataclass
from typing import List, Callable, Optional
import uuid

class SagaState(Enum):
    STARTED = "started"
    RUNNING = "running"
    COMPLETED = "completed"
    COMPENSATING = "compensating"
    FAILED = "failed"

@dataclass
class SagaStep:
    name: str
    action: Callable          # Forward action
    compensation: Callable    # Rollback action

@dataclass
class SagaLog:
    step_name: str
    status: str               # "success" | "failed"
    result: Optional[dict]

class SagaOrchestrator:
    def __init__(self, saga_id: str = None):
        self.saga_id = saga_id or str(uuid.uuid4())
        self.steps: List[SagaStep] = []
        self.executed: List[SagaLog] = []
        self.state = SagaState.STARTED

    def add_step(self, step: SagaStep):
        self.steps.append(step)
        return self

    def execute(self, context: dict) -> bool:
        """Execute saga with automatic compensation on failure"""
        self.state = SagaState.RUNNING

        for step in self.steps:
            try:
                result = step.action(context)
                self.executed.append(SagaLog(step.name, "success", result))
                context.update(result or {})  # Pass data to next step
            except Exception as e:
                self.executed.append(SagaLog(step.name, "failed", {"error": str(e)}))
                self._compensate(context)
                return False

        self.state = SagaState.COMPLETED
        return True

    def _compensate(self, context: dict):
        """Execute compensations in reverse order"""
        self.state = SagaState.COMPENSATING

        # Reverse through executed steps (skip the failed one)
        for log in reversed(self.executed[:-1]):
            step = next(s for s in self.steps if s.name == log.step_name)
            try:
                step.compensation(context)
            except Exception as e:
                # Log compensation failure - may need manual intervention
                print(f"Compensation failed for {step.name}: {e}")

        self.state = SagaState.FAILED

# Usage Example - E-commerce Order Saga
def create_order(ctx):
    order_id = db.orders.insert(ctx['items'], status='pending')
    return {"order_id": order_id}

def cancel_order(ctx):
    db.orders.update(ctx['order_id'], status='cancelled')

def reserve_inventory(ctx):
    for item in ctx['items']:
        db.inventory.decrement(item['sku'], item['qty'])
    return {"inventory_reserved": True}

def release_inventory(ctx):
    for item in ctx['items']:
        db.inventory.increment(item['sku'], item['qty'])

def charge_payment(ctx):
    payment_id = payment_gateway.charge(ctx['user_id'], ctx['amount'])
    return {"payment_id": payment_id}

def refund_payment(ctx):
    payment_gateway.refund(ctx['payment_id'])

# Build and execute saga
saga = SagaOrchestrator()
saga.add_step(SagaStep("create_order", create_order, cancel_order))
saga.add_step(SagaStep("reserve_inventory", reserve_inventory, release_inventory))
saga.add_step(SagaStep("charge_payment", charge_payment, refund_payment))

success = saga.execute({"items": [...], "user_id": "u123", "amount": 99.99})
```

### Key Implementation Principles

| Principle | Implementation |
|-----------|----------------|
| **Idempotency** | Use idempotency keys; check if step already executed |
| **State Persistence** | Store saga state in DB before each step (crash recovery) |
| **Timeout Handling** | Set deadlines per step; auto-compensate on timeout |
| **Retry Logic** | Retry transient failures before compensating |

### Choreography with Events

```python
# Each service publishes and subscribes to events
class OrderService:
    def create_order(self, data):
        order = self.db.create_order(data)
        self.publish("OrderCreated", {"order_id": order.id})

    @subscribe("PaymentFailed")
    def on_payment_failed(self, event):
        self.db.cancel_order(event.order_id)  # Compensation

class PaymentService:
    @subscribe("InventoryReserved")
    def on_inventory_reserved(self, event):
        try:
            self.charge(event.order_id)
            self.publish("PaymentCompleted", event)
        except:
            self.publish("PaymentFailed", event)  # Trigger compensations
```

---

## Real-World Examples

| Company | Use Case |
|---------|----------|
| **Uber** | Ride booking (match driver → payment → trip) |
| **Amazon** | Order fulfillment across warehouses |
| **Airbnb** | Booking + payment + host notification |
| **Netflix** | Content licensing workflows |

### Tools & Frameworks
- **Temporal.io** - Workflow orchestration with saga support
- **Camunda** - BPMN-based saga orchestration
- **Eventuate Tram** - Event-driven sagas for Java/Spring
- **AWS Step Functions** - Serverless saga orchestration

---

## Interview Tips

When discussing Saga in interviews:
1. Compare with 2PC and explain why Saga is preferred
2. Discuss both choreography and orchestration approaches
3. Emphasize compensating transactions
4. Mention idempotency requirements
5. Explain how to handle compensation failures (dead letter queue, manual intervention)

