# RPC vs REST

## What They Are

**REST** (Representational State Transfer) is an architectural style using HTTP methods to operate on resources. **RPC** (Remote Procedure Call) is a protocol for executing functions on remote servers as if they were local.

---

## The Analogy 📬

**REST** is like a **post office**:
- Send letters to addresses (resources)
- Standard operations (send, return, track)
- Stateless interactions

**RPC** is like a **phone call**:
- Call someone and ask them to do something
- Direct conversation (procedure call)
- More expressive communication

---

## REST Overview

### Resource-Oriented
```
REST: "What resources exist?"

GET    /users/123        → Read user
POST   /users            → Create user
PUT    /users/123        → Update user
DELETE /users/123        → Delete user

Resources are nouns, HTTP methods are verbs
```

### REST Characteristics
```
- Stateless: Each request contains all info
- Cacheable: Responses can be cached
- Uniform interface: Standard HTTP methods
- Resource-based: URLs identify resources
- Hypermedia: Links for navigation (HATEOAS)
```

---

## RPC Overview

### Action-Oriented
```
RPC: "What actions can I perform?"

CreateUser(name, email)       → Creates user
GetUser(user_id)              → Returns user
UpdateUserEmail(user_id, email)
DeactivateUser(user_id)

Functions/procedures, not resources
```

### RPC Styles
```
┌─────────────────────────────────────────────────────────────────┐
│                      RPC Implementations                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   JSON-RPC:                                                     │
│   POST /rpc                                                     │
│   {"method": "getUser", "params": [123], "id": 1}              │
│                                                                  │
│   gRPC (Protocol Buffers):                                      │
│   Binary, HTTP/2, streaming support                             │
│                                                                  │
│   XML-RPC / SOAP:                                               │
│   Legacy, verbose XML format                                    │
│                                                                  │
│   Apache Thrift:                                                │
│   Cross-language, binary protocol                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Comparison

| Aspect | REST | RPC (gRPC) |
|--------|------|------------|
| **Style** | Resource-oriented | Action-oriented |
| **Protocol** | HTTP/1.1, HTTP/2 | HTTP/2 |
| **Format** | JSON, XML | Protobuf (binary) |
| **Performance** | Good | Better (binary, HTTP/2) |
| **Browser support** | Native | Needs proxy |
| **Streaming** | Limited | Bi-directional |
| **Contract** | OpenAPI/Swagger | Proto files |
| **Caching** | Easy (HTTP caching) | Harder |
| **Learning curve** | Lower | Higher |

---

## Code Comparison

### REST API
```python
# Flask REST endpoint
@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = db.get_user(user_id)
    return jsonify(user)

@app.route('/users/<int:user_id>/deactivate', methods=['POST'])
def deactivate_user(user_id):
    # Not very RESTful - action on resource
    db.deactivate_user(user_id)
    return jsonify({"status": "deactivated"})

# Client
response = requests.get('http://api.example.com/users/123')
user = response.json()
```

### gRPC
```protobuf
// user.proto
service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc DeactivateUser(DeactivateRequest) returns (DeactivateResponse);
    rpc StreamUpdates(StreamRequest) returns (stream UserUpdate);
}

message GetUserRequest {
    int32 user_id = 1;
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
}
```

```python
# gRPC Server
class UserServicer(user_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        user = db.get_user(request.user_id)
        return user_pb2.User(id=user.id, name=user.name)

# gRPC Client
channel = grpc.insecure_channel('localhost:50051')
stub = user_pb2_grpc.UserServiceStub(channel)
user = stub.GetUser(user_pb2.GetUserRequest(user_id=123))
```

---

## When to Use What

### Choose REST When:
```
✅ Public APIs (broad client support)
✅ Browser-based applications
✅ Simple CRUD operations
✅ Need easy caching
✅ Human-readable debugging needed
✅ Team unfamiliar with gRPC
```

### Choose gRPC When:
```
✅ Microservices communication
✅ Low latency required
✅ Streaming needed
✅ Strong typing important
✅ Polyglot environment
✅ Mobile apps (bandwidth-sensitive)
```

---

## Hybrid Approaches

### gRPC-Gateway
```
┌─────────────────────────────────────────────────────────────────┐
│                      gRPC Gateway                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   [Browser/External] ──REST──▶ [gRPC Gateway] ──gRPC──▶ [Service]│
│                                                                  │
│   [Internal Service] ──────────gRPC─────────────▶ [Service]     │
│                                                                  │
│   Best of both worlds:                                          │
│   - REST for external clients                                   │
│   - gRPC for internal services                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### GraphQL Alternative
```
Neither REST nor RPC - query language
Flexible queries, single endpoint
Good for complex frontend needs
```

---

## Performance

```
gRPC advantages:
1. Binary serialization (10x smaller than JSON)
2. HTTP/2 multiplexing
3. Header compression
4. Streaming (no polling)

Benchmark (typical):
REST JSON:  100μs serialization, 500 bytes
gRPC Proto:  10μs serialization, 50 bytes
```

---

## Real-World Usage

| Company | Approach |
|---------|----------|
| **Google** | gRPC internally, REST externally |
| **Netflix** | REST for public, gRPC for internal |
| **Uber** | gRPC for microservices |
| **Dropbox** | gRPC for mobile apps |
| **Stripe** | REST for developer experience |

---

## Interview Tips

When discussing RPC vs REST:
1. Explain resource vs action orientation
2. Discuss performance trade-offs
3. Mention gRPC benefits (streaming, binary)
4. Know when to choose each
5. Describe hybrid gateway pattern

