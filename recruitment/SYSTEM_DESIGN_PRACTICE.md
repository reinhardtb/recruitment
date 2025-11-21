# System Design Practice Problem - Fintech Focus

## Practice Scenario: Real-Time Loyalty & Rewards Platform for Travel Bookings

**Context**: You're designing a loyalty and rewards system for a large online travel company (similar to Booking.com) that handles millions of bookings per day across hotels, flights, car rentals, and experiences.

### Business Requirements

**Core Functionality**:
- Customers earn points for every booking (hotels, flights, rentals)
- Points awarded based on spend, customer tier (Basic, Silver, Gold, Genius)
- Real-time balance updates (customers see points immediately after booking)
- Points can be redeemed for discounts on future bookings
- Partner integrations (airlines, hotel chains award bonus points)
- Promotional campaigns (2x points weekends, seasonal bonuses)

**Scale Expectations**:
- 100 million active users globally
- 2 million bookings per day (~23 bookings/second average, 100/sec peak)
- Multi-region deployment (Europe, North America, Asia, South America)
- 99.95% availability requirement
- Sub-second balance query latency

**Compliance & Security**:
- GDPR compliance (EU users)
- PCI-DSS (payment references, no card data in loyalty)
- Data residency (EU data stays in EU)
- Audit trail for all point transactions (7+ years retention)
- Fraud detection (suspicious redemptions, account takeovers)

---

## Questions You Should Ask

### Functional Clarifications
1. **Points calculation logic**:
   - Is it always 1:1 (1 euro = 1 point) or variable by product/tier?
   - Do we need to support retroactive point adjustments (e.g., refunds)?
   - How do promotional campaigns work (static rules or dynamic)?

2. **Redemption**:
   - Can points be redeemed during booking or only as discounts?
   - Are there minimum thresholds for redemption?
   - Do points expire? If so, what's the policy?

3. **Partner integrations**:
   - How many external partners (airlines, hotels)?
   - Are partner points awarded synchronously or asynchronously?
   - What happens if a partner system is down?

### Non-Functional Requirements
4. **Consistency model**:
   - Can we use eventual consistency for balance updates?
   - Or must it be strongly consistent (user sees points immediately)?
   - What about redemptions—must they be atomic with booking?

5. **Data retention**:
   - How long must we keep transaction history?
   - Archive strategy for old data?

6. **Disaster recovery**:
   - RTO (Recovery Time Objective)?
   - RPO (Recovery Point Objective)?
   - Multi-region active-active or active-passive?

---

## Expected Architecture Coverage

### 1. High-Level Architecture
Draw a system context diagram showing:
- Booking systems (hotels, flights, rentals)
- Loyalty platform
- Partner systems (airlines, hotel chains)
- Users (customers, support, marketing)
- External systems (CRM, analytics)

### 2. Data Architecture
**Key Decisions**:
- **Event sourcing** vs. **state-based storage**?
- **Database choice**: SQL (ACID), NoSQL (scale), or hybrid?
- **Partitioning strategy**: By user ID? By region?
- **Caching**: Redis for balance lookups?

**Data model**:
- Customer profiles (tier, balance)
- Transactions (bookings → points earned)
- Redemptions (points spent)
- Audit log (immutable)

### 3. Event-Driven Architecture
**Why events?**:
- Decouple loyalty from booking systems
- Enable partner integrations without blocking bookings
- Support promotional campaigns without code changes

**Topics/Streams**:
- `bookings.completed` → Points calculation
- `points.earned` → Balance aggregation
- `points.redeemed` → Booking discounts
- `partners.bonus-points` → External partner events

### 4. Security & Compliance
**PCI-DSS**:
- No payment card data in loyalty system
- Only booking references, user IDs

**GDPR**:
- Right to erasure (tombstone events in Kafka)
- Data minimization (only essential fields)
- Encryption at rest (AES-256) and in transit (TLS 1.3)

**Data Residency**:
- Regional Kafka clusters (EU, US, APAC)
- EU customer data never leaves EU
- Cross-region replication for disaster recovery only

### 5. Reliability & Availability
**Fault Tolerance**:
- Multi-region deployment (active-active or active-passive?)
- Database replication (sync or async?)
- Kafka cluster per region (3-5 nodes, RF=3)

**Disaster Recovery**:
- Automated failover to secondary region
- RTO: < 5 minutes (if active-passive)
- RPO: < 1 minute (event log replay from Kafka)

### 6. Performance & Scale
**Latency Targets**:
- Balance query: < 100ms (P99)
- Points calculation: < 1 second (eventual consistency acceptable?)
- Redemption: < 500ms (synchronous with booking)

**Throughput**:
- 100 bookings/sec peak → 100 events/sec to Kafka
- 10,000 balance queries/sec (read-heavy workload)

**Scaling Strategy**:
- Horizontal scaling (Kafka partitions, service replicas)
- Caching (Redis for hot user balances)
- Database sharding (by user ID or region)

---

## Trade-Offs to Discuss

### 1. Event-Driven vs. Synchronous
**Event-Driven (Recommended)**:
- ✅ Decouples loyalty from booking systems
- ✅ Better scalability, fault tolerance
- ❌ Eventual consistency (user sees points after delay)

**Synchronous**:
- ✅ Immediate consistency (points shown instantly)
- ❌ Tight coupling, booking latency increases

**Decision**: Event-driven with <1 second latency acceptable for points, synchronous for redemptions.

### 2. Database Choice
**SQL (PostgreSQL, Azure SQL)**:
- ✅ ACID transactions (critical for redemptions)
- ✅ Audit trail (7+ years retention)
- ❌ Harder to scale writes globally

