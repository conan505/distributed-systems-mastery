# Low-Level Design Patterns Overview

## What It Is

**Low-Level Design (LLD)** focuses on the detailed design of individual components, classes, and their interactions. It involves applying design patterns to create maintainable, extensible, and testable code.

---

## The Analogy 🏗️

If **High-Level Design** is the building's blueprint (floors, rooms, plumbing routes), **Low-Level Design** is the detailed specifications (door handles, electrical outlets, cabinet designs).

---

## Why LLD Matters

### The Problem: Spaghetti Code
```
Without patterns:
- Tightly coupled classes
- Hard to test
- Difficult to extend
- Copy-paste everywhere
```

### What Good LLD Provides:
- **Maintainability** - Easy to modify
- **Testability** - Easy to unit test
- **Extensibility** - Easy to add features
- **Reusability** - Components work in multiple contexts

---

## SOLID Principles

### S - Single Responsibility
```python
# BAD: One class does everything
class User:
    def save_to_db(self): ...
    def send_email(self): ...
    def generate_report(self): ...

# GOOD: Each class has one job
class User: ...
class UserRepository: def save(user): ...
class EmailService: def send(user, message): ...
class ReportGenerator: def generate(user): ...
```

### O - Open/Closed
```python
# Open for extension, closed for modification

# BAD: Modify existing code for new shapes
def area(shape):
    if shape.type == "circle": return pi * r^2
    if shape.type == "square": return s^2
    # Add new shape = modify this function

# GOOD: Extend without modifying
class Shape:
    def area(self): pass

class Circle(Shape):
    def area(self): return pi * self.r ** 2

class Square(Shape):
    def area(self): return self.s ** 2
```

### L - Liskov Substitution
```python
# Subtypes must be substitutable for base types

# BAD: Square breaks Rectangle contract
class Rectangle:
    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

class Square(Rectangle):  # Violates LSP!
    def set_width(self, w): 
        self.width = self.height = w  # Unexpected!
```

### I - Interface Segregation
```python
# Many specific interfaces > one general interface

# BAD: Fat interface
class Worker:
    def work(self): ...
    def eat(self): ...
    def sleep(self): ...

# GOOD: Segregated interfaces
class Workable: def work(self): ...
class Eatable: def eat(self): ...
class Sleepable: def sleep(self): ...
```

### D - Dependency Inversion
```python
# Depend on abstractions, not concretions

# BAD: High-level depends on low-level
class OrderService:
    def __init__(self):
        self.db = MySQLDatabase()  # Concrete!

# GOOD: Depend on abstraction
class OrderService:
    def __init__(self, db: Database):  # Abstract!
        self.db = db
```

---

## Creational Patterns

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Singleton** | One instance globally | Database connection pool |
| **Factory** | Create objects without specifying class | Payment processor factory |
| **Builder** | Construct complex objects step by step | Query builder, HTTP request builder |
| **Prototype** | Clone existing objects | Game character templates |

---

## Structural Patterns

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Adapter** | Convert interface to another | Legacy API wrapper |
| **Decorator** | Add behavior dynamically | Logging, caching wrappers |
| **Facade** | Simplify complex subsystem | Payment gateway facade |
| **Proxy** | Control access to object | Lazy loading, access control |

---

## Behavioral Patterns

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Strategy** | Swap algorithms at runtime | Sorting, pricing strategies |
| **Observer** | Notify dependents of changes | Event listeners, pub/sub |
| **Command** | Encapsulate request as object | Undo/redo, task queues |
| **State** | Change behavior based on state | Order status, game states |
| **Chain of Responsibility** | Pass request along chain | Middleware, validators |

---

## Common LLD Interview Problems

### 1. Parking Lot
```
Classes: ParkingLot, Floor, Spot, Vehicle, Ticket
Patterns: Factory (vehicle types), Strategy (pricing)
```

### 2. Elevator System
```
Classes: Elevator, Floor, Request, Scheduler
Patterns: State (elevator states), Strategy (scheduling)
```

### 3. Library Management
```
Classes: Library, Book, Member, Loan, Fine
Patterns: Observer (due date notifications)
```

### 4. Rate Limiter
```
Classes: RateLimiter, TokenBucket, SlidingWindow
Patterns: Strategy (algorithm selection)
```

### 5. Cache (LRU)
```
Classes: Cache, Node, DoublyLinkedList
Data structures: HashMap + DLL
```

---

## LLD Interview Approach

```
1. Clarify requirements (5 min)
   - What entities exist?
   - What operations needed?
   - What constraints?

2. Identify classes and relationships (10 min)
   - Nouns → Classes
   - Verbs → Methods
   - Draw class diagram

3. Define interfaces and contracts (5 min)
   - Public methods
   - Input/output types

4. Apply patterns where appropriate (5 min)
   - Don't force patterns
   - Explain why you chose them

5. Write key code (15 min)
   - Core classes
   - Important methods
   - Handle edge cases
```

---

## Interview Tips

When doing LLD:
1. Start with requirements clarification
2. Identify entities and relationships first
3. Apply SOLID principles
4. Use patterns where they fit naturally
5. Write clean, readable code
6. Consider extensibility

