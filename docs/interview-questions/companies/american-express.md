# American Express Interview Experience

## Compensation Snapshot (Gurgaon / Bangalore)
- **Base Salary:** ₹28.6 LPA
- **Variable Bonus:** ₹2.08 LPA
- **Joining Bonus:** ₹3 L
- **Total CTC (1st year):** ~₹33.7 LPA
- **Work Mode:** Hybrid (3 days in office)

---

## Interview Process
- Round 1 – DSA + Machine Coding
- Round 2 – Java, JS, React (Deep Dive)
- Round 3 – Techno-Managerial + HLD Discussion
- HR Round – Culture fit, compensation, timelines

---

## Recently Asked Problems & Topics (With Answers)

### DSA / Coding

#### Combination Sum

??? success "Solution Approach"
    ```python
    def combination_sum(candidates: list[int], target: int) -> list[list[int]]:
        """
        Backtracking with reuse allowed. O(n^(target/min)) time.
        """
        result = []

        def backtrack(start: int, path: list[int], remaining: int):
            if remaining == 0:
                result.append(path[:])
                return
            if remaining < 0:
                return

            for i in range(start, len(candidates)):
                path.append(candidates[i])
                backtrack(i, path, remaining - candidates[i])  # i, not i+1 (reuse)
                path.pop()

        backtrack(0, [], target)
        return result

    # Example: candidates=[2,3,6,7], target=7
    # Output: [[2,2,3], [7]]
    ```

#### Unique Triplets (3Sum)

??? success "Solution Approach"
    ```python
    def three_sum(nums: list[int]) -> list[list[int]]:
        """
        Sort + two pointers. O(n²) time, O(1) space.
        """
        nums.sort()
        result = []

        for i in range(len(nums) - 2):
            if i > 0 and nums[i] == nums[i-1]:
                continue  # Skip duplicates

            left, right = i + 1, len(nums) - 1
            while left < right:
                total = nums[i] + nums[left] + nums[right]

                if total == 0:
                    result.append([nums[i], nums[left], nums[right]])
                    while left < right and nums[left] == nums[left+1]:
                        left += 1
                    while left < right and nums[right] == nums[right-1]:
                        right -= 1
                    left += 1
                    right -= 1
                elif total < 0:
                    left += 1
                else:
                    right -= 1

        return result
    ```

??? success "HashMap Internals & Collision Handling"
    **Java HashMap Structure:**
    ```
    HashMap
    ├── Array of buckets (default 16)
    ├── Each bucket: LinkedList (Java 7) or Tree (Java 8+, when > 8 nodes)
    └── Load factor: 0.75 (resize when 75% full)
    ```

    **Collision Handling:**
    1. **Chaining:** Multiple entries in same bucket (linked list/tree)
    2. **Open Addressing:** Probe for next empty slot (not used in Java)

    **Hash Function:**
    ```java
    // Java 8+ uses bit spreading to reduce collisions
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    // Bucket index
    int index = hash & (n - 1);  // n is power of 2
    ```

### Machine Coding (React)

#### Carousel Component

??? success "Solution Approach"
    ```jsx
    import { useState, useEffect, useCallback, useMemo } from 'react';

    function Carousel({ apiUrl }) {
      const [items, setItems] = useState([]);
      const [currentIndex, setCurrentIndex] = useState(0);
      const [isPaused, setIsPaused] = useState(false);

      // Fetch data on mount
      useEffect(() => {
        fetch(apiUrl)
          .then(res => res.json())
          .then(setItems);
      }, [apiUrl]);

      // Auto-advance every 5 seconds
      useEffect(() => {
        if (isPaused || items.length === 0) return;

        const timer = setInterval(() => {
          setCurrentIndex(prev => (prev + 1) % items.length);
        }, 5000);

        return () => clearInterval(timer);
      }, [isPaused, items.length]);

      // Memoized current item
      const currentItem = useMemo(() => items[currentIndex], [items, currentIndex]);

      // Callbacks for buttons
      const handlePause = useCallback(() => setIsPaused(true), []);
      const handlePlay = useCallback(() => setIsPaused(false), []);
      const handleNext = useCallback(() => {
        setCurrentIndex(prev => (prev + 1) % items.length);
      }, [items.length]);

      if (!currentItem) return <div>Loading...</div>;

      return (
        <div className="carousel">
          <div className="carousel-item">
            <img src={currentItem.image} alt={currentItem.title} />
            <h3>{currentItem.title}</h3>
          </div>
          <div className="controls">
            <button onClick={handlePause} disabled={isPaused}>Pause</button>
            <button onClick={handlePlay} disabled={!isPaused}>Play</button>
            <button onClick={handleNext}>Next</button>
          </div>
          <div className="indicators">
            {items.map((_, i) => (
              <span key={i} className={i === currentIndex ? 'active' : ''} />
            ))}
          </div>
        </div>
      );
    }
    ```

    **Optimizations:**
    - `useMemo` for derived state
    - `useCallback` for stable function references
    - Cleanup in `useEffect` to prevent memory leaks

