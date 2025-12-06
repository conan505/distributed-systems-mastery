# Google Interview Experience

**Leetcode Practice List:** https://leetcode.com/problem-list/2mxn884m/

## Salary Snapshot
- **Base:** ₹45–55 LPA
- **Stocks (RSUs):** ₹30–40L+ over 4 years
- **Performance Bonus:** ~15–20%
- **Total CTC:** ₹85–110 LPA (based on team & location)

---

## Interview Process
1. Online Coding Round – 2 DSA problems (LeetCode Medium-Hard level)
2. 2 Rounds of DSA + Problem Solving
3. Low-Level/System Design Round
4. Googleyness + Behavioral
5. Hiring Committee Review

---

## Favored DSA Problems
- Median in Data Stream
- Serialize and Deserialize a Binary Tree
- Hard Graph Problems (e.g., Word Ladder II, Topo Sort, Bridges)
- Range Minimum Query (Segment Tree)
- Regular Expression Matching
- Merge Intervals / Skyline Problem
- LFU Cache / Least Recently Used (LRU)
- Trie Problems (with Wildcards)

## Low Level Design (LLD)
- Design Google Docs (collaboration)
- Thread Pool
- File System
- Rate Limiter
- Elevator System / Splitwise

## High Level System Design
- Google Search Autocomplete
- YouTube-like Video Streaming
- Google Calendar Backend
- Google Maps Navigation / Live Traffic
- Distributed File Storage System (GFS-like)

---

## Interview Experience #2 (With Answers)

### Round 1: Screening Round – DSA + Optimisation
**Problem:** Given an array of integers, return the length of the longest subarray where the sum is divisible by K.

??? success "Solution Approach"
    **Key Insight:** Use prefix sum + modulo + hashmap

    ```python
    def longest_subarray_divisible_by_k(arr, k):
        """
        Time: O(n), Space: O(k)

        Key insight: If prefix_sum[j] % k == prefix_sum[i] % k,
        then sum(arr[i+1:j+1]) is divisible by k
        """
        prefix_sum = 0
        # Map: remainder -> first index where this remainder occurred
        remainder_index = {0: -1}  # Handle subarray from start
        max_length = 0

        for i, num in enumerate(arr):
            prefix_sum += num
            remainder = prefix_sum % k

            # Handle negative numbers
            if remainder < 0:
                remainder += k

            if remainder in remainder_index:
                max_length = max(max_length, i - remainder_index[remainder])
            else:
                remainder_index[remainder] = i

        return max_length
    ```

    **Why this works:** Two positions with same remainder means the subarray between them sums to a multiple of K.

### Round 2: Screening Round – DSA
**Problem:** Given an array of integers, return the minimum number of swaps required to sort the array in non-decreasing order.

??? success "Solution Approach"
    **Key Insight:** Model as graph - find cycles, swaps = n - cycles

    ```python
    def min_swaps_to_sort(arr):
        """
        Time: O(n log n), Space: O(n)

        Each cycle of length L needs L-1 swaps.
        Total swaps = n - number_of_cycles
        """
        n = len(arr)
        # Create (value, original_index) pairs
        indexed = [(val, i) for i, val in enumerate(arr)]
        indexed.sort(key=lambda x: x[0])

        visited = [False] * n
        swaps = 0

        for i in range(n):
            if visited[i] or indexed[i][1] == i:
                continue

            # Find cycle length
            cycle_length = 0
            j = i
            while not visited[j]:
                visited[j] = True
                j = indexed[j][1]  # Go to original position
                cycle_length += 1

            swaps += cycle_length - 1

        return swaps
    ```

    **Edge cases:** Already sorted (0 swaps), reverse sorted, duplicates (use stable sort).

### Round 3: Onsite – Concurrency + Scheduling
**Problem Statement:** You're given access to a system of worker threads. Each worker can perform a task, but may fail randomly. Implement a robust scheduler that:
- Retries failed tasks
- Ensures a task is only retried up to 3 times
- Distributes tasks evenly across all threads

??? success "Solution Approach"
    ```python
    import threading
    from concurrent.futures import ThreadPoolExecutor
    from queue import Queue
    from dataclasses import dataclass
    from typing import Callable
    import time

    @dataclass
    class Task:
        id: str
        fn: Callable
        args: tuple = ()
        retries: int = 0
        max_retries: int = 3

    class RobustScheduler:
        def __init__(self, num_workers: int = 4):
            self.task_queue = Queue()
            self.dead_letter_queue = Queue()  # Failed after max retries
            self.executor = ThreadPoolExecutor(max_workers=num_workers)
            self.running = True
            self.results = {}
            self.lock = threading.Lock()

        def submit(self, task: Task):
            self.task_queue.put(task)

        def worker(self):
            while self.running:
                try:
                    task = self.task_queue.get(timeout=1)
                except:
                    continue

                try:
                    result = task.fn(*task.args)
                    with self.lock:
                        self.results[task.id] = ("success", result)
                except Exception as e:
                    task.retries += 1
                    if task.retries < task.max_retries:
                        # Exponential backoff
                        time.sleep(2 ** task.retries * 0.1)
                        self.task_queue.put(task)
                    else:
                        self.dead_letter_queue.put(task)
                        with self.lock:
                            self.results[task.id] = ("failed", str(e))
                finally:
                    self.task_queue.task_done()

        def start(self, num_workers: int = 4):
            for _ in range(num_workers):
                self.executor.submit(self.worker)

        def shutdown(self):
            self.running = False
            self.task_queue.join()
            self.executor.shutdown()
    ```

    **Key points:**
    - Thread-safe queue for task distribution
    - Exponential backoff for retries
    - Dead letter queue for failed tasks
    - Lock for shared results dictionary

