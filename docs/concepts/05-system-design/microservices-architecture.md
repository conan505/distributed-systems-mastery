# Microservices Architecture

## What It Is

**Microservices Architecture** is a design approach where an application is built as a collection of small, independent services that communicate over a network, each responsible for a specific business capability.

---

## The Analogy 🏘️

Think of **a city vs a single building**:
- **Monolith**: One giant building with everything inside
- **Microservices**: City with specialized buildings (hospital, school, mall)
- Each building operates independently
- Connected by roads (network)

---

## Why It Exists

### The Problem: Monolith Limitations
```
Monolith issues at scale:
- Deploy entire app for small change
- One bug can crash everything
- Team coordination bottleneck
- Technology lock-in
- Scaling wastes resources (scale everything)
```

### What Microservices Solve:
- **Independent deployment** - Deploy one service without others
- **Fault isolation** - One service failure doesn't crash all
- **Team autonomy** - Teams own their services
- **Technology freedom** - Use best tool for each job
- **Targeted scaling** - Scale only what needs it

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                  Microservices Architecture                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Clients ──▶ [API Gateway] ──▶ [Load Balancer]                 │
│                     │                                            │
│         ┌──────────┼──────────┬──────────┐                      │
│         ▼          ▼          ▼          ▼                      │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│   │  User   │ │  Order  │ │ Payment │ │Inventory│              │
│   │ Service │ │ Service │ │ Service │ │ Service │              │
│   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘              │
│        │           │           │           │                     │
│   ┌────▼────┐ ┌────▼────┐ ┌────▼────┐ ┌────▼────┐              │
│   │ User DB │ │Order DB │ │Pay DB   │ │Inv DB   │              │
│   └─────────┘ └─────────┘ └─────────┘ └─────────┘              │
│                                                                  │
│   Each service:                                                  │
│   - Has its own database                                        │
│   - Deploys independently                                       │
│   - Scales independently                                        │
│   - Can use different tech stack                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Principles

### 1. Single Responsibility
```
Each service does ONE thing well:
- User Service: Authentication, profiles
- Order Service: Order lifecycle
- Payment Service: Payment processing
- Notification Service: Emails, SMS, push
```

### 2. Database per Service
```
❌ Shared database:
   Services coupled through data
   Schema changes affect all

✅ Database per service:
   Services own their data
   Communicate via APIs/events
```

### 3. API-First Design
```
Services communicate via:
- REST APIs (synchronous)
- Message queues (asynchronous)
- gRPC (high performance)
```

---

## Communication Patterns

### Synchronous (REST/gRPC)
```
Order Service ──HTTP──▶ Payment Service
                 │
                 ◀── Response ──

✅ Simple, immediate response
❌ Tight coupling, cascading failures
```

### Asynchronous (Events)
```
Order Service ──publish──▶ [Message Queue]
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        Payment Service  Inventory Service  Email Service

✅ Loose coupling, resilient
❌ Eventual consistency, complex debugging
```

---

## Essential Components

### API Gateway
```
- Single entry point
- Authentication/authorization
- Rate limiting
- Request routing
- Response aggregation
```

### Service Discovery
```
Services register themselves:
  User Service → Registry: "I'm at 10.0.0.5:8080"

Services find each other:
  Order Service → Registry: "Where is User Service?"
  Registry → Order Service: "10.0.0.5:8080"

Tools: Consul, Eureka, Kubernetes DNS
```

### Circuit Breaker
```
Prevent cascade failures:
- Monitor failures
- Open circuit when threshold reached
- Fail fast instead of waiting
- Periodically test if service recovered

See: circuit-breaker.md
```

---

## Challenges

### 1. Distributed Transactions
```
Problem: Order spans multiple services
Solution: Saga pattern (see saga-pattern.md)
```

### 2. Data Consistency
```
Problem: No ACID across services
Solution: Eventual consistency, event sourcing
```

### 3. Service Communication
```
Problem: Network is unreliable
Solution: Retries, timeouts, circuit breakers
```

### 4. Debugging
```
Problem: Request spans multiple services
Solution: Distributed tracing (Jaeger, Zipkin)
```

### 5. Testing
```
Problem: Integration testing is complex
Solution: Contract testing, service virtualization
```

---

## Microservices vs Monolith

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | All or nothing | Independent |
| **Scaling** | Entire app | Per service |
| **Technology** | Single stack | Polyglot |
| **Team size** | Large, coordinated | Small, autonomous |
| **Complexity** | In code | In infrastructure |
| **Latency** | In-process | Network calls |

---

## When to Use

### ✅ Good Fit:
- Large, complex applications
- Multiple teams
- Need independent scaling
- Different tech requirements
- Frequent deployments

### ❌ Avoid When:
- Small team/application
- Unclear domain boundaries
- Limited DevOps capability
- Low latency requirements

---

## Real-World Examples

| Company | Approach |
|---------|----------|
| **Netflix** | 700+ microservices |
| **Amazon** | Service-oriented since 2002 |
| **Uber** | Domain-oriented microservices |
| **Spotify** | Squad-based ownership |

---

## Interview Tips

When discussing Microservices:
1. Explain benefits over monolith
2. Discuss communication patterns
3. Know the challenges (transactions, consistency)
4. Mention supporting infrastructure
5. Discuss when NOT to use microservices

