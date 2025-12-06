# Idempotent APIs

## What It Is

An **idempotent operation** produces the same result regardless of how many times it's executed. For APIs, this means repeated requests with the same parameters have the same effect as a single request.

---

## The Analogy 🔘

Think of an **elevator button**:
- Press "Floor 5" once → Goes to floor 5
- Press "Floor 5" ten times → Still goes to floor 5
- The button is idempotent

Contrast with a **vending machine button**:
- Press once → Get 1 soda
- Press 5 times → Get 5 sodas
- NOT idempotent

---

## Why It Matters

### The Problem: Network Unreliability
```
Client ──request──▶ Server
Client ◀──timeout── (no response)

Did the server:
A) Never receive the request?
B) Process it but response was lost?

If B and client retries: DUPLICATE OPERATION!

Example: Double charge on payment
```

### What Idempotency Solves:
- **Safe retries** - Can retry without side effects
- **Network resilience** - Handle timeouts gracefully
- **Simpler clients** - Retry logic is straightforward
- **At-least-once delivery** - Works with message queues

---

## HTTP Methods Idempotency

```
┌─────────────────────────────────────────────────────────────────┐
│                HTTP Methods                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Method      Idempotent?   Safe?                               │
│   ─────────   ───────────   ─────                               │
│   GET         ✅ Yes        ✅ Yes                               │
│   HEAD        ✅ Yes        ✅ Yes                               │
│   OPTIONS     ✅ Yes        ✅ Yes                               │
│   PUT         ✅ Yes        ❌ No                                │
│   DELETE      ✅ Yes        ❌ No                                │
│   POST        ❌ No         ❌ No                                │
│   PATCH       ❌ No*        ❌ No                                │
│                                                                  │
│   *PATCH can be made idempotent with proper design              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementing Idempotency

### 1. Idempotency Keys
```python
# Client sends unique key with request
POST /payments
Idempotency-Key: abc-123-def-456
{
    "amount": 100,
    "currency": "USD"
}

# Server implementation
class PaymentService:
    def __init__(self):
        self.idempotency_store = {}  # Redis in production
    
    def process_payment(self, idempotency_key: str, request: dict):
        # Check if already processed
        if idempotency_key in self.idempotency_store:
            return self.idempotency_store[idempotency_key]
        
        # Process payment
        result = self._charge_payment(request)
        
        # Store result
        self.idempotency_store[idempotency_key] = result
        
        return result
```

### 2. Natural Idempotency (PUT semantics)
```python
# Instead of:
POST /users/123/increment-balance  # Not idempotent

# Use:
PUT /users/123/balance  # Idempotent
{
    "balance": 150
}

# Multiple identical PUTs = same result
```

### 3. Conditional Operations
```python
# Using ETags
PUT /resources/123
If-Match: "etag-abc123"
{...}

# Server checks ETag before update
# If ETag doesn't match, return 412 Precondition Failed
```

---

## Idempotency Key Design

### Key Generation (Client)
```python
import uuid
import hashlib

def generate_idempotency_key(user_id: str, action: str, params: dict) -> str:
    """Generate deterministic idempotency key."""
    # Option 1: Random UUID (client must store/retry same key)
    return str(uuid.uuid4())
    
    # Option 2: Deterministic from request (automatic dedup)
    content = f"{user_id}:{action}:{json.dumps(params, sort_keys=True)}"
    return hashlib.sha256(content.encode()).hexdigest()
```

### Server Storage
```python
class IdempotencyStore:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.ttl = 86400  # 24 hours
    
    def get_or_set(self, key: str, processor: callable) -> tuple:
        """
        Returns (result, was_cached).
        """
        # Check cache
        cached = self.redis.get(f"idempotency:{key}")
        if cached:
            return json.loads(cached), True
        
        # Acquire lock for this key
        lock = self.redis.lock(f"lock:{key}", timeout=30)
        if not lock.acquire(blocking=True, blocking_timeout=5):
            raise ConcurrentRequestError()
        
        try:
            # Double-check after acquiring lock
            cached = self.redis.get(f"idempotency:{key}")
            if cached:
                return json.loads(cached), True
            
            # Process and cache
            result = processor()
            self.redis.setex(
                f"idempotency:{key}",
                self.ttl,
                json.dumps(result)
            )
            return result, False
        finally:
            lock.release()
```

---

## Common Patterns

### Make POST Idempotent
```python
# Non-idempotent
POST /orders
{items: [...]}

# Idempotent with client-generated ID
POST /orders
{
    "order_id": "ord_abc123",  # Client provides
    "items": [...]
}

# Server: INSERT ... ON CONFLICT DO NOTHING
```

### Idempotent Counter Updates
```python
# Non-idempotent
POST /inventory/decrement
{product_id: 123, quantity: 5}

# Idempotent with request ID
POST /inventory/decrement
{
    "request_id": "req_xyz789",
    "product_id": 123,
    "quantity": 5
}

# Server tracks processed request_ids
```

---

## Best Practices

```
1. Use idempotency keys for mutations
2. Store keys with TTL (24h typical)
3. Return same response for duplicate requests
4. Include status code in stored response
5. Handle concurrent duplicate requests (locking)
6. Log duplicate request detection
7. Document idempotency behavior in API docs
```

---

## Stripe's Implementation

```
Request:
POST /v1/charges
Idempotency-Key: key123
{amount: 2000, currency: "usd"}

First request:
- Process charge
- Store result with key123
- Return 200 with charge object

Retry (same key):
- Find stored result
- Return exact same 200 with same charge object

Different payload, same key:
- Return 400 error (keys are tied to first request)
```

---

## Interview Tips

When discussing Idempotency:
1. Explain the network unreliability problem
2. Describe idempotency keys pattern
3. Discuss storage and TTL considerations
4. Handle concurrent duplicate requests
5. Give examples (payments, order creation)

