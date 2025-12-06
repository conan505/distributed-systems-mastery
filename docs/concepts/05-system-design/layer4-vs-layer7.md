# Layer 4 vs Layer 7 Load Balancing

## What They Are

**Layer 4 (L4)** load balancing operates at the transport layer (TCP/UDP), routing based on IP and port. **Layer 7 (L7)** operates at the application layer, making routing decisions based on content like HTTP headers, URLs, and cookies.

---

## The Analogy 📬

**Layer 4** = Post office sorting by ZIP code:
- Look at destination address only
- Fast, simple routing
- Don't open the package

**Layer 7** = Mail room sorting by department:
- Open and read the contents
- Route to specific desk/person
- More intelligent, more work

---

## OSI Layer Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                    OSI Model Layers                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Layer 7: Application   ← L7 Load Balancing (HTTP, gRPC)       │
│   Layer 6: Presentation                                         │
│   Layer 5: Session                                              │
│   Layer 4: Transport     ← L4 Load Balancing (TCP, UDP)         │
│   Layer 3: Network       (IP)                                   │
│   Layer 2: Data Link     (Ethernet)                             │
│   Layer 1: Physical      (Cables)                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Layer 4 Load Balancing

### How It Works
```
Client                   L4 LB                    Servers
  │                        │                         │
  │──── TCP SYN ──────────►│                         │
  │     (IP: X, Port: 80)  │                         │
  │                        │── Forward (NAT) ───────►│ Server A
  │◄─────────────── TCP SYN-ACK ─────────────────────│
  │──── Data ─────────────►│── Forward ─────────────►│
  │◄──────────────── Response ───────────────────────│

L4 sees: Source IP, Dest IP, Source Port, Dest Port
L4 doesn't see: HTTP headers, URL, cookies, body
```

### L4 Algorithms
```python
class L4LoadBalancer:
    def __init__(self, backends):
        self.backends = backends
        self.connections = {}  # For connection persistence
    
    def select_backend_round_robin(self):
        """Simple round-robin."""
        backend = self.backends[self.current_index % len(self.backends)]
        self.current_index += 1
        return backend
    
    def select_backend_hash(self, client_ip, client_port):
        """Consistent hashing based on connection tuple."""
        key = f"{client_ip}:{client_port}"
        hash_val = hash(key) % len(self.backends)
        return self.backends[hash_val]
    
    def select_backend_least_conn(self):
        """Route to backend with fewest connections."""
        return min(self.backends, key=lambda b: b.active_connections)
```

### L4 Characteristics
```
Pros:
+ Very fast (kernel-level, minimal processing)
+ Protocol agnostic (any TCP/UDP traffic)
+ Lower latency
+ Simpler to configure
+ Hardware acceleration possible (DPDK, XDP)

Cons:
- No content-based routing
- No HTTP-specific features
- Limited session persistence options
- Can't modify request/response
```

---

## Layer 7 Load Balancing

### How It Works
```
Client                   L7 LB                    Servers
  │                        │                         │
  │──── TCP SYN ──────────►│                         │
  │◄─── TCP SYN-ACK ───────│                         │
  │──── HTTP GET /api ────►│                         │
  │                        │ Parse HTTP              │
  │                        │ Check headers           │
  │                        │ Route decision          │
  │                        │                         │
  │                        │── New TCP ─────────────►│ API Server
  │                        │── HTTP GET /api ───────►│
  │◄──────────────── HTTP Response ──────────────────│

L7 sees: Full HTTP request - headers, URL, cookies, body
```

### L7 Routing Examples
```yaml
# Nginx L7 routing example
upstream api_servers {
    server api1:8080;
    server api2:8080;
}

upstream web_servers {
    server web1:80;
    server web2:80;
}

server {
    listen 80;
    
    # Route by URL path
    location /api {
        proxy_pass http://api_servers;
    }
    
    location /static {
        proxy_pass http://cdn_servers;
    }
    
    # Route by header
    location / {
        if ($http_x_api_version = "v2") {
            proxy_pass http://api_v2_servers;
        }
        proxy_pass http://web_servers;
    }
}
```

### L7 Capabilities
```python
class L7LoadBalancer:
    def route_request(self, request):
        # URL-based routing
        if request.path.startswith('/api/v2'):
            return self.api_v2_pool
        elif request.path.startswith('/api'):
            return self.api_v1_pool
        
        # Header-based routing
        if request.headers.get('X-Mobile-App'):
            return self.mobile_pool
        
        # Cookie-based session affinity
        session_id = request.cookies.get('session_id')
        if session_id:
            return self.get_sticky_server(session_id)
        
        # A/B testing
        if hash(request.headers.get('User-Id', '')) % 100 < 10:
            return self.canary_pool
        
        return self.default_pool
```

### L7 Characteristics
```
Pros:
+ Content-aware routing (URL, headers, cookies)
+ SSL/TLS termination
+ Request/response modification
+ Caching capabilities
+ Compression
+ Authentication/authorization
+ Rate limiting per endpoint
+ WebSocket support
+ Health checks at application level

Cons:
- Higher latency (must parse application data)
- More CPU intensive
- Must understand protocol (HTTP, gRPC, etc.)
- More complex configuration
- Terminates and re-establishes connections
```

---

## Comparison Table

```
┌────────────────────────────────────────────────────────────────┐
│          Layer 4 vs Layer 7 Comparison                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Feature          │ Layer 4        │ Layer 7                  │
│   ─────────────────│────────────────│──────────────────────────│
│   Speed            │ Fastest        │ Slower                   │
│   CPU usage        │ Minimal        │ Higher                   │
│   Routing          │ IP/Port only   │ Content-based            │
│   SSL termination  │ No (passthru)  │ Yes                      │
│   Health checks    │ TCP/ping       │ HTTP/app-level           │
│   Session stick    │ IP-based       │ Cookie-based             │
│   Caching          │ No             │ Yes                      │
│   Modify request   │ No             │ Yes                      │
│   Protocol aware   │ No             │ Yes                      │
│   Use case         │ High throughput│ Smart routing            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## When to Use Each

### Use Layer 4 When:
```
✓ Maximum performance needed
✓ Simple TCP/UDP load balancing
✓ Protocol doesn't matter (any TCP)
✓ Don't need content inspection
✓ Database connection pooling
✓ Gaming servers
✓ DNS load balancing
```

### Use Layer 7 When:
```
✓ HTTP/HTTPS services
✓ Need URL-based routing
✓ SSL termination required
✓ Session affinity by cookie
✓ A/B testing, canary deploys
✓ API gateway functionality
✓ WebSocket routing
✓ Need request/response modification
```

---

## Common Tools

| Layer | Tools |
|-------|-------|
| **L4** | HAProxy (TCP mode), AWS NLB, Linux IPVS, Envoy |
| **L7** | Nginx, HAProxy (HTTP mode), AWS ALB, Envoy, Traefik |
| **Both** | HAProxy, Envoy, F5 |

---

## Interview Tips

When discussing L4 vs L7:
1. Explain OSI layer difference clearly
2. Know performance vs features trade-off
3. Give routing examples for L7
4. Discuss SSL termination placement
5. Know when to use each (database vs API)

