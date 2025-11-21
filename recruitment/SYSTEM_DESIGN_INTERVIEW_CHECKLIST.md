# System Design Interview Checklist

**Interview Date**: _______________  
**Interviewer(s)**: _______________  
**Problem Statement**: _______________

---

## Phase 1: Requirements Gathering (0-10 min)

### Functional Requirements
- [ ] Understand core business problem
- [ ] Identify key user personas (customers, support, marketing, etc.)
- [ ] Clarify main use cases (read-heavy vs. write-heavy?)
- [ ] Define success criteria (what does "done" look like?)

**Questions to Ask**:
- What are the primary user actions?
- Are there admin/configuration workflows?
- Any batch processing or reporting needs?

---

### Non-Functional Requirements

#### Scale
- [ ] How many users? (DAU/MAU)
- [ ] Transactions per second? (average vs. peak)
- [ ] Data volume? (GB/TB/PB)
- [ ] Geographic distribution? (single region vs. global)

**Questions to Ask**:
- What's the expected growth rate?
- Are there seasonal spikes?
- Which regions/countries?

---

#### Performance
- [ ] Latency targets? (P50, P95, P99)
- [ ] Throughput requirements?
- [ ] Read vs. write ratio?

**Questions to Ask**:
- What's acceptable latency for critical paths?
- Synchronous or asynchronous processing?
- Real-time or eventual consistency acceptable?

---

#### Availability & Reliability
- [ ] SLA target? (99.9%, 99.95%, 99.99%)
- [ ] Downtime acceptable? (planned maintenance windows?)
- [ ] RTO (Recovery Time Objective)?
- [ ] RPO (Recovery Point Objective)?

**Questions to Ask**:
- What happens if the system is down?
- Multi-region deployment required?
- Active-active or active-passive?

---

#### Security & Compliance
- [ ] Authentication/authorization requirements?
- [ ] Data sensitivity? (PII, financial, health)
- [ ] Regulatory compliance? (GDPR, PCI-DSS, HIPAA, SOC2)
- [ ] Data residency requirements?
- [ ] Audit trail needed?

**Questions to Ask**:
- Who has access to what data?
- How long must data be retained?
- Any encryption requirements?

---

## Phase 2: High-Level Design (10-25 min)

### System Context Diagram
- [ ] Draw main system boundaries
- [ ] Identify external actors (users, external systems)
- [ ] Show key integrations (APIs, events, databases)

**Components to Consider**:
- Users/Clients (web, mobile, partners)
- Core application services
- Data stores (databases, caches)
- External dependencies (payment gateways, APIs)

---

### API Design (if applicable)
- [ ] Define key endpoints (RESTful, GraphQL, gRPC?)
- [ ] Request/response formats
- [ ] Rate limiting strategy
- [ ] Versioning approach

**Example Endpoints**:
```
GET  /users/{userId}/balance
POST /users/{userId}/transactions
POST /redemptions (idempotent)
GET  /campaigns (marketing configuration)
```

---

### Data Flow
- [ ] Trace end-to-end flow for primary use case
- [ ] Identify synchronous vs. asynchronous paths
- [ ] Show event flows (if event-driven)

**Example**: User booking → Points awarded → Balance updated

---

## Phase 3: Detailed Design (25-45 min)

### Data Architecture

#### Database Choice
- [ ] SQL (PostgreSQL, MySQL, Azure SQL)?
- [ ] NoSQL (DynamoDB, Cosmos DB, MongoDB)?
- [ ] Hybrid (SQL for writes, NoSQL for reads)?

**Decision Criteria**:
- ACID transactions needed?
- Query patterns (simple lookups vs. complex joins)?
- Scale (vertical vs. horizontal)?

---

#### Data Model
- [ ] Core entities (users, transactions, balances)
- [ ] Relationships (1:1, 1:N, N:N)
- [ ] Indexes (for query performance)
- [ ] Partitioning/sharding strategy

**Example Schema**:
```sql
Users: userId (PK), tier, createdAt
Transactions: txId (PK), userId (FK), amount, points, timestamp
Balances: userId (PK), balance, lastUpdated
```

---

#### Caching Strategy
- [ ] What to cache? (hot user balances, configuration)
- [ ] Cache technology? (Redis, Memcached)
- [ ] TTL (time-to-live)?
- [ ] Cache invalidation strategy?

---

### Application Architecture

#### Service Decomposition
- [ ] Microservices or monolith?
- [ ] Service boundaries (by domain, capability?)
- [ ] Communication patterns (sync REST, async events?)

**Example Services**:
- Points Calculation Service
- Balance Service
- Redemption Service
- Campaign Configuration Service

---

#### Event-Driven Architecture (if applicable)
- [ ] Event streaming platform (Kafka, Kinesis, Event Hubs)?
- [ ] Key topics/streams
- [ ] Producer/consumer patterns
- [ ] Exactly-once vs. at-least-once semantics?

**Example Topics**:
- `bookings.completed`
- `points.calculated`
- `balances.updated`

---

### Security Architecture

#### Authentication & Authorization
- [ ] Auth mechanism (OAuth 2.0, JWT, API keys)?
- [ ] Role-based access control (RBAC)?
- [ ] Token expiration strategy?

---

#### Data Protection
- [ ] Encryption at rest (AES-256)?
- [ ] Encryption in transit (TLS 1.3)?
- [ ] Field-level encryption (for PII)?
- [ ] Key management (KMS, Vault)?

---

