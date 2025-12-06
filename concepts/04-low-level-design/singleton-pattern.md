# Singleton Pattern

## What It Is

The **Singleton Pattern** ensures a class has only one instance and provides a global point of access to it.

---

## The Analogy 🏛️

Think of a **country's president**:
- Only one president at a time
- Everyone refers to the same person
- Can't create a second president
- Global access: "The President said..."

---

## Why It Exists

### The Problem: Multiple Instances
```python
# Without Singleton
db1 = DatabaseConnection()  # Opens connection
db2 = DatabaseConnection()  # Opens another connection!
db3 = DatabaseConnection()  # And another!
# 100 requests = 100 connections = resource exhaustion
```

### What Singleton Solves:
- **Single instance** - One object for entire application
- **Global access** - Access from anywhere
- **Lazy initialization** - Create only when needed
- **Resource management** - Shared expensive resources

---

## Implementation

### Basic Singleton (Python)
```python
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Usage
s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True - same instance
```

### Thread-Safe Singleton
```python
import threading

class ThreadSafeSingleton:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                # Double-check locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

### Singleton with Decorator
```python
def singleton(cls):
    instances = {}
    
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    
    return get_instance

@singleton
class Database:
    def __init__(self):
        self.connection = self._connect()
```

### Module-Level Singleton (Pythonic)
```python
# database.py
class _Database:
    def __init__(self):
        self.connection = self._connect()

# Module-level instance
db = _Database()

# Usage: from database import db
# Python modules are singletons by default!
```

---

## Real-World Examples

### 1. Database Connection Pool
```python
class ConnectionPool:
    _instance = None
    
    def __new__(cls, max_connections=10):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.pool = []
            cls._instance.max = max_connections
        return cls._instance
    
    def get_connection(self):
        if self.pool:
            return self.pool.pop()
        return self._create_connection()
    
    def release(self, conn):
        if len(self.pool) < self.max:
            self.pool.append(conn)
```

### 2. Logger
```python
class Logger:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.log_file = open("app.log", "a")
        return cls._instance
    
    def log(self, message):
        self.log_file.write(f"{datetime.now()}: {message}\n")
```

### 3. Configuration Manager
```python
class Config:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._load_config()
        return cls._instance
    
    def _load_config(self):
        with open("config.yaml") as f:
            self.settings = yaml.safe_load(f)
    
    def get(self, key):
        return self.settings.get(key)
```

---

## Singleton Anti-Patterns

### ❌ Global State Problems
```python
# Singleton can become a dumping ground
class AppState:
    user = None
    cart = None
    settings = None
    cache = None
    # Everything global = hard to test, reason about
```

### ❌ Hidden Dependencies
```python
def process_order(order):
    db = Database()  # Hidden dependency!
    db.save(order)

# Better: Explicit dependency
def process_order(order, db: Database):
    db.save(order)
```

### ❌ Testing Difficulties
```python
# Hard to test with real singleton
def test_order_processing():
    process_order(order)  # Uses real database!

# Solution: Dependency injection
def test_order_processing():
    mock_db = MockDatabase()
    process_order(order, mock_db)  # Uses mock
```

---

## Singleton vs Dependency Injection

| Aspect | Singleton | DI |
|--------|-----------|-----|
| **Access** | Global | Explicit |
| **Testing** | Difficult | Easy |
| **Coupling** | High | Low |
| **Flexibility** | Low | High |

### Modern Approach: DI Container
```python
# Instead of Singleton, use DI container
container = Container()
container.register(Database, scope="singleton")

# Inject where needed
class OrderService:
    def __init__(self, db: Database):  # Injected
        self.db = db
```

---

## When to Use

### ✅ Good Fit:
- **Connection pools** - Shared resource
- **Loggers** - Single log destination
- **Configuration** - App-wide settings
- **Caches** - Shared cache instance

### ❌ Avoid When:
- Can use dependency injection instead
- Need multiple configurations (test vs prod)
- Object has mutable state that varies
- Makes testing difficult

---

## Interview Tips

When discussing Singleton:
1. Explain the "one instance" guarantee
2. Show thread-safe implementation
3. Discuss the testing problem
4. Mention DI as modern alternative
5. Give valid use cases (pools, loggers)

