# WhatsApp Architecture Design Document

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

## Overview

This document describes the architecture of a large-scale, real-time messaging platform modeled after WhatsApp. The system supports billions of users exchanging text messages, media, voice notes, and calls with low latency, high reliability, and end-to-end encryption (E2EE) as a core guarantee.

Users send one-to-one and group messages, share images, videos, documents, and audio, view presence and delivery status, make voice and video calls, and manage contacts and chat history across mobile and desktop clients. The distributed messaging backbone optimizes persistent connections, efficient fan-out, offline delivery, and privacy-preserving storage.

The architecture prioritizes connection density, message delivery guarantees, horizontal scalability, and security by design. It handles peak events—holidays, major news moments, and reconnect storms—as first-class capacity and resilience concerns.

Although this document uses a consumer messaging product as the reference domain, the same architectural principles apply to many real-time communication systems where persistent sessions, ordered delivery, multi-device sync, and encrypted payloads must coexist at internet scale.

### System Context

The platform sits between client applications, identity and contact discovery, media storage, push notification providers, and business APIs. Clients maintain long-lived sessions with connection and messaging layers. Supporting systems include CDNs, object storage, and push services.

```mermaid
flowchart LR
  subgraph Clients
    Mobile[Mobile Clients]
    Desktop[Desktop Clients]
    Web[Web Client]
  end

  subgraph Platform["Messaging Platform"]
    Edge[Edge / Connection Layer]
    Msg[Messaging Core]
    Media[Media Services]
    Presence[Presence & Status]
  end

  subgraph External["External / Supporting Systems"]
    Push[Push Notification Providers]
    CDN[CDN / Object Storage]
    IdP[Identity / Phone Verification]
    Biz[Business Messaging APIs]
  end

  Mobile --> Edge
  Desktop --> Edge
  Web --> Edge
  Edge --> Msg
  Edge --> Presence
  Msg --> Media
  Msg --> Push
  Media --> CDN
  Edge --> IdP
  Biz --> Msg
```

| Actor / System | Type | Relationship to the Platform |
|----------------|------|------------------------------|
| Mobile Client (iOS / Android) | Primary actor | Maintains persistent sessions; sends/receives messages, media, and call signaling |
| Desktop / Web Client | Primary actor | Multi-device sync of chats; linked-device authentication flows |
| Connection / Edge Layer | Core platform | Terminates TLS, multiplexes sessions, routes frames to messaging services |
| Messaging Core | Core platform | Stores, routes, fans out, and acknowledges message delivery |
| Media Services | Core platform | Handles upload, processing, and retrieval of encrypted media blobs |
| Push Notification Providers | External system | Wakes offline clients (APNs, FCM, and regional equivalents) |
| CDN / Object Storage | Supporting system | Serves media and static assets with high availability and low latency |
| Identity / Phone Verification | Supporting system | Registers users via phone number and issues session credentials |
| Business Messaging APIs | External / partner channel | Enables verified businesses to send templated and conversational messages |

---

## Goals

The primary goals of the architecture are to:

- Support billions of users and hundreds of millions of concurrent sessions
- Deliver one-to-one and group messages with low latency and reliable offline delivery
- Provide clear acknowledgment semantics (sent, delivered, read)
- Scale group fan-out and multi-device sync efficiently
- Enforce E2EE while enabling multi-device support
- Maintain high availability across regions with resilience to failures and reconnect storms
- Keep per-connection and per-message cost low enough for global consumer scale
- Isolate failures so that media or calling degradation does not block text messaging
- Enable independent evolution of messaging, media, presence, and calling capabilities

---

## Architectural Approach

The design begins with **Domain-Driven Design (DDD)** to identify bounded contexts from the business domain. Microservices are derived from those contexts—not the other way around—so that service boundaries reflect business capabilities, ubiquitous language, and invariants. This avoids premature decomposition into technical services that later fight the domain.

Once contexts are clear, each resulting service follows **Clean Architecture** (Presentation → Application → Domain → Infrastructure) so domain rules stay independent of frameworks, databases, and transports. Long-lived sessions (for example WebSocket or custom binary / XMPP-like protocols) still terminate at a horizontally scaled edge layer; connection density remains a first-class runtime concern, but it is an implementation choice sitting on top of domain ownership, not a substitute for it.