#### Compliance
- [ ] GDPR: Right to erasure, data minimization
- [ ] PCI-DSS: No card data in system, tokenization
- [ ] Audit logging (who did what, when)
- [ ] Data retention policy (7+ years for fintech?)

---

### Reliability & Availability

#### Fault Tolerance
- [ ] Single points of failure identified?
- [ ] Database replication (master-slave, multi-master)?
- [ ] Service redundancy (multiple instances)?
- [ ] Health checks and circuit breakers?

---

#### Disaster Recovery
- [ ] Multi-region deployment?
- [ ] Backup strategy (frequency, retention)?
- [ ] Failover mechanism (automated or manual)?
- [ ] Data recovery process (from backups, event replay)?

---

#### Monitoring & Observability
- [ ] Metrics (Prometheus, CloudWatch)?
- [ ] Logging (ELK, Splunk)?
- [ ] Distributed tracing (Jaeger, Zipkin)?
- [ ] Alerting (PagerDuty, Opsgenie)?

**Key Metrics**:
- Request latency (P50, P95, P99)
- Error rate (4xx, 5xx)
- Throughput (requests/sec)
- Database query time
- Cache hit rate

---

### Performance & Scale

#### Horizontal Scaling
- [ ] Stateless services (can add replicas)?
- [ ] Load balancing (round-robin, least connections)?
- [ ] Database sharding (by user ID, region)?

---

#### Vertical Scaling
- [ ] When is vertical scaling appropriate?
- [ ] Instance size limits?
- [ ] Cost implications?

---

#### CDN & Edge Caching
- [ ] Static assets (images, JS, CSS)?
- [ ] API responses cacheable?
- [ ] Geographic distribution?

---

## Phase 4: Trade-Offs Discussion (Ongoing)

### Key Trade-Offs to Address

#### Consistency vs. Availability (CAP Theorem)
- [ ] Strong consistency or eventual consistency?
- [ ] What's the business impact of stale data?

**Example**: Balance queries can be eventually consistent, but redemptions must be strongly consistent (avoid overspending).

---

#### Latency vs. Throughput
- [ ] Optimize for low latency or high throughput?
- [ ] Batch processing acceptable?

---

#### Cost vs. Performance
- [ ] Managed services (higher cost, lower ops) vs. self-managed?
- [ ] Reserved instances vs. on-demand?
- [ ] Storage tiers (hot vs. cold)?

---

#### Complexity vs. Maintainability
- [ ] Microservices (flexible, complex) vs. monolith (simple, rigid)?
- [ ] Event-driven (decoupled, harder to debug) vs. synchronous?

---

## Phase 5: Deep Dive Topics (Be Ready)

### Common Deep Dive Questions

#### Idempotency
- [ ] How do you prevent duplicate processing?
- [ ] Idempotency keys in APIs?
- [ ] Database constraints (unique IDs)?

---

#### Schema Evolution
- [ ] How do you change event schemas without breaking consumers?
- [ ] Schema Registry?
- [ ] Backward/forward compatibility?

---

#### Regional Failover
- [ ] What happens if primary region goes down?
- [ ] DNS failover strategy?
- [ ] Cache warming after failover?

---

#### Rate Limiting
- [ ] How do you prevent abuse?
- [ ] Token bucket algorithm?
- [ ] Per-user or global limits?

---

#### Data Migration
- [ ] How would you migrate from old system to new?
- [ ] Dual-write strategy?
- [ ] Strangler fig pattern?

---

## Phase 6: Wrap-Up (45-60 min)

### Summary
- [ ] Recap key architectural decisions
- [ ] Highlight trade-offs made
- [ ] Acknowledge areas for improvement (given more time)

**Example Summary**:
> "We designed an event-driven loyalty platform using Kafka for decoupling, SQL for transactional consistency, and Redis for fast balance queries. We chose eventual consistency for points calculation (acceptable 1-second delay) but strong consistency for redemptions (prevent overspending). Multi-region active-passive deployment ensures 99.95% availability with <5 min RTO."

---

### Your Questions
- [ ] What tech stack does Booking.com use for similar systems?
- [ ] What are the biggest scalability challenges you face?
- [ ] How does the team handle schema evolution in production?
- [ ] What's the on-call culture like?

---

## Pre-Interview Setup

### Tools
- [ ] Diagramming tool ready (Excalidraw, Draw.io, Miro)
- [ ] Screen sharing tested
- [ ] Backup pen and paper (in case of tech issues)

### Environment
- [ ] Quiet location
- [ ] Good internet connection
- [ ] Camera and mic tested
- [ ] Water nearby (stay hydrated!)

### Mindset
- [ ] Treat it as a conversation, not interrogation
- [ ] Think out loud (let them see your thought process)
- [ ] Ask clarifying questions (shows you're thoughtful)
- [ ] Be honest if you don't know something
- [ ] Relate to your experience (loyalty platform examples)

---

## Post-Interview Reflection

### What Went Well
- _______________________________________________
- _______________________________________________

### What Could Improve
- _______________________________________________
- _______________________________________________

### Follow-Up Items
- _______________________________________________
- _______________________________________________

---

**Remember**: Interviewers want to see:
1. **How you think** (not just the final answer)
2. **Communication skills** (can you explain complex ideas clearly?)
3. **Trade-off awareness** (there's no perfect solution)
4. **Fintech domain knowledge** (security, compliance, scale)
5. **Pragmatism** (balance ideal vs. practical)

Good luck! 🚀
