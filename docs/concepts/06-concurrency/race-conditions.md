# Race Conditions

## What It Is

A **Race Condition** occurs when the behavior of software depends on the timing or interleaving of multiple threads/processes, leading to unpredictable and often incorrect results.

---

## The Analogy 🏃

Think of **two people editing the same document**:
- Alice reads document: "Hello World"
- Bob reads document: "Hello World"
- Alice changes to: "Hello Alice"
- Bob changes to: "Hello Bob"
- Final result: "Hello Bob" (Alice's change lost!)

The outcome depends on who saves last — a race.

---

## Why It's Critical

### The Problem: Non-Atomic Operations
```python
# This looks atomic but ISN'T
balance = 100

# Thread A                    # Thread B
read balance (100)            read balance (100)
add 50                        subtract 30
write balance (150)           write balance (70)

# Expected: 100 + 50 - 30 = 120
# Actual: 70 or 150 (whoever writes last "wins")
```

### Common Manifestations:
- **Lost updates** - Concurrent writes overwrite each other
- **Dirty reads** - Reading uncommitted changes
- **Double-spending** - Same resource used twice
- **TOCTOU** - Time-of-check to time-of-use bugs

---

## Types of Race Conditions

### 1. Check-Then-Act
```python
# WRONG
if file_exists(path):         # Thread B deletes file here!
    open(path)                # Crash!

# CORRECT
try:
    open(path)
except FileNotFoundError:
    handle_missing()
```

### 2. Read-Modify-Write
```python
# WRONG
counter = get_counter()        # Another thread also reads
counter += 1
set_counter(counter)           # Lost update!

# CORRECT
atomic_increment(counter)
# Or use locks
```

### 3. Lazy Initialization (Double-Checked Locking)
```java
// WRONG (without volatile)
if (instance == null) {
    synchronized(this) {
        if (instance == null) {
            instance = new Singleton();  // Partially constructed visible!
        }
    }
}

// CORRECT
private static volatile Singleton instance;
// Or use synchronized initialization
```

---

## Solutions

### 1. Locks (Mutexes)
```python
lock = threading.Lock()

with lock:
    balance = read_balance()
    balance += 50
    write_balance(balance)
# Only one thread can execute this block at a time
```

### 2. Atomic Operations
```python
from atomics import AtomicInt

counter = AtomicInt(0)
counter.fetch_add(1)  # Atomic increment

# Compare-And-Swap (CAS)
while True:
    old = counter.load()
    if counter.compare_exchange(old, old + 1):
        break  # Success
```

### 3. Immutable Data
```python
# Immutable objects can be shared safely
@dataclass(frozen=True)
class Point:
    x: int
    y: int

# Create new instead of modifying
new_point = Point(old_point.x + 1, old_point.y)
```

### 4. Thread-Local Storage
```python
import threading
local = threading.local()

def process_request(user_id):
    local.user_id = user_id  # Thread-local, no sharing
```

### 5. Message Passing
```python
# Instead of shared state, send messages
queue = Queue()

# Producer
queue.put(data)

# Consumer
data = queue.get()
```

---

## Database Race Conditions

### Lost Update Problem:
```sql
-- Transaction A                    -- Transaction B
SELECT balance FROM accounts;       SELECT balance FROM accounts;
-- balance = 100                    -- balance = 100
UPDATE accounts SET balance = 150;  UPDATE accounts SET balance = 70;
-- Expected: 120, Got: 70 or 150
```

### Solutions:
```sql
-- 1. Pessimistic Locking
SELECT balance FROM accounts FOR UPDATE;

-- 2. Optimistic Locking (versioning)
UPDATE accounts 
SET balance = 150, version = 2 
WHERE id = 1 AND version = 1;
-- Fails if version changed

-- 3. Atomic Updates
UPDATE accounts SET balance = balance + 50;
```

---

## Distributed Race Conditions

### Double-Booking Problem:
```
User A: Check seat 5A available → Yes
User B: Check seat 5A available → Yes
User A: Book seat 5A → Success
User B: Book seat 5A → Success (double-booked!)
```

### Solutions:
```
1. Distributed locks (Redis, ZooKeeper)
2. Optimistic concurrency with versions
3. Idempotency keys
4. Database constraints (unique index)
```

---

## Detection Tools

| Tool | Language | Purpose |
|------|----------|---------|
| **ThreadSanitizer** | C/C++ | Runtime detection |
| **Go Race Detector** | Go | `go run -race` |
| **Helgrind** | C/C++ | Valgrind tool |
| **FindBugs/SpotBugs** | Java | Static analysis |

---

## Best Practices

```
1. Minimize shared state
2. Prefer immutable data
3. Use well-tested concurrent primitives
4. Keep critical sections short
5. Document thread-safety guarantees
6. Test with stress/load testing
7. Use race detection tools in CI
```

---

## Interview Tips

When discussing Race Conditions:
1. Give concrete examples (counter, bank balance)
2. Explain check-then-act vs read-modify-write
3. Know multiple solutions (locks, atomics, immutability)
4. Discuss database-level solutions
5. Mention detection tools