**NoSQL (DynamoDB, Cosmos DB)**:
- ✅ Global distribution, low latency
- ✅ Horizontal scaling
- ❌ Eventual consistency challenges
- ❌ Complex transactions (redemptions)

**Decision**: Hybrid—SQL for transactions/audit, NoSQL (or cache) for balance queries.

### 3. Multi-Region Strategy
**Active-Active**:
- ✅ Low latency globally (users read/write locally)
- ❌ Complex conflict resolution (CRDTs?)
- ❌ Higher cost

**Active-Passive**:
- ✅ Simpler consistency model
- ❌ Higher latency for secondary regions
- ❌ Failover delay (RTO)

**Decision**: Active-passive with regional caches (balance queries local, writes to primary).

---

## Sample Solution Architecture

### High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Booking Systems                          │
│     (Hotels, Flights, Rentals, Experiences)                 │
└───────────────────┬─────────────────────────────────────────┘
                    │ Publish booking events
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              Event Streaming Platform (Kafka)               │
│   Topics: bookings.completed, points.earned, redemptions    │
└───────────────────┬─────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┬─────────────────┐
        ▼                       ▼                 ▼
┌───────────────┐      ┌────────────────┐  ┌──────────────┐
│ Points Calc   │      │ Balance        │  │ Redemption   │
│ Service       │      │ Aggregator     │  │ Service      │
│ (Stream)      │      │ (Stream)       │  │ (API)        │
└───────┬───────┘      └────────┬───────┘  └──────┬───────┘
        │                       │                  │
        └───────────────────────┴──────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │  Database Layer       │
                    │  - Transactions (SQL) │
                    │  - Balances (Cache)   │
                    │  - Audit Log (SQL)    │
                    └───────────────────────┘
```

### Key Design Decisions

1. **Transactional Outbox Pattern** (from booking systems):
   - Booking + event published atomically
   - No lost events (guaranteed delivery to Kafka)

2. **Stream Processing** (Kafka Streams or Flink):
   - Points Calculator: Applies tier multipliers, campaigns
   - Balance Aggregator: Maintains running totals per user
   - Exactly-once semantics for financial accuracy

3. **CQRS Pattern**:
   - Write path: Kafka Streams → SQL (transactions)
   - Read path: Redis cache (hot balances) + SQL (cold queries)

4. **API Gateway**:
   - Rate limiting (prevent abuse)
   - Authentication (OAuth 2.0)
   - Request routing (balance queries vs. redemptions)

5. **Fraud Detection**:
   - Real-time ML model on Kafka streams
   - Flags suspicious redemptions (large amounts, rapid changes)
   - Manual review workflow for support team

---

## Deep Dive Topics (Be Ready)

Interviewers may probe on:

### 1. Idempotency
**Question**: "What if the same booking event is processed twice?"

**Answer**:
- Store booking ID in transactions table (unique constraint)
- Kafka Streams exactly-once semantics (transactional writes)
- Idempotent API endpoints (use idempotency keys)

### 2. Schema Evolution
**Question**: "How do you handle changes to event schemas without breaking consumers?"

**Answer**:
- Schema Registry (Confluent, AWS Glue)
- Backward/forward compatibility rules
- Consumer versioning strategy

### 3. Regional Failover
**Question**: "What happens if the EU region goes down?"

**Answer**:
- DNS failover to US region (1-2 min RTO)
- Kafka consumers resume from last committed offset
- Cache warming strategy (pre-load hot user balances)

### 4. Promotional Campaigns
**Question**: "How do you support 2x points weekends without code changes?"

**Answer**:
- Campaign rules stored in database (marketing configures via UI)
- Points Calculator reads rules dynamically
- Real-time activation (no deployment needed)

### 5. Partner Integration Failures
**Question**: "Airline partner API is down—how do you handle bonus points?"

**Answer**:
- Dead letter queue for failed partner events
- Retry with exponential backoff
- Manual reconciliation process (support team)
- SLA with partners (99.9% uptime)

---

## Time Management (60 min Interview)

**0-10 min**: Requirements Gathering
- Ask clarifying questions
- Confirm scale, consistency, compliance needs

**10-15 min**: High-Level Architecture
- Draw system context diagram
- Identify main components (booking, loyalty, partners)

**15-30 min**: Detailed Design
- Data model (transactions, balances, audit)
- Event flows (booking → points → balance)
- API design (query balance, redeem points)

**30-45 min**: Deep Dives
- Interviewer probes specific areas (security, scale, reliability)
- Discuss trade-offs (SQL vs. NoSQL, active-active vs. active-passive)

**45-55 min**: Wrap-Up
- Summarize key decisions
- Acknowledge what you'd improve with more time

**55-60 min**: Your Questions
- Ask about their tech stack, team structure, challenges

---

## Practice Tips

1. **Draw first, explain later**: Start with a simple diagram, then iterate
2. **Think out loud**: Verbalize your thought process
3. **Ask for feedback**: "Does this make sense?" "Should I go deeper here?"
4. **Be honest**: If you don't know, say so—then propose how you'd find out
5. **Relate to experience**: Reference your loyalty platform work when relevant

---

## Your Strengths (Leverage These)

From your loyalty platform experience:
- ✅ Event-driven architecture (Kafka, Kafka Streams)
- ✅ Transactional outbox pattern
- ✅ CQRS (command vs. query separation)
- ✅ Compliance (GDPR, PCI-DSS, gaming regulations)
- ✅ Hybrid cloud (on-prem + cloud)
- ✅ Strangler fig pattern (incremental migration)

Use these as reference points when discussing solutions!

---

Good luck! 🚀