### Round 4: Onsite – System Design + API Thinking
**Task:** Design a calculator library that supports basic arithmetic operations, nested expressions, and variables.

??? success "Solution Approach"
    ```python
    from typing import Dict, Union
    import re

    class Calculator:
        def __init__(self):
            self.variables: Dict[str, float] = {}

        def set(self, name: str, value: Union[float, str]) -> float:
            """Set a variable: calc.set('x', 5) or calc.set('y', 'x + 1')"""
            if isinstance(value, str):
                value = self.evaluate(value)
            self.variables[name] = value
            return value

        def evaluate(self, expression: str) -> float:
            """Evaluate expression with operator precedence"""
            tokens = self._tokenize(expression)
            return self._parse_expression(tokens)

        def _tokenize(self, expr: str) -> list:
            """Convert expression string to tokens"""
            pattern = r'(\d+\.?\d*|[a-zA-Z_]\w*|[+\-*/()])'
            tokens = re.findall(pattern, expr.replace(' ', ''))
            return tokens

        def _parse_expression(self, tokens: list) -> float:
            """Recursive descent parser with precedence"""
            pos = [0]  # Mutable position tracker

            def parse_primary():
                token = tokens[pos[0]]
                if token == '(':
                    pos[0] += 1
                    result = parse_addition()
                    pos[0] += 1  # Skip ')'
                    return result
                elif token.replace('.', '').isdigit():
                    pos[0] += 1
                    return float(token)
                else:  # Variable
                    pos[0] += 1
                    if token not in self.variables:
                        raise ValueError(f"Undefined variable: {token}")
                    return self.variables[token]

            def parse_multiplication():
                left = parse_primary()
                while pos[0] < len(tokens) and tokens[pos[0]] in '*/':
                    op = tokens[pos[0]]
                    pos[0] += 1
                    right = parse_primary()
                    left = left * right if op == '*' else left / right
                return left

            def parse_addition():
                left = parse_multiplication()
                while pos[0] < len(tokens) and tokens[pos[0]] in '+-':
                    op = tokens[pos[0]]
                    pos[0] += 1
                    right = parse_multiplication()
                    left = left + right if op == '+' else left - right
                return left

            return parse_addition()

    # Usage
    calc = Calculator()
    calc.set('x', 10)
    calc.set('y', 'x * 2')
    print(calc.evaluate('(x + y) * 2 - 5'))  # 55.0
    ```

    **Design decisions:**
    - Recursive descent parser for clean precedence handling
    - Variables stored in dict, resolved during evaluation
    - Tokenizer separates parsing concerns

### Round 5: Googliness – Behavioral

??? success "Answer: Decision against team opinion"
    **Situation:** Our team was building a new caching layer and everyone wanted Redis. I advocated for using our existing Memcached infrastructure.

    **Action:** I created a comparison doc with benchmarks, operational costs, and migration complexity. I scheduled a design review where I presented the data objectively, acknowledging Redis's advantages but highlighting that our use case (simple key-value with no complex data structures) didn't need them.

    **Result:** Team agreed to use Memcached for v1, with a clear decision point to migrate if we needed Redis features. Six months later, we still haven't needed to migrate, saving significant operational overhead.

    **Key lesson:** Data-driven arguments + respecting team's concerns = buy-in.

??? success "Answer: Receiving critical feedback"
    **Situation:** My manager told me my code reviews were too harsh and discouraging junior engineers.

    **Action:** I asked for specific examples, then met with affected engineers to understand their perspective. I changed my approach: started with positives, asked questions instead of dictating changes, and held optional pairing sessions for complex reviews.

    **Result:** Three months later, feedback surveys showed improved perception. One junior engineer mentioned my reviews helped them grow the most.

    **Key lesson:** Feedback is a gift. The goal is team success, not being right.

??? success "Answer: Growing juniors while meeting deadlines"
    **My approach:**

    1. **Intentional task assignment:** 70% tasks they can do, 30% stretch tasks
    2. **Pairing on critical path:** They drive, I guide - learning happens, deadline met
    3. **Documentation requirements:** They document as they learn (future reference)
    4. **Weekly 1:1s:** 15 min focused on growth, not status updates
    5. **Protect learning time:** Block 2 hours/week for exploration (no meetings)

    **Concrete example:** Junior needed to implement rate limiting. Instead of doing it myself, I gave them the Stripe rate limiting blog post, 30 min of context, then let them design. Review took longer but they now own that entire system.

