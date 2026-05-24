# The Parking Lot Problem: A Complete System Design Chapter

> *From OOP naivety to distributed concurrent resource allocation — a ground-up exploration*

---

## Preface: Why This Problem Is Harder Than It Looks

Most system design problems are abstractions — you never actually deploy a Twitter or design an Uber from scratch in a single interview. But a parking lot? You've been in one. You've sat in the queue at a mall on a Saturday. You've driven three floors up only to find every spot taken. You've watched two cars try to squeeze past each other in a narrow lane.

That lived experience is your greatest asset here. This chapter will teach you to take what you already feel intuitively — the frustration of a jammed entrance, the relief of finding a spot quickly — and translate it into the language of distributed systems, concurrency, and high-throughput architecture.

By the end, you won't just know *what* to design. You'll understand *why* each piece exists, and you'll be able to reason about it the same way a senior engineer would.

---

## Part 1: Building the Intuition First

### 1.1 What Is a Parking Lot, Really?

Before we write a single class or draw a single diagram, let's think about what a parking lot actually *does* at a mechanical level.

A parking lot is a physical space with a finite number of positions (slots), connected by a network of lanes, that must serve many cars simultaneously — each trying to find, reach, and occupy a free position, and later leave it.

That description immediately surfaces three challenges:

**First: Resource scarcity.** There are more potential cars than slots. At any point in time, you're managing a limited shared resource. This is the same core problem that databases face when managing connection pools, or operating systems face when managing memory pages.

**Second: Concurrency.** Hundreds of cars might arrive in the same five-minute window. They are not taking turns politely. They're all trying to do the same thing at the same time. This means your system must handle simultaneous, competing operations without breaking.

**Third: Physical movement.** Unlike a database, a parking lot has spatial constraints. A slot can be "logically assigned" to a car but the car still has to physically travel there. Two cars can collide. Lanes can deadlock. The software layer cannot simply ignore the physical world.

A clean, one-sentence framing that immediately signals senior thinking:

> "A parking lot is a concurrent resource allocation system with physical movement constraints."

This framing reorients the entire conversation. You're not building a UML diagram of `ParkingLot → Floor → Slot → Vehicle`. You're building something closer to a distributed lock manager with a real-world topology.

---

### 1.2 The Naive Approach — and Why It Fails

Let's first build the obvious thing, understand exactly how it breaks, and then build something better. This is how experienced engineers think: they don't skip to the answer, they trace the failure modes of simpler solutions to motivate the complex one.

**The obvious design looks like this:**

```
[Car arrives at gate]
        ↓
[Gate calls central server: "Find me a free slot"]
        ↓
[Server scans its slot list, finds first free slot]
        ↓
[Server marks slot as occupied, returns slot ID]
        ↓
[Gate opens, car drives to slot]
```

This works fine for a small, quiet parking lot with one gate and twenty cars a day. But let's think about what happens at a large shopping mall on a Sunday afternoon.

**Failure Mode 1: The Central Bottleneck**

Imagine 200 cars arriving in a ten-minute window. Every single one of them must talk to the central server before the gate can open. The server processes requests one at a time (or with limited parallelism). Cars queue up. Gates stay closed. The entrance road backs up onto the street. You've created a single point of serialization — one server that every car must pass through — and it becomes the ceiling on your entire system's throughput.

In engineering terms: you've built a system where throughput is bounded by the capacity of a single component. No matter how fast your cars move or how many gates you have, one slow server kills everything.

**Failure Mode 2: Lock Contention**

The server needs to make sure two cars don't get assigned the same slot. So it uses a lock — a mechanism that says "only one operation can modify the slot list at a time." While the lock is held for Car A, Cars B, C, D, E... all wait.

```
Car A holds lock → updates slot 42 → releases lock
                   [Cars B, C, D, E waiting]
Car B acquires lock → updates slot 18 → releases lock
                      [Cars C, D, E waiting]
... and so on
```

Every car is blocked by every other car, even if they would have taken completely different slots. The lock is too coarse — it protects the entire lot when it only needs to protect individual slots.

**Failure Mode 3: Single Point of Failure**

If the central server crashes, the gates freeze. No car can enter. The physical infrastructure (the lot, the slots, the lanes) is perfectly functional, but because the software has a single brain, the entire system stops. This is why distributed systems engineers are so focused on eliminating single points of failure.

**Failure Mode 4: The Ghost Slot Problem**

Consider this sequence:
1. Server assigns slot 42 to Car A.
2. Car A drives toward slot 42.
3. Before Car A arrives, the server crashes and restarts.
4. The server, having lost its state, thinks slot 42 is free.
5. Server assigns slot 42 to Car B.
6. Car A and Car B both arrive at slot 42.