| Principle | Role in this Platform |
|-----------|------------------------|
| Domain-Driven Design (DDD) | Identifies bounded contexts first; services are mapped from those contexts |
| Clean Architecture | Applied inside each service so business logic does not depend on infrastructure |
| Persistent Connection Model | Uses long-lived sessions for low-latency push to online clients |
| Microservices (derived from domains) | Enables independent scaling and evolution only after domain boundaries are sound |
| Event-Driven Integration | Contexts communicate via domain events (and anti-corruption layers where needed) |
| CQRS (where justified) | Separates write-optimized message ingest from read/sync views for multi-device catch-up |
| Sharding by User / Chat | Partitions state so hot users and large groups can scale independently |
| Security by Design | Treats E2EE, key management, and minimal server-side plaintext as product requirements |
| Cloud-Native / Multi-Region Deployment | Uses containers or equivalent fleets, regional capacity, and elastic connection pools |
| Observability by Design | Treats connection health, queue lag, delivery latency, and drop rates as runtime contracts |

Synchronous request/response remains appropriate for registration, profile updates, and media upload handshakes. Asynchronous fan-out and store-and-forward are preferred for delivery to offline or multi-device recipients where temporary inconsistency of presence or read receipts is acceptable.

---

## Intended Audience

This document is intended for:

- Software Engineers
- Senior Developers
- Technical Leads
- Solution Architects
- Engineering Managers
- Students learning large-scale distributed systems and real-time architecture

Readers should treat this as a design reference for planning, reviews, and onboarding—not as a product tutorial or reverse-engineering of any specific vendor implementation.

---

## Scope

Scope follows the business domains identified in Section 5. Technical capabilities such as connection management and multi-device sync sit inside or beside these domains as needed, without inventing service boundaries ahead of domain analysis.

| Domain (Bounded Context) | Responsibility Summary |
|--------------------------|------------------------|
| Identity | Phone-based signup, authentication, sessions, device linking |
| Contacts / User Profile | Profiles, privacy settings, contact sync, blocking, discovery |
| Messaging (core) | Conversations, send/receive/edit/delete, receipts, offline delivery |
| Group | Membership, admin controls, permissions, fan-out coordination |
| Presence | Online/offline, last seen, typing and related ephemeral indicators |
| Media | Encrypted upload, thumbnails/compression, blob references |
| Notification | Push (APNs/FCM), retries, dead-letter handling |
| Search | Message, group, and contact search indexes and queries |
| Administration | Moderation, reporting, abuse controls |
| Analytics (optional) | Insights and abuse-detection signals; not on the critical messaging path |
| Calling (supporting) | Voice/video signaling and relay coordination (detail limited by out-of-scope rules) |

---

## Out of Scope

- Client UI/UX implementation details
- Full WebRTC media-plane engineering (codecs, congestion control, TURN farm ops)
- Vendor-specific infrastructure scripts and cloud control-plane runbooks
- Content moderation ML model training and policy operations
- WhatsApp Business Platform commercial packaging and pricing

---

# 2. Business Requirements

| Requirement | Description | Business Driver |
|-------------|-------------|-----------------|
| Global real-time communication | Seamless messaging for consumer and business users worldwide | Core product value; network effects |
| Privacy by default | E2EE and minimal server-side retention of message content | Trust, regulatory expectations, competitive differentiation |
| Business monetization | Business APIs and templates without degrading consumer UX | Sustainable revenue adjacent to free consumer messaging |
| Retention through reliability | Low data usage, cross-device support, consistent delivery | Reduce churn in markets with constrained networks |
| Cost-effective hypergrowth | Support explosive volume (for example 100B+ messages/day) | Unit economics at internet scale |

### Business Constraints

- Consumer messaging remains free and must not be held back by business features
- Server-side systems must not require access to plaintext message content
- Growth must be absorbable through horizontal scale rather than vertical “hero” servers alone
- Regional availability and reconnect behavior are product-visible, not only infrastructure metrics

---

# 3. Functional Requirements

