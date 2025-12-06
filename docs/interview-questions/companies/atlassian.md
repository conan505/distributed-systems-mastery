# Atlassian Interview Experience

## Compensation Snapshot (India)
- **Base Salary:** ₹40–50 LPA (avg ~₹45 LPA)
- **Bonus + RSUs:** ~₹25 LPA
- **Total CTC:** ₹65–75 LPA

---

## Interview Process
- Karat Screening (live coding + rapid-fire)
- DSA Round – medium to hard level questions
- Low-Level Design – real-world object-oriented design
- Code Design – implementing real services
- System Design – scalable architecture discussions
- Values Round
- Managerial Round

---

## Recently Asked Problems & Topics

### DSA / Coding
- Snake Game logic with boundary conditions
- File collection: calculate total size & top-K largest
- Word search in 2D grid (only right/down moves)
- Anagram grouping and frequency mapping
- Sliding window + Trie-based problems

### Code Design (LLD)
- API Rate Limiter (Token Bucket / Leaky Bucket)
- Feature Flag Service with caching
- Logger System
- Cab Booking – Classes and Service design

### System Design
- Tagging system for large-scale platforms
- Web Crawler with nested link traversal
- Podcast Search Engine
- Ride-Sharing Database
- Dropbox-like File Storage

### Behavioral & Values
- Resolving team conflict
- Managing tight deadlines
- Ownership and initiative in tough situations

---

## Why Join Atlassian?
- Strong base salary + meaningful equity
- Remote/hybrid flexibility
- Focus on engineering excellence and practical work
- Great work-life balance with a product-first culture

---

## Interview Experience #2 (With Answers)

**Compensation:** 54L + 9
**Position:** Senior Software Engineer
**Application Method:** Direct Application

### Round 1: Scenario Based Technical Questions + Coding

**A. Scenario Questions:**

??? success "Consistent hashing issues for music streaming"
    **Issues:**
    1. **Hot spots:** Popular songs cause uneven load even with virtual nodes
    2. **Large files:** Music files are large; rebalancing on node add/remove is expensive
    3. **Caching inefficiency:** Same song requested from different nodes = cache misses

    **Solutions:**
    - Use CDN for popular content (hot data)
    - Replicate hot content to multiple nodes
    - Separate metadata (consistent hash) from content (CDN)

??? success "Processing file larger than RAM"
    **Approach:**
    1. **Streaming:** Read file in chunks, process incrementally
    2. **External sort:** Split into sorted chunks, merge-sort on disk
    3. **Memory-mapped files:** Let OS handle paging
    4. **MapReduce:** Distribute across machines

    ```python
    def process_large_file(filepath, chunk_size=1024*1024):
        with open(filepath, 'r') as f:
            while chunk := f.read(chunk_size):
                process_chunk(chunk)
    ```

??? success "Multi-country expansion considerations"
    **Backend changes:**
    1. **Data residency:** Store user data in local region (GDPR, etc.)
    2. **Latency:** Deploy services in regional data centers
    3. **Localization:** i18n for currencies, dates, languages
    4. **Compliance:** Payment methods, tax calculations per country
    5. **DNS:** GeoDNS to route to nearest region
    6. **Database:** Multi-region replication with conflict resolution

**B. Coding Round:**

??? success "Find word from jumbled string"
    ```python
    from collections import Counter

    def find(words: list[str], jumbled: str) -> str:
        """
        Check if any word can be formed from jumbled string's characters.
        Time: O(n * m) where n = words, m = avg word length
        """
        available = Counter(jumbled)

        for word in words:
            word_count = Counter(word)
            if all(word_count[c] <= available[c] for c in word_count):
                return word

        return "-"

    # Optimized: Pre-sort words by length (shorter first)
    # Or use Trie with character frequency pruning
    ```

??? success "Word search in matrix (right/down only)"
    ```python
    def word_exists(matrix: list[list[str]], word: str) -> bool:
        """
        DP approach since we can only go right/down.
        Time: O(m * n * len(word)), Space: O(len(word))
        """
        if not matrix or not word:
            return False

        rows, cols = len(matrix), len(matrix[0])

        def dfs(r, c, idx):
            if idx == len(word):
                return True
            if r >= rows or c >= cols or matrix[r][c] != word[idx]:
                return False

            return dfs(r + 1, c, idx + 1) or dfs(r, c + 1, idx + 1)

        for r in range(rows):
            for c in range(cols):
                if dfs(r, c, 0):
                    return True
        return False
    ```

### Round 2: Data Structures Round

