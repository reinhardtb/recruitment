# Loyalty Platform - Data Flow Diagram

```mermaid
graph TB
    subgraph "On-Premises"
        M[Gaming Monolith<br/>.NET/SQL Server]
        OB[(Outbox Table<br/>SQL Server)]
        SQLCLR[SQLCLR Publisher]
        KO[On-Prem Kafka<br/>5-node cluster]
    end
    
    subgraph "Confluent Cloud"
        KC[Kafka Connect]
        T1[wager.events topic]
        T2[loyalty.eligible topic]
        T3[loyalty.calculated topic]
        T4[loyalty.balances topic]
    end
    
    subgraph "Kafka Streams Processing - Azure AKS"
        S1["Filter Service<br/>(Kafka Streams - Java)"]
        S2["Calculator Service<br/>(Kafka Streams - Java)"]
        S3["Aggregator Service<br/>(Kafka Streams - Java)"]
    end
    
    subgraph "Storage - Azure"
        CFG[(Config Store<br/>Cosmos DB)]
        SQL[(Points Balance<br/>SQL Server<br/>7+ years)]
        DW[(Data Warehouse<br/>Long-term Analytics<br/>7+ years)]
    end
    
    subgraph "APIs"
        API[Loyalty API]
    end
    
    subgraph "Clients"
        OP[Operator Portal]
        PA[Player App]
    end
    
    M -->|"1. Transaction:<br/>Wager + Outbox"| OB
    OB -->|"2. SQLCLR reads"| SQLCLR
    SQLCLR -->|"3. Publish Event"| KO
    KO -->|"4. Replicate<br/>(7-day retention)"| KC
    KC -->|"5. Write"| T1
    
    T1 -->|"6. Consume"| S1
    CFG -.->|"Read Rules"| S1
    S1 -->|"7. Filtered Event<br/>(if eligible)"| T2
    
    T2 -->|"8. Consume"| S2
    CFG -.->|"Read Multipliers"| S2
    S2 -->|"9. Calculated Points<br/>{playerId, points}"| T3
    
    T3 -->|"10. Consume"| S3
    S3 -->|"11. Aggregated Balance<br/>{playerId, total}"| T4
    S3 -->|"12. Persist<br/>(idempotent)"| SQL
    
    T1 -->|"13. ETL Pipeline<br/>(all events)"| DW
    T4 -->|"14. Balance Snapshots"| DW
    
    SQL -.->|"15. Query<br/>(real-time)"| API
    DW -.->|"16. Analytics<br/>(historical)"| API
    API -->|"17. Balance"| OP
    API -->|"17. Balance"| PA
    
    style M fill:#1a1a1a,stroke:#fff,stroke-width:3px,color:#fff
    style OB fill:#0066cc,stroke:#fff,stroke-width:3px,color:#fff
    style SQLCLR fill:#00cc66,stroke:#000,stroke-width:3px,color:#000
    style KO fill:#ff6600,stroke:#000,stroke-width:3px,color:#000
    style T1 fill:#ffcc00,stroke:#000,stroke-width:3px,color:#000
    style T2 fill:#ffcc00,stroke:#000,stroke-width:3px,color:#000
    style T3 fill:#ffcc00,stroke:#000,stroke-width:3px,color:#000
    style T4 fill:#ffcc00,stroke:#000,stroke-width:3px,color:#000
    style S1 fill:#cc00cc,stroke:#fff,stroke-width:3px,color:#fff
    style S2 fill:#cc00cc,stroke:#fff,stroke-width:3px,color:#fff
    style S3 fill:#cc00cc,stroke:#fff,stroke-width:3px,color:#fff
    style CFG fill:#00cccc,stroke:#000,stroke-width:3px,color:#000
    style SQL fill:#cc0000,stroke:#fff,stroke-width:3px,color:#fff
    style DW fill:#990000,stroke:#fff,stroke-width:3px,color:#fff
    style API fill:#006600,stroke:#fff,stroke-width:3px,color:#fff
```