This is called a **ghost occupancy** problem — the system believes a slot is free when it isn't (or vice versa). It happens when your state lives in memory and isn't durably committed before the assignment is confirmed.

---

## Part 2: Core Concepts You Need to Understand

Before we build the better system, we need to understand three foundational concepts. These are general computer science ideas, explained here in the context of parking lots.

---

### 2.1 What Is a Mutex (and Why It's a Problem at Scale)?

A **mutex** (short for "mutual exclusion") is a programming construct that ensures only one thread of execution can access a resource at a time. Think of it like a bathroom with a single-key lock: you take the key, use the bathroom, return the key. While you have the key, nobody else can get in.

```
// Pseudocode example
mutex lotLock;

function assignSlot(car):
    lotLock.acquire()          // take the key
    slot = findFreeSlot()      // find a slot
    slots[slot] = OCCUPIED     // mark it
    lotLock.release()          // return the key
    return slot
```

This is safe (no two cars get the same slot) but slow. At scale, with hundreds of cars per minute, threads spend more time waiting for the lock than actually doing work. The lock becomes the bottleneck.

The deeper problem is that a lot-wide mutex is far too coarse. Car A taking slot 42 and Car B taking slot 91 have absolutely no conflict with each other — they're different slots. There's no reason they shouldn't happen simultaneously. But with a single mutex on the whole lot, they can't.

---

### 2.2 What Is an Atomic Operation?

An **atomic operation** is one that completes entirely or not at all, with no intermediate state visible to anyone else. The word comes from the Greek *atomos* — indivisible.

Consider incrementing a counter. Non-atomically, this is three steps:
1. Read the value (say, 5).
2. Add 1 to get 6.
3. Write 6 back.

If two threads do this simultaneously, both might read 5, both compute 6, and both write 6 — so the counter ends up at 6 instead of 7. One increment was lost. This is called a **race condition**.

An atomic increment does all three steps as one indivisible operation. No other thread can see a state "between" the steps.

Modern CPUs have hardware support for atomic operations. The most important one for our purposes is **Compare-And-Swap (CAS)**.

```
CAS(memory_location, expected_value, new_value):
    if *memory_location == expected_value:
        *memory_location = new_value
        return SUCCESS
    else:
        return FAILURE  // someone else changed it first
```

CAS says: "Only update this value if it still equals what I last read. If someone else changed it in the meantime, tell me and I'll try again." This is the fundamental building block of lock-free data structures.

---

### 2.3 What Is Lock-Free Programming?

Lock-free programming means writing concurrent code that makes progress without using mutex locks. Instead of blocking waiting for a lock, threads retry their operations when they detect a conflict.

The key insight: **in a high-concurrency, low-conflict environment, optimistic approaches win.** If conflicts are rare (most of the time, different cars want different slots), it's faster to assume no conflict, try the operation, and retry on the rare occasion that there was one — rather than acquiring a lock every time "just in case."

This is the same philosophy behind optimistic concurrency control in databases (optimistic locking using version numbers vs pessimistic locking using row locks).

---

### 2.4 What Is a Bitmap?

A **bitmap** is an array of bits where each bit represents the state of one item. For a parking lot with 64 slots:

```
Position: 63 62 61 ... 5  4  3  2  1  0
Bit:        1  1  0 ... 1  0  1  1  0  1

1 = free
0 = occupied
```

Reading or writing a single bit is extremely fast — modern CPUs handle 64 bits in a single instruction. More importantly, **a 64-slot floor can be represented in a single 64-bit integer**, and atomic operations on 64-bit integers are natively supported by all modern hardware.

This means: checking if any slot is free, and reserving one, can be done in a single atomic CPU instruction. No database. No network call. No lock. Just one CPU instruction.

---

## Part 3: Lock-Free Bitmap Allocation — The Heart of the Design

Now we can build something much better.

### 3.1 Per-Floor Atomic Bitmaps

Each floor maintains a single 64-bit atomic integer where each bit represents one slot.

```
Floor A:  atomic<uint64_t> freeSlots = 0b11101111_10110111_...
                                         ↑ bit 63       ↑ bit 0
                                         1 = free, 0 = occupied
```

When a car wants to park on Floor A, here's what happens — no server, no lock, just atomic operations:

**Step 1: Load the current bitmap**
```
current = freeSlots.load()
// current = 11101111...
```

**Step 2: Find the lowest free bit**
```
slot = findLowestSetBit(current)
// slot = 0 (bit 0 is set, meaning slot 0 is free)
```

Finding the lowest set bit is a single CPU instruction on x86 (`BSF` — Bit Scan Forward). This is incredibly fast.

