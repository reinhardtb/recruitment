# Hybrid Cloud Leaderboard System - System Context

**Project**: Real-Time Tournament Leaderboard Platform  
**Organization**: Derivco  
**Domain**: Gaming/Fintech  
**Timeline**: [Add dates]  
**Role**: Lead Architect

---

## Business Context

### The Problem

**Business Requirement**: Gaming operators need to run competitive tournaments to increase player engagement and retention.

**Technical Challenges**:
1. **Real-time Rankings**: Players expect to see live leaderboard updates during tournaments
2. **Dynamic Tournament Rules**: Each operator has different eligibility criteria (games, bet sizes, player segments)
3. **Scale**: 10,000+ concurrent players per tournament
4. **Multi-Operator**: Platform must serve multiple casino operators with isolated data
5. **Configuration Complexity**: Tournament setup should be operator self-service, not engineering effort
6. **Global Distribution**: Players worldwide need low-latency access

### Business Impact
- **Player Engagement**: Tournaments drive 30-40% increase in wagering during events
- **Operator Differentiation**: Custom tournaments = competitive advantage
- **Time to Market**: Operators want to launch tournaments in hours, not weeks
- **Revenue**: Tournament-driven engagement directly impacts operator revenue

### Stakeholders
- **Casino Operators**: Configure and manage tournaments
- **Players**: Participate in tournaments, view real-time rankings
- **Compliance Teams**: Ensure fair play, audit tournament results
- **Marketing Teams**: Use tournaments for promotional campaigns
- **Platform Engineering**: Operate and scale the infrastructure

---

## Solution Approach

### Strategic Vision: **Turnkey Tournament Platform**

Provide operators with self-service tournament configuration without requiring engineering involvement.

### Architecture Transformation

**Before (Manual Engineering Effort)**:
```
Operator Request → Engineering Backlog → Code Changes 
→ Deployment → Tournament Live (weeks)
```

**After (Self-Service Platform)**:
```
Operator Portal → ksqlDB Dynamic Query → Tournament Live (minutes)
```

## Event Publishing Strategy

The gaming monolith publishes **best-effort wager events** to Kafka. The leaderboard system determines position changes:

- **Gaming Monolith**: Publishes all wager events as fire-and-forget (best-effort delivery)
- **Separation of Concerns**: Gaming system has no leaderboard awareness (clean architecture)
- **Leaderboard Aggregator**: Subscribes to wager events, calculates positions, maintains state
- **Position Change Detection**: Aggregator publishes position change events only when player rank changes
- **WebSocket Optimization**: 90% fewer WebSocket messages (position changes vs. all wagers)
- **Trade-off**: Leaderboard aggregator must maintain state to detect rank changes

---

## High-Level Architecture

### Hybrid Cloud Event-Driven Platform

