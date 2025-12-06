# Load Balancing

## What It Is

**Load Balancing** distributes incoming network traffic across multiple servers to ensure no single server is overwhelmed, improving availability, responsiveness, and reliability.

---

## The Analogy 🏪

Think of **checkout lanes at a supermarket**:
- Multiple lanes available
- Customers directed to shortest line
- If one lane closes, others handle traffic
- Rush hour = open more lanes

---

## Why It Exists

### The Problem: Single Server Limits
```
Single server:
- 1000 requests/sec capacity
- Traffic spike: 5000 requests/sec
- Result: Slow responses or crashes

With 5 servers + load balancer:
- Each handles 1000 req/sec
- Total capacity: 5000 req/sec
- One fails? Others absorb traffic
```

### What Load Balancing Solves:
- **Scalability** - Add servers as needed
- **Availability** - Survive server failures
- **Performance** - Reduce response times
- **Maintenance** - Roll out updates gracefully

---

## Load Balancing Algorithms

### 1. Round Robin
```
Server A → Server B → Server C → Server A → ...

✅ Simple, fair distribution
❌ Ignores server capacity/load
❌ Session affinity issues
```

### 2. Weighted Round Robin
```
Server A (weight=5): handles 5 requests
Server B (weight=2): handles 2 requests
Server C (weight=3): handles 3 requests

Ratio: A:B:C = 5:2:3
Use when servers have different capacities
```

### 3. Least Connections
```
Route to server with fewest active connections

Server A: 50 connections  ← New request goes here
Server B: 100 connections
Server C: 75 connections

✅ Adapts to actual load
❌ Doesn't consider request complexity
```

### 4. IP Hash
```
hash(client_ip) % num_servers

Same client → Same server (session affinity)

✅ Session persistence
❌ Uneven if client IPs not distributed
```

### 5. Least Response Time
```
Route to server with fastest response + fewest connections

✅ Considers actual performance
❌ Requires monitoring overhead
```

---

## Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer):
```
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 4 Load Balancer                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Routes based on: IP address, TCP/UDP port                     │
│   Does NOT inspect: HTTP headers, cookies, URLs                 │
│                                                                  │
│   Client ──TCP──▶ LB ──TCP──▶ Server                            │
│                                                                  │
│   ✅ Fast (no packet inspection)                                 │
│   ✅ Protocol agnostic                                           │
│   ❌ No content-based routing                                    │
│   ❌ Limited health checks                                       │
│                                                                  │
│   Examples: AWS NLB, HAProxy (TCP mode)                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Layer 7 (Application Layer):
```
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 7 Load Balancer                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Routes based on: URL path, headers, cookies, content          │
│                                                                  │
│   /api/* ──────▶ API servers                                    │
│   /static/* ───▶ CDN/cache                                      │
│   /admin/* ────▶ Admin servers                                  │
│                                                                  │
│   ✅ Content-based routing                                       │
│   ✅ SSL termination                                             │
│   ✅ Rich health checks                                          │
│   ❌ Higher latency                                              │
│   ❌ More resource intensive                                     │
│                                                                  │
│   Examples: AWS ALB, NGINX, HAProxy (HTTP mode)                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Health Checks

```
Types:
1. Passive: Monitor real traffic for errors
2. Active: Periodically probe servers

Active health check example:
GET /health every 5 seconds
- 200 OK → Server healthy
- 500/timeout → Mark unhealthy
- 3 failures → Remove from pool
- 2 successes → Add back to pool
```

---

## Session Persistence (Sticky Sessions)

### Problem:
```
Request 1 → Server A (creates session)
Request 2 → Server B (no session found!)
```

### Solutions:
```
1. Cookie-based: LB injects cookie with server ID
2. IP hash: Same IP → same server
3. Session store: Externalize sessions (Redis)
```

---

## When to Use Each Type

| Use Case | L4 | L7 |
|----------|----|----|
| Simple TCP/UDP services | ✅ | ❌ |
| gRPC, WebSocket | ✅ | ✅ |
| HTTP routing by path | ❌ | ✅ |
| SSL termination | ❌ | ✅ |
| Ultra-low latency | ✅ | ❌ |
| A/B testing | ❌ | ✅ |

---

## Real-World Tools

| Tool | Type | Notes |
|------|------|-------|
| **NGINX** | L7 | Most popular reverse proxy |
| **HAProxy** | L4/L7 | High performance |
| **AWS ALB** | L7 | Application Load Balancer |
| **AWS NLB** | L4 | Network Load Balancer |
| **Envoy** | L7 | Service mesh proxy |
| **Traefik** | L7 | Cloud-native, auto-config |

---

## Global Load Balancing (GSLB)

```
DNS-based routing across regions:

User in Europe → EU datacenter
User in Asia → APAC datacenter

Methods:
- GeoDNS: Route by client location
- Anycast: Same IP, nearest datacenter
- Latency-based: Measure and route to fastest
```

---

## Interview Tips

When discussing Load Balancing:
1. Explain L4 vs L7 differences
2. Know common algorithms (round robin, least connections)
3. Discuss health checks and failover
4. Mention session persistence challenges
5. Give real examples (NGINX, ALB, HAProxy)

