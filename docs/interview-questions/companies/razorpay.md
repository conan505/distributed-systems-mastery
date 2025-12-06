# Razorpay Interview Experience

## Compensation Snapshot (India)
- **Base Salary:** ₹35–45 LPA
- **Bonus + ESOPs:** ₹10–20 LPA
- **Total CTC:** ₹50–70 LPA (depending on experience & negotiation skills)

---

## Interview Process
- Machine Coding Round – Focus on OOP, threading, edge cases
- High-Level Design – End-to-end system & APIs
- Hiring Manager Round – Architecture, past projects, ownership
- HR Round – Compensation, team fit, values

---

## Recently Asked Problems & Topics

### Machine Coding

#### Multithreaded Logger

??? success "Solution Approach"
    ```python
    import threading
    from queue import Queue
    from enum import Enum
    from datetime import datetime
    import os

    class LogLevel(Enum):
        DEBUG = 1
        INFO = 2
        WARN = 3
        ERROR = 4

    class Logger:
        _instance = None
        _lock = threading.Lock()

        def __new__(cls):
            if cls._instance is None:
                with cls._lock:
                    if cls._instance is None:
                        cls._instance = super().__new__(cls)
                        cls._instance._init()
            return cls._instance

        def _init(self):
            self.log_queue = Queue()
            self.min_level = LogLevel.INFO
            self.handlers = []
            self._start_worker()

        def _start_worker(self):
            def worker():
                while True:
                    log_entry = self.log_queue.get()
                    for handler in self.handlers:
                        handler.write(log_entry)

            thread = threading.Thread(target=worker, daemon=True)
            thread.start()

        def log(self, level: LogLevel, message: str):
            if level.value >= self.min_level.value:
                entry = {
                    "timestamp": datetime.now().isoformat(),
                    "level": level.name,
                    "thread": threading.current_thread().name,
                    "message": message
                }
                self.log_queue.put(entry)

        def info(self, msg): self.log(LogLevel.INFO, msg)
        def error(self, msg): self.log(LogLevel.ERROR, msg)

    class FileHandler:
        def __init__(self, filepath):
            self.filepath = filepath
            self.lock = threading.Lock()

        def write(self, entry):
            with self.lock:
                with open(self.filepath, 'a') as f:
                    f.write(f"[{entry['timestamp']}] {entry['level']} - {entry['message']}\n")
    ```

    **Key design points:** Singleton pattern, async queue for non-blocking, thread-safe file writes.

#### Elevator System

??? success "Solution Approach"
    ```python
    from enum import Enum
    from threading import Lock
    from heapq import heappush, heappop

    class Direction(Enum):
        UP = 1
        DOWN = -1
        IDLE = 0

    class Elevator:
        def __init__(self, id: int, total_floors: int):
            self.id = id
            self.current_floor = 0
            self.direction = Direction.IDLE
            self.up_stops = []      # Min-heap for upward stops
            self.down_stops = []    # Max-heap for downward stops
            self.lock = Lock()

        def add_request(self, floor: int, direction: Direction):
            with self.lock:
                if direction == Direction.UP:
                    heappush(self.up_stops, floor)
                else:
                    heappush(self.down_stops, -floor)  # Max-heap trick

        def move(self):
            with self.lock:
                if self.direction == Direction.UP:
                    if self.up_stops:
                        self.current_floor = heappop(self.up_stops)
                    elif self.down_stops:
                        self.direction = Direction.DOWN
                elif self.direction == Direction.DOWN:
                    if self.down_stops:
                        self.current_floor = -heappop(self.down_stops)
                    elif self.up_stops:
                        self.direction = Direction.UP
                else:
                    if self.up_stops:
                        self.direction = Direction.UP
                    elif self.down_stops:
                        self.direction = Direction.DOWN

    class ElevatorController:
        def __init__(self, num_elevators: int, floors: int):
            self.elevators = [Elevator(i, floors) for i in range(num_elevators)]

        def request(self, floor: int, direction: Direction):
            # Find nearest elevator going in same direction
            best = min(self.elevators,
                       key=lambda e: abs(e.current_floor - floor)
                       if e.direction == direction or e.direction == Direction.IDLE
                       else float('inf'))
            best.add_request(floor, direction)
    ```

    **SCAN algorithm:** Elevator moves in one direction, servicing all requests, then reverses.

