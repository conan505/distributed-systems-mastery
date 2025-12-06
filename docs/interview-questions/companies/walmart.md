# Walmart Interview Experience

## Compensation Snapshot
- **Fixed Salary:** ₹24 LPA
- **PF:** ₹1.15 LPA
- **Joining Bonus:** ₹3 L
- **Performance Bonus:** ₹4.8 LPA (20% of fixed)
- **Stock Grant:** ₹6 L over 3 years
- **First-Year CTC:** ~₹35 LPA
- **Standard Walmart Benefits:** Insurance, Wellness, Relocation, Stock Refreshers

---

## Interview Rounds & Questions (With Answers)

### Round 1 – DSA (Java)

#### Kadane's Algorithm (Maximum Sum Subarray)

??? success "Solution Approach"
    ```java
    public int maxSubArray(int[] nums) {
        // Kadane's: O(n) time, O(1) space
        int maxSum = nums[0];
        int currentSum = nums[0];

        for (int i = 1; i < nums.length; i++) {
            // Either extend current subarray or start new
            currentSum = Math.max(nums[i], currentSum + nums[i]);
            maxSum = Math.max(maxSum, currentSum);
        }

        return maxSum;
    }

    // Variation: Return the subarray itself
    public int[] maxSubArrayWithIndices(int[] nums) {
        int maxSum = nums[0], currentSum = nums[0];
        int start = 0, end = 0, tempStart = 0;

        for (int i = 1; i < nums.length; i++) {
            if (nums[i] > currentSum + nums[i]) {
                currentSum = nums[i];
                tempStart = i;
            } else {
                currentSum += nums[i];
            }

            if (currentSum > maxSum) {
                maxSum = currentSum;
                start = tempStart;
                end = i;
            }
        }

        return Arrays.copyOfRange(nums, start, end + 1);
    }
    ```

#### Search in Rotated Sorted Array

??? success "Solution Approach"
    ```java
    public int search(int[] nums, int target) {
        // Modified binary search: O(log n)
        int left = 0, right = nums.length - 1;

        while (left <= right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) return mid;

            // Left half is sorted
            if (nums[left] <= nums[mid]) {
                if (target >= nums[left] && target < nums[mid]) {
                    right = mid - 1;
                } else {
                    left = mid + 1;
                }
            }
            // Right half is sorted
            else {
                if (target > nums[mid] && target <= nums[right]) {
                    left = mid + 1;
                } else {
                    right = mid - 1;
                }
            }
        }

        return -1;
    }
    ```

    **Key insight:** One half is always sorted. Check if target is in sorted half.

### Round 2 – LLD + Core Java

#### Digital Wallet Design

??? success "Solution Approach"
    ```java
    import java.util.concurrent.locks.ReentrantLock;
    import java.math.BigDecimal;

    public class DigitalWallet {
        private final String userId;
        private BigDecimal balance;
        private final ReentrantLock lock = new ReentrantLock();

        public DigitalWallet(String userId, BigDecimal initialBalance) {
            this.userId = userId;
            this.balance = initialBalance;
        }

        public boolean credit(BigDecimal amount) {
            lock.lock();
            try {
                balance = balance.add(amount);
                return true;
            } finally {
                lock.unlock();
            }
        }

        public boolean debit(BigDecimal amount) {
            lock.lock();
            try {
                if (balance.compareTo(amount) < 0) {
                    return false;  // Insufficient balance
                }
                balance = balance.subtract(amount);
                return true;
            } finally {
                lock.unlock();
            }
        }

        // Transfer with deadlock prevention (lock ordering)
        public static boolean transfer(DigitalWallet from, DigitalWallet to, BigDecimal amount) {
            // Always lock in consistent order to prevent deadlock
            DigitalWallet first = from.userId.compareTo(to.userId) < 0 ? from : to;
            DigitalWallet second = first == from ? to : from;

            first.lock.lock();
            try {
                second.lock.lock();
                try {
                    if (from.balance.compareTo(amount) < 0) {
                        return false;
                    }
                    from.balance = from.balance.subtract(amount);
                    to.balance = to.balance.add(amount);
                    return true;
                } finally {
                    second.lock.unlock();
                }
            } finally {
                first.lock.unlock();
            }
        }
    }
    ```

