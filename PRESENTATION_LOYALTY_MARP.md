---
marp: true
theme: default
paginate: true
size: 16:9
style: |
  section {
    background-color: #ffffff;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }
  h1 {
    color: #0066cc;
    border-bottom: 3px solid #0066cc;
    padding-bottom: 10px;
  }
  h2 {
    color: #333333;
  }
  h3 {
    color: #666666;
  }
  code {
    background-color: #f4f4f4;
    padding: 2px 6px;
    border-radius: 3px;
  }
  pre {
    background-color: #f4f4f4;
    padding: 15px;
    border-radius: 5px;
    border-left: 4px solid #0066cc;
  }
  table {
    border-collapse: collapse;
    width: 100%;
  }
  th {
    background-color: #0066cc;
    color: white;
    padding: 10px;
  }
  td {
    padding: 8px;
    border: 1px solid #ddd;
  }
  blockquote {
    border-left: 4px solid #0066cc;
    padding-left: 20px;
    font-style: italic;
    color: #555555;
    background-color: #f9f9f9;
    padding: 15px;
    margin: 15px 0;
  }
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 20px;
  }
---

<!-- _class: lead -->
# Real-Time Event-Driven Fintech Architecture
## Loyalty Points Extraction Platform

**Presented by**: [Your Name]
**Role**: Senior Fintech Architect
**Context**: Derivco (iGaming) - $2B+ annual wager volume

**Today's Focus**: Migrating loyalty processing from legacy SQL Server systems to event-driven Kafka architecture
- 30% latency reduction
- 60% operational cost savings
- 50B events/month

---

# The Business Problem

## Monolithic Bottleneck (2021)

### Before State:
- **Architecture**: Multiple gaming systems per operator (legacy SQL Server instances)
- **Business Logic**: Stored procedures (legacy pattern - majority of logic in database)
- **Performance**: **3-5 second** wager-to-balance latency
- **Scalability**: Vertical scaling only (expensive SQL Server licenses per gaming system)

### Business Impact:
```
Player places $100 wager → 3-5 seconds → Points appear in balance
                        ↓
                   Player frustration
                   Support tickets
```

---

# The Business Problem (cont.)

## Critical Metrics:
- **30B wagers/month** (growing 20% YoY)
- **$180K/year** in operational overhead (Kafka cluster management)
- **99.5% availability** (SLA target: 99.9%)
- **2-4 hours** for dispute resolution (manual SQL queries)

## Executive Mandate:
> "Make it faster, more scalable, and reduce operational burden."

---

# Strategic Architecture Decision

## Hybrid Cloud + Event-Driven Architecture

### Key Decision: Strangler Fig Pattern
- **Keep**: Gaming systems on-premises (multiple per operator, stored procedure logic)
- **Extract**: Loyalty processing to cloud-native event-driven platform
- **Bridge**: Kafka Connect per datacenter (aggregates from multiple gaming systems)

---

# Technology Choices

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| **Java Kafka Streams** | Native streaming, exactly-once semantics | Team upskilling (.NET → Java) |
| **Confluent Cloud** | Managed Kafka, 99.99% SLA, auto-scaling | Vendor lock-in, cost |
| **SQLCLR Outbox** | Transactional consistency, SQL expertise | Performance overhead |
| **7-day Kafka retention** | Optimized for streaming | Need data warehouse |

---

# Communicating the Decision

<div class="columns">

<div>

## Communicating Up (Executives)

- **"60% reduction in Kafka ops = $108K/year savings"**
- **"30% latency = better UX = higher retention"**
- **"Strangler fig = low-risk incremental migration"**

</div>

<div>

## Communicating Down (Engineers)

- **"Kafka Streams = best tool for stateful processing"**
- **"SQLCLR outbox = transactional guarantees"**
- **"Learn industry-standard patterns (Netflix/Uber scale)"**

</div>

</div>

---

# Architecture Deep Dive

## End-to-End Data Flow

1. **Gaming Systems** (multiple per operator) → Insert wager + Outbox
2. **SQLCLR Publisher** → Poll outbox, publish to Kafka
3. **Kafka Topic**: `wagers-topic` (7-day retention)
4. **Filter Service** (Java) → Eligible wagers only
5. **Calculator Service** (Java) → Compute loyalty points
6. **Aggregator Service** (Java) → Materialize balances
7. **SQL Server Read Models** → Fast balance queries
8. **Data Warehouse** → 7+ year retention (compliance)

---

# Outbox Pattern (SQLCLR)

