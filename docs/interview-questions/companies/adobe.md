# Adobe Interview Experience

## Compensation Snapshot
- **Base Salary:** ₹26 LPA
- **Variable Pay:** ₹2.6 LPA (10% of base)
- **Joining Bonus:** ₹2 LPA
- **RSUs:** $68,800 USD (vested over 4 years)
- **Standard Adobe Perks:** Insurance, Wellness, WFH setup, etc.
- **First-Year CTC:** ~₹43 LPA

---

## Interview Rounds & Questions (With Answers)

### Round 1 – DSA + LLD

#### Maximum Product Subarray (with subarray return)

??? success "Solution Approach"
    ```python
    def max_product_subarray(nums: list[int]) -> tuple[int, list[int]]:
        """
        Track both max and min at each position (negative * negative = positive).
        Time: O(n), Space: O(1)
        """
        if not nums:
            return 0, []

        max_prod = min_prod = result = nums[0]
        start = end = temp_start = 0

        for i in range(1, len(nums)):
            if nums[i] < 0:
                max_prod, min_prod = min_prod, max_prod

            if nums[i] > max_prod * nums[i]:
                max_prod = nums[i]
                temp_start = i
            else:
                max_prod = max_prod * nums[i]

            min_prod = min(nums[i], min_prod * nums[i])

            if max_prod > result:
                result = max_prod
                start, end = temp_start, i

        return result, nums[start:end+1]
    ```

#### Pluggable Cache (LRU/LFU) using Strategy Pattern

??? success "Solution Approach"
    ```python
    from abc import ABC, abstractmethod
    from collections import OrderedDict, defaultdict

    class EvictionStrategy(ABC):
        @abstractmethod
        def access(self, key): pass

        @abstractmethod
        def evict(self) -> str: pass

        @abstractmethod
        def add(self, key): pass

    class LRUStrategy(EvictionStrategy):
        def __init__(self):
            self.order = OrderedDict()

        def access(self, key):
            self.order.move_to_end(key)

        def evict(self) -> str:
            return self.order.popitem(last=False)[0]

        def add(self, key):
            self.order[key] = True

    class LFUStrategy(EvictionStrategy):
        def __init__(self):
            self.freq = defaultdict(int)
            self.min_freq = 0
            self.freq_to_keys = defaultdict(OrderedDict)

        def access(self, key):
            f = self.freq[key]
            self.freq[key] += 1
            del self.freq_to_keys[f][key]
            self.freq_to_keys[f + 1][key] = True
            if not self.freq_to_keys[self.min_freq]:
                self.min_freq += 1

        def evict(self) -> str:
            key = next(iter(self.freq_to_keys[self.min_freq]))
            del self.freq_to_keys[self.min_freq][key]
            del self.freq[key]
            return key

        def add(self, key):
            self.freq[key] = 1
            self.freq_to_keys[1][key] = True
            self.min_freq = 1

    class PluggableCache:
        def __init__(self, capacity: int, strategy: EvictionStrategy):
            self.capacity = capacity
            self.strategy = strategy
            self.cache = {}

        def get(self, key):
            if key not in self.cache:
                return -1
            self.strategy.access(key)
            return self.cache[key]

        def put(self, key, value):
            if key in self.cache:
                self.cache[key] = value
                self.strategy.access(key)
                return

            if len(self.cache) >= self.capacity:
                evicted = self.strategy.evict()
                del self.cache[evicted]

            self.cache[key] = value
            self.strategy.add(key)

    # Usage
    lru_cache = PluggableCache(100, LRUStrategy())
    lfu_cache = PluggableCache(100, LFUStrategy())
    ```

### Round 2 – Java + OOPS + LLD

??? success "Interface vs Abstract Class"
    | Aspect | Interface | Abstract Class |
    |--------|-----------|----------------|
    | **Methods** | All abstract (Java 8+: default) | Mix of abstract + concrete |
    | **Variables** | public static final only | Any access modifier |
    | **Inheritance** | Multiple interfaces | Single abstract class |
    | **Constructor** | No | Yes |
    | **Use when** | Defining contract | Sharing code among related classes |

??? success "Thread vs Runnable vs Callable"
    ```java
    // 1. Extend Thread (not recommended - can't extend other class)
    class MyThread extends Thread {
        public void run() { /* task */ }
    }

    // 2. Implement Runnable (preferred - no return value)
    class MyRunnable implements Runnable {
        public void run() { /* task */ }
    }

    // 3. Implement Callable (returns value, throws exception)
    class MyCallable implements Callable<Integer> {
        public Integer call() throws Exception {
            return 42;
        }
    }

    // Usage with ExecutorService
    ExecutorService executor = Executors.newFixedThreadPool(4);
    Future<Integer> future = executor.submit(new MyCallable());
    Integer result = future.get();  // Blocks until complete
    ```