**Step 3: Compute the new bitmap (with that slot cleared)**
```
desired = current & ~(1 << slot)
// Clear bit 0: 11101111... → 11101110...
```

**Step 4: Attempt Compare-And-Swap**
```
success = freeSlots.compare_exchange_weak(current, desired)
```

This says: "If `freeSlots` still equals `current` (nobody changed it since I read it), update it to `desired` (with my slot reserved). Otherwise, tell me it failed."

**Step 5: Handle the result**
```
if success:
    car has slot 0 — done!
else:
    // Someone else grabbed a slot between my read and my CAS
    // Go back to Step 1 and try again with the fresh value
    retry
```

**Why does retry work?** If the CAS fails, it means another car took *some* slot in that instant. But there may still be other free slots. We re-read the bitmap (which `compare_exchange_weak` updates for us on failure), find a new candidate slot, and try again. In practice, retries are rare because conflicts are rare.

**Let's trace two cars arriving simultaneously:**

```
Time →
Car A: read bitmap = 11110000  | compute slot=4, desired=11100000  | CAS succeeds → Car A gets slot 4
Car B: read bitmap = 11110000  | compute slot=4, desired=11100000  | CAS FAILS (A already changed it)
Car B: read bitmap = 11100000  | compute slot=5, desired=11000000  | CAS succeeds → Car B gets slot 5
```

Car A and Car B never blocked each other. Car B just did one retry. Total time: microseconds. Compare this to the mutex approach where Car B would have waited for Car A to complete its entire critical section.

---

### 3.2 The Slot State Machine

Every individual slot transitions through states. Understanding these transitions is critical to avoiding the ghost slot problem.

```
                    ┌─────────────────────────────────────────────┐
                    │                                             │
                    ▼                                             │
              ┌──────────┐    car reserves    ┌──────────────┐   │
              │   FREE   │ ─────────────────► │   RESERVED   │   │
              └──────────┘                    └──────────────┘   │
                    ▲                               │    │        │
                    │                    TTL expires│    │ sensor │
                    │                         (30s) │    │detects │
                    │                               │    │  car   │
                    │                               ▼    ▼        │
                    │                          ┌──────────────┐   │
                    │       car exits          │   OCCUPIED   │   │
                    └──────────────────────────│              │───┘
                                               └──────────────┘
                                                      │
                                               (maintenance)
                                                      │
                                                      ▼
                                               ┌──────────────┐
                                               │  UNAVAILABLE │
                                               └──────────────┘
```

**FREE:** Slot is available. Its bit in the bitmap is 1.

**RESERVED:** A car has been assigned this slot but hasn't physically arrived yet. The bit is 0 (so no other car is assigned here), but we know it's a soft hold. This reservation has a TTL (time-to-live) — typically 2–5 minutes. If the car doesn't arrive in time, the slot automatically reverts to FREE. This prevents a car that got assigned slot 42 but then drove the wrong way from permanently blocking that slot.

**OCCUPIED:** A physical sensor (pressure plate, infrared, camera) has confirmed a car is actually present. The reservation is now committed. This is the "two-phase" commit: Phase 1 is logical reservation, Phase 2 is physical confirmation.

**UNAVAILABLE:** Slot is taken out of service (for maintenance, EV charger installation, etc.). Excluded from all allocation decisions.

The key insight of two phases is fault tolerance. If your server crashes between RESERVED and OCCUPIED, the TTL ensures the slot eventually frees itself. You never end up with permanently "ghost reserved" slots.

---

### 3.3 Slot State Storage

The bitmap handles the binary FREE/NOT-FREE distinction with maximum efficiency. But RESERVED and OCCUPIED are richer states that need more information (who reserved it, when, TTL). These live in a separate lightweight store:

```
Slot State Store (per zone):
{
  "slot_42": {
    "state": "RESERVED",
    "reservation_id": "uuid-abc-123",
    "reserved_by": "car_plate_KA01AB1234",
    "reserved_at": 1716534210,
    "ttl_expires_at": 1716534390,  // +3 minutes
    "slot_type": "STANDARD"
  }
}
```

The bitmap is the fast path: for allocation decisions, you only need to know FREE vs NOT-FREE, and the bitmap gives you that in a single CPU instruction. The state store is the slow path: for auditing, billing, TTL management, and physical confirmation, you consult the richer store.

---

## Part 4: Zone-Based Architecture — Scaling Horizontally

A single floor can have at most 64 slots in our bitmap (since a `uint64_t` is 64 bits). What about larger lots? And more importantly, how do we avoid creating a new bottleneck by having all floors share one allocator?

### 4.1 Zone Decomposition

The answer is to decompose the parking lot into **zones** — independent sub-lots, each managing its own resources, communicating with a lightweight coordinator only for cross-zone awareness.