```
┌─────────────────────────────────────────────────────────────────┐
│                        ON-PREMISES                               │
│  ┌────────────┐                    ┌──────────────┐            │
│  │   Gaming   │──best-effort──────→│ Kafka Cluster│            │
│  │  Systems   │    wager events    │  (on-prem)   │            │
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
│  │  • wager.events (all wagers)                     │          │
│  │  • tournament.eligible.{tournament-id}           │          │
│  │  • leaderboard.updates.{tournament-id}           │          │
│  └─────────────────┬────────────────────────────────┘          │
│                    ↓                                             │
│  ┌──────────────────────────────────────────────────┐          │
│  │            ksqlDB (Dynamic Queries)              │          │
│  │  CREATE STREAM eligible_wagers AS                │          │
│  │  SELECT * FROM wager_events                      │          │
│  │  WHERE game_id IN ('slot-1', 'slot-2')           │          │
│  │    AND wager_amount >= 10.00                     │          │
│  │    AND player_segment = 'VIP';                   │          │
│  └─────────────────┬────────────────────────────────┘          │
│                    ↓                                             │
│  ┌──────────────────────────────────────────────────┐          │
│  │      Azure Durable Functions (Initial)           │          │
│  │      → Kubernetes (After Migration)              │          │
│  │  • Aggregate player scores                       │          │
│  │  • Rank players                                  │          │
│  │  • Publish leaderboard updates                   │          │
│  └─────────────────┬────────────────────────────────┘          │
│                    ↓                                             │
│  ┌──────────────────────────────────────────────────┐          │
│  │    Leaderboard State Storage                     │          │
│  │  • Azure Table Storage (fast reads)              │          │
│  │  • Redis Cache (sub-10ms queries)                │          │
│  └──────────────────────────────────────────────────┘          │
│                                                                  │
└─────────────────────────────────────┬───────────────────────────┘
                                      ↓
                    ┌──────────────────────────────────┐
                    │    Azure Front Door              │
                    │  • Global load balancing         │
                    │  • Regional failover             │
                    │  • CDN for static assets         │
                    └─────────────┬────────────────────┘
                                  ↓
                    ┌──────────────────────────────────┐
                    │  Player UI / Operator Portals    │
                    │  • Real-time leaderboard view    │
                    │  • Tournament configuration      │
                    │  • Analytics dashboard           │
                    └──────────────────────────────────┘
```

---

## Core Components

### 1. Event Source: Position Change Events (from Gaming Monolith)

**Event Publishing**: Gaming monolith (SQL Server + SQLCLR) publishes position change events when a player's rank changes in a tournament.

**Why Position Changes (Not Every Wager)?**
- Only publish when a player's leaderboard position changes
- Dramatically reduces event volume
- Leaderboard updates are what matters to players
- Same SQLCLR outbox pattern as Loyalty Platform

**Event Schema** (simplified):
```json
{
  "positionChangeId": "uuid",
  "playerId": "uuid",
  "tournamentId": "summer-slots-2024",
  "operatorId": "operator-123",
  "oldPosition": 15,
  "newPosition": 12,
  "score": 125000,
  "timestamp": "2024-11-11T10:30:00Z",
  "gameCategory": "slots",
  "playerSegment": "VIP",
  "jurisdiction": "Malta"
}
```

**Event Volume**:
- Peak load: 10,000+ concurrent players
- Event rate: ~5-10 wagers/second/tournament (highly variable)
- Total: Multiple tournaments running concurrently

### 2. Kafka Connect: On-Prem to Cloud Bridge

**Technology**: Kafka Connect (Confluent)

**Purpose**: 
- Replicate position change events from on-prem Kafka to Confluent Cloud
- Bridge network boundary (on-prem → cloud)
- Managed service (reduced operational burden)

### 3. ksqlDB: Dynamic Tournament Routing

**Technology**: ksqlDB (Confluent)

**Purpose**: Route position change events to active tournaments based on operator-configured rules

**Example Tournament Configuration**:
```sql
-- Tournament: "Weekend Slots Challenge"
CREATE STREAM weekend_slots_tournament AS
SELECT 
  playerId,
  positionChangeId,
  score,
  newPosition,
  timestamp
FROM position_change_events
WHERE 
  tournamentId = 'weekend-slots-2024'
  AND gameCategory = 'slots'
  AND playerSegment IN ('VIP', 'Gold')
  AND CAST(timestamp AS DATE) BETWEEN '2024-11-15' AND '2024-11-17';
```

**Why ksqlDB**:
- ✅ **SQL Familiarity**: Operators understand SQL, not code
- ✅ **Real-time**: Queries execute on live event streams
- ✅ **Dynamic**: Create/modify tournament routing without deployment
- ✅ **Scalable**: ksqlDB clusters scale horizontally

**Operator Configuration Flow**:
1. Operator uses UI to configure tournament (dates, games, eligibility)
2. UI generates ksqlDB query from configuration
3. ksqlDB query deployed to cluster
4. Eligible wagers automatically filtered in real-time

### 4. Leaderboard Processing (Evolution Story)

#### Initial: Azure Durable Functions

