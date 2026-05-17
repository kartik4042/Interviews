# 🚗 Uber Interview: Why Does Uber Generate a Different OTP Per Ride?
### *The 4-Layer Systems Design Answer That Separates Senior Engineers From the Rest*

---

> **The Question (as asked in Uber interviews):**
> *"Why does Uber generate a different OTP for every single ride, while Rapido uses the same OTP? The same OTP is simpler, faster, and easier for users to remember — so why is Uber's approach better?"*

> **The weak answer:** "Security."
>
> **The strong answer:** State isolation, ride-level idempotency, fraud containment, concurrency correctness, blast radius reduction, and support reconciliation — all at the scope of the ride as a transactional unit.

---

## Table of Contents

1. [The Mental Model: OTP as User PIN vs OTP as Trip Token](#1-the-mental-model)
2. [System Architecture: How Uber Scopes the OTP](#2-system-architecture)
3. [Layer 1 — State Transition and Ride Lifecycle](#3-layer-1-state-transition)
4. [Layer 2 — Idempotency and Concurrency Correctness](#4-layer-2-idempotency)
5. [Layer 3 — Fraud, Replay Attacks, and Blast Radius](#5-layer-3-fraud)
6. [Layer 4 — Support, Reconciliation, and Audit Trail](#6-layer-4-audit)
7. [Why Rapido's Approach Is a Valid Engineering Trade-off](#7-rapido-tradeoff)
8. [The Backend Complexity Uber Accepts](#8-backend-complexity)
9. [How Real Companies Apply This Pattern](#9-real-world-references)
10. [Bottlenecks Resolved: Summary Matrix](#10-summary-matrix)
11. [The Closing Statement (For the Interview)](#11-closing-statement)

---

## 1. The Mental Model

Before any code or diagram: the entire answer hinges on one conceptual shift.

```mermaid
graph LR
    subgraph Rapido["🛵 Rapido Mental Model"]
        R_OTP["OTP = f(user)"]
        R_DESC["Static per-user PIN\nLike a password\nReusable across rides\nProves: 'I am this user'"]
        R_OTP --> R_DESC
    end

    subgraph Uber["🚗 Uber Mental Model"]
        U_OTP["OTP = f(ride session)"]
        U_DESC["Dynamic per-ride token\nLike a session key\nSingle-use, short TTL\nProves: 'I am THIS rider,\nentering THIS vehicle,\nfor THIS trip, RIGHT NOW'"]
        U_OTP --> U_DESC
    end

    style Rapido fill:#ffeaa7,stroke:#f39c12
    style Uber fill:#d5f4e6,stroke:#27ae60
```

> **The shift:** From `user-scoped identity verification` → `transaction-scoped session authentication`

This is the exact same principle that separates:
- **Session cookies** (user-scoped) from **JWT tokens** (request-scoped)
- **API keys** (account-scoped) from **OAuth tokens** (scope+TTL-scoped)
- **Database passwords** (connection-scoped) from **row-level encryption keys** (record-scoped)

At small scale, user-scoped works fine. At Uber's scale — millions of concurrent rides, driver reassignments, multi-device logins, scheduled rides, family profiles — user-scoped authentication creates ambiguity that breaks everything downstream.

---

## 2. System Architecture

### 2.1 Rapido: User-Scoped OTP (Simplified)

```mermaid
graph TD
    subgraph Rapido["Rapido Architecture (User-Scoped OTP)"]
        UR["Rider Account\nOTP: 4821\n(static, reusable)"]
        DR["Driver App\nEnters: 4821"]
        SVC["OTP Service\nValidates: user_id → OTP match?"]
        RIDE["Ride Started ✅"]

        UR -->|"tells driver"| DR
        DR -->|"submit"| SVC
        SVC -->|"match → start"| RIDE
    end

    subgraph Problems["⚠️ Problems at Scale"]
        P1["Driver from yesterday\nstill knows 4821"]
        P2["Same OTP valid for\nRide A, B, C, D..."]
        P3["Reassigned driver?\nOld driver still valid"]
        P4["Which ride does\nthis OTP start?"]
    end

    Rapido --> Problems

    style Problems fill:#ffeaa7,stroke:#e67e22
```

### 2.2 Uber: Ride-Scoped OTP (Full Architecture)

```mermaid
graph TD
    subgraph Request["Ride Request"]
        RIDER["Rider App\nrider_id: R_789"]
    end

    subgraph Dispatch["Dispatch Service"]
        MATCH["Driver Matching\ndriver_id: D_456\nvehicle_id: V_321"]
    end

    subgraph OTP["OTP Generation Service"]
        GEN["Generate OTP\nInputs:\n• ride_id: RIDE_001\n• rider_id: R_789\n• driver_id: D_456\n• vehicle_id: V_321\n• timestamp: T\n\nOutput: OTP = 6274\nTTL: 10 minutes\nStored in Redis with ride_id key"]
    end

    subgraph Delivery["OTP Delivery"]
        RIDER_OTP["Rider App\nShows: 6274"]
        DRIVER_OTP["Driver App\nWaiting for: 6274"]
    end

    subgraph Verification["Verification at Pickup"]
        RIDER_SHARE["Rider tells driver:\n'6274'"]
        DRIVER_ENTER["Driver enters:\n6274"]
        VERIFY["OTP Service\nValidates:\nride_id + driver_id + OTP = ✅\nState: DRIVER_ARRIVED → IN_PROGRESS"]
    end

    subgraph Downstream["Downstream Systems Triggered"]
        PAY["Payment Service\nride_id → charge rider"]
        PAY_D["Driver Payout\nride_id → credit driver"]
        GPS["GPS Tracking\nride_id → start trace"]
        AUDIT["Audit Log\nOTP verified at T+3min"]
    end

    RIDER --> MATCH
    MATCH --> GEN
    GEN --> RIDER_OTP
    GEN --> DRIVER_OTP
    RIDER_OTP --> RIDER_SHARE
    DRIVER_ENTER --> VERIFY
    RIDER_SHARE --> DRIVER_ENTER
    VERIFY --> PAY
    VERIFY --> PAY_D
    VERIFY --> GPS
    VERIFY --> AUDIT

    style GEN fill:#2980b9,color:#fff
    style VERIFY fill:#27ae60,color:#fff
    style Downstream fill:#dfe6e9
```

---

## 3. Layer 1 — State Transition and Ride Lifecycle

The OTP is not a verification code. It is a **state transition gate**.

```mermaid
stateDiagram-v2
    [*] --> RIDE_REQUESTED : Rider books trip

    RIDE_REQUESTED --> DRIVER_MATCHED : Dispatch assigns driver
    note right of DRIVER_MATCHED
        OTP generated here.
        Tied to: ride_id + driver_id + rider_id
        Stored in Redis with TTL = 10min
    end note

    DRIVER_MATCHED --> DRIVER_EN_ROUTE : Driver accepts

    DRIVER_EN_ROUTE --> DRIVER_ARRIVED : Driver reaches pickup

    DRIVER_ARRIVED --> IN_PROGRESS : OTP verified ✅
    note right of IN_PROGRESS
        This is the ONLY gate.
        No OTP = no state transition.
        Per-ride OTP = one valid transition.
        Same request twice = idempotent return.
    end note

    DRIVER_ARRIVED --> RIDE_CANCELLED : Timeout / OTP expired
    note right of RIDE_CANCELLED
        If driver reassigned:
        New OTP generated.
        Old driver's OTP invalidated.
    end note

    IN_PROGRESS --> COMPLETED : Drop-off reached
    COMPLETED --> [*] : Payment, payout, audit triggered

    RIDE_CANCELLED --> DRIVER_MATCHED : Retry with new driver
```

### The OTP as a State Transition Token

```mermaid
graph LR
    subgraph Before["Before OTP Verified"]
        S1["State: DRIVER_ARRIVED\nPayment: BLOCKED\nGPS trace: BLOCKED\nDriver payout: BLOCKED"]
    end

    subgraph Gate["OTP Verification Gate"]
        G["ride_id + driver_id + OTP\n= valid?\n\n✅ Yes → transition\n❌ No → reject"]
    end

    subgraph After["After OTP Verified"]
        S2["State: IN_PROGRESS\nPayment: AUTHORIZED\nGPS trace: STARTED\nDriver payout: PENDING\nAudit: LOGGED"]
    end

    Before -->|"driver enters code"| Gate
    Gate -->|"atomic state change"| After

    style Gate fill:#2980b9,color:#fff
    style After fill:#27ae60,color:#fff
```

> **Why this matters:** The OTP is the only event that transitions a ride from `DRIVER_ARRIVED` to `IN_PROGRESS`. Every financial, safety, and tracking system downstream trusts this single gate. If that gate is ambiguous (reusable OTP = ambiguous), everything downstream inherits that ambiguity.

---

## 4. Layer 2 — Idempotency and Concurrency Correctness

This is the layer most candidates miss entirely.

### 4.1 The Concurrent Ride Problem

```mermaid
graph TD
    subgraph Scenario["Uber Scale Concurrent Scenarios"]
        S1["👨‍👩‍👧 Family Profile\nSame account, 3 active rides\nRider A, Rider B, Rider C"]
        S2["📅 Scheduled Ride\nBooked 3 hours ago\nDriver assigned now\nOTP generated now"]
        S3["🔄 Driver Reassignment\nDriver A cancels mid-route\nDriver B assigned\nNew OTP needed"]
        S4["📱 Multi-device login\nRider on phone + tablet\nBoth show OTP"]
        S5["🔁 Network Retry\nDriver submits OTP\nNetwork timeout\nSDK retries submission"]
    end

    subgraph StaticOTPProblem["❌ With Static OTP (user-scoped)"]
        P1["Which ride does OTP 4821 start?\nAmbiguous across 3 family rides"]
        P2["Old driver from 6 hours ago\nstill has valid OTP"]
        P3["Retry triggers second ride start\nDouble-charge, corrupt state"]
    end

    subgraph PerRideOTPSolution["✅ With Per-Ride OTP"]
        SOL1["Each ride has unique OTP\nNo ambiguity — ever"]
        SOL2["Reassignment regenerates OTP\nOld driver's code invalid instantly"]
        SOL3["Idempotent verification:\nride_id + OTP = same result\non retry, no double-start"]
    end

    S1 --> P1
    S3 --> P2
    S5 --> P3
    S1 --> SOL1
    S3 --> SOL2
    S5 --> SOL3

    style StaticOTPProblem fill:#ffeaa7
    style PerRideOTPSolution fill:#d5f4e6
```

### 4.2 Idempotency: The Network Retry Problem

```mermaid
sequenceDiagram
    participant Driver as Driver App
    participant API as OTP Verification API
    participant DB as Ride State (Redis + DB)

    Note over Driver,DB: Scenario: Driver submits OTP, network drops

    Driver->>API: POST /verify-otp {ride_id: R001, otp: 6274, driver_id: D456}
    API->>DB: Check: ride_id R001, otp 6274 → valid?
    DB-->>API: Valid ✅, transition to IN_PROGRESS
    API->>DB: SET ride_state[R001] = IN_PROGRESS
    Note over API: Response lost in network ❌

    Driver->>API: POST /verify-otp {ride_id: R001, otp: 6274, driver_id: D456} (RETRY)
    API->>DB: Check: ride_id R001, otp 6274 → valid?
    DB-->>API: Already IN_PROGRESS, OTP already used
    API-->>Driver: 200 OK (idempotent return: "ride already started") ✅

    Note over Driver,DB: No double-start. No duplicate charge. Correct state.

    Note over Driver,DB: What happens with STATIC OTP on retry?
    Driver->>API: POST /verify-otp {user_id: U789, otp: 4821} (RETRY)
    API->>DB: Check: user U789, otp 4821 → valid?
    DB-->>API: Valid ✅ (static OTP still valid!)
    API->>DB: Start ANOTHER ride? Which one? 💥
    Note over DB: Ambiguity. Possible double-start. State corruption.
```

### 4.3 The Idempotency Key Formula

```
With per-ride OTP:
  idempotency_key = (ride_id, driver_id, rider_id, otp)
  → uniquely identifies one valid start event
  → same key submitted twice = safe no-op

With static OTP:
  idempotency_key = (user_id, otp)
  → could match multiple rides
  → retry could start the wrong ride
  → cannot be made safely idempotent
```

---

## 5. Layer 3 — Fraud, Replay Attacks, and Blast Radius

### 5.1 Attack Vectors That Static OTP Creates

```mermaid
graph TD
    subgraph StaticOTPAttacks["❌ Attack Vectors with Static OTP"]
        A1["🎭 Driver learns OTP once\nCan start fake rides later\nwithout rider present"]
        A2["🤝 Collusion fraud\nDriver + fake rider\nshare static OTP\nStart rides with no actual trip"]
        A3["📍 Crowded pickup ambiguity\nWrong driver, same OTP\nWrong rider boards wrong car"]
        A4["🔄 Cancellation fraud\nKnow OTP → cancel ride\nRe-start without rider\nCollect surge fare"]
        A5["🔁 Cross-ride replay\nOTP from RideA valid for RideB\nif not expired"]
    end

    subgraph PerRideOTPDefense["✅ Per-Ride OTP Eliminates These"]
        D1["OTP from yesterday useless\nKnowledge doesn't transfer"]
        D2["Fake ride needs valid ride_id\nGenerated server-side\nCannot be spoofed"]
        D3["OTP is ride-specific\nWrong driver submits wrong OTP\nVerification fails cleanly"]
        D4["After cancellation:\nNew ride_id, new OTP\nOld OTP invalid immediately"]
        D5["OTP only valid for its ride_id\nno cross-ride replay possible"]
    end

    A1 --- D1
    A2 --- D2
    A3 --- D3
    A4 --- D4
    A5 --- D5

    style StaticOTPAttacks fill:#ffeaa7
    style PerRideOTPDefense fill:#d5f4e6
```

### 5.2 Blast Radius Comparison

```mermaid
graph LR
    subgraph StaticBlast["❌ Static OTP Leaked"]
        SB["One leaked OTP\naffects ALL future rides\nfor that user\n\nFraud window:\nunlimited (until user resets)\nImpact: all rides, all drivers,\nall time"]
    end

    subgraph PerRideBlast["✅ Per-Ride OTP Leaked"]
        PB["One leaked OTP\naffects ONLY:\n• This ride_id\n• This driver_id\n• This time window (TTL: 10min)\n\nFraud window:\nexpires automatically\nImpact: exactly one ride"]
    end

    subgraph Principle["Systems Design Principle"]
        PR["Minimize failure domain.\nScope credentials to the\nsmallest possible unit of work."]
    end

    StaticBlast --> Principle
    PerRideBlast --> Principle

    style StaticBlast fill:#e74c3c,color:#fff
    style PerRideBlast fill:#27ae60,color:#fff
    style Principle fill:#2980b9,color:#fff
```

### 5.3 The Crowded Pickup Problem (Wrong-Trip Protection)

```mermaid
graph TD
    subgraph Scene["Airport Pickup — 50 UberX Waiting"]
        R1["Rider: OTP 6274\nFor: Toyota Camry\nPlate: KA-01-AB-1234"]
        D1["Driver A\nToyota Camry ✅\nRide: RIDE_001"]
        D2["Driver B (wrong car)\nHonda City ❌\nRide: RIDE_002"]
        D3["Driver C (fraudster)\nNo booking ❌"]
    end

    subgraph Verification["OTP Verification"]
        V1["Driver A enters 6274\nride_id=RIDE_001 + driver_id=D1 + otp=6274\n→ MATCH ✅ → Ride starts"]
        V2["Driver B enters 6274 (guessed)\nride_id=RIDE_002 + driver_id=D2 + otp=6274\n→ NO MATCH ❌ → Rejected"]
        V3["Driver C enters 6274\nNo valid ride_id for D3\n→ INVALID ❌ → Rejected"]
    end

    R1 --> V1
    D1 --> V1
    D2 --> V2
    D3 --> V3

    style V1 fill:#27ae60,color:#fff
    style V2 fill:#e74c3c,color:#fff
    style V3 fill:#e74c3c,color:#fff
```

---

## 6. Layer 4 — Support, Reconciliation, and Audit Trail

This is the layer almost no candidate mentions. At Uber's scale, it is as important as fraud prevention.

### 6.1 The Clean Audit Trail Per-Ride OTP Creates

```mermaid
graph LR
    subgraph PerRideAudit["✅ Per-Ride OTP Audit Trail"]
        A["T+00:00 — Ride requested\n rider_id: R_789"]
        B["T+02:15 — Driver matched\n driver_id: D_456\n OTP generated: 6274\n Stored: Redis[RIDE_001]"]
        C["T+06:30 — Driver arrived\n GPS: 12.9716° N, 77.5946° E"]
        D["T+06:45 — OTP verified\n ride_id: RIDE_001\n driver_id: D_456\n rider_id: R_789\n OTP: 6274 ✅\n State: IN_PROGRESS"]
        E["T+24:10 — Ride completed\n Distance: 8.3km\n GPS corroborated"]
        F["T+24:12 — Payment triggered\n ₹156 charged\n ride_id: RIDE_001"]

        A --> B --> C --> D --> E --> F
    end

    style D fill:#2980b9,color:#fff
    style F fill:#27ae60,color:#fff
```

### 6.2 Support Resolution: Per-Ride OTP vs Static OTP

```mermaid
graph TD
    subgraph Complaint["Rider complaint: 'I was charged for a ride I didn't take'"]
        COMP["Support ticket opened"]
    end

    subgraph StaticSupport["❌ With Static OTP — Support Investigation"]
        SS1["Was the rider's OTP 4821 used?"]
        SS2["When? Multiple rides may match..."]
        SS3["Was the driver the right one?\nCan't verify — OTP is user-level"]
        SS4["GPS match? Maybe. Timestamp? Maybe."]
        SS5["Resolution: Gut feel + user report\nHigh ambiguity → expensive manual review"]
        SS1 --> SS2 --> SS3 --> SS4 --> SS5
    end

    subgraph PerRideSupport["✅ With Per-Ride OTP — Support Investigation"]
        PS1["ride_id from charge record"]
        PS2["OTP 6274 generated at T+02:15\nfor driver D_456, vehicle V_321"]
        PS3["OTP verified at T+06:45\nGPS: matches pickup location"]
        PS4["GPS corroborates: 8.3km trip completed"]
        PS5["Resolution: Definitive. Automated.\nUnambiguous evidence chain."]
        PS1 --> PS2 --> PS3 --> PS4 --> PS5
    end

    COMP --> StaticSupport
    COMP --> PerRideSupport

    style SS5 fill:#e74c3c,color:#fff
    style PS5 fill:#27ae60,color:#fff
```

> **At Uber's scale:** Even a 0.1% dispute rate at 20 million rides/day = 20,000 disputes/day. Ambiguous audit trails that require manual resolution at that volume cost tens of millions of dollars per year in support staffing.

---

## 7. Why Rapido's Approach Is a Valid Engineering Trade-off

Rapido's choice is not wrong — it is an intentional trade-off based on a different threat model.

```mermaid
graph TD
    subgraph RapidoContext["Rapido's Context (Bike Taxi)"]
        RC1["Avg ride value: ₹40–80\nFraud cost per incident: low"]
        RC2["Ride duration: 5–15 min\nOTP complexity overhead: noticeable UX cost"]
        RC3["Primary market: tier-2/3 cities\nUser familiarity with OTPs: variable"]
        RC4["Backend team size: smaller\nInfrastructure cost: material"]
        RC5["Concurrent rides per user: almost always 1\nAmbiguity risk: low"]
    end

    subgraph ThreatModel["Rapido's Threat Model Math"]
        TM["fraud_cost_per_incident × fraud_rate\nvs\ninfra_cost + UX_friction_cost + support_overhead\n\nAt Rapido's scale and ticket size:\nfraud_cost < system_complexity_cost\n→ Static OTP is rational"]
    end

    subgraph UberContext["Uber's Context (Car + Premium)"]
        UC1["Avg ride value: ₹200–2000+\nFraud cost per incident: high"]
        UC2["Prime Time, surge, airport rides:\nhigh-value fraud targets"]
        UC3["Scheduled rides, family profiles,\nmulti-device: ambiguity is real"]
        UC4["20M+ rides/day: 0.01% fraud\n= 2,000 fraud events/day"]
        UC5["Per-ride OTP cost amortized\nacross massive ride volume"]
    end

    RapidoContext --> ThreatModel
    UberContext --> ThreatModel

    style ThreatModel fill:#2980b9,color:#fff
```

> **The engineering lesson:** There is no universally correct answer. The right design depends on threat model, scale, ticket size, and operational cost. Rapido's static OTP is not a mistake — it is a calibrated trade-off that makes sense at their scale and risk profile. Uber's per-ride OTP makes sense at their scale and risk profile.

---

## 8. The Backend Complexity Uber Accepts

Per-ride OTP is not free. Here is everything Uber's systems must handle that Rapido's don't.

```mermaid
graph TD
    subgraph OTPLifecycle["OTP Lifecycle Management (Uber)"]
        GEN["1. OTP Generation\nCryptographically random\nNot sequential (no guessing)\nStored: Redis with ride_id key + TTL"]
        DIST["2. OTP Distribution\nRider: in-app display\nDriver: waiting state\nConsistency: both get same OTP"]
        TTL["3. TTL Management\nDefault: 10–15 min window\nExpiry = auto-invalidation\nNo manual cleanup needed"]
        REGEN["4. Reassignment Handling\nDriver cancels → old OTP invalidated\nNew driver assigned → new OTP generated\nRace condition: fencing tokens prevent stale OTP use"]
        VERIFY["5. Verification API\nIdempotent: same request = same result\nAtomic: verify + state transition in one operation\nDistributed lock: prevent concurrent verification"]
        AUDIT["6. Audit Logging\nOTP generated: logged\nOTP verified: logged\nOTP expired unused: logged\nAll downstream to S3 + data warehouse"]
        SYNC["7. Offline Driver Sync\nDriver in low connectivity area\nOTP cached locally (encrypted)\nVerification queued, synced on reconnect"]
    end

    GEN --> DIST --> TTL --> REGEN --> VERIFY --> AUDIT --> SYNC

    style GEN fill:#2980b9,color:#fff
    style VERIFY fill:#27ae60,color:#fff
    style REGEN fill:#e67e22,color:#fff
```

### The Redis Data Model

```
Key:   OTP:RIDE_001
Value: {
    otp:        "6274",
    ride_id:    "RIDE_001",
    rider_id:   "R_789",
    driver_id:  "D_456",
    vehicle_id: "V_321",
    status:     "PENDING",        ← changes to USED after verification
    generated:  1718000000,
    ttl:        600               ← 10 minutes in seconds
}
TTL: 600 seconds (auto-expiry)
```

```mermaid
graph LR
    subgraph Redis["Redis OTP Store"]
        K["Key: OTP:RIDE_001\nTTL: 600s"]
        V["Value:\notp: 6274\nride_id: RIDE_001\nrider_id: R_789\ndriver_id: D_456\nstatus: PENDING"]
    end

    subgraph States["OTP Lifecycle States"]
        S1["PENDING\n(generated, not yet used)"]
        S2["USED\n(verified, ride started)\n→ becomes invalid"]
        S3["EXPIRED\n(TTL elapsed, never used)\n→ auto-deleted by Redis"]
        S4["INVALIDATED\n(driver reassigned)\n→ explicit delete + regen"]

        S1 -->|"OTP verified"| S2
        S1 -->|"TTL elapses"| S3
        S1 -->|"driver cancels"| S4
    end

    Redis --> States

    style S2 fill:#27ae60,color:#fff
    style S3 fill:#95a5a6,color:#fff
    style S4 fill:#e74c3c,color:#fff
```

---

## 9. How Real Companies Apply This Pattern

### 9.1 Stripe — Payment Idempotency Keys

Stripe uses a structurally identical pattern for payments. Every charge has an `idempotency_key` that scopes the operation to a single transaction — not to the user.

- Prevents duplicate charges on network retry
- Same key submitted twice = deterministic same result
- Key expires after 24 hours (transaction-scoped TTL)
- Audit trail: every idempotency key is logged with request fingerprint

> This is per-transaction scoping, identical to Uber's per-ride OTP.

📎 [Stripe Idempotent Requests Documentation](https://stripe.com/docs/api/idempotent_requests)

---

### 9.2 AWS STS — Session Tokens (Not Permanent Credentials)

AWS IAM could give every user a permanent access key. Instead, AWS Security Token Service (STS) issues short-lived session tokens.

- Session token = scoped to a session (not the user account)
- TTL: 15 minutes to 36 hours
- Leaked token expires automatically — blast radius contained
- Every session generates unique credentials

> Same principle: credential scoped to the unit of work, not the identity.

📎 [AWS STS Session Token Documentation](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetSessionToken.html)

---

### 9.3 Google — One-Time Authorization Codes (OAuth PKCE)

OAuth 2.0 authorization codes are single-use, short-TTL, and scoped to a specific authorization request. Reusing a code is a security error (RFC 6749 Section 4.1.2).

- Auth code expires in 10 minutes
- Auth code is single-use (invalidated after first exchange)
- Scoped to: client_id + redirect_uri + scope + state parameter
- Not reusable across different client sessions

> Same pattern: session-scoped, single-use, short TTL.

📎 [RFC 6749 — OAuth 2.0 Authorization Code](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1)

---

### 9.4 Ola, Lyft, DiDi — Industry Convergence

All major ride-hailing platforms at scale converged on per-ride OTP (or equivalent):

| Company | Mechanism | Scope | Why |
|---------|-----------|-------|-----|
| **Uber** | Per-ride OTP with Redis TTL | ride_id | Fraud, idempotency, audit |
| **Lyft** | Per-ride PIN | ride_id | Same fraud/safety concerns |
| **Ola** | Per-ride OTP | ride_id | Adopted as market matured |
| **DiDi** | Per-ride verification code | trip_id | Safety mandate, fraud prevention |
| **Rapido** | Static user OTP | user_id | Simpler, valid at current scale |

> Industry convergence is itself a signal: as platforms scale and fraud cost grows, the per-ride scoped model wins.

---

### 9.5 Banking — OTP TTL Best Practices (RBI Guidelines)

The Reserve Bank of India mandates that OTPs for financial transactions:
- Must expire within 30 minutes
- Must be single-use
- Must be transaction-scoped (not session-reusable)

Uber's ride = a financial transaction. The same regulatory logic applies.

📎 [RBI Guidelines on OTP-based Authentication](https://rbi.org.in/scripts/NotificationUser.aspx?Id=11522&Mode=0)

---

## 10. Bottlenecks Resolved: Summary Matrix

| # | Problem | With Static OTP | With Per-Ride OTP | Pattern Used |
|---|---------|----------------|-------------------|--------------|
| 1 | **Wrong-trip boarding** | Wrong driver can use leaked OTP | OTP tied to driver_id — wrong driver rejected | Scope minimization |
| 2 | **Driver reassignment safety** | Old driver retains valid OTP | Reassignment invalidates old OTP, generates new | Credential revocation |
| 3 | **Concurrent rides (family profile)** | OTP is ambiguous across rides | Each ride has unique OTP, no ambiguity | Session isolation |
| 4 | **Network retry → double ride start** | Retry re-validates same OTP, second start possible | Idempotent: ride_id + OTP = one result always | Idempotent state machine |
| 5 | **Replay attack from past rides** | OTP from past ride still valid | OTP expired (TTL 10min), unusable after ride | TTL-based expiry |
| 6 | **Cross-ride fraud (driver collusion)** | Fake rides using known static OTP | ride_id generated server-side, cannot be spoofed | Server-side token generation |
| 7 | **Blast radius on OTP leak** | All future rides compromised | Only one ride affected, expires automatically | Blast radius minimization |
| 8 | **Support reconciliation** | Ambiguous: which ride, which driver? | Clean audit: OTP → ride_id → driver → GPS → payment | Immutable audit trail |
| 9 | **Fraud analytics** | OTP reuse undetectable | OTP reuse = impossible; anomaly detection clean | Deterministic uniqueness |
| 10 | **Regulatory / financial compliance** | OTP reuse violates transaction-scoped norms | Compliant with transaction-scoped OTP standards | Regulatory alignment |

---

## 11. Closing Statement

> *For the Uber interviewer — the complete answer:*

**"The core difference is what the OTP proves. Rapido's OTP proves 'this is the customer.' Uber's OTP proves 'this exact rider is entering this exact vehicle for this exact trip right now.'**

**Uber scopes the OTP to the ride because the ride is the unit of risk — it is a financial transaction, a safety event, a driver payout, a fraud boundary, and a support reconciliation record all at once.**

**A per-ride OTP solves four concrete engineering problems that a static OTP cannot:**

**First, idempotency. Starting a ride is a state transition from DRIVER_ARRIVED to IN_PROGRESS. A per-ride OTP makes this cleanly idempotent — the same OTP submitted twice safely returns the same result, preventing double-starts on network retry.**

**Second, concurrency correctness. Uber handles family profiles, scheduled rides, multi-device logins, and driver reassignments simultaneously. A user-scoped OTP is ambiguous across concurrent rides. A ride-scoped OTP is unambiguous by construction.**

**Third, fraud containment. A static OTP creates cross-ride replay risk, wrong-trip boarding, and driver collusion vectors. A per-ride OTP expires in 10 minutes, is tied to a specific driver_id, and becomes useless the moment the ride ends or the driver is reassigned. Blast radius is exactly one ride.**

**Fourth, support and reconciliation. At 20 million rides per day, even a 0.1% dispute rate is 20,000 cases daily. Per-ride OTPs create an unambiguous audit chain — OTP generated, driver arrived, rider verified, GPS matched, payment triggered — that resolves disputes programmatically rather than manually.**

**Rapido's approach is a rational trade-off. At their scale and ticket size, the engineering cost of per-ride OTP infrastructure exceeds the fraud cost it would prevent. As they scale, they will likely converge on the same model — every major ride-hailing platform at scale has.**

**The deeper systems principle is: credentials should have narrow scope and short lifetime, and they should be scoped to the unit of work, not the identity of the user. The same principle governs AWS session tokens, Stripe idempotency keys, and OAuth authorization codes."**

---

*References:*
- [Stripe Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
- [AWS STS GetSessionToken](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetSessionToken.html)
- [RFC 6749 — OAuth 2.0 Authorization Code Flow](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1)
- [RBI OTP Guidelines for Financial Transactions](https://rbi.org.in/scripts/NotificationUser.aspx?Id=11522&Mode=0)
- [Google SRE Book — Designing for Failure](https://sre.google/sre-book/introduction/)
- [AWS Builders Library — Making Retries Safe with Idempotency](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [Martin Kleppmann — Designing Data-Intensive Applications, Chapter 9 (Consistency & Consensus)](https://dataintensive.net/)