```
┌─────────────────────────────────────────────────────────┐
│                    PARKING LOT                          │
│                                                         │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────┐  │
│   │    Zone A    │   │    Zone B    │   │  Zone C  │  │
│   │  Ground + L1 │   │   L2 + L3   │   │   Roof   │  │
│   │              │   │              │   │          │  │
│   │ Bitmap:      │   │ Bitmap:      │   │ Bitmap:  │  │
│   │ 0xF3A7...    │   │ 0x9C2B...    │   │ 0x1E5D.. │  │
│   │              │   │              │   │          │  │
│   │ Entry: Gate1 │   │ Entry: Gate2 │   │Entry:G3  │  │
│   │ Exit:  Gate4 │   │ Exit: Gate5  │   │Exit: G6  │  │
│   └──────────────┘   └──────────────┘   └──────────┘  │
│                                                         │
│              Zone Coordinator (lightweight)             │
│         "Zone A: 120 free | B: 45 free | C: 8 free"    │
└─────────────────────────────────────────────────────────┘
```

Each zone has:
- Its own atomic bitmap(s) — one per floor within the zone.
- Its own state store for RESERVED/OCCUPIED tracking.
- Its own entry and exit gates (ideally).
- Its own local routing logic.
- Its own sensors and edge controllers.

The Zone Coordinator is a lightweight aggregator — it doesn't handle individual slot assignments. It just maintains zone-level counts ("Zone A has approximately 120 free spots") so that incoming cars can be directed to the right zone before they enter. Once inside a zone, the zone handles everything locally.

This is directly analogous to database sharding: instead of one giant table with a global lock, you split into shards, each managing its own subset of data independently.

### 4.2 Zone Coordinator — What It Does and Doesn't Do

**Does:**
- Maintain approximate free-slot counts per zone (updated periodically, not on every allocation).
- Route incoming cars to the least-congested zone.
- Provide zone-level availability for pre-arrival apps ("Zone B has lots of space, Zone C is nearly full").
- Handle zone-to-zone transfer reservations (rare edge case where a car changes its mind).

**Does NOT:**
- Handle individual slot assignments (each zone does this locally).
- Maintain a lock that all zones must acquire.
- Become a bottleneck — it handles queries, not transactions.

The coordinator's count is **approximate** and **eventually consistent** — zones push updates every few seconds rather than on every allocation. This means the coordinator might say "Zone A has 30 free spots" when it actually has 28 or 32. That's fine. Its job is to give cars a direction to drive, not an exact count. The actual slot reservation happens locally in the zone, with exact atomic guarantees.

---

## Part 5: Movement and Physical Deadlock Prevention

We've solved the logical allocation problem. Now the harder problem: physical movement.

### 5.1 What Is a Physical Deadlock?

In narrow parking lot lanes, two cars can face each other head-on with no room to pass. Neither can move forward. Neither can easily reverse (especially with cars behind them). This is a **physical deadlock** — the software equivalent of two threads each holding a resource the other needs, both waiting forever.

```
      ←←←←
   [Car B] 🚗         🚗 [Car A]
                →→→→

   Both trying to occupy the same narrow lane
   from opposite ends simultaneously
```

Unlike software deadlocks, physical deadlocks can't be "killed and restarted." They require human intervention (tow trucks, angry honking, a traffic marshal). Your system design should make them impossible, not just handle them after the fact.

### 5.2 One-Way Traffic Graph

The most elegant solution: **make all lanes one-directional**.

Model the parking lot as a **directed graph** where nodes are intersections and edges are lane segments. Each edge has a direction — a car can only traverse it in one specific direction.

```
ENTRY
  │
  ▼
[Gate 1] ──► [Ramp A] ──► [Floor 1 Main Lane] ──► [Slot Cluster A1]
                                │
                                ▼
                         [Floor 1 Cross] ──► [Slot Cluster A2]
                                │
                                ▼
                         [Ramp to Floor 2]
                                │
                                ▼
                         [Floor 2 Main Lane] ──► ...
```

With a directed graph, cycles are impossible by design — and deadlocks require cycles. Cars always flow in one direction. You never have two cars facing each other in the same lane because they approach from the same direction.

Real-world parking lots often implement this with physical one-way signs and painted arrows. Your software model must reflect and enforce this topology.

### 5.3 Lane Segment Leases — Fine-Grained Movement Control

For added safety (and for autonomous vehicle scenarios where precision matters), each lane segment can issue **micro-leases**: short-lived, lightweight locks on a segment of lane.

Think of it like railway blocks: a train can only enter a section of track if that section is clear, and it "owns" that section until it exits. This prevents two trains (or cars) from occupying the same stretch at the same time.

