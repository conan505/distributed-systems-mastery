# System Design Concepts

## Core Concepts to Master
- **Idempotent APIs** – the key to safe retries in distributed systems
- **Redis Use Cases** – beyond caching: rate limiting, queues, and more
- **Saga Design Pattern** – handling failures in microservices with grace
- **Protocol Buffers vs JSON** – speed, size, and efficiency battle
- **Concurrency vs Parallelism** – not the same, but often confused
- **Consistent Hashing** – the backbone of scalable distributed systems
- **Service Discovery** – how microservices find each other
- **Monolith vs Microservices Architecture** – tradeoffs that matter
- **What Happens When You Type a URL** – the classic, never gets old
- **How Databases Store Passwords Securely** – hash, salt, repeat
- **API Gateway** – your system's smart traffic cop
- **Modular Monolith Architecture** – best of both monolith & microservices
- **Bloom Filters** – tiny, fast, and surprisingly powerful
- **Microservices Lessons from Netflix** – scale isn't just about size

---

## Bottleneck Detection Question

*Asked at: Confluent, MakeMyTrip, and other high-scale product firms*

**"How would you detect bottlenecks in a system and how would you fix them?"**

### Bottleneck Detection:
- Analyze load balancer logs (like ALB, NLB, etc.) to find uneven traffic or latency issues.
- Monitor server metrics — high CPU/memory could mean you need to autoscale or offload work to queues.
- Use APM tools (Datadog, X-Ray, New Relic) to trace latency across services.
- Check DB slow queries, replica lag, and connection pool saturation.
- Watch for queue backlogs or consumer lag in Kafka/SQS.

### Remedies:
- Remove single points of failure via multi-AZ, multi-region infra.
- Introduce auto-scaling groups and geolocation-based routing.
- Use CQRS to scale read and write paths independently.
- Break monoliths into bounded microservices with clear ownership.
- Retry failed events using DLQ, idempotency, and backoff strategies.
- Automate everything using Terraform/IaC for reproducible, scalable infra.

---

## Common System Design Problems by Category

### Read-Heavy Systems
*Focus on scale, latency, and efficient data fetching.*

1. **Design a URL Shortener (Bitly)**
   - Talk about key generation, collisions, and DB storage.
   - Add caching and DB sharding if traffic is high.

2. **Design an Image Hosting Service**
   - Talk about object storage (S3, GCS) + CDN usage.
   - Consider image deduplication and resizing strategies.

3. **Design a Social Media Platform (Twitter/Facebook)**
   - Talk about posts, timelines, relationships (follows, friends).
   - Focus on denormalized storage and sharding.

4. **Design a NewsFeed System (Hard)**
   - Push vs Pull models, Fanout on Write vs Read.
   - Caching, pagination, and ranking algorithms.

### Write-Heavy Systems
*Durability, throughput, and ingestion speed are critical.*

5. **Design a Rate Limiter**
   - Token bucket or leaky bucket algorithms.
   - Redis-backed counters + TTL logic.

6. **Design a Log Collection and Analysis System**
   - Use Kafka for ingestion, and something like ELK for processing.
   - Talk about partitioning, buffering, and real-time querying.

7. **Design a Voting System**
   - Idempotency, fraud prevention, and result aggregation.
   - Real-time vs eventual vote count updates.

8. **Design a Trending Topics System**
   - Use count-min sketch or approximate counting.
   - Talk about sliding window aggregation + ranking.

### Strong Consistency Systems
*Transactional integrity and failure handling become the focus.*

9. **Design an Online Ticket Booking System**
   - Handle race conditions with locking or optimistic concurrency.
   - Talk about seat reservation + payment flow.

10. **Design an E-Commerce Website (Amazon)**
    - Cover product catalog, cart service, order processing.
    - Include DB consistency, checkout idempotency.

11. **Design an Online Messaging App (WhatsApp/Slack)**
    - Talk about message queues, delivery receipts, retries.
    - Offline storage, notification delivery, scaling chat infra.

12. **Design a Task Management Tool**
    - CRUD APIs, user auth, task assignment.
    - Background jobs, status updates, and audit trails.

### Scheduler Services
*Timing, reliability, and eventual execution are tested here.*

13. **Design a Web Crawler**
    - BFS vs DFS for crawling, politeness rules.
    - Distributed queues, duplicate URL filters.

14. **Design a Task Scheduler**
    - Job queues, retry logic, cron-based triggers.
    - Priority queues and task deduplication.

15. **Design a Real-Time Notification System**
    - Push vs Polling, webhooks, and device token mgmt.
    - Scale delivery across millions of users.

### Trie / Proximity Systems
*Efficient data structures and latency-optimized retrieval.*

16. **Design a Search Autocomplete System**
    - Trie or Ternary Search Tree backed by frequency rank.
    - Debouncing, caching, and typo-tolerance.

17. **Design a Ride-Sharing App (Uber/Lyft)**
    - Matchmaking engine, real-time location tracking.
    - Talk about ETA algorithms, surge pricing, DB design.

