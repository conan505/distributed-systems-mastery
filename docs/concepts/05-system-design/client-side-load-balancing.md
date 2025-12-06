# Client-Side Load Balancing

## What It Is

**Client-Side Load Balancing** is a pattern where the client maintains a list of available servers and decides which server to send each request to, rather than relying on a central load balancer.

---

## The Analogy 🎯

Think of **choosing a checkout lane at a grocery store**:
- You see all the lanes (service discovery)
- You pick the shortest one (load balancing decision)
- No central person directing you (no proxy)
- You make the choice yourself (client-side)

---

## Why It Exists

### Traditional (Server-Side) Load Balancing
```
┌─────────────────────────────────────────────────────────────────┐
│                Server-Side Load Balancer                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [Client] ──▶ [Load Balancer] ──▶ [Server 1]                   │
│                      │                                           │
│                      ├──────────▶ [Server 2]                    │
│                      │                                           │
│                      └──────────▶ [Server 3]                    │
│                                                                  │
│   Problems:                                                      │
│   - Single point of failure                                     │
│   - Extra network hop (latency)                                 │
│   - Scalability bottleneck                                      │
│   - Extra cost (hardware/cloud LB)                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Client-Side Load Balancing
```
┌─────────────────────────────────────────────────────────────────┐
│                Client-Side Load Balancing                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [Service Registry]                                            │
│         │                                                        │
│         ▼ (discover servers)                                    │
│   ┌──────────┐                                                  │
│   │  Client  │ ──────────────▶ [Server 1]                       │
│   │  (with   │                                                  │
│   │   LB)    │ ──────────────▶ [Server 2]                       │
│   │          │                                                  │
│   └──────────┘ ──────────────▶ [Server 3]                       │
│                                                                  │
│   Benefits:                                                      │
│   - No SPOF                                                     │
│   - No extra hop                                                │
│   - Scales with clients                                         │
│   - Smarter routing decisions                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Basic Client-Side Load Balancer
```python
import random
from typing import List, Optional
import time

class ServiceInstance:
    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.healthy = True
        self.weight = 1
        self.active_requests = 0
        self.response_times = []

class ClientLoadBalancer:
    def __init__(self, service_registry):
        self.registry = service_registry
        self.instances: List[ServiceInstance] = []
        self.current_index = 0
    
    def refresh_instances(self):
        """Get latest instances from service registry."""
        self.instances = self.registry.get_instances()
    
    def get_healthy_instances(self) -> List[ServiceInstance]:
        return [i for i in self.instances if i.healthy]
    
    # Load balancing strategies below...
```

### Round Robin
```python
def round_robin(self) -> Optional[ServiceInstance]:
    """Simple rotation through instances."""
    healthy = self.get_healthy_instances()
    if not healthy:
        return None
    
    instance = healthy[self.current_index % len(healthy)]
    self.current_index += 1
    return instance
```

### Weighted Round Robin
```python
def weighted_round_robin(self) -> Optional[ServiceInstance]:
    """Distribute based on weights."""
    healthy = self.get_healthy_instances()
    if not healthy:
        return None
    
    total_weight = sum(i.weight for i in healthy)
    point = random.uniform(0, total_weight)
    
    current = 0
    for instance in healthy:
        current += instance.weight
        if current >= point:
            return instance
    
    return healthy[-1]
```

### Least Connections
```python
def least_connections(self) -> Optional[ServiceInstance]:
    """Route to instance with fewest active requests."""
    healthy = self.get_healthy_instances()
    if not healthy:
        return None
    
    return min(healthy, key=lambda i: i.active_requests)
```

### Least Response Time
```python
def least_response_time(self) -> Optional[ServiceInstance]:
    """Route to fastest responding instance."""
    healthy = self.get_healthy_instances()
    if not healthy:
        return None
    
    def avg_response_time(instance):
        if not instance.response_times:
            return float('inf')
        return sum(instance.response_times[-10:]) / len(instance.response_times[-10:])
    
    return min(healthy, key=avg_response_time)
```

---

## Health Checking

```python
class HealthChecker:
    def __init__(self, lb: ClientLoadBalancer, interval_seconds: float = 10):
        self.lb = lb
        self.interval = interval_seconds
    
    async def check_health(self, instance: ServiceInstance) -> bool:
        try:
            response = await http_client.get(
                f"http://{instance.host}:{instance.port}/health",
                timeout=5
            )
            return response.status_code == 200
        except Exception:
            return False
    
    async def run(self):
        while True:
            for instance in self.lb.instances:
                instance.healthy = await self.check_health(instance)
            await asyncio.sleep(self.interval)
```

---

## Real-World Implementations

| Library | Language | Features |
|---------|----------|----------|
| **Ribbon** | Java | Netflix, Spring Cloud |
| **gRPC** | Multi | Built-in client LB |
| **Envoy** | Sidecar | Service mesh pattern |
| **go-micro** | Go | Microservices toolkit |

### gRPC Example
```python
import grpc

# Client-side load balancing with round_robin
channel = grpc.insecure_channel(
    'dns:///my-service:50051',
    options=[
        ('grpc.lb_policy_name', 'round_robin'),
    ]
)
```

---

## Comparison

| Aspect | Client-Side | Server-Side |
|--------|-------------|-------------|
| **Latency** | Lower (no hop) | Higher |
| **Complexity** | In each client | Centralized |
| **Failure** | Distributed | SPOF risk |
| **Intelligence** | Can use local metrics | Limited visibility |
| **Deployment** | Update all clients | Update one place |

---

## Best Practices

```
1. Implement health checking
2. Use circuit breakers with LB
3. Cache service discovery results
4. Handle stale server lists gracefully
5. Implement retry with different server
6. Monitor load distribution
7. Use weighted algorithms for heterogeneous servers
```

---

## Interview Tips

When discussing Client-Side Load Balancing:
1. Compare with server-side LB (pros/cons)
2. Explain service discovery integration
3. Describe health checking importance
4. Discuss algorithm choices (round robin, least connections)
5. Mention gRPC and service mesh patterns

