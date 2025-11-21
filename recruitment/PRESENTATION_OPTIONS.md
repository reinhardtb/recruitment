# Presentation Case Options - Architecture Deep Dive

**Decision Required**: Choose which system to present in 25-minute architecture interview

---

## Option 1: Loyalty Extraction Platform ⭐ Primary Recommendation

### System Overview
Event-driven platform extracting loyalty calculations from legacy monolith using Kafka event streams, enabling real-time fintech processing at scale.

### Key Stats
- **Scale**: 50 billion events/month (~69 million events/hour average, ~139M peak)
- **Distribution**: Multi-region (Malta, US, UK)
- **Architecture**: Hybrid cloud (on-prem + Azure)
- **Domain**: Fintech (gaming industry loyalty rewards)

### Complexity Indicators
- ✅ Event sourcing with Kafka
- ✅ CQRS pattern for read/write separation
- ✅ Saga pattern for distributed transactions
- ✅ Strangler fig pattern for monolith extraction
- ✅ Regulatory compliance (Malta Gaming Authority, GDPR)
- ✅ Multi-tenant (multiple casino operators)
- ✅ Real-time financial calculations

### Why This Is Strong
1. **Horizontal Platform**: Served multiple product teams (aligns with FinTech Foundations role)
2. **Cloud Migration**: Hybrid architecture story
3. **Fintech Relevance**: Financial calculations, compliance, audit requirements
4. **Pattern Rich**: Multiple advanced patterns (event sourcing, CQRS, saga)
5. **Scale**: Enterprise-grade performance requirements
6. **Stakeholder Complexity**: Gaming operators, compliance teams, product teams

### Alignment with Interviewers
- **Leila (Fintech Director)**: ✅ Financial calculations, compliance, enabling product teams
- **Igor (Principal Architect)**: ✅ Distributed systems, event-driven patterns, scalability

### Potential Weaknesses
- May be less familiar domain to Booking.com (gaming vs travel)
- Requires explaining gaming loyalty context

---

## Option 2: Hybrid Cloud Leaderboard System 🎯 Alternative Choice

### System Overview
Real-time tournament leaderboard engine ingesting wager events, applying dynamic eligibility rules, and updating player rankings with low latency and high reliability.

### Architecture Flow
```
Player → On-Prem Kafka → Kafka Connect → Confluent Cloud 
→ ksqlDB (dynamic queries) → Azure Durable Functions 
→ Leaderboard State → Azure Front Door
```

### Key Stats
- **Scale**: 10,000+ concurrent players
- **Latency**: Sub-second leaderboard updates
- **Availability**: 99.99% uptime during peak tournaments
- **Architecture**: Hybrid cloud (on-prem Kafka → Confluent Cloud → Azure)

### Complexity Indicators
- ✅ Hybrid cloud architecture (on-prem + cloud)
- ✅ Dynamic query engine (ksqlDB) for operator-driven configuration
- ✅ Event streaming at scale (Kafka)
- ✅ Stateful serverless (Azure Durable Functions)
- ✅ Migration story: Azure Durable Functions → Kubernetes
- ✅ Global distribution (Azure Front Door)
- ✅ Multi-tenant (multiple casino operators)

### Key Features
- **Turnkey Platform**: Operators configure tournaments dynamically via ksqlDB
- **Elastic Processing**: Evolution from serverless to Kubernetes
- **Global Reliability**: Azure Front Door for cross-region failover
- **Observability**: Unified monitoring across cloud and on-prem

### Critical Design Decisions

#### 1. Hybrid Cloud Architecture
- **Problem**: On-prem reliability + cloud elasticity needed
- **Solution**: Kafka Connect bridge from on-prem to Confluent Cloud
- **Trade-offs**: Network latency vs operational flexibility

#### 2. Dynamic ksqlDB Queries
- **Problem**: Tournament rules vary by operator and change frequently
- **Solution**: ksqlDB for real-time, operator-configurable filtering
- **Benefit**: Turnkey platform without code deployments

#### 3. Azure Durable Functions → Kubernetes Migration
- **Initial Choice**: Azure Durable Functions for orchestration and state
- **Problem**: Scaling limits, performance unpredictability under load
- **Migration**: Kubernetes for fine-grained control
- **Learning**: Serverless abstractions can obscure performance bottlenecks

#### 4. Azure Front Door for Global Distribution
- **Problem**: Tournament players worldwide, need low latency
- **Solution**: Azure Front Door for load balancing and regional failover
- **Outcome**: 99.99% availability

### Lessons Learned
- **Serverless Trade-offs**: Durable Functions great for orchestration, but scaling limits hit at high load
- **Abstraction vs Control**: Closer control (Kubernetes) needed when performance is critical
- **Dynamic Configuration**: ksqlDB enabled operator flexibility without constant deployments
- **Pre-warming**: Essential for tournament spikes (learned the hard way)
- **Deduplication**: Critical for correctness in at-least-once event processing

