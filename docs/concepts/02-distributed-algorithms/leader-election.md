# Leader Election

## What It Is

**Leader Election** is a distributed algorithm that selects one node from a cluster to act as the coordinator or leader, responsible for critical tasks like writes, scheduling, or coordination.

---

## The Analogy 👑

Think of **electing a class president**:
- Any student can run for president
- Students vote
- Highest votes wins
- If president leaves school → new election
- Only one president at a time

---

## Why It Exists

### The Problem: Who's in Charge?
```
Scenario: Database cluster with 5 nodes
- All need to know who handles writes
- If leader fails, need new leader
- Can't have two leaders (split-brain)

Without leader election:
- Split-brain: Two nodes think they're leader
- No leader: System stalls
- Conflicting decisions
```

### What Leader Election Solves:
- **Single point of coordination** - One decision maker
- **Fault tolerance** - Automatic failover
- **Consistency** - Avoid conflicts
- **Partition handling** - Only majority can elect

---

## Common Algorithms

### 1. Bully Algorithm
```
┌─────────────────────────────────────────────────────────────────┐
│                    Bully Algorithm                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Nodes: 1, 2, 3, 4, 5 (higher ID = higher priority)           │
│   Current leader: 5                                              │
│                                                                  │
│   5 fails:                                                       │
│   1. Node 3 detects, sends ELECTION to 4, 5                     │
│   2. Node 4 responds OK (bullies 3)                              │
│   3. Node 5 no response (dead)                                   │
│   4. Node 4 sends ELECTION to 5                                  │
│   5. No response → 4 becomes LEADER                             │
│   6. Node 4 broadcasts COORDINATOR message                       │
│                                                                  │
│   + Simple, deterministic                                        │
│   - High message complexity O(n²)                                │
│   - Assumes reliable failure detection                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2. Ring Algorithm
```
┌─────────────────────────────────────────────────────────────────┐
│                    Ring Algorithm                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│              1 ──────▶ 2                                        │
│              ▲          │                                        │
│              │          ▼                                        │
│              5          3                                        │
│              ▲          │                                        │
│              │          ▼                                        │
│              └──── 4 ◀──┘                                       │
│                                                                  │
│   1. Node detects failure, sends ELECTION with its ID           │
│   2. Each node adds its ID, forwards to next                    │
│   3. Message returns to initiator with all alive IDs            │
│   4. Highest ID elected, COORDINATOR broadcasted                │
│                                                                  │
│   + Lower message complexity O(n)                                │
│   - Single ring failure problematic                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Raft-based Election
```
┌─────────────────────────────────────────────────────────────────┐
│                    Raft Leader Election                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   States: FOLLOWER → CANDIDATE → LEADER                         │
│                                                                  │
│   1. Followers wait for heartbeat (random timeout)              │
│   2. On timeout, become CANDIDATE, increment term               │
│   3. Vote for self, request votes from others                   │
│   4. Win with majority → become LEADER                          │
│   5. Leader sends heartbeats to prevent new elections           │
│                                                                  │
│   Term numbers prevent split-brain:                              │
│   - Higher term always wins                                      │
│   - Stale leaders step down                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Split-Brain Prevention

### Problem:
```
Network partition:
[Node1, Node2] | [Node3, Node4, Node5]

Without protection:
- Left side elects Node2 as leader
- Right side elects Node5 as leader
- Two leaders = data corruption!
```

### Solutions:
```
1. Quorum requirement: Need majority (n/2 + 1) to elect
   - Left: 2 nodes, need 3 → can't elect
   - Right: 3 nodes, need 3 → can elect

2. Fencing tokens: Each leader gets unique, increasing token
   - Resources only accept highest token

3. Lease-based: Leader holds time-limited lease
   - Must renew before expiration
```

---

## When to Use

### ✅ Good Fit:
- **Database primary selection** - Write coordination
- **Distributed locks** - Lock manager election
- **Job scheduling** - Single scheduler
- **Consensus groups** - Raft/Paxos leader

### ❌ Avoid When:
- Can use external coordinator (ZooKeeper, etcd)
- Leaderless design works (Cassandra, DynamoDB)
- Single node sufficient

---

## Real-World Implementations

| System | Approach |
|--------|----------|
| **ZooKeeper** | ZAB protocol, built-in election |
| **etcd** | Raft leader election |
| **Kafka** | Controller election via ZooKeeper |
| **Redis Sentinel** | Quorum-based election |
| **Consul** | Raft for server leadership |

---

## Implementation Deep Dive

### Raft-based Leader Election (Production-Ready)

```python
import random
import time
import threading
from enum import Enum
from dataclasses import dataclass
from typing import Dict, Optional, Callable

