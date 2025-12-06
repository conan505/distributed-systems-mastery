# Uber Interview Experience

## Salary Snapshot
- **Base:** ₹45-50 LPA
- **Stocks (RSUs):** ₹15–25L+
- **Performance Bonus:** ~15%
- **Total CTC:** ₹75–90 LPA (varies by experience & location)

---

## Interview Rounds
1. Online Assessment – 2 DSA problems (medium-hard)
2. 2 Rounds of DSA + Problem Solving
3. LLD/System Design Round
4. Hiring Manager Round
5. Behavioral + Bar Raiser

---

## Common DSA Problems

#### Subarray with at most K distinct elements

??? success "Solution Approach"
    ```python
    def subarrays_with_k_distinct(nums: list[int], k: int) -> int:
        """
        Count subarrays with AT MOST k distinct =
        atMost(k) - atMost(k-1) gives EXACTLY k distinct
        """
        def at_most(k):
            count = 0
            left = 0
            freq = {}

            for right, num in enumerate(nums):
                freq[num] = freq.get(num, 0) + 1

                while len(freq) > k:
                    freq[nums[left]] -= 1
                    if freq[nums[left]] == 0:
                        del freq[nums[left]]
                    left += 1

                count += right - left + 1  # All subarrays ending at right

            return count

        return at_most(k)  # Or at_most(k) - at_most(k-1) for exactly k
    ```

    **Time:** O(n), **Space:** O(k)

#### Meeting Rooms II (Minimum Platforms)

??? success "Solution Approach"
    ```python
    import heapq

    def min_meeting_rooms(intervals: list[list[int]]) -> int:
        """
        Min-heap tracks end times of ongoing meetings.
        Time: O(n log n), Space: O(n)
        """
        if not intervals:
            return 0

        intervals.sort(key=lambda x: x[0])  # Sort by start time
        heap = []  # Min-heap of end times

        for start, end in intervals:
            if heap and heap[0] <= start:
                heapq.heappop(heap)  # Reuse room
            heapq.heappush(heap, end)

        return len(heap)
    ```

    **Alternative:** Two-pointer on sorted start/end arrays.

#### Merge K Sorted Lists

??? success "Solution Approach"
    ```python
    import heapq

    def merge_k_lists(lists):
        """
        Min-heap approach: O(N log k) where N = total nodes
        """
        heap = []
        for i, lst in enumerate(lists):
            if lst:
                heapq.heappush(heap, (lst.val, i, lst))

        dummy = ListNode(0)
        curr = dummy

        while heap:
            val, i, node = heapq.heappop(heap)
            curr.next = node
            curr = curr.next
            if node.next:
                heapq.heappush(heap, (node.next.val, i, node.next))

        return dummy.next
    ```

## Low-Level Design (LLD) Topics

#### Parking Lot System

??? success "Solution Approach"
    ```python
    from abc import ABC, abstractmethod
    from enum import Enum
    from typing import Optional
    import heapq

    class VehicleType(Enum):
        MOTORCYCLE = 1
        CAR = 2
        TRUCK = 3

    class ParkingSpot:
        def __init__(self, id: str, spot_type: VehicleType, floor: int):
            self.id = id
            self.spot_type = spot_type
            self.floor = floor
            self.vehicle = None

        def is_available(self) -> bool:
            return self.vehicle is None

        def can_fit(self, vehicle_type: VehicleType) -> bool:
            return vehicle_type.value <= self.spot_type.value

    class ParkingLot:
        def __init__(self):
            self.spots = {}  # spot_id -> ParkingSpot
            self.available = {vt: [] for vt in VehicleType}  # Min-heap by floor

        def add_spot(self, spot: ParkingSpot):
            self.spots[spot.id] = spot
            heapq.heappush(self.available[spot.spot_type], (spot.floor, spot.id))

        def park(self, vehicle_type: VehicleType) -> Optional[str]:
            # Find smallest spot that fits (prefer lower floors)
            for vt in VehicleType:
                if vt.value >= vehicle_type.value and self.available[vt]:
                    floor, spot_id = heapq.heappop(self.available[vt])
                    self.spots[spot_id].vehicle = vehicle_type
                    return spot_id
            return None  # No spot available

        def unpark(self, spot_id: str) -> bool:
            spot = self.spots.get(spot_id)
            if spot and spot.vehicle:
                spot.vehicle = None
                heapq.heappush(self.available[spot.spot_type], (spot.floor, spot_id))
                return True
            return False
    ```

    **Key patterns:** Strategy for pricing, Observer for notifications, Singleton for ParkingLot.

## System Design

#### Design Uber Backend

??? success "Solution Approach"
    **Core Components:**
    ```
    ┌─────────────┐     ┌──────────────┐     ┌─────────────┐
    │   Mobile    │────▶│   API GW     │────▶│ Ride Service│
    │    App      │     │  (Kong/Envoy)│     └─────────────┘
    └─────────────┘     └──────────────┘            │
          │                                         ▼
          │ WebSocket              ┌──────────────────────────┐
          ▼                        │     Matching Service     │
    ┌─────────────┐               │  (Geospatial + ML)       │
    │  Location   │◀──────────────└──────────────────────────┘
    │   Service   │                        │
    └─────────────┘                        ▼
          │                        ┌─────────────┐
          ▼                        │   Driver    │
    ┌─────────────┐               │   Service   │
    │    Redis    │               └─────────────┘
    │  (GeoHash)  │
    └─────────────┘
    ```

    **Key Design Decisions:**

    1. **Location Tracking:**
       - Drivers send location every 4 seconds via WebSocket
       - Store in Redis using GeoHash: `GEOADD drivers:city lng lat driver_id`
       - Query nearby: `GEORADIUS drivers:city lng lat 5 km`

    2. **Ride Matching Algorithm:**
       ```python
       def match_driver(rider_location, radius_km=5):
           # 1. Find nearby available drivers
           nearby = redis.georadius("drivers", lng, lat, radius_km)

           # 2. Filter by availability, rating, acceptance rate
           candidates = [d for d in nearby if d.is_available and d.rating > 4.0]

           # 3. Score by ETA (not just distance)
           for driver in candidates:
               driver.eta = routing_service.get_eta(driver.location, rider_location)

           # 4. Return best match
           return min(candidates, key=lambda d: d.eta)
       ```

    3. **Surge Pricing:**
       - Track demand (ride requests) vs supply (available drivers) per geohash cell
       - Surge multiplier = demand / supply (capped at 3x)

    4. **Reliability:**
       - Saga pattern for ride lifecycle (request → match → pickup → complete)
       - Dead letter queue for failed notifications
       - Circuit breaker for payment service

#### Design Real-time Location Tracking

??? success "Solution Approach"
    **Architecture:**
    ```
    Driver App ──WebSocket──▶ Location Ingestion ──Kafka──▶ Location Processor
                                                                    │
                                                                    ▼
    Rider App ◀──WebSocket── Notification Service ◀── Redis (GeoHash)
    ```

    **Key decisions:**
    - **WebSocket** for bi-directional real-time updates
    - **Kafka** for buffering high-volume location updates
    - **Redis GeoHash** for efficient spatial queries
    - **Cell-based subscription:** Rider subscribes to driver's geohash cell, not individual driver

    **Optimization:**
    - Batch location updates (every 4s, not every 1s)
    - Delta compression (send only if moved > 10m)
    - Hierarchical geohash (coarse for far, fine for near)

