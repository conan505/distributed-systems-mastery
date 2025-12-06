# Temporal Workflow Pattern

## What It Is

**Temporal** is a workflow orchestration platform that provides durable execution of long-running business processes, handling failures, retries, and state management automatically.

---

## The Analogy 📋

Think of a **project manager with perfect memory**:
- Tracks every task and its status
- If someone fails, reassigns automatically
- Never forgets where things left off
- Can resume after any interruption

Temporal is that project manager for your code.

---

## Why It Exists

### The Problem: Distributed Workflow Complexity
```python
# Without Temporal - fragile workflow
def process_order(order):
    try:
        payment = charge_payment(order)  # What if this fails?
        inventory = reserve_inventory(order)  # What if this fails?
        shipping = create_shipment(order)  # What if this fails?
        notify_customer(order)  # What if this fails?
    except Exception:
        # How do we know what succeeded?
        # How do we compensate?
        # What if we crash mid-way?
        rollback_somehow()  # 😰
```

### What Temporal Solves:
- **Durable execution** - Survives crashes, restarts
- **Automatic retries** - Configurable retry policies
- **State management** - Tracks workflow progress
- **Visibility** - See workflow status anytime
- **Timeouts** - Handle stuck operations

---

## Core Concepts

### Workflow
```python
# Workflow definition - the orchestration logic
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order: Order) -> str:
        # Each activity is durably executed
        payment = await workflow.execute_activity(
            charge_payment,
            order,
            start_to_close_timeout=timedelta(minutes=5)
        )
        
        inventory = await workflow.execute_activity(
            reserve_inventory,
            order,
            start_to_close_timeout=timedelta(minutes=2)
        )
        
        shipment = await workflow.execute_activity(
            create_shipment,
            order,
            start_to_close_timeout=timedelta(minutes=10)
        )
        
        return f"Order {order.id} completed"
```

### Activity
```python
# Activity - the actual work (can fail, will be retried)
@activity.defn
async def charge_payment(order: Order) -> PaymentResult:
    # This can fail - Temporal will retry
    return await payment_gateway.charge(
        order.customer_id,
        order.total
    )

@activity.defn
async def reserve_inventory(order: Order) -> InventoryResult:
    return await inventory_service.reserve(order.items)
```

### Worker
```python
# Worker - executes workflows and activities
async def main():
    client = await Client.connect("localhost:7233")
    
    worker = Worker(
        client,
        task_queue="order-processing",
        workflows=[OrderWorkflow],
        activities=[charge_payment, reserve_inventory, create_shipment]
    )
    
    await worker.run()
```

---

## How Durability Works

```
┌─────────────────────────────────────────────────────────────────┐
│                   Temporal Durability                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Workflow starts                                            │
│      → Event: WorkflowExecutionStarted                          │
│                                                                  │
│   2. Activity scheduled                                         │
│      → Event: ActivityTaskScheduled                             │
│                                                                  │
│   3. Activity completes                                         │
│      → Event: ActivityTaskCompleted                             │
│                                                                  │
│   [Worker crashes here]                                         │
│                                                                  │
│   4. New worker picks up                                        │
│      → Replays events 1-3                                       │
│      → Continues from step 4                                    │
│                                                                  │
│   All state is in event history, not in memory!                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Features

### Retry Policies
```python
retry_policy = RetryPolicy(
    initial_interval=timedelta(seconds=1),
    backoff_coefficient=2.0,
    maximum_interval=timedelta(minutes=5),
    maximum_attempts=10,
    non_retryable_error_types=["InvalidInputError"]
)

await workflow.execute_activity(
    charge_payment,
    order,
    retry_policy=retry_policy
)
```

### Timeouts
```python
await workflow.execute_activity(
    long_running_task,
    start_to_close_timeout=timedelta(hours=1),  # Max execution time
    schedule_to_start_timeout=timedelta(minutes=5),  # Max queue time
    heartbeat_timeout=timedelta(minutes=1)  # Must heartbeat
)
```

### Signals and Queries
```python
@workflow.defn
class OrderWorkflow:
    def __init__(self):
        self.status = "pending"
    
    @workflow.signal
    async def cancel_order(self):
        self.status = "cancelled"
    
    @workflow.query
    def get_status(self) -> str:
        return self.status
```

---

## Saga Pattern with Temporal

```python
@workflow.defn
class OrderSaga:
    @workflow.run
    async def run(self, order: Order):
        compensations = []
        
        try:
            # Step 1: Payment
            payment = await workflow.execute_activity(charge_payment, order)
            compensations.append((refund_payment, payment))
            
            # Step 2: Inventory
            reservation = await workflow.execute_activity(reserve_inventory, order)
            compensations.append((release_inventory, reservation))
            
            # Step 3: Shipping
            shipment = await workflow.execute_activity(create_shipment, order)
            
            return "Success"
            
        except Exception as e:
            # Compensate in reverse order
            for compensate_fn, data in reversed(compensations):
                await workflow.execute_activity(compensate_fn, data)
            raise
```

---

## Temporal vs Alternatives

| Feature | Temporal | Message Queue | Cron Jobs |
|---------|----------|---------------|-----------|
| **Durability** | Built-in | Manual | None |
| **Retries** | Automatic | Manual | Manual |
| **Visibility** | Full | Limited | Limited |
| **Long-running** | Native | Complex | Not suited |
| **Compensation** | Easy | Complex | Complex |

---

## When to Use

### ✅ Good Fit:
- Long-running business processes
- Multi-step transactions (sagas)
- Scheduled/recurring workflows
- Human-in-the-loop processes
- Microservice orchestration

### ❌ Consider Alternatives:
- Simple request/response
- Sub-second latency requirements
- Stateless operations

---

## Interview Tips

When discussing Temporal:
1. Explain durable execution concept
2. Distinguish workflows vs activities
3. Show how it handles failures
4. Compare with message queues
5. Give saga implementation example

