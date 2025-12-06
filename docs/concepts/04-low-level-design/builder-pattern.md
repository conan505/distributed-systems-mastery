# Builder Pattern

## What It Is

The **Builder Pattern** separates the construction of a complex object from its representation, allowing the same construction process to create different representations.

---

## The Analogy 🍔

Think of **ordering a custom burger**:
- Start with base (bun)
- Add patty (beef, chicken, veggie)
- Add toppings (lettuce, tomato, cheese)
- Add sauce (ketchup, mayo, special)
- Build step by step, get custom result

---

## Why It Exists

### The Problem: Telescoping Constructors
```python
# BAD: Too many constructor parameters
class Pizza:
    def __init__(self, size, cheese, pepperoni, mushrooms, 
                 onions, bacon, olives, extra_cheese, 
                 thin_crust, stuffed_crust):
        # 10 parameters! Hard to remember order
        pass

# Calling is error-prone
pizza = Pizza("large", True, True, False, True, False, 
              True, False, True, False)  # What's what?
```

### What Builder Solves:
- **Readable construction** - Named methods instead of positional args
- **Optional parameters** - Only set what you need
- **Immutable objects** - Build then freeze
- **Validation** - Validate before building

---

## Implementation

### Basic Builder
```python
class Pizza:
    def __init__(self):
        self.size = "medium"
        self.cheese = False
        self.pepperoni = False
        self.mushrooms = False

class PizzaBuilder:
    def __init__(self):
        self._pizza = Pizza()
    
    def size(self, size: str):
        self._pizza.size = size
        return self  # Enable chaining
    
    def add_cheese(self):
        self._pizza.cheese = True
        return self
    
    def add_pepperoni(self):
        self._pizza.pepperoni = True
        return self
    
    def add_mushrooms(self):
        self._pizza.mushrooms = True
        return self
    
    def build(self) -> Pizza:
        return self._pizza

# Usage - fluent interface
pizza = (PizzaBuilder()
         .size("large")
         .add_cheese()
         .add_pepperoni()
         .build())
```

### Builder with Validation
```python
class HttpRequestBuilder:
    def __init__(self):
        self._method = None
        self._url = None
        self._headers = {}
        self._body = None
    
    def method(self, method: str):
        if method not in ["GET", "POST", "PUT", "DELETE"]:
            raise ValueError(f"Invalid method: {method}")
        self._method = method
        return self
    
    def url(self, url: str):
        if not url.startswith("http"):
            raise ValueError("URL must start with http")
        self._url = url
        return self
    
    def header(self, key: str, value: str):
        self._headers[key] = value
        return self
    
    def body(self, body: str):
        self._body = body
        return self
    
    def build(self) -> HttpRequest:
        if not self._method or not self._url:
            raise ValueError("Method and URL are required")
        return HttpRequest(
            self._method, 
            self._url, 
            self._headers, 
            self._body
        )

# Usage
request = (HttpRequestBuilder()
           .method("POST")
           .url("https://api.example.com/users")
           .header("Content-Type", "application/json")
           .body('{"name": "John"}')
           .build())
```

---

## Real-World Examples

### 1. SQL Query Builder
```python
class QueryBuilder:
    def __init__(self):
        self._select = "*"
        self._from = None
        self._where = []
        self._order_by = None
        self._limit = None
    
    def select(self, *columns):
        self._select = ", ".join(columns)
        return self
    
    def from_table(self, table: str):
        self._from = table
        return self
    
    def where(self, condition: str):
        self._where.append(condition)
        return self
    
    def order_by(self, column: str, desc=False):
        self._order_by = f"{column} {'DESC' if desc else 'ASC'}"
        return self
    
    def limit(self, n: int):
        self._limit = n
        return self
    
    def build(self) -> str:
        query = f"SELECT {self._select} FROM {self._from}"
        if self._where:
            query += " WHERE " + " AND ".join(self._where)
        if self._order_by:
            query += f" ORDER BY {self._order_by}"
        if self._limit:
            query += f" LIMIT {self._limit}"
        return query

# Usage
query = (QueryBuilder()
         .select("id", "name", "email")
         .from_table("users")
         .where("age > 18")
         .where("status = 'active'")
         .order_by("created_at", desc=True)
         .limit(10)
         .build())
# SELECT id, name, email FROM users WHERE age > 18 AND status = 'active' ORDER BY created_at DESC LIMIT 10
```

### 2. Test Data Builder
```python
class UserBuilder:
    def __init__(self):
        self._user = User()
        # Sensible defaults
        self._user.name = "Test User"
        self._user.email = "test@example.com"
        self._user.role = "user"
    
    def with_name(self, name: str):
        self._user.name = name
        return self
    
    def with_email(self, email: str):
        self._user.email = email
        return self
    
    def as_admin(self):
        self._user.role = "admin"
        return self
    
    def build(self) -> User:
        return self._user

# In tests - readable and maintainable
admin = UserBuilder().with_name("Admin").as_admin().build()
regular = UserBuilder().with_email("user@test.com").build()
```

---

## Builder vs Other Patterns

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| **Builder** | Complex construction | Many optional params |
| **Factory** | Create objects | Hide concrete classes |
| **Prototype** | Clone objects | Copy existing objects |

---

## When to Use

### ✅ Good Fit:
- Many constructor parameters
- Many optional parameters
- Complex object construction
- Need immutable objects
- Test data creation

### ❌ Avoid When:
- Simple objects (few params)
- All parameters required
- No optional configuration

---

## Interview Tips

When discussing Builder Pattern:
1. Explain the telescoping constructor problem
2. Show fluent interface with method chaining
3. Mention validation in build()
4. Give real examples (SQL, HTTP, test data)
5. Compare with Factory pattern

