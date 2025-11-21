# Loyalty Extraction Platform - System Context

**Project**: Loyalty Points Calculation Platform Extraction  
**Organization**: Derivco  
**Domain**: Gaming/Fintech  
**Timeline**: [Add dates]  
**Role**: Lead Architect

---

## Executive Summary

### Key Stats
- **Scale**: 50 billion events/month (~69 million events/hour average, ~139M peak)
- **Distribution**: Multi-region (Malta, Canada, Isle of Man, Europe, Asia, Oceania)
- **Architecture**: Hybrid cloud (on-prem + Azure)
- **Domain**: Fintech (gaming industry loyalty rewards)
- **Impact**: 30% wager latency reduction, 99.9%+ availability

---

## Business Context

### The Problem

**Legacy State**: Loyalty points calculation was tightly coupled within the core wager processing monolith.

**Pain Points**:
1. **Performance Bottleneck**: Heavy loyalty calculation logic inside critical wager path
2. **Wager Latency**: Every wager transaction slowed by synchronous loyalty processing
3. **Scalability Limits**: Monolithic architecture couldn't scale loyalty independently
4. **Operational Risk**: Loyalty failures could impact core wagering (revenue risk)
5. **Feature Velocity**: Loyalty enhancements required changes to monolith (slow, risky deployments)
6. **Multi-Operator Complexity**: Different operators had different loyalty rules, all embedded in monolith

### Business Impact
- **Wager latency** directly affects player experience and operator revenue
- Loyalty was identified as a **"big-hitter"** in performance profiling
- Streamlining wager process = competitive advantage for operators
- Loyalty = customer retention mechanism (critical business value)

### Stakeholders
- **Gaming Operators**: Configure loyalty rules, redemption policies
- **Players**: Earn and redeem points for prizes, bonuses
- **Compliance Teams**: Audit trail requirements for loyalty transactions
- **Product Teams**: Need to innovate on loyalty features quickly
- **Platform Engineering**: Need to scale wager processing independently

---

## Solution Approach

### Strategic Pattern: **Strangler Fig**

Extract loyalty processing from monolith while maintaining business continuity.

### Architecture Transformation

**Before (Synchronous, Coupled)**:
```
Player Wager → Monolith (Wager Process + Loyalty Calculation) → Wager Result + Loyalty Points
```

**After (Asynchronous, Decoupled)**:
```
Player Wager → Monolith (Streamlined Wager) → Wager Result
                    ↓ (best-effort event)
                Kafka Event Stream
                    ↓
            Loyalty Platform (async) → Points Balance Updated
```

### Key Architectural Decision: **Async via Events**