| Capability | Requirement |
|------------|-------------|
| Conversations | One-to-one and group conversations |
| Acknowledgments | Sent, delivered, and read receipts |
| Media sharing | Images, videos, audio, and documents with upload/download |
| Offline delivery | Persistent storage and delivery when recipients reconnect |
| Push | Notifications for offline users via APNs/FCM (or equivalents) |
| Presence | Online/last-seen and typing indicators |
| Calling | Voice/video call signaling |
| Multi-device | Sync across linked devices and contact management |
| Groups | Admin controls (add/remove, roles, settings) |
| Business messaging | Templated and conversational business messages |

### Primary User Flows

1. **Online send** — Sender writes ciphertext → edge → message service stores and routes → recipient edge pushes → acks flow back (sent → delivered → read).
2. **Offline send** — Message is durably queued; push wakes the device; catch-up on reconnect delivers pending ciphertext.
3. **Group send** — Membership resolved → fan-out to members (online push / offline store) using group encryption primitives (for example Sender Keys).
4. **Media share** — Client uploads encrypted blob → media service returns media ID → message references media ID → recipients download ciphertext.

---

# 4. Non-Functional Requirements

| Category | Target |
|----------|--------|
| Latency | Sub-second one-to-one delivery under normal load |
| Consistency | Ordered delivery for messages; eventual consistency acceptable for presence/read receipts |
| Availability | High (for example 99.99%), with tolerance for temporary presence inconsistency |
| Security | Mandatory E2EE; servers operate on ciphertext for user content |
| Scalability | 2B+ users, 100B+ messages/day, millions of concurrent connections |
| Reliability | Durable storage, offline queuing, fault isolation across domains |
| Performance | Low bandwidth via efficient protocols/compression; usable on poor networks |

### Capacity Estimates (Illustrative)

| Metric | Example Order of Magnitude |
|--------|----------------------------|
| Message storage growth | ~10 TB/day (policy-dependent retention) |
| Aggregate messaging bandwidth | ~926 Mb/s (highly dependent on media mix) |
| Chat / connection servers | Hundreds to thousands, depending on connection density |
| Connections per server | Millions feasible with efficient runtimes (for example Erlang-style soft real-time) |

These figures are planning anchors, not SLOs. Actual sizing depends on retention policy, media ratio, average message size, and regional traffic mix.

---

# 5. Domain Analysis (DDD)

Domain analysis is the foundation of this architecture. Bounded contexts are identified from the business domain first; service decomposition in Sections 6–7 is derived from that model. Starting with microservices without this step tends to produce technical slices that couple unrelated business rules and slow independent evolution.

## Step 1: Business Domains / Bounded Contexts

Each context owns its ubiquitous language, models, and invariants:

```text
WhatsApp
│
├── Identity
├── Contacts (or User Profile)
├── Messaging (core)
├── Group
├── Presence
├── Media
├── Notification
├── Search
├── Administration
└── Analytics (optional, for insights and abuse detection)
```

| Bounded Context | Responsibility | Notes |
|-----------------|----------------|-------|
| Identity | Registration, phone verification, login, sessions, device linking | Auth credentials and session lifecycle |
| Contacts / User Profile | Profiles, avatars, privacy settings, contact sync, blocking, favorites, discovery | May consolidate related user-facing profile concerns |
| Messaging (core) | Conversations, send/receive/edit/delete/reply/forward, receipts, offline delivery | Highest write and fan-out load |
| Group | Create groups, membership, admin controls, permissions | Membership invariants live here |
| Presence | Online/offline, last seen, typing, voice-recording indicators | Separated due to high-frequency ephemeral updates |
| Media | Upload, thumbnails, compression; metadata vs blob ownership | Blobs in object storage; metadata in context store |
| Notification | Push (APNs/FCM), email/SMS where used, retries, DLQs | Reacts to domain events from Messaging/Presence |
| Search | Message, group, and contact search | Eventually consistent indexes from domain events |
| Administration | Moderation, reporting, abuse controls | Cross-cutting policies without owning message plaintext |
| Analytics (optional) | Insights and abuse-detection signals | Off the critical delivery path |

Connection management (heartbeats, reconnect, session affinity) is a platform/edge concern that supports these domains; it is not a substitute business domain. Multi-device sync is modeled primarily inside Messaging (catch-up queues) with Identity owning device linking.