**Technology**: Azure Durable Functions (Serverless)

**Why Initially Chosen**:
- ✅ Serverless (no infrastructure management)
- ✅ Built-in orchestration (stateful workflows)
- ✅ Automatic scaling (pay per execution)
- ✅ Retry logic (resilience)

**Processing Logic**:
```csharp
// Simplified orchestration
[FunctionName("UpdateLeaderboard")]
public static async Task Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var tournamentId = context.GetInput<string>();
    
    // Fan-out: Process eligible wagers
    var wagers = await context.CallActivityAsync<List<Wager>>(
        "FetchEligibleWagers", tournamentId);
    
    // Aggregate player scores
    var scores = await context.CallActivityAsync<Dictionary<string, decimal>>(
        "AggregateScores", wagers);
    
    // Rank players
    var leaderboard = await context.CallActivityAsync<List<LeaderboardEntry>>(
        "RankPlayers", scores);
    
    // Publish updates
    await context.CallActivityAsync("PublishLeaderboard", leaderboard);
}
```

**Challenges Encountered**:
- ❌ **Scaling Unpredictability**: Under high load, instances scaled erratically
- ❌ **Performance Bottlenecks**: Orchestration overhead added latency
- ❌ **Cold Starts**: Tournament spikes hit cold start delays
- ❌ **Debugging Complexity**: Distributed traces across functions difficult
- ❌ **Cost**: High execution count = high cost at scale

#### Migration: Kubernetes

**Technology**: Kubernetes (Azure AKS)

**Why Migrated**:
- ✅ **Predictable Scaling**: HPA (Horizontal Pod Autoscaler) with CPU/memory targets
- ✅ **Fine-Grained Control**: Resource limits, concurrency, thread pools
- ✅ **Performance Tuning**: Optimize processing without serverless abstractions
- ✅ **Pre-Warming**: Pods ready before tournament starts (no cold starts)
- ✅ **Cost Efficiency**: Reserved instances cheaper than high-execution-count functions

**Processing Architecture (Kubernetes)**:
```
Kafka Consumer (Deployment) 
→ Score Aggregator (StatefulSet with local state)
→ Ranker (Deployment)
→ State Publisher (Deployment)
→ Redis + Table Storage
```

**Key Improvements**:
- Sub-second leaderboard updates (previously 2-5 seconds)
- 99.99% availability during peak tournaments
- 60% cost reduction vs Durable Functions
- Predictable scaling behavior

**Lessons Learned**:
- Serverless is excellent for bursty, low-concurrency workloads
- High-throughput, latency-sensitive processing benefits from closer control
- Abstraction trade-offs: Ease of use vs performance optimization

### 5. Leaderboard State Storage

**Dual Storage Strategy**:

#### Azure Table Storage
- **Purpose**: Durable leaderboard snapshots
- **Write Pattern**: Periodic snapshots (every 10 seconds)
- **Read Pattern**: Operator analytics, historical queries
- **Partition Key**: `tournamentId`
- **Row Key**: `playerId`

#### Redis Cache
- **Purpose**: Ultra-fast leaderboard queries
- **Write Pattern**: Real-time updates on rank changes
- **Read Pattern**: Player UI, live leaderboard display
- **Data Structure**: Sorted Set (ZADD for scores, ZREVRANGE for rankings)
- **TTL**: Tournament duration + 24 hours

**Example Redis Operations**:
```redis
# Add/update player score
ZADD tournament:123:leaderboard 4500.00 player-456

# Get top 10 players
ZREVRANGE tournament:123:leaderboard 0 9 WITHSCORES

# Get player rank
ZREVRANK tournament:123:leaderboard player-456
```

### 6. Azure Front Door: Global Distribution

**Technology**: Azure Front Door (CDN + Load Balancer)

**Features Used**:
- **Global Load Balancing**: Route players to nearest Azure region
- **Regional Failover**: Automatic failover if region unavailable
- **SSL Termination**: Centralized certificate management
- **Caching**: Static assets (UI, images) cached globally
- **WAF**: Web Application Firewall for security