??? success "File collection report"
    ```python
    from collections import defaultdict
    import heapq

    class FileSystem:
        def __init__(self):
            self.files = {}  # file_id -> (size, collection_ids)
            self.collection_sizes = defaultdict(int)

        def add_file(self, file_id: str, size: int, collection_ids: list[str]):
            self.files[file_id] = (size, set(collection_ids))
            for cid in collection_ids:
                self.collection_sizes[cid] += size

        def total_size(self) -> int:
            return sum(size for size, _ in self.files.values())

        def top_n_collections(self, n: int) -> list[tuple[str, int]]:
            # Use heap for O(m log n) where m = collections
            return heapq.nlargest(n, self.collection_sizes.items(), key=lambda x: x[1])

    # Multithreaded version: Use RWLock
    from threading import RLock

    class ThreadSafeFileSystem(FileSystem):
        def __init__(self):
            super().__init__()
            self.lock = RLock()

        def add_file(self, *args):
            with self.lock:
                super().add_file(*args)
    ```

### Round 3: Code Design Round

??? success "Rate Limiter with credits"
    ```python
    import time
    from threading import Lock

    class CreditBasedRateLimiter:
        """
        Unused requests carry over as credits (up to max_credits).
        """
        def __init__(self, rate_per_second: float, max_credits: int = 100):
            self.rate = rate_per_second
            self.max_credits = max_credits
            self.credits = 0
            self.tokens = rate_per_second
            self.last_update = time.time()
            self.lock = Lock()

        def allow_request(self) -> bool:
            with self.lock:
                now = time.time()
                elapsed = now - self.last_update

                # Refill tokens
                new_tokens = elapsed * self.rate
                self.tokens = min(self.rate, self.tokens + new_tokens)

                # Add unused tokens to credits
                unused = self.tokens - 1 if self.tokens >= 1 else 0
                self.credits = min(self.max_credits, self.credits + unused * 0.1)

                self.last_update = now

                # Try to use token, then credit
                if self.tokens >= 1:
                    self.tokens -= 1
                    return True
                elif self.credits >= 1:
                    self.credits -= 1
                    return True
                return False
    ```

### Round 4: System Design Round

??? success "Web Scraper System"
    **Architecture:**
    ```
    ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
    │  URL Queue  │────▶│   Workers    │────▶│  Results DB │
    │   (Redis)   │     │  (Scrapers)  │     │ (Postgres)  │
    └─────────────┘     └──────────────┘     └─────────────┘
          ▲                    │
          │                    ▼
          │              ┌──────────────┐
          └──────────────│  URL Filter  │ (Dedup + Robots.txt)
                         └──────────────┘
    ```

    **Implementation:**
    ```python
    from queue import Queue
    from concurrent.futures import ThreadPoolExecutor
    from urllib.parse import urljoin
    import requests
    from bs4 import BeautifulSoup

    class WebScraper:
        def __init__(self, max_depth: int = 3, max_workers: int = 10):
            self.max_depth = max_depth
            self.visited = set()
            self.results = {}  # url -> [image_urls]
            self.queue = Queue()

        def scrape(self, seed_urls: list[str]) -> dict:
            for url in seed_urls:
                self.queue.put((url, 0))

            with ThreadPoolExecutor(max_workers=10) as executor:
                while not self.queue.empty():
                    url, depth = self.queue.get()
                    if url in self.visited or depth > self.max_depth:
                        continue

                    self.visited.add(url)
                    executor.submit(self._process_url, url, depth)

            return self.results

        def _process_url(self, url: str, depth: int):
            try:
                resp = requests.get(url, timeout=10)
                soup = BeautifulSoup(resp.text, 'html.parser')

                # Extract images
                images = [img['src'] for img in soup.find_all('img', src=True)]
                self.results[url] = [urljoin(url, img) for img in images]

                # Queue nested URLs
                for link in soup.find_all('a', href=True):
                    self.queue.put((urljoin(url, link['href']), depth + 1))
            except Exception as e:
                pass  # Log and retry queue
    ```

    **Fault tolerance:** Retry queue with exponential backoff, dead letter queue for permanent failures.

### Round 5: Managerial Round

??? success "Handling vague requirements"
    **Situation:** Asked to "improve search" with no specific metrics or scope.

    **Action:**
    1. **Clarify:** Met with PM to understand user pain points
    2. **Define success:** Proposed metrics (latency p99, relevance score)
    3. **Prototype:** Built quick A/B test with one improvement
    4. **Iterate:** Used data to refine scope

    **Result:** Reduced search latency by 40%, improved click-through by 15%.

    **Key lesson:** When requirements are vague, define measurable outcomes first.

??? success "Mentoring a team member"
    **Situation:** Junior engineer struggling with system design.

    **Action:**
    1. **Paired weekly:** 1-hour design sessions on real problems
    2. **Assigned stretch tasks:** Let them lead small design decisions
    3. **Provided resources:** Curated reading list, mock interviews
    4. **Gave feedback:** Specific, actionable, timely

    **Result:** Within 6 months, they led their first system design and got promoted.

    **Key lesson:** Growth happens through practice + feedback, not just teaching.