```
Lane Segment: "F1-Lane-Section-3"
  Lease held by: Car_KA01AB1234
  Lease issued at: 14:22:05.123
  Lease expires: 14:22:35.123  (30 second TTL)
  Status: OCCUPIED_IN_TRANSIT
```

These leases are managed by **edge controllers** — small computers embedded in the parking lot infrastructure near each lane segment. The edge controller uses sensors (cameras, pressure plates, RFID readers) to:
1. Detect when a car enters a lane segment → issue lease.
2. Detect when a car exits a lane segment → release lease.
3. Deny entry to a car trying to enter an occupied segment.
4. Auto-release leases if the TTL expires (car broke down, sensor malfunction).

Importantly, edge controllers operate locally. They don't need to contact a central server to issue or release leases. This keeps lane management fast and decentralized.

---

## Part 6: The Complete Allocation Flow — End to End

Let's trace a single car's journey through the fully designed system, from arrival to parking.

### Phase 1: Pre-Arrival (Optional but Powerful)

The driver opens an app before leaving home.

```
Driver App ──► Zone Coordinator API: "I want to park at Mall X around 3pm"
Zone Coordinator: "Zone B has 45 free spots, Zone A has 120. Recommending Zone A."
Driver App: "Reserve slot in Zone A for 3:05 PM – 6:00 PM"
Zone Coordinator ──► Zone A Allocator: "Soft-reserve one slot for this booking"
Zone A Allocator: CAS on bitmap → bit cleared → slot 23 reserved
Reservation ID returned: "RES-uuid-789"
```

The reservation lives in Zone A's state store with a TTL. If the driver doesn't show up within 15 minutes of the reservation time, it expires.

### Phase 2: Arrival and Gate Entry

Car arrives at Gate 1.

```
Car approaches Gate 1
Gate 1 camera reads license plate OR driver scans QR code
Gate 1 controller checks Zone Coordinator: "Which zone should this car go to?"
Zone Coordinator returns: "Zone A, already has reservation RES-uuid-789"
  (or, if no reservation: "Zone A, ~120 free spots, enter there")
Gate 1 displays on screen: "Welcome! Proceed to Zone A. Slot A-23"
Gate opens
```

Note: the gate does not assign the slot — it only directs the car. Slot assignment already happened atomically in Phase 1. If no prior reservation, the slot is assigned right now via the same CAS-on-bitmap mechanism.

### Phase 3: In-Lot Routing

Navigation signage throughout the lot (dynamic LED signs, app turn-by-turn) guides the car along the directed lane graph to the assigned slot. Because lanes are one-directional, there's no ambiguity about the path.

### Phase 4: Physical Confirmation (Two-Phase Commit)

Car arrives at Slot A-23. A pressure sensor in the slot detects a car.

```
Slot A-23 pressure sensor: "Weight detected"
Edge controller for Slot A-23:
  → Reads reservation: RES-uuid-789, expected car: KA01AB1234
  → Camera confirms plate matches
  → Updates slot state: RESERVED → OCCUPIED
  → Updates bitmap: bit already 0 (was cleared at reservation time), no change needed
  → Notifies Zone A state store: slot 23 now OCCUPIED, TTL removed
  → Notifies billing system: parking session started at 14:22:07
```

The transition from RESERVED to OCCUPIED is the "commit" of the two-phase allocation. Only now is the parking session fully confirmed in a durable, billing-ready state.

### Phase 5: Exit

Car is ready to leave. Driver walks back to car, drives to exit lane.

```
Car approaches exit gate (Gate 4)
Gate 4 camera reads license plate: KA01AB1234
Gate 4 controller: lookup active session for this plate
  → Session found: Zone A, Slot 23, started 14:22:07
  → Current time: 17:45:33
  → Duration: 3 hours 23 minutes
  → Charge calculated: ₹85
Driver taps pay (UPI/card/pre-paid balance)
Payment confirmed
Gate 4 opens

Simultaneously:
  Slot A-23 pressure sensor: "Weight gone"
  Edge controller: slot 23 state: OCCUPIED → FREE
  CAS on Zone A bitmap: bit 23 set back to 1 (free)
  Zone A notifies Zone Coordinator: free count updated
```

The exit is just as atomic as the entry. The slot's bit is set back to 1 via CAS — no lock required, just one atomic operation.

---

## Part 7: Handling Edge Cases and Failures

### 7.1 What If the Zone Allocator Crashes?

The system must be **idempotent** under failures. If the allocator crashes mid-reservation:

```
Scenario: Allocator crashes after CAS (bit cleared) but before writing to state store.

Recovery:
  New allocator instance starts up.
  Scans state store: slot 23 has no reservation record.
  Scans bitmap: slot 23 bit is 0 (occupied in bitmap).
  Discrepancy detected → slot 23 flagged as "ORPHANED".
  After 5-minute reconciliation window (no sensor confirmation):
    Slot 23 bit reset to 1 (free).
  Slot recovered.
```

