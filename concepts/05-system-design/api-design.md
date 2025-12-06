# API Design

## What It Is

**API Design** is the process of defining how software components communicate, including endpoints, data formats, authentication, versioning, and error handling.

---

## The Analogy 🔌

Think of **electrical outlets**:
- Standard interface (plug shape)
- Consistent voltage/frequency
- Works with any compatible device
- Well-documented specifications

Good APIs are like standard outlets - predictable and reliable.

---

## REST API Principles

### 1. Resource-Based URLs
```
✅ Good (nouns):
GET    /users           # List users
GET    /users/123       # Get user 123
POST   /users           # Create user
PUT    /users/123       # Update user 123
DELETE /users/123       # Delete user 123

❌ Bad (verbs):
GET    /getUsers
POST   /createUser
POST   /deleteUser/123
```

### 2. HTTP Methods
```
GET    - Read (idempotent, safe)
POST   - Create (not idempotent)
PUT    - Replace (idempotent)
PATCH  - Partial update (idempotent)
DELETE - Remove (idempotent)
```

### 3. Status Codes
```
2xx Success:
  200 OK              - Success
  201 Created         - Resource created
  204 No Content      - Success, no body

4xx Client Error:
  400 Bad Request     - Invalid input
  401 Unauthorized    - Not authenticated
  403 Forbidden       - Not authorized
  404 Not Found       - Resource doesn't exist
  409 Conflict        - State conflict
  429 Too Many Requests - Rate limited

5xx Server Error:
  500 Internal Error  - Server bug
  502 Bad Gateway     - Upstream error
  503 Service Unavailable - Overloaded
```

---

## Request/Response Design

### Request Body
```json
// POST /users
{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user"
}
```

### Response Body
```json
// 201 Created
{
  "id": "usr_123",
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Error Response
```json
// 400 Bad Request
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid email format",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address"
      }
    ]
  }
}
```

---

## Pagination

### Offset-Based
```
GET /users?offset=20&limit=10

Response:
{
  "data": [...],
  "pagination": {
    "offset": 20,
    "limit": 10,
    "total": 150
  }
}

❌ Problem: Inconsistent with real-time data
```

### Cursor-Based
```
GET /users?cursor=eyJpZCI6MTIzfQ&limit=10

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTMzfQ",
    "has_more": true
  }
}

✅ Consistent, handles insertions/deletions
```

---

## Filtering and Sorting

```
# Filtering
GET /users?status=active&role=admin

# Sorting
GET /users?sort=created_at&order=desc

# Combined
GET /users?status=active&sort=-created_at

# Field selection
GET /users?fields=id,name,email
```

---

## Versioning

### URL Versioning
```
GET /v1/users
GET /v2/users

✅ Clear, easy to route
❌ Not RESTful (version isn't a resource)
```

### Header Versioning
```
GET /users
Accept: application/vnd.api+json; version=2

✅ Clean URLs
❌ Harder to test/debug
```

### Query Parameter
```
GET /users?version=2

✅ Easy to use
❌ Optional parameter confusion
```

---

## Authentication

### API Keys
```
GET /users
X-API-Key: sk_live_abc123

Simple, good for server-to-server
```

### JWT (Bearer Token)
```
GET /users
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

Good for user authentication
```

### OAuth 2.0
```
1. User authorizes app
2. App receives authorization code
3. App exchanges code for access token
4. App uses token for API calls

Good for third-party access
```

---

## Rate Limiting

```
Response Headers:
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000

When exceeded:
HTTP 429 Too Many Requests
Retry-After: 60
```

---

## Idempotency

```
Problem: Network retry causes duplicate action

Solution: Idempotency key
POST /payments
Idempotency-Key: unique-request-id-123
{
  "amount": 100,
  "currency": "USD"
}

Server stores result by key, returns same response on retry
```

---

## GraphQL vs REST

| Aspect | REST | GraphQL |
|--------|------|---------|
| **Endpoints** | Multiple | Single |
| **Data fetching** | Fixed response | Client specifies |
| **Over-fetching** | Common | Avoided |
| **Caching** | HTTP caching | Complex |
| **Learning curve** | Lower | Higher |

---

## Best Practices

```
1. Use consistent naming (snake_case or camelCase)
2. Return appropriate status codes
3. Provide meaningful error messages
4. Version your API from day one
5. Document with OpenAPI/Swagger
6. Implement rate limiting
7. Use HTTPS always
8. Support idempotency for mutations
9. Paginate list endpoints
10. Use standard date formats (ISO 8601)
```

---

## Interview Tips

When discussing API Design:
1. Explain RESTful principles
2. Know HTTP methods and status codes
3. Discuss pagination strategies
4. Mention versioning approaches
5. Cover authentication options