**Availability Impact**:
- 99.99% uptime during tournaments (vs 99.5% without Front Door)
- <100ms latency for 95% of global players
- Transparent regional failover (no player impact)

---

## Key Design Decisions

### Decision 1: ksqlDB for Dynamic Tournament Configuration

**Context**: Tournaments have varying rules (games, bet sizes, player segments, dates).

**Alternatives Considered**:
1. **Hardcoded Logic**: Tournament rules in application code
2. **Rule Engine**: Custom DSL for tournament configuration
3. **ksqlDB**: SQL queries on event streams

**Choice**: ksqlDB

**Rationale**:
- Operators already know SQL (low learning curve)
- Real-time query execution (no batch processing)
- Confluent-managed (reduced operational burden)
- Scales with Kafka (horizontal scaling)

**Trade-offs**:
- ❌ ksqlDB learning curve for engineering team
- ❌ Query performance depends on stream partitioning
- ✅ Operator self-service (time to market: weeks → minutes)
- ✅ Unlimited tournament variations without code changes

### Decision 2: Azure Durable Functions → Kubernetes Migration

**Context**: Initial serverless choice hit scaling/performance limits.

**Initial Choice**: Azure Durable Functions
- **Pros**: Serverless, orchestration, auto-scaling
- **Cons**: Unpredictable scaling, latency overhead, high cost

**Migration Choice**: Kubernetes (Azure AKS)
- **Pros**: Predictable scaling, performance control, cost efficiency
- **Cons**: Operational complexity, need Kubernetes expertise

**Rationale for Migration**:
- Tournament load is predictable (scheduled events)
- Performance requirements tightened (sub-second updates)
- Cost at scale became prohibitive
- Team had Kubernetes expertise available

**Migration Strategy**:
- Dual-run: Functions and K8s side-by-side
- Shadow mode: K8s processed events, results compared to Functions
- Gradual cutover: Tournament by tournament
- Decommission: Functions retired after validation

**Outcome**:
- 50% latency reduction (5s → <1s leaderboard updates)
- 60% cost reduction
- 99.99% availability (up from 99.5%)

### Decision 3: Hybrid Cloud Architecture

**Context**: Gaming systems on-prem, want cloud elasticity.

**Alternatives Considered**:
1. **Full On-Prem**: Keep all processing on-prem
2. **Full Cloud Migration**: Move gaming systems to cloud
3. **Hybrid**: Bridge on-prem to cloud

**Choice**: Hybrid with Kafka Connect

**Rationale**:
- Gaming systems migration too risky (revenue-critical)
- Leaderboards less critical (graceful degradation acceptable)
- Cloud elasticity needed for tournament spikes
- Confluent Cloud reduces Kafka operational burden