## Messaging Context: Aggregates, Entities, Value Objects, and Events

| Kind | Examples |
|------|----------|
| Aggregate root | `Conversation` |
| Entities | `Message`, `Attachment`, `Reaction` |
| Value objects | `MessageContent` (ciphertext envelope), `MessageStatus`, `MessageType` |
| Domain events | `MessageSent`, `MessageDelivered`, `MessageRead`, `MessageDeleted`, `MessageEdited` |

Similar modeling applies elsewhere—for example a `Group` aggregate with membership invariants, or Presence with ephemeral status updates that intentionally avoid strong consistency with Messaging.

Contexts integrate through **domain events** or **anti-corruption layers** when models differ (for example Notification consuming `MessageSent` without importing Messaging’s full aggregate graph).

```mermaid
flowchart LR
  Identity --- Contacts
  Contacts --- Messaging
  Messaging --- Group
  Messaging --- Media
  Messaging --- Presence
  Messaging --- Notification
  Messaging --- Search
  Group --- Notification
  Administration --- Messaging
  Analytics --- Messaging
```

This DDD foundation keeps technical decomposition aligned with business capabilities before any microservice map is drawn.

---

# 6. High-Level Architecture

## Step 2: Mapping Bounded Contexts to Microservices

Bounded contexts map to independently deployable services. Some related contexts may be consolidated for simplicity and performance (for example closely related user/profile concerns), but consolidation is an explicit trade-off—not the default starting point. An **API Gateway** (plus connection edge for persistent sessions) fronts the system for routing, authentication, and rate limiting.

```text
                         API Gateway / Connection Edge
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
 Identity Service              User Service                 Contact Service
        │                             │                             │
        └─────────────────────────────┼─────────────────────────────┘
                                      │
                         Messaging Service (Core)
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                                   │
              Group Service                      Presence Service
                    │                                   │
              Media Service                   Notification Service
                    │                                   │
              Search Service              File Storage (supports Media)
                    │
            Administration Service
```

Clients still use persistent sessions (WebSocket or custom protocol) to the connection edge. Messages route through the Messaging service; fan-out and side effects travel on events (Kafka/RabbitMQ). Offline messages are stored and delivered on reconnect. Media uses object storage plus CDN. Presence and session maps use caching (Redis). E2EE uses the Signal Protocol family.

### Service Responsibilities (Aligned to Domains)

| Service | Derived From | Responsibility |
|---------|--------------|----------------|
| Identity Service | Identity | Authentication, registration, login, JWT/refresh tokens, MFA, sessions, device linking |
| User Service | Contacts / User Profile | Profiles, avatars, privacy settings, status text |
| Contact Service | Contacts | Contact sync, blocking, favorites, discovery |
| Messaging Service | Messaging | Send/receive/edit/delete/reply/forward, read receipts, delivery status, offline queues |
| Group Service | Group | Create/add/remove members, admin controls, permissions |
| Presence Service | Presence | Online/offline, last seen, typing, recording indicators |
| Media Service | Media | Upload, thumbnails, compression; metadata in DB; blobs via File Storage |
| File Storage Service | Media (infra) | Object storage (S3-like) supporting Media |
| Notification Service | Notification | Push (APNs/FCM), retries, dead-letter queues |
| Search Service | Search | Message/group/contact search indexes and queries |
| Administration Service | Administration | Moderation, reporting, abuse controls |

Cross-service communication prefers **events** for loose coupling (for example `MessageSent` → Notification and Search). Synchronous calls are reserved for requests that must complete on the user-facing path and cannot tolerate eventual consistency.

### Typical Message Flow

**Sender → Edge → Messaging Service (store + enqueue) → Recipient Edge (push or store) → Ack back to sender**

```mermaid
sequenceDiagram
  participant S as Sender Client
  participant SE as Sender Edge
  participant MS as Messaging Service
  participant Q as Event Bus
  participant NS as Notification Service
  participant RE as Recipient Edge
  participant R as Recipient Client

  S->>SE: Encrypted message frame
  SE->>MS: Authenticated route + persist
  MS->>MS: Conversation aggregate: apply MessageSent
  MS->>Q: Publish MessageSent / delivery work
  Q->>RE: Deliver to recipient session
  Q->>NS: Trigger push if offline
  alt Recipient online
    RE->>R: Push message frame
    R->>RE: Delivered ack
  else Recipient offline
    MS->>MS: Retain for catch-up
  end
  RE->>MS: Ack status → MessageDelivered
  MS->>SE: Update sender
  SE->>S: Ack notification
```