```sql
-- Deployed across multiple gaming systems
-- Business logic in stored procedures (legacy)
CREATE PROCEDURE PublishWagerEvent
    @PlayerId INT,
    @WagerAmount DECIMAL(18,2),
    @GamingSystemId VARCHAR(50)
AS
BEGIN TRANSACTION
    -- Gaming transaction (stored procedure business logic)
    INSERT INTO Wagers (PlayerId, Amount, GamingSystemId) 
    VALUES (@PlayerId, @Amount, @GamingSystemId)
    
    -- Outbox for Kafka (each gaming system has own outbox)
    INSERT INTO OutboxEvents (EventType, Payload, GamingSystemId) 
    VALUES ('WagerPlaced', JSON_OBJECT(...), @GamingSystemId)
COMMIT TRANSACTION
```

**Why**: Transactional consistency. No dual-write problem. Works with legacy stored procedures.

---

# Kafka Streams State Management

```java
// Stateful aggregation with exactly-once semantics
KStream<String, Wager> wagers = builder.stream("wagers-topic");

wagers.groupByKey()
    .aggregate(
        () -> new Balance(0),  // Initial state
        (key, wager, balance) -> balance.add(wager.points),
        Materialized.as("balance-store")  // RocksDB state store
    )
    .toStream()
    .to("balances-topic");
```

**Why**: 
- Stateful processing with automatic checkpointing
- Fault tolerance with RocksDB state stores
- Exactly-once guarantees (critical for financial accuracy)

---

# Data Retention Strategy

| Layer | Retention | Purpose | Compliance |
|-------|-----------|---------|------------|
| **Kafka** | 7 days | Stream processing, replay | Short-term |
| **SQL Server** | 7+ years | Operational queries (balances) | GDPR, PCI-DSS |
| **Data Warehouse** | 7+ years | Analytics, disputes, audits | Regulatory |

### Communicating Up:
> "We separate hot path (Kafka) from cold path (DW) for cost optimization while ensuring regulatory compliance."

### Communicating Down:
> "Kafka = fast streaming. SQL = operational queries. DW = compliance. Right tool for each layer."

---

# Critical Decision: Java in .NET Shop

## ADR-001: Java Kafka Streams vs. .NET Alternatives

**Context**: Derivco is 100% .NET/C# with stored procedure business logic. Introducing Java was controversial.

**Alternatives Considered**:
- ❌ **C# + Kafka Client**: No stateful processing, manual state management
- ❌ **Azure Stream Analytics**: Vendor lock-in, limited state support
- ✅ **Java Kafka Streams**: Native framework, exactly-once, proven at scale

---

# Justifying Java to Leadership

## Communicating Up (CTO):

> "While introducing Java adds technology diversity to our .NET/SQL Server ecosystem, Kafka Streams is the **battle-tested standard** for stateful stream processing. Used by LinkedIn, Netflix, Uber. The engineering investment (2 months training) is justified by **long-term maintainability** and **access to community patterns**. Critically, this lets us **extract loyalty without refactoring legacy gaming systems** (stored procedure business logic across multiple systems per operator)."

---

# Implementation Strategy

## Communicating Down (Team):

1. **Training Plan**: 2-month Java/Kafka Streams bootcamp (Confluent certification)
2. **Pair Programming**: Java experts paired with .NET engineers
3. **Shared Ownership**: Cross-functional team owns both .NET APIs and Java streams
4. **Career Growth**: "Learn industry-standard streaming - valued at FAANG companies"

---

# Results of Java Decision

## Outcomes:

