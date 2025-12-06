# CDN and Edge Computing

## What They Are

**CDN (Content Delivery Network)** is a geographically distributed network of servers that cache and deliver content from locations close to users.

**Edge Computing** extends this by running application logic at edge locations, not just caching.

---

## The Analogy 📦

Think of **Amazon warehouses**:
- **Without CDN**: Ship everything from one central warehouse
- **With CDN**: Local warehouses in every city
- **Edge Computing**: Local warehouses that can also customize orders

---

## Why They Exist

### The Problem: Latency
```
User in Tokyo → Server in New York

Physical distance: 10,800 km
Speed of light: ~200,000 km/s in fiber
Minimum latency: ~54ms one way, ~108ms round trip

Add routing, processing: 200-300ms total

With CDN (Tokyo edge):
Distance: ~10 km
Latency: <10ms
```

### What CDN Solves:
- **Reduced latency** - Content closer to users
- **Reduced origin load** - Edge handles most requests
- **Better availability** - Multiple edge locations
- **DDoS protection** - Distributed absorption

---

## How CDN Works

```
┌─────────────────────────────────────────────────────────────────┐
│                      CDN Architecture                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User Request: example.com/image.jpg                           │
│                                                                  │
│   1. DNS resolves to nearest edge                               │
│      example.com → 203.0.113.50 (Tokyo edge)                    │
│                                                                  │
│   2. Edge checks cache                                          │
│      ┌─────────────────────────────────────────────┐            │
│      │ Cache HIT: Return cached content            │            │
│      │ Cache MISS: Fetch from origin, cache, return│            │
│      └─────────────────────────────────────────────┘            │
│                                                                  │
│   ┌─────────┐     ┌─────────┐     ┌─────────┐                   │
│   │ Tokyo   │     │ London  │     │ NYC     │                   │
│   │  Edge   │     │  Edge   │     │  Edge   │                   │
│   └────┬────┘     └────┬────┘     └────┬────┘                   │
│        │               │               │                         │
│        └───────────────┼───────────────┘                         │
│                        │                                         │
│                   ┌────▼────┐                                    │
│                   │ Origin  │                                    │
│                   │ Server  │                                    │
│                   └─────────┘                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Caching Strategies

### Cache-Control Headers
```http
# Cache for 1 hour
Cache-Control: public, max-age=3600

# Cache for 1 year (immutable assets)
Cache-Control: public, max-age=31536000, immutable

# Don't cache
Cache-Control: no-store

# Revalidate before using
Cache-Control: no-cache
```

### Cache Invalidation
```
Methods:
1. TTL expiration (time-based)
2. Purge API (immediate invalidation)
3. Versioned URLs (image-v2.jpg)
4. Stale-while-revalidate
```

---

## Edge Computing

### Beyond Caching
```
Traditional CDN:
  Edge caches static content

Edge Computing:
  Edge runs application code

Examples:
- A/B testing at edge
- Authentication/authorization
- Image resizing on-demand
- Personalization
- API responses
```

### Edge Functions
```javascript
// Cloudflare Worker example
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  // Run at edge, not origin
  const country = request.cf.country
  
  if (country === 'CN') {
    return Response.redirect('https://cn.example.com')
  }
  
  return fetch(request)
}
```

---

## CDN Use Cases

### 1. Static Assets
```
Images, CSS, JavaScript, fonts
- High cache hit ratio
- Long TTL
- Versioned filenames
```

### 2. Video Streaming
```
- Chunked delivery
- Adaptive bitrate
- Geographic distribution
- Reduces buffering
```

### 3. API Acceleration
```
- Cache GET responses
- Edge authentication
- Response compression
- SSL termination
```

### 4. Dynamic Content
```
- Edge-side includes (ESI)
- Personalization at edge
- A/B testing
- Geolocation-based content
```

---

## CDN Providers

| Provider | Strengths |
|----------|-----------|
| **Cloudflare** | DDoS, Workers, free tier |
| **AWS CloudFront** | AWS integration, Lambda@Edge |
| **Akamai** | Enterprise, largest network |
| **Fastly** | Real-time purge, VCL |
| **Google Cloud CDN** | GCP integration |

---

## Performance Metrics

```
Cache Hit Ratio:
  hits / (hits + misses)
  Target: >90% for static content

Time to First Byte (TTFB):
  Time until first byte received
  Target: <100ms

Origin Offload:
  % of requests served from edge
  Target: >80%
```

---

## Common Patterns

### 1. Pull-Based (Lazy Loading)
```
First request: Cache miss → Fetch from origin → Cache
Subsequent: Cache hit → Serve from edge

✅ Simple, automatic
❌ First user gets slow response
```

### 2. Push-Based (Pre-warming)
```
Deploy: Push content to all edges
Request: Always cache hit

✅ Consistent performance
❌ More complex, storage costs
```

### 3. Stale-While-Revalidate
```
Cache-Control: max-age=60, stale-while-revalidate=300

- Serve stale content immediately
- Fetch fresh content in background
- Update cache for next request

✅ Fast responses, fresh content
```

---

## When to Use

### ✅ Good Fit:
- Global user base
- Static content heavy
- High traffic volume
- Latency-sensitive applications

### ❌ Consider Alternatives:
- Single region users
- Highly dynamic content
- Real-time requirements
- Small scale

---

## Interview Tips

When discussing CDN:
1. Explain the latency reduction benefit
2. Know caching headers and strategies
3. Discuss cache invalidation challenges
4. Mention edge computing capabilities
5. Give real examples (video streaming, static assets)