| Layer | Responsibility |
|-------|----------------|
| Clients | E2EE, UI, local store, reconnect/catch-up |
| API Gateway / Connection Edge | TLS, auth, routing, rate limits, session affinity |
| Domain services | Own aggregates, invariants, and domain events per context |
| Event bus | Fan-out, push triggers, search indexing, analytics |
| Data & cache | Per-service stores; Redis for sessions/presence |
| File storage / CDN | Encrypted media blobs |

---

# 7. Core Components & Services

Services listed in Section 6 are the runtime components. Each microservice internally follows Clean Architecture so domain rules do not depend on controllers, ORMs, or brokers.

## Clean Architecture Inside a Service (Messaging Example)

```text
Messaging Service
│
├── Presentation (API / Controllers / WebSocket handlers)
│
├── Application
│   ├── Commands (e.g., SendMessageCommand)
│   ├── Queries
│   ├── DTOs
│   └── Validators
│
├── Domain
│   ├── Entities / Aggregates (Conversation, Message)
│   ├── Value Objects
│   ├── Domain Events
│   └── Interfaces (repositories, domain services)
│
├── Infrastructure
│   ├── Persistence (EF Core, Cassandra client, Redis, etc.)
│   ├── Messaging (Kafka / RabbitMQ)
│   ├── Repository implementations
│   └── External integrations
│
└── Tests (unit / integration)
```

| Layer | Dependency Rule |
|-------|-----------------|
| Domain | No dependencies on Application, Presentation, or Infrastructure |
| Application | Orchestrates use cases; depends only on Domain abstractions |
| Infrastructure | Implements Domain/Application ports (persistence, bus, push adapters) |
| Presentation | Translates HTTP/WebSocket frames into commands/queries |

The same layering applies to Identity, Group, Presence, Media, and other services. Framework choices (EF Core vs another ORM, Kafka vs RabbitMQ) stay in Infrastructure and can change without rewriting Conversation or Group invariants.

### Design Notes

- **Do not start with microservices** — derive them from bounded contexts; consolidate only with an explicit reason (team size, latency, transactional needs).
- **Connection edge** stays thin: auth, heartbeats, routing—not Conversation invariants.
- **Messaging Service** owns delivery guarantees and ack state transitions; it must remain available if Media, Search, or Calling degrade.
- **Group Service** owns membership consistency; Messaging and fan-out workers consume membership snapshots or membership events.
- **Presence Service** is separate because update frequency and consistency needs differ from durable Messaging.
- **Notification Service** is event-driven; payloads stay privacy-conscious (often opaque wake-ups).
- **Search / Analytics** consume events asynchronously and must not sit on the critical send path.

---

# 8. Database Design

| Data Class | Storage Choice | Rationale |
|------------|----------------|-----------|
| Users / metadata | PostgreSQL or MySQL (sharded) | Structured profiles, groups metadata, transactional updates |
| Messages | Cassandra, DynamoDB, or Mnesia-like wide-column / time-series | High write throughput; sharded by user or chat |
| Sessions / presence | Redis | Fast user → server mapping and online status |
| Media blobs | Object storage (S3-like) | Cheap, durable large-object storage |
| Media metadata | Relational or document store | Lookup by media ID, ownership, expiry |
| Groups | SQL + cache | Membership and admin state with hot-path caching |

### Sharding and Retention

- Shard by **user ID** or **conversation ID** to localize hot partitions and support independent scale-out.
- Apply retention policies: delete after successful delivery where product policy allows, or retain for a bounded period (for example 30 days) for multi-device catch-up—never longer than privacy and product requirements demand.
- Store **ciphertext + routing metadata** only for user content; avoid schemas that imply server-side plaintext bodies.

```text
MessageRecord (logical)
-----------------------
message_id
chat_id
sender_id
ciphertext
media_ref?          # optional
created_at
ack_state           # per recipient / device as needed
expiry_at
```