- ✅ Team achieved **Confluent certification in 6 weeks**
- ✅ **Exactly-once processing** delivered (would've been months of custom .NET code)
- ✅ **Zero production incidents** related to Java/Kafka Streams in 18 months
- ✅ **2 engineers hired by AWS** for Kafka expertise

## Lesson:
> Sometimes the **best technical decision** isn't the easiest organizational decision. Clear communication of trade-offs and investment in people makes it successful.

---

# Security & Compliance

## Multi-Layered Security Architecture

### Data Protection:
- **AES-256** at rest
- **TLS 1.3** in transit
- **Field-level encryption** for PII (KMS-managed keys)

### Access Control:
- **Azure AD B2C** for player authentication
- **RBAC** on Kubernetes
- **Kafka ACLs** for topic permissions
- **SQL Row-Level Security**

---

# Compliance Requirements

## PCI-DSS (Payment Card Industry):
- No credit card data in loyalty platform (player references only)
- Separate Kafka partitions for payment vs. loyalty events
- All access logged to immutable Data Warehouse

## GDPR (Data Privacy):
- **Right to Erasure**: Tombstone events in Kafka
- **Data Minimization**: Only essential fields in events
- **Breach Notification**: Automated alerting

## Gaming Regulations:
- **7+ year retention** in Data Warehouse (immutable)
- **Dispute resolution**: Replay events to reconstruct balances
- **Data residency**: Regional Kafka clusters (Malta data stays in EU)

---

# Communicating Compliance

<div class="columns">

<div>

## To Regulators/Auditors:

> "Our architecture provides **cryptographic guarantees** of data integrity. Every loyalty point transaction is logged to an immutable data warehouse with SHA-256 checksums. We can **reconstruct any player's balance** from event history, proving fair play."

</div>

<div>

## To Engineers:

> "When publishing to Kafka, **never include PII in event keys**. Use player IDs, encrypt sensitive fields with KMS. For GDPR deletes, publish tombstone events—Kafka's log compaction handles cleanup."

</div>

</div>

---

# Performance & Scale

## Business Impact Metrics

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Wager-to-Balance Latency** | 3-5 seconds | <1 second | **70-80% reduction** |
| **Throughput** | 30B/month | 50B/month | **67% increase** |
| **Operational Overhead** | 2 FTE | 0.8 FTE | **60% reduction** |
| **System Availability** | 99.5% | 99.95% | **0.45pp improvement** |
| **Dispute Resolution** | 2-4 hours | 15 minutes | **88% reduction** |
| **Monthly Cost** | $10K | $4K | **60% savings** |

---

# Scale Achievements

## Production Statistics:

- **50B events/month** = 69M events/hour = **19K events/second** sustained
- **5-node Kafka clusters** across 6 regions (Malta, Canada, Asia, Europe, Oceania, Isle of Man)
- **Auto-scaling**: Kubernetes HPA scales stream processors 2-20 pods
- **Peak Traffic**: 3x normal during major tournaments (handled seamlessly)

---

# Communicating Impact

<div class="columns">

<div>

## Up (CFO/COO):

> "We've reduced infrastructure costs by **$72K/year** while **increasing capacity by 67%**. Faster loyalty updates correlate with **15% increase in player session duration**—players stay engaged when they see points instantly."

</div>

<div>

## Down (Engineering):

> "Your code processes **19K events/second** in production. Exactly-once guarantees mean you never worry about duplicate points. Kafka Streams handles partitioning, state distribution, failover automatically."

</div>

</div>

---

# Lessons Learned

## What Went Well ✅

1. **Incremental Migration** (Strangler Fig)
   - Extracted loyalty without touching gaming systems
   - Reduced risk: Could rollback at any phase

2. **Team Upskilling Investment**
   - Java training paid off: Zero production incidents
   - Engineers more valuable (multi-language skills)

3. **Managed Services** (Confluent Cloud)
   - Eliminated on-call burden for Kafka management
   - 99.99% SLA better than our on-prem (99.5%)

---

# What We'd Do Differently ⚠️

1. **Distributed Tracing from Day 1**
   - Initially relied on Kafka metrics alone
   - Added OpenTelemetry later (should've been upfront)

2. **Schema Registry from Start**
   - Started with JSON (flexible but error-prone)
   - Migrated to Avro after 6 months (2 weeks of effort)

3. **Chaos Engineering Quarterly**
   - Assumed Kafka Streams handled all failures
   - Discovered edge cases in production (network partitions)

---

# Future Roadmap

## Next Innovations:

1. **Machine Learning Integration**
   - Real-time fraud detection on wager patterns (Kafka → Azure ML)

2. **Multi-Currency Support**
   - Extend points calculation for crypto rewards (Bitcoin, Ethereum)

3. **Event Sourcing Migration**
   - Move SQL Server read models to full event sourcing
   - Rebuild any player balance from event history

---

# Why This Matters

## We Built a Platform, Not Just a Solution

> "The same event-driven architecture now powers **leaderboards, tournaments, and bonusing**. Future features leverage existing infrastructure—**marginal cost** to add new capabilities."

**Value Delivered**:
- Extract loyalty from legacy (no gaming system refactoring)
- Handle 50B events/month with exactly-once guarantees
- Reduce costs while increasing capacity
- Enable rapid feature development

---

# Key Takeaways

## Architecture Principles That Delivered Success

1. **Incremental Migration > Big Bang**
   - Strangler fig pattern reduced risk

2. **Best Tool > Organizational Comfort**
   - Introduced Java for Kafka Streams despite .NET culture

3. **Managed Services > DIY Infrastructure**
   - Confluent Cloud: 60% cost reduction, better SLA

4. **Compliance as Architecture**
   - Data residency, encryption, audit logging built-in

5. **Communicate Trade-offs**
   - **Up**: Business impact, ROI
   - **Down**: Technical rationale, career growth

---

# Final Numbers

| Metric | Impact |
|--------|--------|
| **Latency** | 70% reduction (better UX) |
| **Cost** | 60% savings ($108K/year) |
| **Scale** | 67% throughput increase (50B events/month) |
| **Reliability** | 99.95% availability |
| **Team** | 100% Java certification, zero attrition |

---

<!-- _class: lead -->
# Closing Statement

> "This wasn't just a technology migration—it was a **transformation** in how we think about data, scalability, and team capability. We built a platform that powers real-time fintech experiences for millions of players while maintaining the highest compliance and security standards."

> "I'm excited to bring this experience to Booking.com's fintech challenges—loyalty programs, dynamic pricing, fraud detection—where **milliseconds matter** and **scale is massive**."

---

<!-- _class: lead -->
# Questions?

**Thank you for your time**

[Your Name]
Senior Fintech Architect

---

# Backup Slides

## Appendix: Additional Technical Details

(Not presented unless specifically asked)

---

# A1. Detailed Cost Breakdown

## Infrastructure Costs

| Component | Before (On-Prem) | After (Cloud) | Savings |
|-----------|------------------|---------------|---------|
| **Kafka Clusters** | $8,000/month | $3,000/month | $5,000 |
| **Compute (AKS)** | N/A | $1,000/month | N/A |
| **Operations** | $2,000/month (2 FTE) | $800/month (0.8 FTE) | $1,200 |
| **Total** | $10,000/month | $4,800/month | **$5,200/month** |

**Annual Savings**: $62,400 + reduced FTE costs = **$108K/year**

---

# A2. Kafka Cluster Topology

## 5-Node Architecture per Region

- **3 Brokers**: Data replication (RF=3)
- **2 ZooKeeper Nodes**: Cluster coordination (migrating to KRaft)
- **Replication Factor**: 3 (no data loss on single node failure)
- **Min In-Sync Replicas**: 2 (ensures durability)
- **Partitions**: 30 per topic (parallelism for consumers)

**Why 5 nodes?**
- Balance between cost and availability
- Survives 1 node failure without data loss
- Handles 2x traffic spikes without re-provisioning

---

# A3. Kubernetes Resource Allocation

## Pod Configurations

### Wager Filter Service:
- **CPU**: 500m (request), 2000m (limit)
- **Memory**: 1Gi (request), 4Gi (limit)
- **Replicas**: 2-10 (HPA based on consumer lag)

### Points Calculator:
- **CPU**: 1000m (request), 4000m (limit)
- **Memory**: 2Gi (request), 8Gi (limit)
- **Replicas**: 2-20 (HPA based on CPU)

### Balance Aggregator:
- **CPU**: 500m (request), 2000m (limit)
- **Memory**: 1Gi (request), 4Gi (limit)
- **Replicas**: 2-10 (HPA based on consumer lag)

---

# A4. Monitoring Dashboards

## Key Metrics (Prometheus + Grafana)

### Kafka Metrics:
- Consumer lag per partition
- Message throughput (in/out)
- Replication lag (cross-region)

### Application Metrics:
- Processing latency (P50, P95, P99)
- Error rate per service
- State store size (RocksDB)

### Business Metrics:
- Wager-to-balance latency (end-to-end)
- Points awarded per minute
- Balance query response time

---

# A5. Incident Post-Mortem Example

## Kafka Partition Rebalance (May 2023)

**Incident**: Consumer group rebalanced during peak traffic, causing 5-minute processing delay.

**Root Cause**: Pod restart triggered by OOM (RocksDB state store too large).

**Resolution**:
1. Increased RocksDB cache size (1GB → 2GB)
2. Tuned Kafka Streams `commit.interval.ms` (30s → 10s)
3. Added memory limits to prevent OOM

**Lessons**:
- Monitor RocksDB state store growth
- Tune Kafka Streams memory allocation
- Implement gradual pod rollouts (avoid simultaneous restarts)

---

<!-- _class: lead -->
# End of Backup Slides

Return to main presentation or Q&A