The reconciliation process runs periodically (every few minutes) comparing bitmap state vs state store vs sensor readings. Discrepancies are resolved conservatively (when in doubt, treat as free only after sensor confirms absence).

Reservation IDs are globally unique (UUID) so that retried reservation requests can be deduplicated — even if the client retries after a timeout, the allocator can recognize "I've already processed RES-uuid-789" and return the same result without double-allocating.

### 7.2 What If a Car Parks Without a Reservation?

Walk-in cars are handled by the same CAS mechanism — just without the pre-arrival phase. At the gate, the gate controller triggers an immediate allocation from the zone's bitmap.

```
Walk-in car at Gate 1:
  Zone Coordinator: "Zone A has ~120 free, send them there"
  Gate 1 → Zone A allocator: "Allocate one slot now"
  Zone A: CAS on bitmap → finds free bit → clears it → returns slot 31
  Gate 1: "Welcome! Proceed to Zone A, Slot A-31"
  Gate opens
  Walk-in treated identically to reserved car from here
```

### 7.3 What If a Car Takes the Wrong Slot?

Happens more than you'd think. Driver parks in A-23 but their reservation was for A-31.

```
Sensor at A-23 detects car, reads plate: KA01AB1234
State store lookup: A-23 is currently RESERVED by KA05XY9999
  But the detected plate is KA01AB1234 whose reservation is for A-31.

Resolution:
  1. Check if A-31 sensor has detected anything: No (still empty).
  2. Check if KA01AB1234 is allocated to A-31: Yes.
  3. Determine: car parked in wrong slot.
  4. Options:
     a. If zone has capacity, auto-reassign KA01AB1234 to A-23 (where they actually are),
        and free A-31 back to pool.
     b. Alert system operator / display message to driver to move car.
  5. KA05XY9999's reservation remains valid; system routes them to next available slot.
```

### 7.4 Hotspot Prevention

A hotspot occurs when many cars compete for the same set of slots simultaneously, creating CAS retry loops even though the lot has plenty of capacity elsewhere.

This happens naturally if your allocation always picks the "first free bit" (lowest-numbered slot) — because all cars try slot 0, then slot 1, etc. You get many CAS conflicts on the low-numbered bits even when high-numbered slots are perfectly free.

**Solution: Randomized Bit Probing**

Instead of always starting from bit 0, each allocator request starts from a random offset in the bitmap:

```
function allocateSlot(bitmap):
  start = random(0, 63)                // pick a random starting position
  for i in 0..63:
    slot = (start + i) % 64            // wrap around
    if bit[slot] is set (free):
      attempt CAS to clear this bit
      if CAS succeeds: return slot     // success
      else: continue to next bit       // someone else got here first

  return NO_SLOT_AVAILABLE
```