### Java / JS / React

??? success "Java Stream APIs"
    ```java
    // Terminal operations (trigger execution)
    List<String> names = users.stream()
        .filter(u -> u.getAge() > 18)           // Intermediate
        .map(User::getName)                      // Intermediate
        .sorted()                                // Intermediate
        .collect(Collectors.toList());           // Terminal

    // Common terminal ops
    .collect()    // Gather results
    .forEach()    // Side effects
    .reduce()     // Aggregate to single value
    .count()      // Count elements
    .findFirst()  // Get first match
    .anyMatch()   // Boolean check
    ```

??? success "JavaScript Promises & Structured Cloning"
    ```javascript
    // Promise chaining
    fetch('/api/user')
      .then(res => res.json())
      .then(user => fetch(`/api/orders/${user.id}`))
      .then(res => res.json())
      .catch(err => console.error(err));

    // Async/await (preferred)
    async function getOrders() {
      try {
        const user = await fetch('/api/user').then(r => r.json());
        const orders = await fetch(`/api/orders/${user.id}`).then(r => r.json());
        return orders;
      } catch (err) {
        console.error(err);
      }
    }

    // Structured cloning (deep copy)
    const clone = structuredClone(original);
    // Works with: objects, arrays, Maps, Sets, Dates, RegExp
    // Doesn't work with: functions, DOM nodes, symbols
    ```

??? success "React Virtual DOM & Reconciliation"
    **Virtual DOM:**
    - Lightweight JS representation of actual DOM
    - Changes are batched and diffed before applying

    **Reconciliation Algorithm:**
    1. **Diffing:** Compare old and new virtual DOM trees
    2. **Keys:** Use `key` prop to identify elements (avoid index as key)
    3. **Batching:** Multiple setState calls batched into single re-render

    **Fiber Architecture (React 16+):**
    - Incremental rendering (can pause/resume)
    - Priority-based updates (user input > data fetch)
    - Concurrent mode for smoother UX

### System Design (HLD)

#### Payment Gateway + UPI Flow

??? success "Solution Approach"
    **UPI Payment Flow:**
    ```
    1. User initiates payment on Merchant App
    2. Merchant → Payment Gateway (create order)
    3. Gateway → UPI PSP (payment request)
    4. PSP → User's Bank App (collect request)
    5. User authenticates (PIN/biometric)
    6. Bank → NPCI → Beneficiary Bank (settlement)
    7. Callback: Bank → PSP → Gateway → Merchant
    ```

    **Architecture:**
    ```
    ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
    │  Merchant   │────▶│   Gateway    │────▶│  UPI PSP    │
    │    App      │     │   (Razorpay) │     │  (PhonePe)  │
    └─────────────┘     └──────────────┘     └─────────────┘
          ▲                    │                    │
          │                    ▼                    ▼
          │              ┌───────────┐        ┌───────────┐
          └──────────────│  Webhook  │        │   NPCI    │
            (callback)   │  Service  │        │  Switch   │
                         └───────────┘        └───────────┘
    ```

    **Key Design Decisions:**
    1. **Idempotency:** Every transaction has unique `order_id`
    2. **Webhooks:** Async status updates (don't poll)
    3. **Retry logic:** Exponential backoff for failed callbacks
    4. **Reconciliation:** Daily batch to match gateway records with bank

### Behavioral & Culture

??? success "Why do you want to join Amex?"
    **Structure your answer:**

    1. **Company:** "Amex is a leader in fintech with global scale and trust"
    2. **Role:** "This role combines my Java backend skills with system design challenges"
    3. **Growth:** "I'm excited about learning from cross-functional global teams"
    4. **Impact:** "Building payment systems that millions rely on is meaningful work"

    **Avoid:** Generic answers like "good salary" or "brand name"

    **Tip:** Research recent Amex tech blog posts or initiatives to show genuine interest

---

## Why Apply to Amex?
- Fintech stability + strong brand
- Cross-functional work with global teams
- Frontend + Java + design-focused roles
- Competitive pay for 2–5 YOE engineers
- Structured onboarding + learning pathways

