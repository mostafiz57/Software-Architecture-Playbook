# Mobility & Delivery Platform — Enterprise Architecture

**Architecture Design Document (ADD)**

A simple, production architecture for **rides** and **package delivery** on one platform. Shared **auth**, location, matching, pricing, payment, and notifications. Separate trip and delivery rules.

Reference product: **MobilityFlow**.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Business Requirements](#2-business-requirements)
3. [Functional Requirements](#3-functional-requirements)
4. [Non-Functional Requirements](#4-non-functional-requirements)
5. [Domain Analysis (DDD)](#5-domain-analysis-ddd)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Core Components / Services](#7-core-components--services)
8. [Database Design](#8-database-design)
9. [API Design](#9-api-design)
10. [Communication Patterns](#10-communication-patterns)
11. [Scalability Strategy](#11-scalability-strategy)
12. [Performance Considerations](#12-performance-considerations)
13. [Security Considerations](#13-security-considerations)
14. [Reliability & Fault Tolerance](#14-reliability--fault-tolerance)
15. [Deployment Strategy](#15-deployment-strategy)
16. [Monitoring & Observability](#16-monitoring--observability)
17. [Trade-offs & Design Decisions](#17-trade-offs--design-decisions)
18. [Future Improvements](#18-future-improvements)
19. [Conclusion](#19-conclusion)

---

# 1. Introduction

## What we are building

MobilityFlow is a two-sided marketplace:

- **Mobility** — passenger ride-hailing
- **Delivery** — package courier
- **Shared platform** — **auth (all user types)**, location, matching, pricing, payment, notification

Ride-hailing is the core MVP. Courier is in the same MVP because both products use the same drivers and the same dispatch pipeline. We are **not** building food delivery, grocery, freight, or the rest of the Uber catalog.

**One sentence:** one auth for every user; share location, matching, and pay; keep Trip and Delivery separate.

```mermaid
flowchart LR
  subgraph Demand
    Passenger["Passenger"]
    Sender["Sender"]
  end

  subgraph Platform["Shared Platform"]
    Auth["Auth Identity"]
    Loc["Location"]
    Match["Matching"]
    Price["Pricing"]
    Pay["Payment"]
    Notify["Notification"]
  end

  subgraph Supply
    Driver["Driver"]
    Courier["Courier"]
    Ops["Operations"]
  end

  Passenger --> Auth
  Sender --> Auth
  Driver --> Auth
  Courier --> Auth
  Ops --> Auth
  Auth --> Match
  Passenger --> Match
  Sender --> Match
  Match --> Driver
  Match --> Courier
  Loc --> Match
  Price --> Match
  Match --> Pay
  Match --> Notify
```

---

## System context

Clients talk only to the edge. Maps, card processing, and push are outside the platform.

```mermaid
flowchart TB
  subgraph Clients
    CApp["Consumer App"]
    DApp["Driver Courier App"]
    Ops["Operations"]
  end

  subgraph Edge["Edge"]
    REST["API Gateway JWT"]
    WS["WebSocket Gateway JWT"]
  end

  subgraph Core["MobilityFlow"]
    Auth["Auth Identity"]
    Mobility["Mobility Domain"]
    DeliveryDom["Delivery Domain"]
    Shared["Shared Platform"]
  end

  subgraph External["External"]
    Maps["Maps"]
    Pay["Payment Provider"]
    Push["Push SMS Email"]
    KYC["Verification Vendor"]
  end

  CApp --> REST
  CApp --> WS
  DApp --> REST
  DApp --> WS
  Ops --> REST
  REST --> Auth
  REST --> Mobility
  REST --> DeliveryDom
  REST --> Shared
  WS --> Auth
  WS --> Shared
  Auth --> KYC
  Mobility --> Shared
  DeliveryDom --> Shared
  Shared --> Maps
  Shared --> Pay
  Shared --> Push
```

| Who | Talks to the platform to |
|-----|--------------------------|
| Consumer App | Register, verify phone, request a ride or delivery, track, pay, rate |
| Driver / Courier App | Register, complete driver verification, go online, stream GPS, accept offers |
| Operations | Register or invite, disputes, safety, approval exceptions |
| Auth / Identity | Signup, OTP, login, JWT, roles, driver KYC status |
| Maps | Distance, ETA, routes |
| Payment Provider | Authorize and capture (we never store raw cards) |
| Push / SMS / Email | OTP, offers, status, receipts |
| Verification Vendor | License, background check, vehicle check for drivers |

---

## Architecture in four layers

Keep this picture in your head. Everything else is a walk through these layers.

```mermaid
flowchart TB
  subgraph L1["1. Edge and Ingestion"]
    GW["API Gateway"]
    WSG["WebSocket Gateway"]
  end

  subgraph L2["2. Shared Platform"]
    Auth["Auth Identity"]
    Loc["Location Service"]
    ME["Matching Engine"]
    PP["Pricing and Payment"]
    Ntf["Notification"]
  end

  subgraph L3["3. Product Domains"]
    Mob["Mobility Domain"]
    Del["Delivery Domain"]
  end

  subgraph L4["4. Data and Messaging"]
    PG["PostgreSQL"]
    RD["Redis"]
    MQ["Message Broker"]
  end

  GW --> Auth
  WSG --> Auth
  Auth --> GW
  GW --> Mob
  GW --> Del
  GW --> PP
  WSG --> Loc
  Loc --> ME
  ME --> Mob
  ME --> Del
  Auth --> PG
  Mob --> PG
  Del --> PG
  Loc --> RD
  ME --> RD
  Auth --> MQ
  Mob --> MQ
  Del --> MQ
  MQ --> Ntf
```

| Layer | Components | What it does |
|-------|------------|----------------|
| Edge and ingestion | API Gateway, WebSocket Gateway | Validate JWT, route REST, keep driver WebSocket sessions |
| Shared platform | **Auth / Identity**, Location, Matching, Pricing and Payment, Notification | Register and verify every user type, issue JWT, driver spatial index, dispatch, pay |
| Product domains | Mobility, Delivery | Product rules: passenger trips vs packages with proof of delivery |
| Data and messaging | PostgreSQL, Redis, RabbitMQ or Kafka | Postgres = system of record. Redis = hot supply and sessions. Broker = async events |

No product API runs until Auth has issued a JWT and the gateway has checked it. REST is for requests the user is waiting on. WebSockets are for GPS and live tracking. The broker is for work that can wait (OTP send, push, receipts, payment capture).

---

## MVP — what ships

```mermaid
flowchart TB
  subgraph SharedMVP["Shared Platform"]
    S0["Auth Identity"]
    S1["Registration all user types"]
    S2["Verification OTP and driver KYC"]
    S3["Login JWT roles"]
    S4["Location index"]
    S5["Matching engine"]
    S6["Pricing and payment"]
    S7["Notification"]
  end

  subgraph MobilityMVP["Mobility"]
    M1["Ride request and matching"]
    M2["Trip lifecycle"]
    M3["Fare payment rating"]
  end

  subgraph DeliveryMVP["Delivery"]
    D1["Package pickup destination"]
    D2["Delivery request and matching"]
    D3["Tracking proof of delivery"]
  end

  S0 --> S1
  S1 --> S2
  S2 --> S3
  SharedMVP --- MobilityMVP
  SharedMVP --- DeliveryMVP
```

Not in MVP: food delivery, grocery, freight, transit, shared rides, scheduled rides, multi-region active-active.

---

## Auth platform — every user type

Registration and verification live on the **Shared Platform**, not inside Mobility or Delivery. One Identity service. Many roles.

```mermaid
flowchart TB
  subgraph Users["All user types"]
    P["Passenger"]
    S["Sender"]
    D["Driver"]
    C["Courier"]
    O["Operations"]
  end

  subgraph AuthPlat["Shared Auth Identity"]
    Reg["Register"]
    Ver["Verify"]
    Login["Login"]
    JWT["Issue JWT"]
    Roles["Assign roles"]
  end

  P --> Reg
  S --> Reg
  D --> Reg
  C --> Reg
  O --> Reg
  Reg --> Ver
  Ver --> Login
  Login --> JWT
  JWT --> Roles
```

| User type | App | Registration | Verification before they can act |
|-----------|-----|--------------|----------------------------------|
| Passenger | Consumer App | Phone, name, payment method token | Phone OTP |
| Sender | Consumer App | Same account can add sender profile | Phone OTP |
| Driver | Driver App | Phone, name, license, vehicle | Phone OTP + license + background + vehicle check |
| Courier | Driver App | Same driver account; courier capability | Same as driver; vehicle must match parcel rules |
| Operations | Admin | Invite by existing admin | Email or SSO + MFA |

Same person can hold more than one role (passenger and sender is normal; driver and courier is normal). Auth stores **one user**, many **roles**. Product domains only see “is this token allowed to request a ride / go online / approve KYC?”

```mermaid
sequenceDiagram
  participant App as Any App
  participant GW as API Gateway
  participant Auth as Auth Identity
  participant SMS as SMS Email
  participant KYC as Verification Vendor
  participant DB as PostgreSQL

  App->>GW: Register phone and role
  GW->>Auth: Create user
  Auth->>DB: Save user pending verify
  Auth->>SMS: Send OTP
  App->>GW: Submit OTP
  GW->>Auth: Confirm OTP
  alt Driver or Courier
    Auth->>KYC: License background vehicle
    KYC-->>Auth: Approved or rejected
    Auth->>DB: Save KYC status
  end
  Auth->>DB: Mark verified
  App->>GW: Login
  GW->>Auth: Check credentials
  Auth-->>App: JWT with roles
```

After login, every call is the same:

```mermaid
flowchart LR
  App["Any App"] --> GW["Gateway"]
  GW --> Auth["Validate JWT"]
  Auth --> Role{"Role on token"}
  Role -->|"Passenger Sender"| Product["Mobility or Delivery APIs"]
  Role -->|"Driver Courier"| Supply["Go online GPS offers"]
  Role -->|"Operations"| Admin["Admin APIs"]
```

| Auth state | Meaning |
|------------|---------|
| Registered | Account exists, OTP not done |
| Verified | Consumer may request rides or deliveries |
| KYC pending | Driver or courier cannot go online yet |
| KYC approved | Driver or courier may go online and receive offers |
| Suspended | Token rejected at the gateway |

Drivers are **not** verified inside the Mobility domain. Mobility and Delivery only check: this JWT is a driver, and Auth says KYC is approved.

---

## How a job actually runs

Auth first. Then three pipelines. Same for rides and deliveries. Only completion differs.

```mermaid
flowchart LR
  P0["0. Register verify login"] --> P1["1. Location stream"]
  P1 --> P2["2. Matching"]
  P2 --> P3["3. Track and complete"]
```

### 1. Location telemetry

Drivers send GPS every 4–5 seconds. That traffic never hits PostgreSQL on the write path.

```mermaid
flowchart LR
  App["Driver App"] --> WS["WebSocket Gateway"]
  WS --> Loc["Location Service"]
  Loc --> Redis["Redis H3 spatial index"]

  Loc -.->|"not on this path"| PG["PostgreSQL"]
```

| Step | What happens |
|------|----------------|
| Stream | Driver App sends GPS over the WebSocket Gateway |
| Validate | Location Service checks coordinates |
| Index | Write to Redis geospatial index (H3) for fast nearby lookup |
| Skip SQL | High-frequency pings stay in memory. Postgres is not on this write path |

Postgres gets location only as a snapshot when a job completes, or in a later batch. That keeps GPS from becoming a database bottleneck.

### 2. Low-latency matching

A passenger or sender requests a job over HTTP. Matching uses Redis, not a SQL scan of every driver.

```mermaid
sequenceDiagram
  participant C as Consumer App
  participant API as API Gateway
  participant Price as Pricing
  participant Match as Matching Engine
  participant Redis as Redis
  participant D as Driver App
  participant Dom as Mobility or Delivery

  C->>API: Submit ride or delivery request
  API->>Price: Quote distance ETA job type
  Price-->>API: Fare or delivery fee
  API->>Match: Find nearby eligible drivers
  Match->>Redis: Radius query by capability
  Redis-->>Match: Candidate drivers
  Match->>Redis: 15 second exclusive offer lock
  Match->>D: Push offer over WebSocket
  D->>Match: First accept wins
  Match->>Dom: Update Trip or Delivery state
  Match->>Match: Emit JobMatched
```

| Step | What happens |
|------|----------------|
| Trigger | Passenger or sender submits a request over REST |
| Price | Deterministic quote from distance, ETA, and job-type coefficients |
| Spatial query | Matching Engine queries Redis for nearby drivers filtered by capability (car, van, courier) |
| Offer | Exclusive offer over WebSocket, 15-second lock in Redis |
| Accept | First accept wins the lock, domain state updates, `JobMatched` is emitted |

Rides and deliveries use this same engine. Eligibility and completion rules stay in the product domain.

```mermaid
flowchart TB
  ME["Matching Engine"] --> RideReq["Ride Request"]
  ME --> DelReq["Delivery Request"]
  RideReq --> Trip["Passenger Trip"]
  DelReq --> Parcel["Package Delivery"]
```

| | Ride | Courier |
|---|------|---------|
| Customer | Passenger | Sender |
| Provider | Driver | Driver / courier |
| Goal | Move a person | Move a package |
| Vehicle filter | Car, bike, ... | Bike, car, van, ... |
| Tracking key | Trip ID | Delivery ID |
| Done when | Passenger drop-off | Proof of delivery |

Do **not** put both products in one `Job` table. Share the index and the offer pipeline. Keep Trip and Delivery as separate records.

### 3. Live tracking and completion

While the job is in progress, GPS is forwarded to the consumer over a WebSocket channel keyed by Trip ID or Delivery ID.

```mermaid
flowchart TB
  DApp["Driver GPS"] --> WS["WebSocket Gateway"]
  WS --> Loc["Location Service"]
  Loc --> Redis["Redis"]
  Loc --> Chan["Live channel by TripID or DeliveryID"]
  Chan --> CApp["Passenger or Recipient App"]

  DApp --> Done{"Completion"}
  Done -->|"Mobility"| Drop["Driver marks drop-off"]
  Done -->|"Delivery"| PoD["PIN photo or signature"]
  Drop --> Event["Completed event"]
  PoD --> Event
  Event --> Workers["Async workers"]
  Workers --> Pay["Capture payment"]
  Workers --> Rcpt["Receipt"]
  Workers --> Pool["Release driver to available pool"]
```

| Step | What happens |
|------|----------------|
| Live broadcast | In-transit GPS goes to the consumer on a channel keyed by Trip ID or Delivery ID |
| Mobility complete | Driver marks drop-off at the destination |
| Delivery complete | Driver uploads proof of delivery (PIN, photo, or signature) |
| After | Background workers capture the pre-authorized payment, send the receipt, and put the driver back in the available pool |

Notifications, receipts, and payment capture are async. Core state change (matched, picked up, completed) stays on the request path, targeting **under 100ms**.

---

## Domain state — two products, two machines

Shared matching. Separate aggregates and completion rules.

```mermaid
flowchart LR
  subgraph Mobility["Mobility aggregate: Trip"]
    T1["Requested"] --> T2["Matched"]
    T2 --> T3["Arriving"]
    T3 --> T4["In Transit"]
    T4 --> T5["Completed"]
  end
```

```mermaid
flowchart LR
  subgraph Delivery["Delivery aggregate: Delivery"]
    D1["Requested"] --> D2["Matched"]
    D2 --> D3["Pickup En Route"]
    D3 --> D4["Picked Up"]
    D4 --> D5["In Transit"]
    D5 --> D6["Completed"]
  end
```

| Domain | Core record | Lifecycle | Done when |
|--------|-------------|-----------|-----------|
| Mobility | Trip | Requested → Matched → Arriving → In Transit → Completed | Passenger drop-off confirmed |
| Delivery | Delivery | Requested → Matched → Pickup En Route → Picked Up → In Transit → Completed | Proof-of-delivery artifact validated |

---

## Three rules that keep this simple

```mermaid
flowchart TB
  R0["One Auth for every user type"]
  R1["Share location and matching"]
  R2["Keep Trip and Delivery separate"]
  R3["GPS in Redis, facts in Postgres"]
  R4["Slow work on the broker"]
```

1. **Share the kernel, isolate the records.** One Auth, one location index, one matching pipeline. No signup inside Mobility or Delivery. No single `Job` table that mixes passenger rules with proof of delivery.
2. **Offload telemetry.** GPS stays in Redis. Write to Postgres on job completion (and optional later snapshots).
3. **Async the side effects.** Push, receipts, and payment capture run on workers so the state transition stays fast.

---

## How it is deployed at launch

One region. A few services. Not thirty microservices.

```mermaid
flowchart TB
  LB["Load Balancer"] --> REST["API Gateway"]
  LB --> WS["WebSocket Gateway"]
  REST --> Auth["Auth Identity"]
  REST --> Mob["Mobility"]
  REST --> Del["Delivery"]
  REST --> Plat["Platform modules"]
  WS --> Auth
  WS --> Plat
  Auth --> PG["PostgreSQL"]
  Mob --> PG
  Del --> PG
  Plat --> PG
  Auth --> Redis["Redis"]
  Plat --> Redis
  Auth --> MQ["Message Broker"]
  Mob --> MQ
  Del --> MQ
  Plat --> MQ
```

| Stage | When | Shape |
|-------|------|--------|
| 1. Launch | First city | One region, one primary Postgres, Redis, broker, several API instances |
| 2. Growth | Traffic rises | More API nodes, read replicas, workers, rate limits, better dashboards |
| 3. Large scale | Many cities | Regional routing, regional data and matching, sharding, failover |

Stage 1 is the default in this document. Later sections say what changes at Stage 2 and 3 — not a rewrite of Trip or Delivery.

---

## Goals

- Ship rides **and** courier in one region, as a real production system
- One **Auth / Identity** platform for passenger, sender, driver, courier, and operations
- Reuse location, matching, pricing, payment, and notification
- Keep Trip and Delivery rules separate
- Match nearby eligible drivers quickly
- Never lose an in-progress trip or delivery on app crash
- Complete rides on drop-off; complete deliveries on proof of delivery
- Pay without storing raw card data
- Scale later (replicas, then regions) without changing the domain meanings

---

## Intended audience

Software engineers, tech leads, architects, and engineering managers on this team. Use this for design reviews and onboarding — not as an interview cheat sheet.

---

## Scope

| Area | Owns |
|------|------|
| Auth / Identity | Register, OTP, login, JWT, roles for passenger, sender, driver, courier, ops; driver KYC |
| Location | Driver GPS index, pickup and drop points |
| Matching | Nearby supply, offers, locks, re-match |
| Pricing | Ride fare and delivery fee |
| Payment | Authorize, capture, refund via the provider |
| Notification | Push, SMS, email from domain events |
| Mobility | Ride request, Trip, fare, rating (no user signup here) |
| Delivery | Package, Delivery, proof of delivery, rating (no user signup here) |

---

## Out of scope

- Mobile UI implementation
- Food delivery, grocery, freight, transit, pool rides, scheduled rides
- Building our own maps or card processor
- Multi-region active-active (Stage 3)
- Cloud vendor runbooks and CI/CD internals

---

## How we record decisions

| Step | Meaning |
|------|---------|
| Problem | What hurts |
| MVP solution | What we ship now |
| Why | Why that is enough now |
| Scaling problem | What breaks later |
| Evolution | Next architecture, same domain |
| Trade-off | What we gain and what gets harder |

Example:

- **Problem:** GPS every few seconds would crush Postgres.
- **MVP:** Redis H3 for live location.
- **Why:** Launch traffic does not need a GPS history database on the hot path.
- **Later:** Snapshots and regional indexes.
- **Trade-off:** Fast matching; weaker historical GPS until we add snapshots.

---

## Document conventions

- **MobilityFlow** = the product. **Platform** = shared capabilities. **Mobility** / **Delivery** = product domains (not automatically separate deployables).
- Stage 1 is the default. Stages 2 and 3 appear when a section talks about growth.
- Diagrams are the source of truth for flows. Tables hold the rules.
- Section 5 will deepen Trip vs Delivery. Section 11 will deepen scale.

---

<!-- Sections 2–19 will be added after review of Section 1. -->