**Why Async**:
- Loyalty calculation removed from critical wager path
- Wager latency reduced (streamlined)
- Independent scaling of loyalty processing
- Failure isolation (loyalty issues don't break wagering)

**Trade-off**: Eventual consistency
- Players don't see points update immediately
- Acceptable: Loyalty is rewards mechanism, not transactional requirement
- Mitigation: Fast async processing (near real-time), clear player communication

---

## High-Level Architecture

### Hybrid Cloud Event-Driven Platform

```
┌─────────────────────────────────────────────────────────────────┐
│                        ON-PREMISES                               │
│  ┌────────────┐                    ┌──────────────┐            │
│  │  Monolith  │──best-effort──────→│ Kafka Cluster│            │
│  │   (Wager)  │    events          │  (on-prem)   │            │
│  └────────────┘                    └──────┬───────┘            │
│                                            │                     │
└────────────────────────────────────────────┼─────────────────────┘
                                             │ Kafka Connect
                                             ↓
┌─────────────────────────────────────────────────────────────────┐
│                    CONFLUENT CLOUD (Azure)                       │
│                                                                  │
│  ┌──────────────────────────────────────────────────┐          │
│  │           Kafka Topics (Confluent Cloud)         │          │
│  │  • wager.events (raw)                            │          │
│  │  • loyalty.eligible (filtered)                   │          │
│  │  • loyalty.calculated (points)                   │          │
│  └─────────────────┬────────────────────────────────┘          │
│                    ↓                                             │
│  ┌──────────────────────────────────────────────────┐          │
│  │       Kafka Streams Processing                   │          │
│  │  1. Eligibility Filter (operator rules)          │          │
│  │  2. Points Calculation (configurable logic)      │          │
│  │  3. Aggregation (player balance)                 │          │
│  └─────────────────┬────────────────────────────────┘          │
│                    ↓                                             │
│  ┌──────────────────────────────────────────────────┐          │
│  │    Materialized View → SQL Server (Azure)        │          │
│  │         Player Points Balance                    │          │
│  └──────────────────────────────────────────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                                   ↓
                    ┌──────────────────────────────┐
                    │  Operator Portals / APIs     │
                    │  • Balance queries           │
                    │  • Redemption processing     │
                    │  • Rule configuration        │
                    └──────────────────────────────┘
```

---

## Core Components

### 1. Event Source: Wager Events (Best-Effort)

**Pattern**: Event-Carried State Transfer via Outbox Pattern

**Event Publishing**: 
- Monolith uses SQL Server with SQLCLR stored procedures
- Outbox pattern: Events written to outbox table in same transaction as wager
- SQLCLR publishes events to Kafka asynchronously
- Ensures transactional consistency (wager + event publication)

**Event Schema** (simplified):
```json
{
  "wagerId": "uuid",
  "playerId": "uuid",
  "operatorId": "operator-123",
  "gameId": "game-456",
  "wagerAmount": 100.00,
  "currency": "USD",
  "timestamp": "2024-11-11T10:30:00Z",
  "gameCategory": "slots",
  "jurisdiction": "Malta"
}
```

**Best-Effort Semantics**:
- Monolith publishes wager events after wager processing
- Fire-and-forget (don't wait for confirmation)
- At-least-once delivery (Kafka guarantees)

### 2. Kafka Connect: On-Prem to Cloud Bridge

**Technology**: Kafka Connect (Confluent)

**Why**:
- Reliable event replication from on-prem Kafka to Confluent Cloud
- Managed by Confluent (reduced operational overhead)
- Automatic backpressure and retry handling

**Configuration**:
- Source: On-prem Kafka cluster
- Destination: Confluent Cloud Kafka
- Topics: `wager.events` → replicated

### 3. Kafka Streams Processing

**Technology**: Kafka Streams (Java)

**Major Technology Shift**: Derivco is a .NET-focused organization. Introducing Java for Kafka Streams was a significant strategic decision - choosing the best tool for the job even though it meant stepping outside the .NET ecosystem.

**Processing Stages**:

#### Stage 1: Eligibility Filter
```java
// Java Kafka Streams
StreamsBuilder builder = new StreamsBuilder();

builder.stream("wager.events", Consumed.with(Serdes.String(), wagerEventSerde))
  .filter((key, event) -> {
    // Operator-specific eligibility rules
    return operatorConfig.get(event.getOperatorId()).isEligible(event.getGameCategory());
  })
  .filter((key, event) -> {
    // Minimum wager amount threshold
    return event.getWagerAmount() >= operatorConfig.get(event.getOperatorId()).getMinWagerAmount();
  })
  .to("loyalty.eligible", Produced.with(Serdes.String(), wagerEventSerde));
```

**Operator Configuration** (externalized):
- Eligible game categories
- Minimum wager amounts
- Excluded players (self-excluded, etc.)
- Jurisdiction-specific rules

#### Stage 2: Points Calculation
```java
// Java Kafka Streams
builder.stream("loyalty.eligible", Consumed.with(Serdes.String(), wagerEventSerde))
  .mapValues(event -> {
    var config = operatorConfig.get(event.getOperatorId());
    var pointsMultiplier = config.getMultiplier(event.getGameCategory());
    
    return new PointsCalculated(
      event.getPlayerId(),
      event.getOperatorId(),
      event.getWagerAmount() * pointsMultiplier,
      event.getWagerId(),
      event.getTimestamp()
    );
  })
  .to("loyalty.calculated", Produced.with(Serdes.String(), pointsCalculatedSerde));
```

**Calculation Rules** (operator-configurable):
- Points multiplier per game category (slots = 0.1, table games = 0.05)
- Tiered loyalty levels (bronze, silver, gold, platinum)
- Promotional bonuses (double points weekends, etc.)

#### Stage 3: Aggregation & Materialization
```java
// Java Kafka Streams
builder.stream("loyalty.calculated", Consumed.with(Serdes.String(), pointsCalculatedSerde))
  .groupByKey()
  .aggregate(
    () -> new PlayerBalance(0.0),
    (playerId, newPoints, currentBalance) -> 
      new PlayerBalance(currentBalance.getPoints() + newPoints.getPoints()),
    Materialized.<String, PlayerBalance, KeyValueStore<Bytes, byte[]>>as("player-balances-store")
      .withKeySerde(Serdes.String())
      .withValueSerde(playerBalanceSerde)
  )
  .toStream()
  .to("loyalty.balances", Produced.with(Serdes.String(), playerBalanceSerde));
```

**Materialized View**:
- Kafka Streams state store (RocksDB) for in-memory aggregation
- Change Data Capture (CDC) to SQL Server for persistence
- SQL Server = source of truth for points balance queries

### 4. SQL Server Points Balance Store

**Technology**: Azure SQL Database

**Schema** (simplified):
```sql
CREATE TABLE PlayerLoyaltyBalance (
  PlayerId UNIQUEIDENTIFIER PRIMARY KEY,
  OperatorId VARCHAR(50) NOT NULL,
  PointsBalance DECIMAL(18,2) NOT NULL,
  LastUpdated DATETIME2 NOT NULL,
  Version BIGINT NOT NULL -- Optimistic locking
);

CREATE TABLE LoyaltyTransaction (
  TransactionId UNIQUEIDENTIFIER PRIMARY KEY,
  PlayerId UNIQUEIDENTIFIER NOT NULL,
  WagerId UNIQUEIDENTIFIER NOT NULL,
  OperatorId VARCHAR(50) NOT NULL,
  PointsEarned DECIMAL(18,2) NOT NULL,
  Timestamp DATETIME2 NOT NULL,
  EventOffset BIGINT NOT NULL -- Kafka offset for idempotency
);
```

**Why SQL Server**:
- Operators already use SQL Server (familiarity)
- Strong consistency for balance queries (required for redemptions)
- ACID transactions for redemption processing
- Rich query capabilities for reporting/analytics

**Idempotency**:
- `EventOffset` column ensures duplicate events don't double-count points
- Kafka Streams `exactly-once` semantics within processing
- At-least-once delivery + idempotent sink = effectively exactly-once

---

## Key Design Decisions

### Decision 1: Async Extraction via Best-Effort Events

**Context**: Loyalty was synchronous, blocking wager processing.

**Alternatives Considered**:
1. **Keep synchronous**: Refactor but stay in wager path
2. **Request-response async**: Wager calls loyalty service, waits for response
3. **Best-effort events**: Fire-and-forget, async processing

**Choice**: Best-effort events

**Rationale**:
- Maximizes wager performance (no waiting)
- Failure isolation (loyalty issues don't break wagering)
- Enables independent scaling
- Eventual consistency acceptable for loyalty rewards

**Trade-offs**:
- ❌ Players don't see points immediately
- ❌ Need to handle duplicate/late events (idempotency)
- ✅ Wager latency reduced by ~30% (measured)
- ✅ Loyalty can be down without revenue impact

### Decision 2: Kafka Streams for Processing (Java)

**Context**: Need to process events with business logic (eligibility, calculation). Derivco is a .NET-focused organization.

**Alternatives Considered**:
1. **Azure Functions**: Serverless, event-triggered
2. **Custom .NET microservices**: REST APIs consuming Kafka
3. **Kafka Streams (Java)**: Best-in-class stream processing
4. **Apache Flink**: Alternative stream processor

**Choice**: Kafka Streams (Java)

**Rationale**:
- Best tool for Kafka-based stream processing
- Built for Kafka (native integration)
- Stateful processing (aggregation, windowing)
- Exactly-once semantics
- Local state stores (low latency)
- Horizontal scaling via partitioning
- Mature ecosystem and community

**Trade-offs**:
- ❌ JVM-based (team needed to learn Java - major shift for .NET organization)
- ❌ Operational complexity (managing JVM containers)
- ❌ More complex deployment than Functions
- ✅ Best performance for high-throughput stream processing
- ✅ Strong consistency guarantees
- ✅ Mature and well-supported

**Strategic Decision**: Introduced Java to the organization for the right use case. Worth the technology shift for stateful stream processing at scale.

### Decision 3: Hybrid Cloud (On-Prem + Confluent Cloud)

**Context**: Monolith is on-prem, want cloud elasticity.

**Alternatives Considered**:
1. **Full on-prem**: Keep everything on-prem
2. **Full cloud migration**: Move monolith to cloud
3. **Hybrid**: Bridge on-prem to cloud

**Choice**: Hybrid with Kafka Connect

**Rationale**:
- Monolith migration is multi-year effort (too slow)
- Loyalty extraction is first step in broader decomposition
- Confluent Cloud offers managed Kafka (reduced ops burden)
- Azure integration for SQL Server, monitoring, etc.

**Trade-offs**:
- ❌ Network latency on-prem → cloud
- ❌ Operational complexity (two environments)
- ✅ Cloud elasticity for new services
- ✅ Start cloud migration without big-bang rewrite

### Decision 4: Materialized View in SQL Server

**Context**: Need fast balance queries for redemptions and operator portals.

**Alternatives Considered**:
1. **Query Kafka topics directly**: Real-time, no separate store
2. **Event sourcing only**: Rebuild state from events on demand
3. **Materialized view in SQL**: Pre-computed, queryable

**Choice**: Materialized view in SQL Server

**Rationale**:
- Balance queries must be fast (<100ms)
- Operators need SQL for reporting, analytics
- Redemption processing requires ACID transactions
- SQL Server already standard in organization

**Trade-offs**:
- ❌ Eventual consistency (Kafka → SQL lag)
- ❌ Dual write complexity (Kafka + SQL)
- ✅ Fast queries (indexed, optimized)
- ✅ Familiar tooling for operators

---

## Operator Configuration

### Configurable Parameters

**Eligibility Rules**:
- Eligible game categories (slots, table games, poker, etc.)
- Minimum wager amount threshold
- Excluded player segments (self-excluded, bonus abusers, etc.)
- Jurisdiction-specific rules (e.g., Malta vs US regulations)

**Points Calculation**:
- Base points multiplier per game category
- Tiered loyalty levels (bronze → platinum with increasing multipliers)
- Promotional periods (double points weekends, seasonal bonuses)
- Currency conversion rates (for multi-currency operators)

**Configuration Storage**:
- **Azure Cosmos DB** (or SQL Server): Operator config store
- **Kafka Streams GlobalKTable**: Config synced to stream processors
- **Config versioning**: Track changes for audit/rollback

**Configuration Update Flow**:
```
Operator Portal → Config API → Cosmos DB → Kafka Config Topic 
→ Kafka Streams GlobalKTable → Processing logic updated
```

---

## Scale & Performance

### Metrics Achieved

- **Event Throughput**: 50 billion events/month (~69M events/hour average, ~139M peak)
- **Processing Latency**: <2 seconds (event publish → balance update)
- **Wager Latency Reduction**: ~30% improvement (loyalty removed from critical path)
- **Availability**: 99.9%+ (loyalty failures don't impact wagering)
- **Data Volume**: 50+ billion loyalty transactions monthly

### Scalability Patterns

**Kafka Partitioning**:
- Partition by `playerId` for ordered processing per player
- Enables horizontal scaling (add more Kafka Streams instances)

**Kafka Streams Scaling**:
- Each instance processes subset of partitions
- Auto-rebalancing on instance failure/addition
- Stateful processing via local RocksDB stores

**SQL Server Scaling**:
- Read replicas for balance queries
- Write master for updates and redemptions
- Connection pooling, query optimization

---

## Regulatory & Compliance

### Audit Requirements

**Immutable Event Log**:
- All wager events stored in Kafka (long retention)
- Loyalty transactions table = full audit trail
- Can replay/reconstruct any player balance from events

**Data Retention**:
- Kafka: 7 days max (event stream retention)
- SQL Server: 7+ years (regulatory requirement)
- Data Warehouse: Long-term event storage and analytics

**GDPR Compliance**:
- Right to erasure: Tombstone events in Kafka, anonymize SQL data
- Data export: Generate player loyalty history report
- Consent tracking: Player opt-in/opt-out for loyalty program

**Jurisdictional Rules**:
- Malta Gaming Authority requirements
- Canadian provincial gambling regulations
- Isle of Man Gambling Supervision Commission
- UK Gambling Commission standards
- European jurisdictions (various)
- Implemented via operator-specific configuration

---

## Monitoring & Observability

### Key Metrics

**Event Processing**:
- Kafka consumer lag (alerts if > threshold)
- Event processing rate (events/sec)
- Error rate (DLQ messages)

**Business Metrics**:
- Points earned per operator/player
- Redemption rate (points → prizes)
- Average points balance

**Infrastructure**:
- Kafka Streams instance health
- SQL Server query performance
- Network latency (on-prem → cloud)

### Tooling

- **Confluent Control Center**: Kafka cluster monitoring
- **Azure Monitor**: Cloud infrastructure and SQL Server
- **Application Insights**: Custom metrics, distributed tracing
- **Grafana + Prometheus**: Unified dashboards

---

## Migration Strategy (Strangler Fig)

### Phased Rollout

**Phase 1: Dual Write (Shadow Mode)**
- Monolith continues synchronous loyalty calculation
- Also publishes wager events to Kafka
- New platform processes events but results not used
- Validate: Compare old vs new points calculations

**Phase 2: Gradual Cutover (Operator by Operator)**
- Select pilot operator (low risk)
- Switch to async loyalty for that operator
- Monitor closely (rollback plan ready)
- Expand to more operators incrementally

**Phase 3: Decommission Old Code**
- All operators on new platform
- Remove synchronous loyalty code from monolith
- Archive old loyalty database (audit compliance)

**Rollback Plan**:
- Feature flag to switch back to synchronous mode
- Dual write continues until confident in new platform
- Kafka events retained for replay if needed

---

## Learnings & Retrospective

### What Went Well

✅ **Performance Impact**: 30% wager latency reduction (measurable business value)

✅ **Failure Isolation**: Loyalty issues no longer crash wagering (reduced outages)

✅ **Feature Velocity**: Loyalty enhancements now independent deployments (weeks → days)

✅ **Kafka Streams**: Excellent fit for stateful stream processing

✅ **Operator Adoption**: Configuration flexibility won over operators

### What I'd Do Differently

❌ **Observability from Day 1**: 
- Initially underestimated distributed tracing complexity
- Retrofitting OpenTelemetry was harder than building it in
- **Lesson**: Observability is architectural, not operational

❌ **Event Schema Evolution**:
- Early schemas were too rigid
- Breaking changes forced dual-schema support
- **Lesson**: Design for schema evolution (Avro, protobuf) from start

❌ **Chaos Engineering**:
- Discovered failure modes in production (Kafka Connect network failures)
- Should have proactively tested network partitions, broker failures
- **Lesson**: Chaos engineering should be part of architecture validation

❌ **Configuration Versioning**:
- Operator config changes sometimes caused unexpected behavior
- Added versioning and rollback capability later
- **Lesson**: Configuration is code, needs same rigor (version control, testing)

### Transferable Insights

🎯 **Event-Driven Architectures Excel for Horizontal Platforms**:
- Decouples producers (monolith) from consumers (loyalty platform)
- Enables independent evolution and scaling
- Perfect for enabling multiple product teams

🎯 **Hybrid Cloud Requires Careful Network Architecture**:
- Latency matters: On-prem → cloud adds 20-50ms
- Network failures are normal: Design for intermittent connectivity
- Cost considerations: Egress bandwidth charges

🎯 **Regulatory Compliance Must Be Architectural**:
- Can't bolt on audit trails later
- Immutability (event sourcing) simplifies compliance
- Different jurisdictions = configuration, not code branches

🎯 **Eventual Consistency Is a Feature, Not a Bug**:
- Initially seen as limitation vs synchronous processing
- Enabled massive performance improvements
- Education and communication key to stakeholder buy-in

🎯 **Strangler Fig Pattern Works for Large Systems**:
- Gradual extraction reduces risk vs big-bang rewrite
- Dual write enables validation and rollback
- Operator-by-operator rollout provided safety net

---

## Relevance to Booking.com FinTech Foundations

### Alignment with Role Responsibilities

**1. Shape Architectural Direction**:
- Designed platform that enabled product teams to innovate on loyalty features
- Established event-driven patterns for future extractions

**2. Bridge Theory and Practice**:
- Applied event sourcing, CQRS, saga patterns to real-world fintech problem
- Balanced academic rigor (patterns) with pragmatic delivery (operator needs)

**3. Simplify Technology Landscape**:
- Reduced monolith complexity by extracting high-impact service
- Standardized on Kafka for event streaming (organizational pattern)

**4. Champion Best Practices**:
- Introduced Kafka Streams to organization (new capability)
- Documented patterns, mentored teams on event-driven architecture

**5. Influence Stakeholders**:
- Convinced operators to accept eventual consistency (business trade-off)
- Worked with compliance teams on audit requirements
- Collaborated with platform engineering on Kafka operations

### Horizontal Service Parallels

**Loyalty Platform = Foundational Service**:
- ✅ Enables multiple product teams (operators)
- ✅ Configurable, not customizable (turnkey)
- ✅ Abstracts complexity (event processing, state management)
- ✅ Platform thinking (not product delivery)

**Similar to FinTech Foundations**:
- Payment processing platform for Booking.com product teams
- Configurable by region/business unit (like operator config)
- Abstracts regulatory complexity (like gambling compliance)
- Enables innovation without reinventing the wheel

---

*Last Updated: November 11, 2025*  
*Status: Ready for presentation preparation*
