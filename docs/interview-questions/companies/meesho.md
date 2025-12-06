# Meesho Interview Experience

## Compensation Snapshot (Bangalore)
- **Base Salary:** ₹43–48 LPA
- **Signing + Retention Bonus:** ₹3–5 L
- **ESOPs:** ₹35–40 L (4-yr vesting)
- **Total CTC (1st year):** ₹56–58 LPA

---

## Interview Process
- Round 1 – Machine Coding (LLD + working code)
- Round 2 – High-Level Design (Scalability + Microservices)
- Round 3 – Bar Raiser (Distributed Systems + Scenario-based)
- HR Round – Culture fit, ESOPs, timelines

---

## Recently Asked Problems & Topics (With Answers)

### DSA / Coding

#### Median of Two Sorted Arrays

??? success "Solution Approach"
    ```python
    def find_median_sorted_arrays(nums1: list[int], nums2: list[int]) -> float:
        """
        Binary search on smaller array. O(log(min(m,n)))
        """
        if len(nums1) > len(nums2):
            nums1, nums2 = nums2, nums1

        m, n = len(nums1), len(nums2)
        left, right = 0, m

        while left <= right:
            partition1 = (left + right) // 2
            partition2 = (m + n + 1) // 2 - partition1

            max_left1 = float('-inf') if partition1 == 0 else nums1[partition1 - 1]
            min_right1 = float('inf') if partition1 == m else nums1[partition1]
            max_left2 = float('-inf') if partition2 == 0 else nums2[partition2 - 1]
            min_right2 = float('inf') if partition2 == n else nums2[partition2]

            if max_left1 <= min_right2 and max_left2 <= min_right1:
                if (m + n) % 2 == 0:
                    return (max(max_left1, max_left2) + min(min_right1, min_right2)) / 2
                return max(max_left1, max_left2)
            elif max_left1 > min_right2:
                right = partition1 - 1
            else:
                left = partition1 + 1

        raise ValueError("Arrays not sorted")
    ```

#### Number of Islands (DFS)

??? success "Solution Approach"
    ```python
    def num_islands(grid: list[list[str]]) -> int:
        """
        DFS to mark visited cells. O(m*n) time and space.
        """
        if not grid:
            return 0

        rows, cols = len(grid), len(grid[0])
        count = 0

        def dfs(r, c):
            if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
                return
            grid[r][c] = '0'  # Mark visited
            dfs(r + 1, c)
            dfs(r - 1, c)
            dfs(r, c + 1)
            dfs(r, c - 1)

        for r in range(rows):
            for c in range(cols):
                if grid[r][c] == '1':
                    count += 1
                    dfs(r, c)

        return count
    ```

### Code Design (LLD)

#### Splitwise

??? success "Solution Approach"
    ```python
    from collections import defaultdict
    from dataclasses import dataclass
    from enum import Enum
    from typing import Dict, List

    class SplitType(Enum):
        EQUAL = "equal"
        EXACT = "exact"
        PERCENT = "percent"

    @dataclass
    class Expense:
        id: str
        paid_by: str
        amount: float
        split_type: SplitType
        participants: Dict[str, float]  # user_id -> share

    class Splitwise:
        def __init__(self):
            # balances[A][B] = amount A owes B
            self.balances: Dict[str, Dict[str, float]] = defaultdict(lambda: defaultdict(float))

        def add_expense(self, expense: Expense):
            shares = self._calculate_shares(expense)

            for user, share in shares.items():
                if user != expense.paid_by:
                    self.balances[user][expense.paid_by] += share
                    self.balances[expense.paid_by][user] -= share

        def _calculate_shares(self, expense: Expense) -> Dict[str, float]:
            if expense.split_type == SplitType.EQUAL:
                share = expense.amount / len(expense.participants)
                return {user: share for user in expense.participants}
            elif expense.split_type == SplitType.EXACT:
                return expense.participants
            elif expense.split_type == SplitType.PERCENT:
                return {user: expense.amount * pct / 100
                        for user, pct in expense.participants.items()}

        def get_balance(self, user: str) -> Dict[str, float]:
            """Returns simplified balances (net amounts)"""
            result = {}
            for other, amount in self.balances[user].items():
                net = amount - self.balances[other].get(user, 0)
                if abs(net) > 0.01:
                    result[other] = net
            return result

        def simplify_debts(self) -> List[tuple]:
            """Minimize number of transactions using greedy approach"""
            # Calculate net balance for each user
            net = defaultdict(float)
            for user, debts in self.balances.items():
                for other, amount in debts.items():
                    net[user] -= amount
                    net[other] += amount

            # Separate creditors and debtors
            creditors = [(u, a) for u, a in net.items() if a > 0]
            debtors = [(u, -a) for u, a in net.items() if a < 0]

            transactions = []
            i, j = 0, 0
            while i < len(debtors) and j < len(creditors):
                debtor, debt = debtors[i]
                creditor, credit = creditors[j]

                amount = min(debt, credit)
                transactions.append((debtor, creditor, amount))

                debtors[i] = (debtor, debt - amount)
                creditors[j] = (creditor, credit - amount)

                if debtors[i][1] < 0.01: i += 1
                if creditors[j][1] < 0.01: j += 1

            return transactions
    ```