class NodeState(Enum):
    FOLLOWER = "follower"
    CANDIDATE = "candidate"
    LEADER = "leader"

@dataclass
class VoteRequest:
    term: int
    candidate_id: str
    last_log_index: int
    last_log_term: int

@dataclass
class VoteResponse:
    term: int
    vote_granted: bool

class RaftNode:
    """
    Simplified Raft leader election implementation.
    """

    def __init__(self, node_id: str, peers: list[str], rpc_client: Callable):
        self.node_id = node_id
        self.peers = peers
        self.rpc = rpc_client

        # Persistent state
        self.current_term = 0
        self.voted_for: Optional[str] = None
        self.log = []

        # Volatile state
        self.state = NodeState.FOLLOWER
        self.leader_id: Optional[str] = None
        self.votes_received = set()

        # Timing
        self.election_timeout = self._random_timeout()
        self.last_heartbeat = time.time()
        self.heartbeat_interval = 0.15  # 150ms

        self._lock = threading.Lock()
        self._running = True

    def _random_timeout(self) -> float:
        """Random timeout between 150-300ms to prevent split votes."""
        return random.uniform(0.15, 0.30)

    def start(self):
        """Start the election timer and heartbeat threads."""
        threading.Thread(target=self._election_timer, daemon=True).start()
        threading.Thread(target=self._heartbeat_sender, daemon=True).start()

    def _election_timer(self):
        """Check for election timeout and start election if needed."""
        while self._running:
            time.sleep(0.05)  # Check every 50ms

            with self._lock:
                if self.state == NodeState.LEADER:
                    continue

                if time.time() - self.last_heartbeat > self.election_timeout:
                    self._start_election()

    def _start_election(self):
        """Transition to candidate and request votes."""
        self.state = NodeState.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id
        self.votes_received = {self.node_id}
        self.election_timeout = self._random_timeout()
        self.last_heartbeat = time.time()

        print(f"[{self.node_id}] Starting election for term {self.current_term}")

        # Request votes from all peers
        for peer in self.peers:
            threading.Thread(
                target=self._request_vote,
                args=(peer,),
                daemon=True
            ).start()

    def _request_vote(self, peer: str):
        """Send vote request to a peer."""
        request = VoteRequest(
            term=self.current_term,
            candidate_id=self.node_id,
            last_log_index=len(self.log) - 1,
            last_log_term=self.log[-1].term if self.log else 0
        )

        try:
            response = self.rpc(peer, "request_vote", request)
            self._handle_vote_response(response)
        except Exception as e:
            print(f"[{self.node_id}] Failed to get vote from {peer}: {e}")

    def _handle_vote_response(self, response: VoteResponse):
        with self._lock:
            if response.term > self.current_term:
                self._step_down(response.term)
                return

            if self.state != NodeState.CANDIDATE:
                return

            if response.vote_granted:
                self.votes_received.add(response.voter_id)

                # Check if we have majority
                if len(self.votes_received) > (len(self.peers) + 1) // 2:
                    self._become_leader()

    def _become_leader(self):
        """Transition to leader state."""
        self.state = NodeState.LEADER
        self.leader_id = self.node_id
        print(f"[{self.node_id}] Became LEADER for term {self.current_term}")

        # Send immediate heartbeat to establish authority
        self._send_heartbeats()

    def _step_down(self, new_term: int):
        """Step down to follower when seeing higher term."""
        self.current_term = new_term
        self.state = NodeState.FOLLOWER
        self.voted_for = None
        self.leader_id = None

    def _heartbeat_sender(self):
        """Send periodic heartbeats as leader."""
        while self._running:
            time.sleep(self.heartbeat_interval)

            with self._lock:
                if self.state == NodeState.LEADER:
                    self._send_heartbeats()

    def _send_heartbeats(self):
        """Send heartbeat (empty AppendEntries) to all peers."""
        for peer in self.peers:
            threading.Thread(
                target=lambda p: self.rpc(p, "heartbeat", self.current_term),
                args=(peer,),
                daemon=True
            ).start()

    def handle_vote_request(self, request: VoteRequest) -> VoteResponse:
        """Handle incoming vote request from candidate."""
        with self._lock:
            if request.term > self.current_term:
                self._step_down(request.term)

            vote_granted = False
            if request.term >= self.current_term:
                if self.voted_for is None or self.voted_for == request.candidate_id:
                    # Check log is at least as up-to-date
                    if self._is_log_up_to_date(request):
                        self.voted_for = request.candidate_id
                        vote_granted = True
                        self.last_heartbeat = time.time()

            return VoteResponse(term=self.current_term, vote_granted=vote_granted)

    def handle_heartbeat(self, leader_term: int):
        """Handle heartbeat from leader."""
        with self._lock:
            if leader_term >= self.current_term:
                self.current_term = leader_term
                self.state = NodeState.FOLLOWER
                self.last_heartbeat = time.time()