Now Car A starts probing from bit 37, Car B starts from bit 12, Car C starts from bit 55. They spread out across the bitmap naturally. CAS conflicts become rare even under heavy load. This is directly analogous to how tcmalloc (Google's high-performance memory allocator) spreads allocations across thread-local arenas to minimize contention.

---

## Part 8: Phases of Scale — Growing the System

A well-designed system doesn't start at "city-wide distributed parking network." It grows through phases as demand increases. Here's how to think about each phase.

---

### Phase 1: Single Lot, Single Zone (0 – 500 slots)

**Architecture:**
- One zone, no zone coordinator needed.
- One atomic bitmap per floor.
- Simple state store (can be Redis or even an in-memory map for very small lots).
- Gates talk directly to the allocator process.
- Manual sensor integration or simple RFID.

**Appropriate for:** Small office building, boutique hotel, neighborhood shopping complex.

**Characteristics:**
- Peak throughput: ~20 cars/minute at entry.
- Single-region deployment.
- Downtime acceptable (maintenance windows at night).
- Simple billing.

---

### Phase 2: Single Lot, Multiple Zones (500 – 5000 slots)

**What's new:**
- Decompose floors into 3-5 zones.
- Add lightweight Zone Coordinator.
- Introduce per-zone state stores (can still be a single Redis instance with keyspace prefixes).
- Add dynamic signage at entry to direct cars to zones.
- Mobile app for reservations.

**Architecture additions:**

```
                    Zone Coordinator
                   /        |        \
               Zone A    Zone B    Zone C
              (Bitmap)  (Bitmap)  (Bitmap)
                 |          |         |
             Gate 1-2   Gate 3-4  Gate 5-6
```

**Characteristics:**
- Peak throughput: ~100 cars/minute across all gates.
- Zone-level failure isolation: if Zone B allocator crashes, A and C keep working.
- Pre-reservation system adds predictable demand.

---

### Phase 3: Multi-Lot, City-District Scale (5 – 50 lots)

**What's new:**
- Each lot is now an independent distributed node with its own coordinator.
- A Regional Coordinator aggregates availability across lots.
- City-level app integration: "Find parking near your destination."
- Event routing: "Stadium event in 2 hours — pre-fill nearby lots."
- Integration with navigation apps (Google Maps, Apple Maps, Waze) via API.

**Architecture:**

```
         Regional Coordinator
        /          |          \
     Lot X       Lot Y       Lot Z
      |            |            |
 Zone Coord    Zone Coord   Zone Coord
  /  |  \       /  |  \      /  |  \
 ZA  ZB  ZC   ZA  ZB  ZC   ZA  ZB  ZC
```

**New challenges at this scale:**
- **Cross-lot consistency:** If the regional coordinator says "Lot X has 50 free spots," that number must be fresh enough to be useful. Too stale and you'll route cars to a full lot.
- **Network partitions:** What if Lot Y can't reach the Regional Coordinator? It should still work — just operate in "island mode," accepting cars based on local state.
- **Data aggregation latency:** Zone-level counts propagate to lot level (~1 second), then to regional level (~5 seconds). At the regional level, displayed counts are always slightly stale. That's acceptable — a 5-second-old count for directional routing is fine.

---

### Phase 4: City-Wide Infrastructure (50+ lots)

**What's new:**
- Predictive allocation: ML model predicts demand based on time-of-day, events, weather, historical patterns. Pre-reserves clusters of slots before demand arrives.
- Dynamic pricing: slots priced based on real-time demand (like Uber surge pricing).
- Autonomous vehicle integration: AVs communicate directly with the parking system to negotiate slots and navigation paths.
- CRDT-based occupancy maps: eventual consistency for city-wide dashboards without coordination overhead.
- Kafka event streams: every slot state change emits an event; downstream consumers (billing, analytics, city traffic management) subscribe independently.

**Regarding CRDTs:** A **Conflict-free Replicated Data Type** is a data structure designed to be replicated across multiple nodes with no coordination required — nodes can update independently and their states will eventually converge. For parking, an occupancy counter can be a CRDT: each zone updates its own counter, and the city-wide total is computed by summing all zone counters without any node needing to "agree" on a single value at a single moment.

---

## Part 9: The Complete High-Level Architecture

Here is the full system architecture at Phase 3 (multi-lot, district scale), which is the level most appropriate for a senior engineering interview:

```
╔══════════════════════════════════════════════════════════════════════════╗
║                         CITY / REGIONAL LAYER                           ║
║                                                                          ║
║   ┌─────────────────────────────────────────────────────────────────┐   ║
║   │              Regional Parking Coordinator                       │   ║
║   │   [REST/gRPC API]  [Availability Cache]  [Event Bus (Kafka)]    │   ║
║   └─────────────┬──────────────────┬──────────────────┬────────────┘   ║
║                 │                  │                  │                  ║
╚═════════════════╪══════════════════╪══════════════════╪══════════════════╝
                  │                  │                  │
╔═════════════════╪══════╗  ╔════════╪═══════╗  ╔══════╪════════════════╗
║    PARKING LOT X       ║  ║  PARKING LOT Y  ║  ║  PARKING LOT Z      ║
║                        ║  ║                 ║  ║                     ║
║  ┌──────────────────┐  ║  ║ ...             ║  ║ ...                 ║
║  │  Lot Coordinator │  ║  ╚═════════════════╝  ╚═════════════════════╝
║  │  (Zone routing,  │  ║
║  │  lot-level avail)│  ║
║  └──────┬─────┬─────┘  ║
║         │     │        ║
║  ┌──────┘     └──────┐ ║
║  ▼                   ▼ ║
║  ┌──────────┐  ┌──────────┐  ┌──────────┐
║  │  Zone A  │  │  Zone B  │  │  Zone C  │
║  │──────────│  │──────────│  │──────────│
║  │ Bitmap   │  │ Bitmap   │  │ Bitmap   │
║  │ StateDB  │  │ StateDB  │  │ StateDB  │
║  │ Sensors  │  │ Sensors  │  │ Sensors  │
║  │ EdgeCtrl │  │ EdgeCtrl │  │ EdgeCtrl │
║  └────┬─────┘  └────┬─────┘  └────┬─────┘
║       │              │              │
║  ┌────┘    PHYSICAL LAYER           └─────┐
║  │                                        │
║  [Gate 1]   [Gate 2]   [Gate 3]   [Gate 4-Exit]
║     │          │          │
║  [Lanes]   [Lanes]   [Lanes]  → [One-Way Directed Graph]
║     │          │          │
║  [Slots]   [Slots]   [Slots]  → [Sensors, Cameras, RFID]
╚════════════════════════════════════════════════════════╝

CLIENT LAYER:
  [Driver Mobile App] → [Pre-Reservation API] → [Regional Coordinator]
  [Navigation Apps] → [Availability API] → [Regional Coordinator]
  [Operator Dashboard] → [Analytics Service] → [Kafka Consumers]
```

---

## Part 10: Concurrency Summary — The Full Mental Model

Let's consolidate the concurrency guarantees at each layer:

| Layer | Mechanism | Guarantee |
|---|---|---|
| Individual slot reservation | Atomic CAS on bitmap | Exactly one car per slot, no locks |
| Cross-slot isolation | Per-zone bitmaps | Cars wanting different slots never contend |
| Two-phase commit | TTL + sensor confirmation | Ghost slots auto-expire |
| Lane movement | Directed graph + edge leases | Physical deadlock impossible |
| Zone-level routing | Eventually consistent counts | Fast routing, tolerable staleness |
| Failure recovery | Idempotent UUIDs | Safe retries, no double-allocation |
| Cross-lot coordination | Regional coordinator (read-heavy) | Availability routing, not transaction processing |
| City-wide analytics | Kafka + CRDT aggregates | Scalable, coordination-free data pipelines |

---

## Part 11: Interview Strategy — How to Present This

### The Opening (First 60 Seconds)

Don't start with classes. Don't start with a UML diagram. Start here:

> "Before I define any classes or APIs, I want to establish the core insight: a parking lot is fundamentally a concurrent resource allocation system with physical movement constraints. That framing changes everything about how we design it."

Then immediately surface the three core problems: efficient slot discovery, collision-free allocation, and movement deadlock prevention. This shows you've thought about the problem, not just its surface representation.

### The Pivot (When Asked to Go Deeper)

When the interviewer asks "how does concurrent reservation work?", this is your moment to introduce the bitmap CAS mechanism. Walk through the example slowly — show the bit operations, explain why CAS avoids locking, trace the two-car simultaneous arrival scenario.

### The Scaling Discussion (Final Phase)

When asked "how would you scale this?", walk through the four phases. Each phase should feel motivated by a new failure mode of the previous phase — not arbitrary additional complexity.

### The Closing Statement

> "The real challenge in parking lot design isn't object modeling — it's building a low-contention concurrent resource allocator that also respects physical movement constraints. Standard OOP decomposition handles neither of those well. That's why we need atomic bitmaps, zone-based sharding, directed movement graphs, and two-phase allocation."

---

## Part 12: Concepts Introduced — Quick Reference

| Concept | Simple Explanation |
|---|---|
| **Mutex** | A lock that only one thread can hold at a time. Fast for single-threaded access; terrible at scale. |
| **Atomic CAS** | CPU instruction: update a value only if it still matches what you last read. Foundation of lock-free code. |
| **Lock-free bitmap** | A 64-bit integer where each bit = one slot's availability. CAS on this integer = lock-free slot reservation. |
| **Compare-And-Swap** | "Update this only if unchanged since I last read it; otherwise tell me and I'll retry." |
| **Two-phase allocation** | Phase 1: logical reservation (CAS bitmap). Phase 2: physical confirmation (sensor). Prevents ghost slots. |
| **TTL (Time-To-Live)** | A timeout on soft state. Reservations not confirmed within TTL auto-expire. |
| **Zone sharding** | Split the lot into independent zones, each managing its own allocation. Analogous to database sharding. |
| **Directed graph** | Lane network with one-way edges. Eliminates cyclic paths. Deadlock requires cycles. No cycles → no deadlock. |
| **Edge controller** | Small embedded computer at each lane segment. Issues micro-leases. Operates independently of central server. |
| **Idempotency** | Same operation, applied multiple times, produces the same result. Enables safe retries after failures. |
| **Hotspot** | A resource that many threads compete for simultaneously. Randomized probing spreads contention across slots. |
| **CRDT** | Data structure that can be updated independently on multiple nodes and will eventually reach the same state without coordination. |
| **Eventual consistency** | Different nodes may temporarily disagree on values, but will converge given no new updates. Acceptable for routing; not for reservations. |

---

*This chapter covers the parking lot problem at the level expected of senior engineers — not as an exercise in object-oriented modeling, but as a practical lesson in concurrent resource allocation, distributed coordination, and scalable system architecture. The same patterns appear in memory allocators, database connection pools, distributed locks, and inventory management systems. Master the parking lot, and you've mastered a dozen other problems.*