### Round 4 – Directorial

??? success "MySQL to Cassandra Migration"
    **Key Differences:**

    | Aspect | MySQL | Cassandra |
    |--------|-------|-----------|
    | **Model** | Relational (tables, joins) | Wide-column (denormalized) |
    | **Scaling** | Vertical (master-slave) | Horizontal (peer-to-peer) |
    | **Consistency** | ACID | Tunable (eventual default) |
    | **Query** | Flexible SQL | Query-driven schema |
    | **Keys** | Primary, Foreign | Partition key + Clustering key |

    **Migration Strategy:**
    1. **Denormalize:** Flatten joins into single tables
    2. **Design for queries:** One table per query pattern
    3. **Partition key:** Choose for even distribution (user_id, not date)
    4. **Dual-write:** Write to both during migration, then cutover

??? success "Design a Water Bottle (OOP)"
    ```python
    from abc import ABC, abstractmethod

    class Container(ABC):
        def __init__(self, capacity_ml: int):
            self.capacity = capacity_ml
            self.current_level = 0

        @abstractmethod
        def pour_out(self, amount: int) -> int: pass

    class Lid(ABC):
        @abstractmethod
        def open(self): pass
        @abstractmethod
        def close(self): pass

    class ScrewLid(Lid):
        def __init__(self):
            self.is_open = False
        def open(self): self.is_open = True
        def close(self): self.is_open = False

    class WaterBottle(Container):
        def __init__(self, capacity_ml: int, lid: Lid, material: str):
            super().__init__(capacity_ml)
            self.lid = lid
            self.material = material  # "plastic", "steel", "glass"
            self.is_insulated = material == "steel"

        def fill(self, amount: int):
            if not self.lid.is_open:
                raise Exception("Open lid first")
            self.current_level = min(self.capacity, self.current_level + amount)

        def pour_out(self, amount: int) -> int:
            if not self.lid.is_open:
                raise Exception("Open lid first")
            poured = min(amount, self.current_level)
            self.current_level -= poured
            return poured
    ```

### Round 5 – DSA

??? success "Count anagram sentences"
    ```python
    from collections import defaultdict
    from itertools import product

    def count_anagram_sentences(sentence: str, word_list: list[str]) -> int:
        """
        Replace each word with any of its anagrams from word_list.
        Count total possible sentences.
        """
        # Group words by sorted characters (anagram signature)
        anagram_groups = defaultdict(list)
        for word in word_list:
            key = ''.join(sorted(word))
            anagram_groups[key].append(word)

        words = sentence.split()
        total = 1

        for word in words:
            key = ''.join(sorted(word))
            # Number of anagram options for this word
            options = len(anagram_groups.get(key, [word]))
            total *= options

        return total
    ```

### Round 6 – LLD

??? success "Notification System Design"
    ```python
    from abc import ABC, abstractmethod
    from enum import Enum
    from dataclasses import dataclass

    class Channel(Enum):
        SMS = "sms"
        WHATSAPP = "whatsapp"
        PUSH = "push"
        EMAIL = "email"

    @dataclass
    class Notification:
        user_id: str
        title: str
        body: str
        channel: Channel

    class NotificationSender(ABC):
        @abstractmethod
        def send(self, notification: Notification) -> bool: pass

    class SMSSender(NotificationSender):
        def send(self, notification: Notification) -> bool:
            # Call Twilio API
            return True

    class WhatsAppSender(NotificationSender):
        def send(self, notification: Notification) -> bool:
            # Call WhatsApp Business API
            return True

    class PushSender(NotificationSender):
        def send(self, notification: Notification) -> bool:
            # Call FCM/APNs
            return True

    class NotificationService:
        def __init__(self):
            self.senders = {
                Channel.SMS: SMSSender(),
                Channel.WHATSAPP: WhatsAppSender(),
                Channel.PUSH: PushSender(),
            }
            self.activity_log = []  # In production: write to DB/Kafka

        def send(self, notification: Notification) -> bool:
            sender = self.senders.get(notification.channel)
            if not sender:
                return False

            success = sender.send(notification)
            self._log_activity(notification, success)
            return success

        def _log_activity(self, notification: Notification, success: bool):
            self.activity_log.append({
                "user_id": notification.user_id,
                "channel": notification.channel.value,
                "success": success,
                "timestamp": datetime.now()
            })
    ```

    **Key patterns:** Strategy (senders), Factory (sender selection), Observer (activity tracking).