### Why This Is Strong
1. **Pure Cloud Migration Story**: On-prem → Cloud hybrid architecture (nice to have!)
2. **Evolution Narrative**: Azure Functions → Kubernetes shows learning and adaptation
3. **Real-time Performance**: Sub-second latency requirements
4. **Multi-tenant Platform**: Turnkey solution for multiple operators
5. **Modern Stack**: Kafka, ksqlDB, Kubernetes, Azure services
6. **Clear Learnings**: Serverless limitations, migration rationale well-articulated

### Alignment with Interviewers
- **Leila (Fintech Director)**: ⚠️ Less fintech-specific (gaming leaderboards vs payments), but platform thinking still relevant
- **Igor (Principal Architect)**: ✅✅ Strong technical depth - hybrid cloud, serverless evolution, Kubernetes, distributed systems

### Potential Weaknesses
- Less fintech domain alignment (leaderboards vs financial systems)
- Gaming context may be less familiar to Booking.com interviewers
- Needs to connect platform thinking to fintech use cases

---

## Side-by-Side Comparison

| Criteria | Loyalty Extraction Platform | Hybrid Cloud Leaderboard |
|----------|----------------------------|-------------------------|
| **Cloud Migration Story** | ✅ Hybrid (on-prem + Azure) | ✅✅ Full migration journey |
- **Fintech Relevance** | ✅✅ Financial calculations, compliance | ⚠️ Gaming domain |
| **Scale Complexity** | ✅✅ 69M events/hour (50B/month) | ✅ 10K+ concurrent users |
| **Pattern Richness** | ✅✅ Event sourcing, CQRS, Saga, Strangler | ✅ Event streaming, stateful serverless |
| **Horizontal Platform** | ✅✅ Enabled product teams | ✅ Turnkey for operators |
| **Learnings/Evolution** | ✅ Monolith extraction, observability | ✅✅ Durable Functions → K8s migration |
| **Regulatory/Compliance** | ✅✅ Malta Gaming, GDPR, audit | ⚠️ Less emphasis |
| **Leila Alignment** | ✅✅ Fintech director | ⚠️ Less fintech-specific |
| **Igor Alignment** | ✅✅ Technical depth | ✅✅ Technical depth + evolution |
| **Modern Tech Stack** | ✅ Kafka, Azure, microservices | ✅✅ ksqlDB, K8s, Confluent Cloud |

---

## Recommendation Matrix

### Choose **Loyalty Extraction Platform** if:
- ✅ You want maximum fintech alignment with Leila
- ✅ You have strong compliance/regulatory examples ready
- ✅ You want to emphasize financial calculation patterns
- ✅ You have detailed CQRS/Saga pattern diagrams prepared
- ✅ You want to position as fintech domain expert

### Choose **Hybrid Cloud Leaderboard** if:
- ✅ You want a pure cloud migration story (nice to have!)
- ✅ You have compelling Durable Functions → K8s migration narrative
- ✅ You want to showcase evolution and learning from mistakes
- ✅ You have strong ksqlDB dynamic configuration story
- ✅ Igor is the primary decision-maker (technical depth focus)

---

## Hybrid Strategy (Advanced)

### Option 3: Reference Both Systems
**Primary**: Loyalty Extraction Platform (25 min presentation)  
**Secondary**: Mention Leaderboard as supporting example

**Example**:
> "Similar to this Loyalty platform, I also architected a real-time leaderboard system where we evolved from Azure Durable Functions to Kubernetes when we hit scaling limits—teaching me valuable lessons about when serverless abstractions help vs. hinder."

**Benefits**:
- Shows breadth of experience
- Demonstrates pattern recognition across domains
- Provides backup discussion material for Q&A

**Risks**:
- May dilute focus
- Could confuse narrative
- Only use if naturally fits

---

## Final Decision Factors

### Interview Context Priorities
1. **Leila's vote matters most** - She's your future fintech stakeholder
2. **Fintech domain knowledge** - Critical for FinTech Foundations team
3. **Cloud migration** - Nice to have, not mandatory
4. **Complexity** - Both systems qualify
5. **Learning/evolution** - Both have strong narratives

### My Recommendation: **Loyalty Extraction Platform** ⭐

**Rationale**:
- Stronger fintech alignment (critical for Leila)
- Richer pattern catalog (event sourcing, CQRS, saga)
- Compliance narrative (Malta Gaming, GDPR)
- Higher event volume (100K+ vs 10K+)
- Horizontal platform positioning (exact role fit)
- Financial integrity focus (audit logs, consistency)

**Use Leaderboard System**:
- As Q&A backup example
- To demonstrate breadth if asked about cloud migration evolution
- For serverless scaling lessons learned discussion

---

## Next Actions

1. **Make final decision** - Loyalty Platform (recommended) or Leaderboard
2. **Start architecture documentation** - Populate templates in `architecture/` folder
3. **Create diagrams** - C4, sequence, deployment for chosen system
4. **Prepare compliance narrative** - If Loyalty Platform (for Leila)
5. **Prepare evolution narrative** - If Leaderboard (Functions → K8s)
6. **Draft presentation outline** - Use `INTERVIEW_REQUIREMENTS.md` structure

---

*Document Created: November 11, 2025*  
*Decision Deadline: Before starting diagram creation*