### System Design

#### Google Meet Architecture

??? success "Solution Approach"
    **Components:**
    ```
    ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
    │   Client    │────▶│  Signaling   │────▶│   TURN/STUN │
    │  (WebRTC)   │     │   Server     │     │   Servers   │
    └─────────────┘     └──────────────┘     └─────────────┘
          │                                        │
          │ Media (UDP)                            │
          ▼                                        ▼
    ┌─────────────────────────────────────────────────────┐
    │                    SFU (Selective Forwarding Unit)   │
    │         Routes media streams between participants    │
    └─────────────────────────────────────────────────────┘
    ```

    **Key Design Decisions:**

    1. **SFU vs MCU:**
       - SFU: Forward streams without transcoding (lower latency, scales better)
       - MCU: Mix streams into one (higher CPU, simpler client)
       - **Choice:** SFU for scalability

    2. **Signaling:** WebSocket for real-time session negotiation (SDP exchange)

    3. **Media Transport:** WebRTC with SRTP encryption

    4. **Scaling:**
       - Horizontal SFU scaling with consistent hashing by room_id
       - Cascading SFUs for large meetings (>50 participants)

    5. **Quality Adaptation:**
       - Simulcast: Client sends multiple quality levels
       - SFU selects based on receiver's bandwidth

#### Facebook Feed Service

??? success "Solution Approach"
    **Architecture:**
    ```
    ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
    │   Client    │────▶│   API GW     │────▶│ Feed Service│
    └─────────────┘     └──────────────┘     └─────────────┘
                                                    │
                               ┌────────────────────┼────────────────────┐
                               ▼                    ▼                    ▼
                        ┌───────────┐        ┌───────────┐        ┌───────────┐
                        │  Post DB  │        │ Graph DB  │        │   Cache   │
                        │(Cassandra)│        │ (follows) │        │  (Redis)  │
                        └───────────┘        └───────────┘        └───────────┘
    ```

    **Feed Generation Strategies:**

    1. **Pull Model (Fan-out on read):**
       - Query followers' posts at read time
       - Good for: Users with many followers (celebrities)
       - Latency: Higher at read time

    2. **Push Model (Fan-out on write):**
       - Pre-compute feed when post is created
       - Good for: Users with few followers
       - Storage: Higher (duplicate posts in feeds)

    3. **Hybrid (Facebook's approach):**
       - Push for normal users
       - Pull for celebrities (>10K followers)

    **Data Model:**
    ```sql
    -- Posts table (Cassandra)
    CREATE TABLE posts (
        user_id UUID,
        post_id TIMEUUID,
        content TEXT,
        created_at TIMESTAMP,
        PRIMARY KEY (user_id, post_id)
    ) WITH CLUSTERING ORDER BY (post_id DESC);

    -- Pre-computed feed (Redis)
    ZADD feed:{user_id} {timestamp} {post_id}
    ZREVRANGE feed:{user_id} 0 20  -- Get latest 20 posts
    ```

### Behavioral & Sensibility

??? success "Handling system failures"
    **My framework:**

    1. **Detect:** Alerts from monitoring (PagerDuty, Datadog)
    2. **Communicate:** Status page update within 5 minutes
    3. **Triage:** Identify blast radius, form war room if needed
    4. **Mitigate:** Rollback, feature flag off, or scale up
    5. **Fix:** Root cause analysis, permanent fix
    6. **Learn:** Blameless post-mortem, action items

    **Example:** Payment service went down during sale. Detected via error rate spike. Mitigated by enabling fallback to backup provider. Root cause: connection pool exhaustion. Fixed with auto-scaling and circuit breaker.

??? success "Prioritization under pressure"
    **My approach:**

    1. **Impact vs Effort matrix:** Quick wins first
    2. **Stakeholder alignment:** Get PM/manager buy-in on priorities
    3. **Timeboxing:** Set hard deadlines, cut scope if needed
    4. **Daily standups:** 15-min syncs to unblock quickly
    5. **Say no:** Protect focus by declining non-critical requests

    **Example:** Three critical bugs + feature deadline. Prioritized: P0 bug (customer-facing), feature (committed), P1 bugs (next sprint). Communicated trade-offs to PM, got alignment.

---

## Why Join Meesho?
- Rocketship growth in social commerce
- Ownership across problem-solving
- High-impact tech culture
- Transparent ESOP structure
- Generous joining/relocation perks