**Trade-offs**:
- ❌ Network latency (on-prem → cloud: 20-50ms)
- ❌ Operational complexity (two environments)
- ✅ Cloud benefits without gaming system migration
- ✅ Isolated failure domain (leaderboard issues don't break gaming)

### Decision 4: Redis + Table Storage Dual Storage

**Context**: Need fast player queries + durable operator analytics.

**Alternatives Considered**:
1. **SQL Database Only**: Single source of truth
2. **Redis Only**: In-memory, fast but volatile
3. **Dual Storage**: Redis for speed, Table Storage for durability

**Choice**: Dual Storage

**Rationale**:
- Players need <10ms leaderboard queries (Redis)
- Operators need historical analytics (Table Storage)
- Different access patterns (real-time vs batch)
- Cost optimization (Redis expensive for large datasets)

**Trade-offs**:
- ❌ Dual-write complexity (consistency challenges)
- ❌ Increased operational burden
- ✅ Optimal performance for both use cases
- ✅ Cost-effective (Redis only for active tournaments)

---

## Operator Configuration

### Tournament Configuration Parameters

**Eligibility Rules**:
- Eligible games (by game ID or category)
- Minimum/maximum wager amounts
- Player segments (VIP, Gold, Silver, All)
- Jurisdiction restrictions
- Time period (start/end dates)

**Scoring Rules**:
- Points per wager (e.g., 1 point per $1 wagered)
- Multipliers (e.g., 2x points for specific games)
- Win-based scoring (points for winning wagers only)
- Frequency caps (max points per hour)

**Leaderboard Settings**:
- Update frequency (real-time, every 10s, every 1min)
- Prize distribution (top 10, top 100, percentile-based)
- Tie-breaking rules (timestamp, total wagers)

### Configuration UI

**Operator Portal**:
1. Select tournament template (pre-configured common setups)
2. Customize rules via form (translates to ksqlDB query)
3. Preview ksqlDB query (power users can edit directly)
4. Validate configuration (dry-run on sample data)
5. Schedule tournament (start/end times)
6. Activate (ksqlDB query deployed to cluster)

**Example UI → ksqlDB Translation**:
```
UI Form:
- Games: Slots (all)
- Min Bet: $10
- Players: VIP only
- Dates: Nov 15-17, 2024

Generated ksqlDB:
CREATE STREAM tournament_123 AS
SELECT playerId, wagerAmount, timestamp
FROM wager_events
WHERE gameCategory = 'slots'
  AND wagerAmount >= 10.00
  AND playerSegment = 'VIP'
  AND timestamp BETWEEN '2024-11-15T00:00:00Z' AND '2024-11-17T23:59:59Z';
```

---

## Scale & Performance

### Metrics Achieved

- **Concurrent Players**: 10,000+ per tournament
- **Leaderboard Update Latency**: <1 second (event → UI update)
- **Query Performance**: <10ms (95th percentile Redis queries)
- **Availability**: 99.99% during peak tournaments
- **Throughput**: ~5-10 wagers/second/tournament (highly variable)

### Pre-Warming Strategy

**Challenge**: Tournament starts create instant load spike.

**Solution**: Pre-Warm Infrastructure
- Kubernetes: Scale up pods 30 minutes before tournament start
- Redis: Pre-populate cache with tournament metadata
- ksqlDB: Validate queries pre-deployment (avoid runtime errors)

**Impact**:
- Zero cold-start delays at tournament launch
- Smooth player experience from minute one

### Deduplication & Idempotency

**Challenge**: At-least-once event delivery can cause duplicate scoring.

**Solution**: Idempotency Keys
- Each wager event has unique `wagerId`
- Track processed `wagerId` in state store
- Ignore duplicate events (don't double-count scores)

**Implementation**:
```
Wager Event → Check if wagerId processed
              ↓                      ↓
            Yes (skip)             No (process)
                                    ↓
                        Update score, store wagerId
```

---

## Monitoring & Observability

### Key Metrics

**Business Metrics**:
- Active tournaments count
- Players per tournament
- Average score per tournament
- Prize distribution tracking

**Technical Metrics**:
- Kafka consumer lag (alert if >10s behind)
- ksqlDB query performance (processing time per event)
- Leaderboard update latency (event → Redis write)
- Redis query latency (P50, P95, P99)
- Kubernetes pod CPU/memory utilization

### Tooling

- **Confluent Control Center**: Kafka and ksqlDB monitoring
- **Azure Monitor**: AKS cluster, Front Door, Table Storage
- **Redis Insights**: Cache hit rates, query patterns
- **Grafana Dashboards**: Unified view across stack
- **Application Insights**: Distributed tracing (post-migration)

---

## Lessons Learned

### What Went Well

✅ **ksqlDB for Dynamic Configuration**: 
- Operators could launch tournaments in minutes (previously weeks)
- Zero engineering backlog for tournament requests

✅ **Hybrid Cloud Strategy**: 
- Achieved cloud benefits without risky gaming system migration
- Leaderboards independently scalable

✅ **Redis Performance**: 
- <10ms queries enabled real-time player experience
- Simple data model (sorted sets) easy to operate

✅ **Azure Front Door**: 
- 99.99% availability vs 99.5% without it
- Global players had consistent low latency

### What I'd Do Differently

❌ **Start with Kubernetes, Not Serverless**: 
- Durable Functions migration cost 6 months of rework
- Performance requirements were clear from start
- **Lesson**: Serverless great for unpredictable workloads, not predictable high-throughput

❌ **Observability from Day 1**: 
- Initially underestimated distributed tracing complexity
- Debugging ksqlDB + Functions + Kafka without tracing was painful
- **Lesson**: Observability is architectural, invest early

❌ **Chaos Engineering for Network Failures**: 
- Kafka Connect network partition caused silent data loss (missed events)
- Discovered in production during tournament
- **Lesson**: Test network failures explicitly (on-prem → cloud bridge)

❌ **Load Testing with Realistic Patterns**: 
- Initial load tests used steady traffic (unrealistic)
- Tournament spikes are bursty (10x load in 30 seconds)
- Missed pre-warming opportunity initially
- **Lesson**: Load test with production traffic patterns

### Transferable Insights

🎯 **Abstraction Trade-offs**: 
- Serverless hides infrastructure but obscures performance bottlenecks
- When latency/throughput critical, closer control (Kubernetes) wins

🎯 **Dynamic Configuration as Competitive Advantage**: 
- ksqlDB self-service = operator time-to-market advantage
- Platform thinking: Enable users, don't block on engineering

🎯 **Hybrid Cloud for Gradual Migration**: 
- Don't need big-bang cloud migration
- Extract services incrementally while preserving existing systems

🎯 **Pre-Warming for Predictable Spikes**: 
- Tournaments are scheduled (not random)
- Pre-scale infrastructure before load arrives = smooth experience

🎯 **Dual Storage for Different Access Patterns**: 
- Redis (fast, ephemeral) + Table Storage (durable, queryable)
- Optimize for use case, not uniformity

---

## Relevance to Booking.com FinTech Foundations

### Alignment with Role Responsibilities

**1. Shape Architectural Direction**:
- Designed turnkey platform enabling operator self-service
- Established ksqlDB as organizational pattern for dynamic configuration

**2. Bridge Theory and Practice**:
- Applied event-driven architecture to real-world problem
- Migrated from serverless to Kubernetes based on performance data (not dogma)

**3. Simplify Technology Landscape**:
- ksqlDB abstracted complex event filtering logic
- Operators configure via SQL, not custom code

**4. Champion Best Practices**:
- Introduced ksqlDB to organization
- Documented migration rationale (serverless → K8s) for future projects

**5. Influence Stakeholders**:
- Convinced operators to accept <1s latency (initially wanted instant)
- Collaborated with platform engineering on Kafka operations

### Horizontal Service Parallels

**Leaderboard Platform = Foundational Service**:
- ✅ Turnkey for multiple operators (multi-tenant)
- ✅ Configurable via self-service UI (not custom engineering)
- ✅ Abstracts complexity (event processing, state management, global distribution)
- ✅ Platform thinking (enable operators, don't build per-operator)

**Similar to FinTech Foundations**:
- Payment processing platform for Booking.com product teams
- Self-service configuration by business units
- Abstracts regulatory/compliance complexity
- Enables rapid experimentation without platform team bottleneck

### Cloud Migration Story

**Pure Hybrid Cloud Migration**:
- ✅ On-prem → Confluent Cloud → Azure (exactly "cloud migration" requirement)
- ✅ Evolution story (Functions → Kubernetes shows learning)
- ✅ Pragmatic approach (hybrid, not big-bang)

**Lessons Applicable to Booking.com**:
- Gradual cloud adoption reduces risk
- Hybrid architectures bridge legacy and cloud-native
- Right-size compute (serverless vs containers) based on workload

---

*Last Updated: November 11, 2025*  
*Status: Ready for presentation preparation*