```

### Using ZooKeeper for Leader Election

```python
from kazoo.client import KazooClient
from kazoo.recipe.election import Election

class ZooKeeperLeaderElection:
    """
    Production-ready leader election using ZooKeeper.
    Uses ephemeral sequential nodes for automatic failover.
    """

    def __init__(self, zk_hosts: str, election_path: str, node_id: str):
        self.zk = KazooClient(hosts=zk_hosts)
        self.election_path = election_path
        self.node_id = node_id
        self.is_leader = False
        self._my_node = None

    def start(self):
        self.zk.start()
        self.zk.ensure_path(self.election_path)
        self._join_election()

    def _join_election(self):
        """Create ephemeral sequential node and check leadership."""
        # Create ephemeral sequential node
        self._my_node = self.zk.create(
            f"{self.election_path}/node-",
            value=self.node_id.encode(),
            ephemeral=True,
            sequence=True
        )

        self._check_leadership()

    def _check_leadership(self):
        """Check if we're the leader (lowest sequence number)."""
        children = sorted(self.zk.get_children(self.election_path))
        my_name = self._my_node.split("/")[-1]

        if children[0] == my_name:
            self.is_leader = True
            self._on_become_leader()
        else:
            # Watch the node before us
            my_index = children.index(my_name)
            prev_node = children[my_index - 1]

            @self.zk.DataWatch(f"{self.election_path}/{prev_node}")
            def watch_prev(data, stat):
                if stat is None:  # Node deleted
                    self._check_leadership()
                return not self.is_leader  # Stop watching if we're leader

    def _on_become_leader(self):
        """Called when this node becomes leader."""
        print(f"[{self.node_id}] I am now the LEADER!")
        # Start leader duties...

    def resign(self):
        """Voluntarily give up leadership."""
        if self._my_node:
            self.zk.delete(self._my_node)
            self.is_leader = False
```

### Redis-based Leader Election (Simpler Alternative)

```python
import redis
import time
import threading

class RedisLeaderElection:
    """
    Simple leader election using Redis with lease-based approach.
    Good for simpler use cases where ZooKeeper is overkill.
    """

    def __init__(self, redis_client: redis.Redis, key: str,
                 node_id: str, lease_seconds: int = 30):
        self.redis = redis_client
        self.key = key
        self.node_id = node_id
        self.lease_seconds = lease_seconds
        self.is_leader = False
        self._running = True

    def start(self):
        """Start the election loop."""
        threading.Thread(target=self._election_loop, daemon=True).start()

    def _election_loop(self):
        while self._running:
            try:
                # Try to acquire leadership
                acquired = self.redis.set(
                    self.key,
                    self.node_id,
                    nx=True,  # Only if not exists
                    ex=self.lease_seconds
                )

                if acquired:
                    self.is_leader = True
                    self._on_become_leader()
                else:
                    # Check if we're still the leader
                    current_leader = self.redis.get(self.key)
                    if current_leader and current_leader.decode() == self.node_id:
                        # Renew lease
                        self.redis.expire(self.key, self.lease_seconds)
                        self.is_leader = True
                    else:
                        self.is_leader = False

            except redis.RedisError as e:
                print(f"Redis error: {e}")
                self.is_leader = False

            time.sleep(self.lease_seconds / 3)  # Renew at 1/3 of lease time

    def _on_become_leader(self):
        print(f"[{self.node_id}] Acquired leadership")
```

---

## Interview Tips

When discussing Leader Election:
1. Explain why single leader is needed (coordination, consistency)
2. Describe split-brain problem and quorum solution
3. Know Raft's approach (terms, heartbeats, random timeouts)
4. Mention practical tools (ZooKeeper, etcd, Redis)
5. Discuss trade-offs (availability during election, complexity)
6. Explain fencing tokens for preventing stale leaders

