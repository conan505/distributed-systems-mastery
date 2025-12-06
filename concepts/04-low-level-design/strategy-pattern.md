# Strategy Pattern

## What It Is

The **Strategy Pattern** defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from clients that use it.

---

## The Analogy 🗺️

Think of **navigation apps**:
- Same destination, different routes
- "Fastest route" strategy
- "Avoid tolls" strategy
- "Scenic route" strategy
- Swap strategies without changing the app

---

## Why It Exists

### The Problem: Conditional Explosion
```python
# BAD: Giant if-else chain
def calculate_shipping(order, method):
    if method == "standard":
        return order.weight * 0.5
    elif method == "express":
        return order.weight * 1.5 + 10
    elif method == "overnight":
        return order.weight * 3.0 + 25
    elif method == "drone":
        return 50 if order.weight < 5 else float('inf')
    # Adding new method = modify this function!
```

### What Strategy Solves:
- **Open/Closed Principle** - Add strategies without modifying existing code
- **Single Responsibility** - Each strategy handles one algorithm
- **Runtime flexibility** - Swap algorithms dynamically
- **Testability** - Test each strategy in isolation

---

## Implementation

### Basic Structure
```python
from abc import ABC, abstractmethod

# Strategy interface
class ShippingStrategy(ABC):
    @abstractmethod
    def calculate(self, order) -> float:
        pass

# Concrete strategies
class StandardShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 0.5

class ExpressShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 1.5 + 10

class OvernightShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 3.0 + 25

# Context
class Order:
    def __init__(self, weight: float):
        self.weight = weight
        self._shipping_strategy: ShippingStrategy = StandardShipping()
    
    def set_shipping_strategy(self, strategy: ShippingStrategy):
        self._shipping_strategy = strategy
    
    def get_shipping_cost(self) -> float:
        return self._shipping_strategy.calculate(self)

# Usage
order = Order(weight=10)
print(order.get_shipping_cost())  # 5.0 (standard)

order.set_shipping_strategy(ExpressShipping())
print(order.get_shipping_cost())  # 25.0 (express)
```

---

## Real-World Examples

### 1. Payment Processing
```python
class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: pass

class CreditCardPayment(PaymentStrategy):
    def __init__(self, card_number: str):
        self.card_number = card_number
    
    def pay(self, amount: float) -> bool:
        # Process credit card payment
        return True

class PayPalPayment(PaymentStrategy):
    def __init__(self, email: str):
        self.email = email
    
    def pay(self, amount: float) -> bool:
        # Process PayPal payment
        return True

class Checkout:
    def __init__(self, payment_strategy: PaymentStrategy):
        self.payment_strategy = payment_strategy
    
    def complete(self, amount: float):
        return self.payment_strategy.pay(amount)
```

### 2. Compression Algorithms
```python
class CompressionStrategy(ABC):
    @abstractmethod
    def compress(self, data: bytes) -> bytes: pass

class GzipCompression(CompressionStrategy):
    def compress(self, data: bytes) -> bytes:
        import gzip
        return gzip.compress(data)

class LZ4Compression(CompressionStrategy):
    def compress(self, data: bytes) -> bytes:
        import lz4.frame
        return lz4.frame.compress(data)

class FileProcessor:
    def __init__(self, compression: CompressionStrategy):
        self.compression = compression
    
    def save(self, data: bytes, path: str):
        compressed = self.compression.compress(data)
        with open(path, 'wb') as f:
            f.write(compressed)
```

### 3. Sorting Strategies
```python
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data: list) -> list: pass

class QuickSort(SortStrategy):
    def sort(self, data: list) -> list:
        # Quick sort implementation
        return sorted(data)  # Simplified

class MergeSort(SortStrategy):
    def sort(self, data: list) -> list:
        # Merge sort implementation
        return sorted(data)  # Simplified

class DataProcessor:
    def __init__(self, sort_strategy: SortStrategy):
        self.sort_strategy = sort_strategy
    
    def process(self, data: list) -> list:
        return self.sort_strategy.sort(data)
```

---

## Strategy with Functions (Python)

```python
# Strategies as functions (simpler for simple cases)
def standard_shipping(order):
    return order.weight * 0.5

def express_shipping(order):
    return order.weight * 1.5 + 10

class Order:
    def __init__(self, weight: float):
        self.weight = weight
        self.shipping_calculator = standard_shipping
    
    def get_shipping_cost(self) -> float:
        return self.shipping_calculator(self)

# Usage
order = Order(10)
order.shipping_calculator = express_shipping
print(order.get_shipping_cost())
```

---

## Strategy vs Other Patterns

| Pattern | Purpose | Key Difference |
|---------|---------|----------------|
| **Strategy** | Swap algorithms | Client chooses strategy |
| **State** | Change behavior by state | State determines behavior |
| **Template Method** | Define algorithm skeleton | Inheritance-based |
| **Command** | Encapsulate request | Focus on action/undo |

---

## When to Use

### ✅ Good Fit:
- Multiple algorithms for same task
- Need to switch algorithms at runtime
- Avoid conditional statements for algorithm selection
- Algorithms have different trade-offs

### ❌ Avoid When:
- Only one algorithm exists
- Algorithms rarely change
- Simple conditional is clearer

---

## Interview Tips

When discussing Strategy Pattern:
1. Explain the "family of algorithms" concept
2. Show how it eliminates conditionals
3. Give real examples (payment, shipping, sorting)
4. Mention runtime swapping capability
5. Compare with State pattern