??? success "HashMap vs ConcurrentHashMap"
    | Aspect | HashMap | ConcurrentHashMap |
    |--------|---------|-------------------|
    | **Thread-safe** | No | Yes |
    | **Null keys/values** | Allowed | Not allowed |
    | **Locking** | None | Segment-level (Java 7) / Node-level (Java 8+) |
    | **Performance** | Faster single-thread | Better concurrent access |
    | **Iteration** | Fail-fast | Weakly consistent |

    ```java
    // ConcurrentHashMap atomic operations
    ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
    map.putIfAbsent("key", 1);
    map.compute("key", (k, v) -> v == null ? 1 : v + 1);  // Atomic increment
    ```

??? success "Garbage Collection Algorithms"
    | Algorithm | Description | Use Case |
    |-----------|-------------|----------|
    | **Serial GC** | Single-threaded, stop-the-world | Small apps, single CPU |
    | **Parallel GC** | Multi-threaded young gen | Throughput-focused |
    | **CMS** | Concurrent mark-sweep | Low latency (deprecated) |
    | **G1 GC** | Region-based, predictable pauses | Default in Java 9+ |
    | **ZGC** | Sub-millisecond pauses | Ultra-low latency |

    ```bash
    # G1 GC tuning
    java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -Xmx4g MyApp
    ```

### Round 3 – Hiring Manager Round

??? success "Dynamic Dispatch in Java"
    **Definition:** Method call resolved at runtime based on actual object type, not reference type.

    ```java
    class Animal {
        void speak() { System.out.println("Animal speaks"); }
    }

    class Dog extends Animal {
        @Override
        void speak() { System.out.println("Dog barks"); }
    }

    Animal a = new Dog();
    a.speak();  // Output: "Dog barks" (dynamic dispatch)
    ```

    **How it works:** JVM uses vtable (virtual method table) to look up actual method at runtime.

??? success "Java 8 vs Java 17 Key Differences"
    | Feature | Java 8 | Java 17 |
    |---------|--------|---------|
    | **Records** | No | `record Person(String name, int age) {}` |
    | **Pattern matching** | No | `if (obj instanceof String s) { use(s); }` |
    | **Sealed classes** | No | `sealed class Shape permits Circle, Square {}` |
    | **Text blocks** | No | `"""multi-line string"""` |
    | **Switch expressions** | No | `int result = switch(x) { case 1 -> 10; };` |
    | **GC** | G1 default | ZGC/Shenandoah production-ready |

??? success "Debugging in Production"
    **My approach:**

    1. **Logs first:** Check structured logs with correlation IDs
    2. **Metrics:** Look at dashboards (latency, error rates, throughput)
    3. **Traces:** Use distributed tracing (Jaeger/Zipkin) for request flow
    4. **Thread dumps:** `jstack <pid>` for deadlocks/hangs
    5. **Heap dumps:** `jmap -dump:format=b,file=heap.bin <pid>` for memory issues
    6. **Profiling:** Async-profiler for CPU/allocation hotspots

    **Example:** Production OOM. Took heap dump, analyzed with Eclipse MAT, found unbounded cache. Fixed with TTL eviction.

??? success "Scaling Java Applications"
    **GC Tuning:**
    ```bash
    # For low latency
    -XX:+UseZGC -Xmx8g -XX:+UseStringDeduplication

    # For throughput
    -XX:+UseParallelGC -XX:ParallelGCThreads=4
    ```

    **Thread Optimization:**
    - Use virtual threads (Java 21) for I/O-bound workloads
    - Size thread pools: CPU-bound = cores, I/O-bound = cores * (1 + wait/compute)
    - Use `CompletableFuture` for async composition

    **Memory:**
    - Off-heap caching (Caffeine with off-heap)
    - Object pooling for expensive objects
    - Avoid autoboxing in hot paths

