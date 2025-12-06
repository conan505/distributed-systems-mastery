# Observer Pattern

## What It Is

The **Observer Pattern** defines a one-to-many dependency between objects so that when one object (subject) changes state, all its dependents (observers) are notified and updated automatically.

---

## The Analogy 📰

Think of a **newspaper subscription**:
- Publisher (subject) produces newspapers
- Subscribers (observers) receive updates
- Subscribe/unsubscribe anytime
- Publisher doesn't know subscriber details

---

## Why It Exists

### The Problem: Tight Coupling
```python
# BAD: Subject knows all observers
class Order:
    def complete(self):
        self.status = "completed"
        # Directly calling all dependents
        EmailService().send_confirmation(self)
        InventoryService().update_stock(self)
        AnalyticsService().log_order(self)
        # Adding new observer = modify Order class!
```

### What Observer Solves:
- **Loose coupling** - Subject doesn't know observer details
- **Open/Closed** - Add observers without modifying subject
- **Dynamic relationships** - Subscribe/unsubscribe at runtime
- **Broadcast communication** - One-to-many updates

---

## Implementation

### Basic Structure
```python
from abc import ABC, abstractmethod
from typing import List

# Observer interface
class Observer(ABC):
    @abstractmethod
    def update(self, subject) -> None:
        pass

# Subject (Observable)
class Subject:
    def __init__(self):
        self._observers: List[Observer] = []
    
    def attach(self, observer: Observer) -> None:
        self._observers.append(observer)
    
    def detach(self, observer: Observer) -> None:
        self._observers.remove(observer)
    
    def notify(self) -> None:
        for observer in self._observers:
            observer.update(self)

# Concrete Subject
class Order(Subject):
    def __init__(self, order_id: str):
        super().__init__()
        self.order_id = order_id
        self._status = "pending"
    
    @property
    def status(self):
        return self._status
    
    @status.setter
    def status(self, value):
        self._status = value
        self.notify()  # Notify all observers

# Concrete Observers
class EmailNotifier(Observer):
    def update(self, subject: Order) -> None:
        print(f"Sending email: Order {subject.order_id} is {subject.status}")

class InventoryUpdater(Observer):
    def update(self, subject: Order) -> None:
        if subject.status == "completed":
            print(f"Updating inventory for order {subject.order_id}")

class AnalyticsLogger(Observer):
    def update(self, subject: Order) -> None:
        print(f"Logging: Order {subject.order_id} changed to {subject.status}")
```

### Usage
```python
# Create subject
order = Order("ORD-123")

# Attach observers
order.attach(EmailNotifier())
order.attach(InventoryUpdater())
order.attach(AnalyticsLogger())

# Change state - all observers notified
order.status = "completed"

# Output:
# Sending email: Order ORD-123 is completed
# Updating inventory for order ORD-123
# Logging: Order ORD-123 changed to completed
```

---

## Push vs Pull Model

### Push Model
```python
# Subject pushes data to observers
class Observer(ABC):
    @abstractmethod
    def update(self, order_id: str, status: str, total: float):
        pass

# Observer receives all data it might need
```

### Pull Model
```python
# Subject notifies, observer pulls what it needs
class Observer(ABC):
    @abstractmethod
    def update(self, subject: Order):
        pass

# Observer queries subject for needed data
def update(self, subject: Order):
    status = subject.status  # Pull what you need
    total = subject.total
```

---

## Event-Based Implementation

```python
from typing import Callable, Dict, List

class EventEmitter:
    def __init__(self):
        self._listeners: Dict[str, List[Callable]] = {}
    
    def on(self, event: str, callback: Callable):
        if event not in self._listeners:
            self._listeners[event] = []
        self._listeners[event].append(callback)
    
    def off(self, event: str, callback: Callable):
        if event in self._listeners:
            self._listeners[event].remove(callback)
    
    def emit(self, event: str, *args, **kwargs):
        if event in self._listeners:
            for callback in self._listeners[event]:
                callback(*args, **kwargs)

# Usage
class Order(EventEmitter):
    def complete(self):
        self.status = "completed"
        self.emit("order_completed", self)

order = Order()
order.on("order_completed", lambda o: print(f"Order {o.id} completed"))
order.on("order_completed", lambda o: send_email(o))
order.complete()
```

---

## Real-World Examples

### 1. UI Event Handling
```python
button.on_click(lambda: print("Clicked!"))
button.on_click(lambda: analytics.log("button_click"))
```

### 2. Stock Price Updates
```python
class StockTicker(Subject):
    def set_price(self, symbol, price):
        self.prices[symbol] = price
        self.notify()

ticker.attach(TradingBot())
ticker.attach(PriceDisplay())
ticker.attach(AlertSystem())
```

### 3. Message Queues
```python
# Pub/Sub is Observer at scale
queue.subscribe("orders", OrderProcessor())
queue.subscribe("orders", AuditLogger())
queue.publish("orders", order_data)
```

---

## Observer vs Pub/Sub

| Aspect | Observer | Pub/Sub |
|--------|----------|---------|
| **Coupling** | Subject knows observers | Decoupled via broker |
| **Scale** | In-process | Distributed |
| **Delivery** | Synchronous | Often async |
| **Use case** | UI events, local | Microservices, messaging |

---

## When to Use

### ✅ Good Fit:
- One-to-many dependencies
- State changes need broadcast
- Observers change dynamically
- Loose coupling needed

### ❌ Avoid When:
- Simple direct calls suffice
- Order of notification matters
- Observers need guaranteed delivery

---

## Common Pitfalls

```python
# ❌ Memory leak: forgetting to detach
observer = MyObserver()
subject.attach(observer)
# observer goes out of scope but still referenced!

# ✅ Always detach when done
subject.detach(observer)

# ❌ Infinite loops
class A(Observer):
    def update(self, subject):
        subject.value += 1  # Triggers another update!

# ✅ Guard against re-entry
def update(self, subject):
    if not self._updating:
        self._updating = True
        # ... do work
        self._updating = False
```

---

## Interview Tips

When discussing Observer Pattern:
1. Explain the one-to-many relationship
2. Show loose coupling benefit
3. Discuss push vs pull models
4. Mention real examples (UI events, pub/sub)
5. Know the memory leak pitfall

