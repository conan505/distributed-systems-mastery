# Factory Pattern

## What It Is

The **Factory Pattern** provides an interface for creating objects without specifying their exact classes. It delegates the instantiation logic to subclasses or factory methods.

---

## The Analogy 🏭

Think of a **pizza restaurant**:
- You order "Margherita" or "Pepperoni"
- Kitchen (factory) creates the right pizza
- You don't know the exact recipe or process
- Just get the pizza you ordered

---

## Why It Exists

### The Problem: Tight Coupling
```python
# BAD: Client knows all concrete classes
def process_payment(payment_type, amount):
    if payment_type == "credit":
        processor = CreditCardProcessor()
    elif payment_type == "paypal":
        processor = PayPalProcessor()
    elif payment_type == "crypto":
        processor = CryptoProcessor()
    # Adding new type = modify this code!
    
    processor.process(amount)
```

### What Factory Solves:
- **Decoupling** - Client doesn't know concrete classes
- **Single Responsibility** - Creation logic in one place
- **Open/Closed** - Add new types without modifying existing code
- **Testability** - Easy to mock factories

---

## Types of Factory Patterns

### 1. Simple Factory
```python
class PaymentProcessorFactory:
    @staticmethod
    def create(payment_type: str) -> PaymentProcessor:
        if payment_type == "credit":
            return CreditCardProcessor()
        elif payment_type == "paypal":
            return PayPalProcessor()
        elif payment_type == "crypto":
            return CryptoProcessor()
        raise ValueError(f"Unknown type: {payment_type}")

# Usage
processor = PaymentProcessorFactory.create("credit")
processor.process(100)
```

### 2. Factory Method
```python
from abc import ABC, abstractmethod

class PaymentProcessorFactory(ABC):
    @abstractmethod
    def create_processor(self) -> PaymentProcessor:
        pass
    
    def process_payment(self, amount):
        processor = self.create_processor()
        return processor.process(amount)

class CreditCardFactory(PaymentProcessorFactory):
    def create_processor(self):
        return CreditCardProcessor()

class PayPalFactory(PaymentProcessorFactory):
    def create_processor(self):
        return PayPalProcessor()

# Usage
factory = CreditCardFactory()
factory.process_payment(100)
```

### 3. Abstract Factory
```python
class UIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: pass
    
    @abstractmethod
    def create_checkbox(self) -> Checkbox: pass

class WindowsFactory(UIFactory):
    def create_button(self):
        return WindowsButton()
    
    def create_checkbox(self):
        return WindowsCheckbox()

class MacFactory(UIFactory):
    def create_button(self):
        return MacButton()
    
    def create_checkbox(self):
        return MacCheckbox()

# Usage: Create family of related objects
factory = WindowsFactory()  # or MacFactory()
button = factory.create_button()
checkbox = factory.create_checkbox()
```

---

## Pattern Comparison

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| **Simple Factory** | Centralize creation | Few types, simple logic |
| **Factory Method** | Defer to subclasses | Need inheritance hierarchy |
| **Abstract Factory** | Create families | Related objects together |

---

## Real-World Examples

### Database Connections
```python
class DatabaseFactory:
    @staticmethod
    def create(db_type: str, config: dict) -> Database:
        if db_type == "postgres":
            return PostgresConnection(config)
        elif db_type == "mysql":
            return MySQLConnection(config)
        elif db_type == "mongodb":
            return MongoConnection(config)
```

### Logger Factory
```python
class LoggerFactory:
    @staticmethod
    def get_logger(name: str, level: str = "INFO") -> Logger:
        logger = Logger(name)
        if level == "DEBUG":
            logger.add_handler(ConsoleHandler())
        else:
            logger.add_handler(FileHandler())
        return logger
```

### Document Parser
```python
class DocumentParserFactory:
    _parsers = {
        ".pdf": PDFParser,
        ".docx": WordParser,
        ".txt": TextParser,
    }
    
    @classmethod
    def create(cls, filename: str) -> DocumentParser:
        ext = os.path.splitext(filename)[1]
        parser_class = cls._parsers.get(ext)
        if not parser_class:
            raise ValueError(f"Unsupported format: {ext}")
        return parser_class()
```

---

## Factory with Registry

```python
class PaymentProcessorFactory:
    _registry = {}
    
    @classmethod
    def register(cls, name: str, processor_class):
        cls._registry[name] = processor_class
    
    @classmethod
    def create(cls, name: str) -> PaymentProcessor:
        processor_class = cls._registry.get(name)
        if not processor_class:
            raise ValueError(f"Unknown processor: {name}")
        return processor_class()

# Registration (can be in separate modules)
PaymentProcessorFactory.register("credit", CreditCardProcessor)
PaymentProcessorFactory.register("paypal", PayPalProcessor)

# Usage
processor = PaymentProcessorFactory.create("credit")
```

---

## When to Use

### ✅ Good Fit:
- Object creation is complex
- Need to decouple client from concrete classes
- Want to centralize creation logic
- Creating families of related objects

### ❌ Avoid When:
- Simple object creation (just use constructor)
- Only one implementation exists
- Over-engineering simple code

---

## Common Mistakes

```python
# ❌ Factory that just wraps constructor
class UserFactory:
    def create(self, name):
        return User(name)  # Pointless!

# ❌ God factory that creates everything
class Factory:
    def create_user(self): ...
    def create_order(self): ...
    def create_payment(self): ...
    # Too many responsibilities!

# ✅ Focused factory with clear purpose
class PaymentProcessorFactory:
    def create(self, type): ...  # Only payment processors
```

---

## Interview Tips

When discussing Factory Pattern:
1. Explain the decoupling benefit
2. Know Simple vs Factory Method vs Abstract Factory
3. Give real examples (DB connections, parsers)
4. Mention registry pattern for extensibility
5. Discuss when NOT to use it