**Data Flow Legend** (Colorblind-friendly):
- **Black (#1a1a1a)**: Gaming monolith (.NET/SQL Server)
- **Blue (#0066cc)**: Outbox table (SQL Server)
- **Green (#00cc66)**: SQLCLR publisher
- **Orange (#ff6600)**: Kafka infrastructure (7-day retention)
- **Yellow (#ffcc00)**: Kafka topics
- **Purple (#cc00cc)**: Stream processing services (Java/Kafka Streams)
- **Cyan (#00cccc)**: Configuration (Cosmos DB)
- **Red (#cc0000)**: SQL Server persistence (7+ years)
- **Dark Red (#990000)**: Data Warehouse (7+ years)
- **Dark Green (#006600)**: Loyalty API

**Key Architectural Points**:
1. **Kafka retention: 7 days** - Optimized for stream processing, not long-term storage
2. **SQL Server materialized views** - Real-time OLTP queries, 7+ years compliance retention
3. **Data Warehouse** - Long-term analytics, regulatory compliance, business intelligence
4. **ETL from Kafka** - All events archived for audit trail and historical analysis

## Event Schemas

### 1. Wager Event (wager.events)
```json
{
  "wagerId": "uuid-123",
  "playerId": "player-456",
  "operatorId": "operator-789",
  "gameId": "slots-ultimate",
  "gameCategory": "slots",
  "wagerAmount": 100.00,
  "currency": "USD",
  "playerSegment": "VIP",
  "jurisdiction": "Malta",
  "timestamp": "2024-11-11T10:30:00Z"
}
```

### 2. Eligible Event (loyalty.eligible)
```json
{
  "wagerId": "uuid-123",
  "playerId": "player-456",
  "operatorId": "operator-789",
  "wagerAmount": 100.00,
  "gameCategory": "slots",
  "timestamp": "2024-11-11T10:30:00Z",
  "eligibilityRuleVersion": "v2.3"
}
```

### 3. Calculated Points (loyalty.calculated)
```json
{
  "wagerId": "uuid-123",
  "playerId": "player-456",
  "operatorId": "operator-789",
  "pointsEarned": 10.0,
  "wagerAmount": 100.00,
  "multiplier": 0.1,
  "timestamp": "2024-11-11T10:30:01Z"
}
```

### 4. Balance Update (loyalty.balances)
```json
{
  "playerId": "player-456",
  "operatorId": "operator-789",
  "pointsBalance": 460.0,
  "lastWagerId": "uuid-123",
  "lastUpdated": "2024-11-11T10:30:02Z",
  "version": 123
}
```

## Processing Stages Detail

### Stage 1: Eligibility Filter
**Input**: wager.events  
**Output**: loyalty.eligible

**Logic**:
```kotlin
wagerEventsStream
  .filter { event -> 
    val rules = operatorConfig[event.operatorId]
    rules.eligibleGames.contains(event.gameCategory) &&
    event.wagerAmount >= rules.minWagerAmount &&
    rules.eligibleSegments.contains(event.playerSegment)
  }
  .to("loyalty.eligible")
```

### Stage 2: Points Calculation
**Input**: loyalty.eligible  
**Output**: loyalty.calculated

**Logic**:
```kotlin
eligibleEventsStream
  .mapValues { event ->
    val config = operatorConfig[event.operatorId]
    val multiplier = config.getMultiplier(
      gameCategory = event.gameCategory,
      playerTier = getPlayerTier(event.playerId)
    )
    
    PointsCalculated(
      playerId = event.playerId,
      pointsEarned = event.wagerAmount * multiplier,
      multiplier = multiplier
    )
  }
  .to("loyalty.calculated")
```

### Stage 3: Balance Aggregation
**Input**: loyalty.calculated  
**Output**: loyalty.balances + SQL persist

**Logic**:
```kotlin
calculatedPointsStream
  .groupByKey { it.playerId }
  .aggregate(
    initializer = { PlayerBalance(points = 0.0) },
    aggregator = { playerId, newPoints, currentBalance ->
      currentBalance.copy(
        points = currentBalance.points + newPoints.pointsEarned,
        lastUpdated = newPoints.timestamp,
        version = currentBalance.version + 1
      )
    }
  )
  .toStream()
  .foreach { playerId, balance ->
    persistToSQL(playerId, balance) // Idempotent write
  }
```

## Outbox Pattern Implementation

**Problem**: Ensure transactional consistency between wager processing and event publishing

**Solution**: Outbox pattern with SQLCLR

```sql
-- Wager processing stored procedure
CREATE PROCEDURE ProcessWager
  @PlayerId UNIQUEIDENTIFIER,
  @WagerAmount DECIMAL(18,2),
  @GameId VARCHAR(50)
AS
BEGIN TRANSACTION
  -- 1. Process wager
  INSERT INTO Wagers (PlayerId, Amount, GameId, Timestamp)
  VALUES (@PlayerId, @WagerAmount, @GameId, GETUTCDATE())
  
  -- 2. Write to outbox (same transaction)
  INSERT INTO WagerOutbox (WagerId, EventData, Published)
  VALUES (
    SCOPE_IDENTITY(),
    JSON_QUERY('{"playerId":"' + CAST(@PlayerId AS VARCHAR(36)) + '",...}'),
    0  -- Not yet published
  )
COMMIT TRANSACTION

-- SQLCLR stored procedure (runs asynchronously)
CREATE PROCEDURE PublishOutboxEvents
AS EXTERNAL NAME KafkaPublisher.OutboxPublisher.Publish
```

**Benefits**:
- Transactional consistency (wager + event in same TX)
- At-least-once delivery guarantee
- No event loss (outbox persisted before Kafka publish)
- Asynchronous publishing doesn't block wager processing

## Idempotency Strategy

**Problem**: At-least-once delivery means duplicate events possible

**Solution**: Track processed wagerIds

```sql
-- Before processing
SELECT wagerId FROM LoyaltyTransaction WHERE wagerId = ?

-- If not exists:
BEGIN TRANSACTION
  INSERT INTO LoyaltyTransaction (
    wagerId, playerId, points, eventOffset, timestamp
  ) VALUES (?, ?, ?, ?, ?)
  
  UPDATE PlayerBalance 
  SET points = points + ?,
      lastUpdated = ?,
      version = version + 1
  WHERE playerId = ?
COMMIT
```

## Scalability Patterns

### Partitioning Strategy
- **Partition Key**: `playerId`
- **Partition Count**: 100
- **Rationale**: 
  - Ensures all events for same player processed in order
  - Enables horizontal scaling (100 parallel consumers)
  - State locality (player balance in same partition)

### Consumer Scaling
```
Kafka Streams Instances: 3-10 (auto-scale based on lag)
Partitions per Instance: 10-33
State Store: RocksDB (local SSD)
Rebalancing: Automatic on instance add/remove
```

### Performance Characteristics
- **Throughput**: 69M events/hour = 19,166 events/second
- **Per Partition**: ~192 events/second/partition
- **Processing Time**: <10ms per event (filter + calculate + aggregate)
- **End-to-End Latency**: <2 seconds (event publish → SQL persist)

### Data Retention Strategy

**Why Short Kafka Retention (7 days)?**
- Kafka optimized for streaming, not long-term storage
- Reduces storage costs and operational complexity
- Event stream used for processing, not historical queries

**Materialized Views (SQL Server)**:
- Real-time query layer (points balances)
- 7+ years retention for compliance
- Optimized for OLTP queries

**Data Warehouse**:
- Long-term event storage and analytics
- Batch ETL from Kafka topics
- Historical analysis, business intelligence
- Regulatory compliance archives

**Multi-Region Data Centers**:
- Malta (primary)
- Canada (North America)
- Isle of Man
- Additional regions: Europe, Asia, Oceania
- Each with local Kafka cluster (7-day retention)