### High-Level Design

#### RazorpayX Payout System

??? success "Solution Approach"
    **Components:**
    ```
    ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
    │   API GW    │────▶│ Payout Svc   │────▶│  Ledger DB  │
    └─────────────┘     └──────────────┘     └─────────────┘
                               │
                               ▼
                        ┌──────────────┐     ┌─────────────┐
                        │ Bank Adapter │────▶│  Bank APIs  │
                        └──────────────┘     └─────────────┘
                               │
                               ▼
                        ┌──────────────┐
                        │ Webhook Svc  │ (Status updates)
                        └──────────────┘
    ```

    **Key design decisions:**

    1. **Idempotency:** Every payout has unique `idempotency_key`
    2. **Double-entry ledger:** Every payout = debit from source + credit to destination
    3. **State machine:** CREATED → PROCESSING → COMPLETED/FAILED
    4. **Bank adapter pattern:** Abstract different bank APIs (IMPS/NEFT/RTGS)
    5. **Async processing:** Queue payouts, process with retries
    6. **Reconciliation:** Daily batch to match our records with bank statements

#### Feature Flag Platform

??? success "Solution Approach"
    **Architecture:**
    ```
    ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
    │  Admin Panel  │────▶│  Flag Service │────▶│   Redis/DB    │
    └───────────────┘     └───────────────┘     └───────────────┘
                                  │
                                  ▼ (Push via SSE/WebSocket)
                          ┌───────────────┐
                          │ SDK (in-app)  │ ← Local cache
                          └───────────────┘
    ```

    **Implementation:**
    ```python
    class FeatureFlagService:
        def __init__(self):
            self.cache = {}  # Local cache with TTL

        def is_enabled(self, flag: str, user_id: str, context: dict) -> bool:
            flag_config = self.get_flag(flag)
            if not flag_config:
                return False

            # Check kill switch
            if flag_config.get("kill_switch"):
                return False

            # Check percentage rollout
            if "percentage" in flag_config:
                bucket = hash(f"{flag}:{user_id}") % 100
                if bucket >= flag_config["percentage"]:
                    return False

            # Check targeting rules
            for rule in flag_config.get("rules", []):
                if self._evaluate_rule(rule, context):
                    return rule["enabled"]

            return flag_config.get("default", False)
    ```

    **Key features:** Percentage rollout, targeting rules, kill switch, SDK caching.

### Behavioral & Leadership

#### Dealing with production outages

??? success "Sample Answer"
    **Situation:** Payment gateway went down during Diwali sale, affecting thousands of transactions.

    **Action:**
    1. **Immediate:** Activated incident response, formed war room
    2. **Communicate:** Sent status page update within 5 minutes
    3. **Triage:** Identified root cause (DB connection pool exhaustion)
    4. **Fix:** Scaled up connections, added circuit breaker for failover
    5. **Recover:** Replayed failed transactions from dead letter queue

    **Result:** 45-minute recovery. Post-mortem led to auto-scaling policies and better monitoring.

    **Key lesson:** Blameless post-mortems + runbooks prevent repeat incidents.

#### Handling architectural disagreements

??? success "Sample Answer"
    **Approach:**

    1. **Understand first:** Ask clarifying questions to understand their perspective fully
    2. **Document options:** Create an RFC with pros/cons of each approach
    3. **Data over opinions:** Run benchmarks, find similar case studies
    4. **Disagree and commit:** If team decides otherwise, support fully

    **Example:** Disagreed about using Kafka vs SQS. Created comparison doc, ran load tests. Data showed SQS met our needs with lower ops overhead. Team agreed. Later, we migrated to Kafka when scale demanded it—right tool at right time.

---

## Why Join Razorpay?
- Fintech-first engineering culture
- Ownership of products at scale
- Strong compensation + ESOPs
- Hypergrowth + internal mobility
- Flexible work and fast execution teams