---

# 9. API Design

APIs split into control-plane REST (or similar) for uploads and catch-up, and a real-time session protocol for messaging frames.

### REST Examples

| Method & Path | Purpose |
|---------------|---------|
| `POST /messages` | Send message (`sender_id`, `receiver_id`, `type`, content or `media_id`) |
| `GET /messages?user_id=...` | Fetch unread / offline messages for catch-up |
| `POST /v1/media` | Upload media → return `media_id` |
| `GET /v1/media/download?file_id=...` | Download encrypted media blob |

### WebSocket / Real-Time Frames

| Frame Type | Purpose |
|------------|---------|
| `message.send` | Transmit encrypted chat message |
| `message.ack` | Sent / delivered / read acknowledgments |
| `presence.update` | Online / last-seen updates |
| `typing` | Typing indicators (often batched/throttled) |
| `call.signal` | Voice/video signaling messages |

Contracts should version carefully: clients on slow upgrade cycles must remain compatible with older frame schemas while new encryption or multi-device features roll out.

---

# 10. Communication Patterns

| Pattern | Where Used |
|---------|------------|
| Persistent bidirectional sessions | Online messaging, presence, typing, call signaling |
| Store-and-forward | Offline recipients and multi-device catch-up |
| Event-driven fan-out (Kafka or equivalent) | Group delivery, push triggers, analytics |
| Pub/Sub (Redis or equivalent) | Presence and cross-server session routing hints |
| Asynchronous processing | Media processing and large-group fan-out |

### Sync vs Async Guidance

- **Synchronous (session path):** auth handshake, online message push to a connected recipient, ack updates on the hot path.
- **Asynchronous:** group fan-out expansion, offline push, media post-processing, abuse scoring.

Prefer async when work can complete after the sender receives “sent,” and when temporary lag on read receipts or presence is acceptable.

---

# 11. Scalability Strategy

| Strategy | Application |
|----------|-------------|
| Horizontal connection scale | Add edge servers; target millions of connections per server with efficient runtimes |
| Sharding / partitioning | Partition users and chats; isolate hot groups |
| Message queues | Decouple ingest from fan-out and push |
| Multi-region + geo-routing | Place edges near users; keep regional failure domains |
| Caching + CDN | Session maps, presence, media downloads |

### Scaling Dimensions

1. **Connections** — Scale edge fleets independently of storage.
2. **Write path** — Scale message ingest and durable stores by shard.
3. **Fan-out** — Scale group workers separately; consider membership-size tiers (small sync fan-out vs large async fan-out).
4. **Media** — Scale object storage and CDN independently of chat servers.

---

# 12. Performance Considerations

- Use efficient protocols, compression, and binary framing to reduce bandwidth on constrained mobile networks.
- Geo-distribute edge PoPs so most users have a nearby connection terminator.
- Batch and throttle high-churn signals (especially typing indicators).
- Optimize database writes for append-heavy message workloads; preserve FIFO ordering per conversation where required.
- Keep edge handlers allocation-light; push heavy work to async workers.
- Prefer incremental catch-up (since watermark) over full history sync on reconnect.

---

# 13. Security Considerations

| Control | Implementation Focus |
|---------|----------------------|
| E2EE | Signal Protocol (X3DH, Double Ratchet; Sender Keys for groups); servers never see plaintext content |
| Transport security | TLS for all client and inter-service links on the public path |
| Identity | Phone-based registration with verification; device linking with explicit user consent |
| Abuse controls | Rate limiting, spam detection, reporting workflows |
| Data minimization | Minimal retention of ciphertext and metadata; clear expiry policies |

### Security Design Notes

- Key material for content encryption lives on client devices; server stores only what is required for routing, spam defense, and multi-device ciphertext delivery.
- Push payloads should avoid leaking message bodies; prefer opaque wake-ups plus encrypted catch-up over the secure channel.
- Business APIs need stricter rate limits, template controls, and abuse isolation so they cannot degrade consumer messaging capacity.

---

# 14. Reliability & Fault Tolerance

