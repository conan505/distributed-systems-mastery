# Flipkart Interview Experience

## Compensation Snapshot (India)
- **Base Salary:** ₹35–45 LPA
- **Bonus + RSUs:** ₹10–20 LPA
- **Total CTC:** ₹50–65 LPA

---

## Interview Process
- Online Assessment (Hackerrank) – 2 DSA problems
- Technical Round 1 – DSA + Complexity Analysis
- Technical Round 2 – LLD + small coding task
- Technical Round 3 – System Design (mid-scale app)
- Hiring Manager Round – Tech + Behavioral
- HR Round – Culture, compensation, fit

---

## Recently Asked Problems & Topics

### DSA / Coding

#### LRU Cache

??? success "Solution Approach"
    ```python
    from collections import OrderedDict

    class LRUCache:
        """
        Time: O(1) for get and put
        Space: O(capacity)
        """
        def __init__(self, capacity: int):
            self.capacity = capacity
            self.cache = OrderedDict()

        def get(self, key: int) -> int:
            if key not in self.cache:
                return -1
            self.cache.move_to_end(key)  # Mark as recently used
            return self.cache[key]

        def put(self, key: int, value: int) -> None:
            if key in self.cache:
                self.cache.move_to_end(key)
            self.cache[key] = value
            if len(self.cache) > self.capacity:
                self.cache.popitem(last=False)  # Remove oldest
    ```

    **Alternative:** Doubly linked list + HashMap for O(1) operations without OrderedDict.

#### Trapping Rain Water

??? success "Solution Approach"
    ```python
    def trap(height: list[int]) -> int:
        """
        Two-pointer approach: O(n) time, O(1) space
        Water at position i = min(max_left, max_right) - height[i]
        """
        if not height:
            return 0

        left, right = 0, len(height) - 1
        left_max, right_max = height[left], height[right]
        water = 0

        while left < right:
            if left_max < right_max:
                left += 1
                left_max = max(left_max, height[left])
                water += left_max - height[left]
            else:
                right -= 1
                right_max = max(right_max, height[right])
                water += right_max - height[right]

        return water
    ```

    **Key insight:** We only need the smaller of the two maxes to calculate water at current position.

#### Kth Largest in Stream

??? success "Solution Approach"
    ```python
    import heapq

    class KthLargest:
        """
        Min-heap of size k: O(log k) per add, O(n log k) init
        """
        def __init__(self, k: int, nums: list[int]):
            self.k = k
            self.heap = nums
            heapq.heapify(self.heap)
            while len(self.heap) > k:
                heapq.heappop(self.heap)

        def add(self, val: int) -> int:
            heapq.heappush(self.heap, val)
            if len(self.heap) > self.k:
                heapq.heappop(self.heap)
            return self.heap[0]  # Kth largest is min of top-k
    ```

### Code Design (LLD)

#### Payment Gateway

??? success "Solution Approach"
    ```python
    from abc import ABC, abstractmethod
    from enum import Enum
    from dataclasses import dataclass
    from typing import Optional
    import uuid

    class PaymentStatus(Enum):
        PENDING = "pending"
        PROCESSING = "processing"
        SUCCESS = "success"
        FAILED = "failed"
        REFUNDED = "refunded"

    @dataclass
    class PaymentRequest:
        amount: float
        currency: str
        method: str  # "card", "upi", "netbanking"
        customer_id: str
        idempotency_key: str

    @dataclass
    class PaymentResponse:
        payment_id: str
        status: PaymentStatus
        message: Optional[str] = None

    # Strategy Pattern for payment methods
    class PaymentProcessor(ABC):
        @abstractmethod
        def process(self, request: PaymentRequest) -> PaymentResponse:
            pass

    class CardProcessor(PaymentProcessor):
        def process(self, request: PaymentRequest) -> PaymentResponse:
            # Tokenize card, call card network, handle 3DS
            return PaymentResponse(str(uuid.uuid4()), PaymentStatus.SUCCESS)

    class UPIProcessor(PaymentProcessor):
        def process(self, request: PaymentRequest) -> PaymentResponse:
            # Generate VPA intent, wait for callback
            return PaymentResponse(str(uuid.uuid4()), PaymentStatus.PENDING)

    class PaymentGateway:
        def __init__(self):
            self.processors = {
                "card": CardProcessor(),
                "upi": UPIProcessor(),
            }
            self.idempotency_store = {}  # Redis in production

        def create_payment(self, request: PaymentRequest) -> PaymentResponse:
            # Idempotency check
            if request.idempotency_key in self.idempotency_store:
                return self.idempotency_store[request.idempotency_key]

            processor = self.processors.get(request.method)
            if not processor:
                return PaymentResponse("", PaymentStatus.FAILED, "Invalid method")

            response = processor.process(request)
            self.idempotency_store[request.idempotency_key] = response
            return response
    ```

    **Key patterns:** Strategy (payment methods), Idempotency, State machine.

### System Design

#### Flipkart Wishlist Service

??? success "Solution Approach"
    **Requirements:**
    - Add/remove items to wishlist
    - View wishlist with pagination
    - Price drop notifications
    - Cross-device sync

    **Architecture:**
    ```
    ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
    │   API GW    │────▶│ Wishlist Svc │────▶│  Cassandra  │
    └─────────────┘     └──────────────┘     └─────────────┘
                               │                    │
                               ▼                    ▼
                        ┌──────────────┐     ┌─────────────┐
                        │    Redis     │     │ Price Svc   │
                        │   (Cache)    │     │ (Subscribe) │
                        └──────────────┘     └─────────────┘
    ```

    **Data Model (Cassandra):**
    ```sql
    CREATE TABLE wishlists (
        user_id UUID,
        product_id UUID,
        added_at TIMESTAMP,
        price_at_add DECIMAL,
        PRIMARY KEY (user_id, added_at, product_id)
    ) WITH CLUSTERING ORDER BY (added_at DESC);
    ```

    **Key decisions:**
    - Cassandra for high write throughput + time-ordered reads
    - Redis cache for hot users
    - Pub/sub for price drop notifications
    - Pagination via `added_at` cursor

### Behavioral & Values

#### Disagreeing with a senior

??? success "Sample Answer"
    **Situation:** Senior architect proposed microservices for a new feature. I believed a modular monolith was better for our 3-person team.

    **Action:**
    1. Scheduled 1:1 to understand their reasoning (scalability concerns)
    2. Created comparison doc: deployment complexity, debugging overhead, team velocity
    3. Proposed compromise: modular monolith now, clear boundaries for future extraction

    **Result:** We shipped 2 months faster. Six months later, extracted one module to microservice when it genuinely needed independent scaling.

    **Key lesson:** Disagree with data, not ego. Focus on outcomes, not being right.

#### Handling tight delivery timelines

??? success "Sample Answer"
    **My framework:**

    1. **Scope ruthlessly:** What's MVP vs nice-to-have? Get PM alignment.
    2. **Parallelize:** Identify independent workstreams, assign to team members
    3. **Cut corners wisely:** Skip tests for non-critical paths (add tech debt ticket)
    4. **Communicate early:** If timeline is unrealistic, raise flag immediately with data
    5. **Daily standups:** 15-min syncs to unblock quickly

    **Example:** 2-week deadline for payment integration. Scoped to single payment method, used existing SDK, skipped admin panel (manual DB updates). Shipped on time, iterated post-launch.

---

## Why Join Flipkart?
- Engineering-led culture in India's top product company
- Solving scale at 100M+ users
- Clear growth paths + internal mobility
- Hybrid model + employee-first benefits

