# Strangler Fig Pattern

## What It Is

The **Strangler Fig Pattern** is a migration strategy that gradually replaces a legacy system by incrementally building new functionality around it. Over time, the new system "strangles" the old one until it can be decommissioned.

---

## The Analogy 🌳

Named after the **strangler fig tree**:
- Seeds land on a host tree
- Vines grow down, roots take hold
- Fig slowly wraps around host
- Eventually, host tree dies/decomposes
- Fig stands independently

In software:
- New system grows alongside legacy
- Traffic gradually shifts
- Legacy functionality migrated piece by piece
- Eventually, legacy is completely replaced

---

## Why It Exists

### The Big Bang Problem:
```
Traditional Rewrite:
┌──────────────┐                ┌──────────────┐
│   Legacy     │  ──── 2 years ───▶  │    New       │
│   System     │   rewrite          │   System     │
└──────────────┘                └──────────────┘

Problems:
- 2 years of parallel development
- "Freeze" legacy while rewriting
- Big bang switch-over (risky!)
- All-or-nothing failure
- Often fails or gets cancelled
```

### Strangler Fig Solution:
```
Gradual Migration:
Month 1:  [==========Legacy==========]
Month 6:  [===Legacy===][==New==]
Month 12: [=Legacy=][======New======]
Month 18: [      ========New=========]

Benefits:
- Continuous delivery
- Reduced risk
- Rollback capability
- Team learns gradually
```

---

## How It Works

### Architecture Evolution:

#### Phase 1: Intercept
```
┌─────────────────────────────────────────────────────────────────┐
│                         Phase 1: Facade                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Users ──▶ ┌──────────────┐ ──▶ ┌──────────────┐               │
│             │   Facade /   │     │    Legacy    │               │
│             │    Proxy     │     │    System    │               │
│             └──────────────┘     └──────────────┘               │
│                                                                  │
│   All traffic goes through facade (routes 100% to legacy)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Phase 2: Migrate Incrementally
```
┌─────────────────────────────────────────────────────────────────┐
│                     Phase 2: Gradual Migration                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Users ──▶ ┌──────────────┐     ┌──────────────┐               │
│             │   Facade /   │ ──▶ │    Legacy    │ (80%)         │
│             │    Proxy     │     └──────────────┘               │
│             │              │     ┌──────────────┐               │
│             │              │ ──▶ │     New      │ (20%)         │
│             └──────────────┘     │   Service    │               │
│                                  └──────────────┘               │
│                                                                  │
│   Route by: feature flag, URL path, user segment, etc.          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Phase 3: Complete Migration
```
┌─────────────────────────────────────────────────────────────────┐
│                      Phase 3: Decommission                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Users ──▶ ┌──────────────┐     ┌ ─ ─ ─ ─ ─ ─ ┐               │
│             │   Facade     │       Legacy       (0%)            │
│             │              │     │  (removed)   │               │
│             │              │     └ ─ ─ ─ ─ ─ ─ ┘               │
│             │              │ ──▶ ┌──────────────┐               │
│             │              │     │     New      │ (100%)        │
│             └──────────────┘     │   Services   │               │
│                                  └──────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Strategies

### 1. URL-based Routing
```nginx
# Nginx configuration
location /api/v2/users {
    proxy_pass http://new-service;
}
location /api/ {
    proxy_pass http://legacy-service;
}
```

### 2. Feature Flags
```java
if (featureFlags.isEnabled("new-checkout")) {
    return newCheckoutService.process(order);
} else {
    return legacyCheckout.process(order);
}
```

### 3. Domain-based Migration
- Migrate one domain at a time
- Users → Payments → Orders → Inventory

---

## When to Use It

### ✅ Good Fit:
- **Legacy modernization** - Monolith to microservices
- **High-risk systems** - Can't afford downtime
- **Large codebases** - Too big for big-bang rewrite
- **Team learning curve** - Need time to understand legacy
- **Continuous business operations** - Can't freeze features

### ❌ Avoid When:
- Small, simple applications
- Clean slate is possible (no users/data)
- Legacy is too coupled to separate
- Short timelines with dedicated rewrite team

---

## How to Use It Effectively

### Best Practices:

1. **Start with a facade**
   - All traffic goes through it
   - Makes routing decisions transparent

2. **Migrate by business domain**
   - Not by technical layer
   - Complete vertical slices

3. **Keep legacy and new in sync**
   - Dual-write if needed temporarily
   - Data migration strategy

4. **Monitor both systems**
   - Compare results between old and new
   - Detect discrepancies early

5. **Have rollback capability**
   - Feature flags for quick switch-back
   - Shadow traffic testing

---

## Real-World Examples

| Company | Migration |
|---------|-----------|
| **Amazon** | Monolith to microservices |
| **Shopify** | Ruby monolith modularization |
| **LinkedIn** | Various service extractions |
| **Netflix** | Data center to cloud |

---

## Interview Tips

When discussing Strangler Fig:
1. Start with the tree analogy
2. Contrast with big-bang rewrite risks
3. Explain facade/proxy approach
4. Mention feature flags for gradual rollout
5. Discuss data migration challenges