| Mechanism | Purpose |
|-----------|---------|
| Store-and-forward + durable queues | Survive recipient offline periods and transient edge loss |
| Replication | Survive node and zone failures for message and metadata stores |
| Reconnect + catch-up | Restore sessions and drain pending ciphertext after network flaps |
| Circuit breakers / isolation | Media or calling failures must not block text messaging |
| Resilient runtime practices | “Let it crash” supervision in runtimes suited to soft real-time messaging |

### Failure Modes Explicitly Designed For

- Regional edge outage → geo-reroute new sessions; drain or fail over sticky maps carefully
- Queue lag spike → backpressure, shed non-critical presence traffic first
- Reconnect storm → admission control, staggered reconnect, prioritized auth/catch-up capacity
- Hot group → isolate fan-out workers and protect shared message ingest

---

# 15. Deployment Strategy

| Aspect | Approach |
|--------|----------|
| Runtime packaging | Cloud-native containers / orchestration where appropriate; specialized OS/runtime for connection-dense cores |
| Topology | Multi-region; edge PoPs for connections and media/call relays |
| Releases | Blue-green or canary for edge and messaging services |
| Core servers | Erlang/FreeBSD-style (or equivalent) fleets historically favored for connection density and soft real-time |

Deploy connection and messaging planes with independent release trains so a media or calling change cannot force a global messaging outage window.

---

# 16. Monitoring & Observability

| Signal | Why It Matters |
|--------|----------------|
| Connection health / churn | Detect reconnect storms and bad client builds |
| Queue lag | Early warning for fan-out or offline delivery backlog |
| Delivery latency | User-visible messaging SLO |
| Drop / error rates | Protocol bugs, auth failures, shard hotspots |
| Regional saturation | Capacity planning and failover decisions |

### Observability Stack Expectations

- Metrics, structured logs, and distributed tracing across edge → message service → queue → recipient edge
- Dashboards analogous to Prometheus/Grafana for SLOs and capacity
- Alerting on reconnect storms, regional error budget burn, and sustained queue lag

Trace IDs should survive ack paths so “sent but never delivered” investigations can correlate sender and recipient hops without inspecting ciphertext bodies.

---

# 17. Trade-offs & Design Decisions

| Decision | Choice | Trade-off |
|----------|--------|-----------|
| Domains before microservices | Derive services from bounded contexts; consolidate only deliberately | More upfront modeling; fewer accidental service boundaries later |
| Consistency vs availability | Favor availability; eventual consistency for presence/read receipts | Simpler global operation; receipts may lag briefly |
| Stateful connections | Required for low-latency push | Needs careful session maps, sticky routing, and reconnect design |
| SQL vs NoSQL | Relational for metadata; wide-column for messages | Operational complexity of two storage models; better fit per workload |
| Custom vs off-the-shelf | Optimized protocols/runtimes for extreme scale | Higher expertise cost vs. faster commodity stacks |
| E2EE | Non-negotiable for user content | Adds multi-device and group-key complexity; limits server-side features |

### Alternatives Considered

- **Start with a microservice template per technical layer** — Faster to sketch boxes; tends to couple unrelated business rules and force distributed transactions across contexts.
- **Pure request/response polling** — Simpler ops, unacceptable latency and battery/network cost at this scale.
- **Single shared SQL store for all messages** — Stronger ad-hoc queryability, poor write scalability and hotspot behavior.
- **Server-side searchable plaintext** — Enables features, incompatible with E2EE product requirements.

---

# 18. Future Improvements

- Enhanced multi-device support (more linked devices, richer sync semantics)
- Privacy-preserving AI features (smart replies, summarization) that do not require server plaintext
- Further protocol optimizations for emerging networks and constrained devices
- Deeper integration with business platforms while protecting consumer capacity
- Improved calling relay scalability and potential AR/VR communication extensions

Each improvement should be evaluated against E2EE constraints, connection-plane capacity, and failure isolation before adoption.

---

# 19. Conclusion

This architecture delivers a reliable, private, and scalable messaging platform for internet-scale demand by starting from Domain-Driven Design—identifying bounded contexts and aggregates before mapping them to services—and applying Clean Architecture inside each service. Persistent connections, event-driven integration, efficient storage, and E2EE sit on that foundation. The result balances user experience with operational feasibility and remains a practical reference for similar real-time systems: domains first, microservices second, availability and ciphertext-first design throughout.
