---
title: "Mobility & Delivery Platform — Enterprise Architecture (ADD)"
description: "Production-ready architecture for rides and package delivery on a unified platform. Shared Auth, Location, Matching, Pricing, Payment, and Notifications with isolated Trip and Delivery domains."
keywords: [mobility platform, delivery platform, ride-hailing architecture, courier architecture, microservices, DDD, Redis H3, WebSocket, RabbitMQ, MobilityFlow]
robots: index, follow
---

# Mobility & Delivery Platform — Enterprise Architecture

**Architecture Design Document (ADD)**

A production-ready, scalable architecture for **rides** and **package delivery** on a unified platform. The system leverages shared capabilities (Auth, Location, Matching, Pricing, Payment, Notifications) while strictly isolating the specific business rules for trips and deliveries.

**Reference Product:** MobilityFlow

---

## Table of Contents

1. [Introduction & Vision](#1-introduction--vision)
2. [Core Design Principles](#2-core-design-principles)
3. [System Context & Scope](#3-system-context--scope)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Unified Identity & Access Management](#5-unified-identity--access-management)
6. [The Core Job Lifecycle](#6-the-core-job-lifecycle)
7. [Deployment & Scaling Strategy](#7-deployment--scaling-strategy)
8. [Design Decision Records (ADR)](#8-design-decision-records-adr)
9. [Document Conventions & Roadmap](#9-document-conventions--roadmap)

---

## 1. Introduction & Vision

### What we are building

MobilityFlow is a two-sided marketplace connecting users with drivers:

* **Mobility:** Passenger ride-hailing.
* **Delivery:** Package courier services.
* **Shared Platform:** Centralized services for Auth, Location, Matching, Pricing, Payment, and Notifications.

**One-Sentence Summary:** One unified Auth system for all user types; shared location, matching, and payment infrastructure; strictly isolated domain rules for Trips and Deliveries.

```mermaid
flowchart LR
  subgraph Demand
    Passenger["Passenger"]
    Sender["Sender"]
  end

  subgraph Platform["Shared Platform"]
    Auth["Auth / Identity"]
    Loc["Location"]
    Match["Matching"]
    Price["Pricing"]
    Pay["Payment"]
    Notify["Notifications"]
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

### Goals

* **Ship a Dual-Sided MVP:** Launch both rides and courier in a single region as a real production system.
* **Cost-Effective Scale:** Support **5K to 50K users** at launch with a lean footprint (one backend, one Postgres, one Redis, RabbitMQ).
* **Unified Identity:** One Auth/Identity platform managing passengers, senders, drivers, couriers, and operations.
* **High Availability:** Never lose an in-progress trip or delivery on app crash.
* **Future-Proofing:** Scale to millions of users (replicas, then regions) without changing core domain meanings.

---

## 2. Core Design Principles

These three rules govern all architectural decisions to keep the system simple and performant:

```mermaid
flowchart TB
  R1["Share the Kernel, Isolate the Domains"]
  R2["Offload High-Frequency Telemetry"]
  R3["Asynchronous Side Effects"]
  R1 --> R2 --> R3
```

1. **Share the Kernel, Isolate the Domains:**
   One Auth service, one location index, one matching pipeline. User signup never happens inside the Mobility or Delivery domains. We avoid a single monolithic `Job` table that mixes passenger rules with proof-of-delivery logic.
2. **Offload High-Frequency Telemetry:**
   Driver GPS pings (every 4–5 seconds) are written to an in-memory spatial index (Redis/H3), not directly to the relational database. Postgres only receives location snapshots upon job completion.
3. **Asynchronous Side Effects:**
   Non-critical actions (push notifications, receipt generation, payment capture) are offloaded to background workers. This ensures the main transaction state transitions remain fast and low-latency.

---

## 3. System Context & Scope

### System Context

Clients communicate exclusively with the edge. Maps, card processing, and push notifications are external dependencies.

```mermaid
flowchart TB
  subgraph Clients
    CApp["Consumer App"]
    DApp["Driver / Courier App"]
    Ops["Operations"]
  end

  subgraph Edge["Edge & Ingestion"]
    REST["API Gateway — JWT"]
    WS["WebSocket Gateway — JWT"]
  end

  subgraph Core["MobilityFlow"]
    Auth["Auth / Identity"]
    Mobility["Mobility Domain"]
    DeliveryDom["Delivery Domain"]
    Shared["Shared Platform"]
  end

  subgraph External["External Dependencies"]
    Maps["Maps Provider"]
    Pay["Payment Provider"]
    Push["Messaging Provider"]
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

| Actor | Interaction with Platform |
| :--- | :--- |
| **Consumer App** | Register, verify phone, request job, track live, pay, rate. |
| **Driver / Courier App** | Register, complete KYC, go online, stream GPS, accept offers. |
| **Operations (Admin)** | Manage disputes, safety issues, manual KYC approvals. |
| **Maps Provider** | Distance calculation, ETA, routing. |
| **Payment Provider** | Authorize and capture funds (we never store raw card data). |
| **Messaging Provider** | OTP, push notifications, SMS, emails. |
| **Verification Vendor** | Background checks, license verification for drivers. |

### Scope & Out of Scope

| Area | Owned by Platform | Out of Scope |
| :--- | :--- | :--- |
| **Auth / Identity** | Register, OTP, login, JWT, role management, driver KYC. | Mobile UI implementation. |
| **Location** | Driver GPS spatial index, pickup/drop geocoding. | Building proprietary maps. |
| **Matching** | Nearby supply query, offer locks, re-match logic. | Scheduled rides, pool rides. |
| **Pricing & Payment** | Fare calculation, authorize/capture/refund via provider. | Building a custom payment gateway. |
| **Mobility Domain** | Trip lifecycle, ride-specific rules, ratings. | Food delivery, grocery, freight, transit. |
| **Delivery Domain** | Package lifecycle, proof of delivery (PIN/Photo), ratings. | Multi-region active-active (Reserved for Stage 3). |

---

## 4. High-Level Architecture

The system is divided into four distinct logical layers:

```mermaid
flowchart TB
  subgraph L1["1. Edge & Ingestion"]
    GW["API Gateway"]
    WSG["WebSocket Gateway"]
  end

  subgraph L2["2. Shared Platform"]
    Auth["Auth / Identity"]
    Loc["Location Service"]
    ME["Matching Engine"]
    Price["Pricing Service"]
    PaySvc["Payment Service"]
    Ntf["Notifications"]
  end

  subgraph L3["3. Product Domains"]
    Mob["Mobility Domain"]
    Del["Delivery Domain"]
  end

  subgraph L4["4. Data & Messaging"]
    PG["PostgreSQL — System of Record"]
    RD["Redis — Hot Spatial Data"]
    MQ["RabbitMQ — Async Domain Events"]
  end

  GW --> Auth
  WSG --> Auth
  Auth --> GW
  GW --> Mob
  GW --> Del
  GW --> Price
  GW --> PaySvc
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

| Layer | Components | Responsibility |
| :--- | :--- | :--- |
| **1. Edge & Ingestion** | API Gateway, WebSocket Gateway | JWT validation, REST routing, persistent WebSocket sessions for live tracking. |
| **2. Shared Platform** | Auth, Location, Matching, Pricing, Payment, Notifications | Cross-cutting enterprise capabilities used by both product domains. |
| **3. Product Domains** | Mobility, Delivery | Specific business logic (e.g., passenger trip states vs. package proof-of-delivery). |
| **4. Data & Messaging** | PostgreSQL, Redis, RabbitMQ | Postgres (System of Record), Redis (Hot spatial data), RabbitMQ (Async domain events). |

No product API runs until Auth has issued a JWT and the gateway has validated it. REST handles user-facing requests; WebSockets carry GPS and live tracking; RabbitMQ handles work that can wait (OTP, push, receipts, payment capture).

---

## 5. Unified Identity & Access Management

Registration and verification live on the **Shared Platform**, not inside the product domains. The Auth service manages **one user** with **multiple roles**.

```mermaid
flowchart TB
  subgraph Users["All User Types"]
    P["Passenger"]
    S["Sender"]
    D["Driver"]
    C["Courier"]
    O["Operations"]
  end

  subgraph AuthPlat["Shared Auth / Identity"]
    Reg["Register"]
    Ver["Verify"]
    Login["Login"]
    JWT["Issue JWT"]
    Roles["Assign Roles"]
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

| User Type | App | Registration Requirements | Verification to Act |
| :--- | :--- | :--- | :--- |
| **Passenger** | Consumer | Phone, Name, Payment Token | Phone OTP |
| **Sender** | Consumer | Same account as Passenger | Phone OTP |
| **Driver** | Driver | Phone, Name, License, Vehicle | Phone OTP + License + Background + Vehicle Check |
| **Courier** | Driver | Same account as Driver | Same as Driver; vehicle must match parcel rules |
| **Operations** | Admin | Invite by existing admin | Email/SSO + MFA |

Same person can hold multiple roles (passenger + sender, or driver + courier). Auth stores **one user**, many **roles**.

*Note: Product domains only query Auth to verify: "Is this JWT allowed to request a ride / go online / approve KYC?"*

```mermaid
sequenceDiagram
  participant App as Any App
  participant GW as API Gateway
  participant Auth as Auth / Identity
  participant SMS as Messaging Provider
  participant KYC as Verification Vendor
  participant DB as PostgreSQL

  App->>GW: Register phone and role
  GW->>Auth: Create user
  Auth->>DB: Save user (pending verify)
  Auth->>SMS: Send OTP
  App->>GW: Submit OTP
  GW->>Auth: Confirm OTP
  alt Driver or Courier
    Auth->>KYC: License, background, vehicle check
    KYC-->>Auth: Approved or rejected
    Auth->>DB: Save KYC status
  end
  Auth->>DB: Mark verified
  App->>GW: Login
  GW->>Auth: Check credentials
  Auth-->>App: JWT with roles
```

After login, every API call follows the same path:

```mermaid
flowchart LR
  App["Any App"] --> GW["Gateway"]
  GW --> Auth["Validate JWT"]
  Auth --> Role{"Role on Token"}
  Role -->|"Passenger / Sender"| Product["Mobility or Delivery APIs"]
  Role -->|"Driver / Courier"| Supply["Go online, GPS, offers"]
  Role -->|"Operations"| Admin["Admin APIs"]
```

| Auth State | Meaning |
| :--- | :--- |
| **Registered** | Account exists; OTP not completed |
| **Verified** | Consumer may request rides or deliveries |
| **KYC Pending** | Driver or courier cannot go online |
| **KYC Approved** | Driver or courier may go online and receive offers |
| **Suspended** | Token rejected at the gateway |

---

## 6. The Core Job Lifecycle

*Note: We use the generic term **"Job"** to refer to either a Ride or a Delivery. The pipeline is identical until the completion phase.*

```mermaid
flowchart LR
  P0["0. Register / Verify / Login"] --> P1["1. Location Telemetry"]
  P1 --> P2["2. Low-Latency Matching"]
  P2 --> P3["3. Live Tracking & Completion"]
```

### Phase 1: Location Telemetry (Write Path)

Drivers stream GPS data every 4–5 seconds. This traffic bypasses PostgreSQL to prevent write bottlenecks.

```mermaid
flowchart LR
  App["Driver App"] --> WS["WebSocket Gateway"]
  WS --> Loc["Location Service"]
  Loc --> Redis["Redis H3 Spatial Index"]
  Loc -.->|"not on this path"| PG["PostgreSQL"]
```

1. **Stream:** Driver App sends GPS coordinates over the WebSocket Gateway.
2. **Validate:** Location Service verifies coordinate bounds.
3. **Index:** Data is written to a Redis Geospatial Index (using H3) for fast nearby lookups.
4. **Skip SQL:** High-frequency pings stay in memory. Postgres only receives a snapshot when the job completes.

### Phase 2: Low-Latency Matching

```mermaid
sequenceDiagram
  participant C as Consumer App
  participant API as API Gateway
  participant Price as Pricing Service
  participant Match as Matching Engine
  participant Redis as Redis
  participant D as Driver App
  participant Dom as Mobility or Delivery

  C->>API: Submit ride or delivery request
  API->>Price: Quote (distance, ETA, job type)
  Price-->>API: Fare or delivery fee
  API->>Match: Find nearby eligible drivers
  Match->>Redis: Radius query by capability
  Redis-->>Match: Candidate drivers
  Match->>Redis: 15-second exclusive offer lock
  Match->>D: Push offer over WebSocket
  D->>Match: First accept wins
  Match->>Dom: Update Trip or Delivery state
  Match->>Match: Emit JobMatched event
```

1. **Trigger:** User submits a request via REST API.
2. **Price:** Pricing Service calculates a deterministic quote (Distance + ETA + Job-Type Coefficients).
3. **Spatial Query:** Matching Engine queries Redis for nearby drivers, filtered by capability (car, van, courier).
4. **Offer & Lock:** An exclusive offer is sent via WebSocket. A **15-second lock** is placed in Redis.
5. **Accept:** The first driver to accept wins the lock. Domain state updates, and a `JobMatched` event is emitted to RabbitMQ.

Rides and deliveries share the matching engine. Eligibility and completion rules stay in the product domain.

```mermaid
flowchart TB
  ME["Matching Engine"] --> RideReq["Ride Request"]
  ME --> DelReq["Delivery Request"]
  RideReq --> Trip["Passenger Trip"]
  DelReq --> Parcel["Package Delivery"]
```

| | Ride | Courier |
| :--- | :--- | :--- |
| **Customer** | Passenger | Sender |
| **Provider** | Driver | Driver / Courier |
| **Goal** | Move a person | Move a package |
| **Vehicle filter** | Car, bike, … | Bike, car, van, … |
| **Tracking key** | Trip ID | Delivery ID |
| **Done when** | Passenger drop-off | Proof of delivery |

### Phase 3: Live Tracking & Completion

```mermaid
flowchart TB
  DApp["Driver GPS"] --> WS["WebSocket Gateway"]
  WS --> Loc["Location Service"]
  Loc --> Redis["Redis"]
  Loc --> Chan["Live channel by Job ID"]
  Chan --> CApp["Consumer App"]

  DApp --> Done{"Completion"}
  Done -->|"Mobility"| Drop["Driver marks drop-off"]
  Done -->|"Delivery"| PoD["PIN, photo, or signature"]
  Drop --> Event["Completed event"]
  PoD --> Event
  Event --> Workers["Async workers via RabbitMQ"]
  Workers --> Pay["Capture payment"]
  Workers --> Rcpt["Send receipt"]
  Workers --> Pool["Return driver to available pool"]
```

1. **Broadcast:** In-transit GPS is forwarded to the Consumer App on a WebSocket channel keyed by Job ID.
2. **Completion:**
   * *Mobility:* Driver marks drop-off at the destination.
   * *Delivery:* Driver uploads Proof of Delivery (PIN, photo, or signature).
3. **Post-Job Async:** Background workers capture the pre-authorized payment, send the receipt, and return the driver to the available pool.

Core state transitions (matched, picked up, completed) stay on the request path, targeting **under 100ms**. Notifications, receipts, and payment capture are async.

### Domain State — Two Products, Two State Machines

Shared matching pipeline. Separate aggregates and completion rules.

```mermaid
flowchart LR
  subgraph Mobility["Mobility Aggregate: Trip"]
    T1["Requested"] --> T2["Matched"]
    T2 --> T3["Arriving"]
    T3 --> T4["In Transit"]
    T4 --> T5["Completed"]
  end
```

```mermaid
flowchart LR
  subgraph Delivery["Delivery Aggregate: Delivery"]
    D1["Requested"] --> D2["Matched"]
    D2 --> D3["Pickup En Route"]
    D3 --> D4["Picked Up"]
    D4 --> D5["In Transit"]
    D5 --> D6["Completed"]
  end
```

| Domain | Core Record | Lifecycle | Done When |
| :--- | :--- | :--- | :--- |
| **Mobility** | Trip | Requested → Matched → Arriving → In Transit → Completed | Passenger drop-off confirmed |
| **Delivery** | Delivery | Requested → Matched → Pickup En Route → Picked Up → In Transit → Completed | Proof-of-delivery artifact validated |

---

## 7. Deployment & Scaling Strategy

**Stage 1 Target:** 5K to 50K users. Keep the cloud bill small and operations simple. One region, one backend, one database.

```mermaid
flowchart TB
  subgraph Stage1["Stage 1 — Launch (5K–50K)"]
    LB["Load Balancer"] --> App["2 App Instances"]
    App --> PG["1 PostgreSQL"]
    App --> Redis["1 Redis"]
    App --> MQ["1 RabbitMQ"]
  end
```

Inside the backend, modules remain separate in code (Auth, Mobility, Delivery, Location, Matching, Pricing, Payment, Notifications) but deploy as **one unit** at launch — not separate clusters.

```mermaid
flowchart TB
  LB["Load Balancer"] --> App["API + WebSocket"]
  subgraph AppBox["Single Deployable"]
    Auth["Auth / Identity"]
    Mob["Mobility"]
    Del["Delivery"]
    Plat["Location · Matching · Pricing · Payment · Notifications"]
  end
  App --> Auth
  App --> Mob
  App --> Del
  App --> Plat
  Auth --> PG["PostgreSQL"]
  Mob --> PG
  Del --> PG
  Plat --> PG
  Plat --> Redis["Redis"]
  Auth --> Redis
  Plat --> MQ["RabbitMQ"]
```

| Stage | User Scale | Infrastructure Shape |
| :--- | :--- | :--- |
| **Stage 1: Launch** | **5K – 50K** | 2 App Instances, 1 Postgres, 1 Redis, 1 RabbitMQ. Pay-as-you-go external APIs. |
| **Stage 2: Growth** | **50K – 500K** | Split hot services, Redis Cluster, Postgres Read Replicas, multiple API gateways. |
| **Stage 3: Scale** | **Millions** | Regional routing, regional data/matching silos, database sharding. |

```mermaid
flowchart LR
  S1["Stage 1: 5K–50K"] --> S2["Stage 2: 50K–500K"]
  S2 --> S3["Stage 3: Millions"]
```

| Metric | Stage 1 | Stage 2 | Stage 3 |
| :--- | :--- | :--- | :--- |
| **Registered users** | 5K start, design for 50K | 50K – ~500K | Millions |
| **Concurrent apps open** | ~500 – 5K | Tens of thousands | Millions, by region |
| **Online drivers (GPS)** | ~200 – 2K | Tens of thousands | 100K+ per region |
| **Peak new jobs / min** | Tens to a few hundred | Thousands | Tens of thousands per region |

**Do not buy at Stage 1:** Kafka, Redis Cluster, Kubernetes multi-AZ microservices, read replicas, multi-region routing. Split only when 50K concurrent load actually hurts.

| Bottleneck (post-50K) | Why It Breaks |
| :--- | :--- |
| WebSocket instances | More drivers pinging every 4–5 seconds |
| Redis memory / CPU | Spatial index and offer locks |
| Postgres connections | Auth, trips, payments |
| Single region | Second city in another timezone |

---

## 8. Design Decision Records (ADR)

When documenting specific architectural choices in future sections, we use the following format:

* **Problem:** What is the bottleneck or pain point?
* **MVP Solution:** What are we shipping right now?
* **Why:** Why is this sufficient for the current stage?
* **Scaling Problem:** What will break when we grow?
* **Evolution:** What is the next architectural iteration?
* **Trade-off:** What do we gain, and what becomes harder?

*Example:*

> **Problem:** Writing GPS every 4 seconds will crush Postgres.
> **MVP Solution:** Use Redis H3 for live location.
> **Why:** Launch traffic doesn't require historical GPS on the hot path.
> **Scaling Problem:** Redis memory pressure at 100K+ concurrent drivers.
> **Evolution:** Batch GPS snapshots to Postgres; regional Redis silos.
> **Trade-off:** Extremely fast matching, but weaker historical GPS tracking until we implement batch snapshots.

---

## 9. Document Conventions & Roadmap

* **Terminology:** **MobilityFlow** = the product. **Platform** = shared capabilities. **Mobility / Delivery** = specific product domains.
* **Staging:** Stage 1 is the default baseline. Stages 2 and 3 are discussed only when addressing scale.
* **Diagrams vs Tables:** Diagrams are the source of truth for flows. Tables hold the business rules.

### Upcoming Sections (Roadmap)

*The following sections will be expanded in subsequent updates:*

| # | Section |
| :--- | :--- |
| 10 | Database Schema Design (ERD) |
| 11 | API Contracts (REST & WebSockets) |
| 12 | Scalability & Performance Deep-Dive |
| 13 | Security & Fault Tolerance |
| 14 | Monitoring, Observability & CI/CD |

---

<!-- Sections 10–14 follow the ADD template and will be added after review of Sections 1–9. -->
