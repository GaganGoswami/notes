# KYC Ecosystem AWS Rewrite: Architecture & Design Document

**Classification:** Engineering-Grade Principal Design Document  
**Version:** 1.0  
**Scope:** Full KYC Platform Rewrite — AWS-Native, Multi-Region Active-Active  
**Team Skillset:** Java  
**Scale:** 300k+ KYCs/day | 2500+ domain fields | 150 processing fields | Multi-region

---

## Table of Contents

1. DynamoDB vs Relational DB Decision Framework
2. Designing Queries Against DynamoDB for Scheduled Processing
3. Architecture Options for Daily + Event-Driven Processing
4. Full End-to-End Target Architecture
5. Business Rules Rewrite Strategy
6. Data Flow & Sequence Designs
7. Multi-Region Active-Active
8. Development & Operational Excellence
9. Migration Plan
10. Final Recommendation

---

## 1. DynamoDB vs Relational DB Decision Framework

### 1.1 Core Question

Should 300k+ daily KYC processing records be read directly from DynamoDB, or does the access pattern mandate a relational tier?

### 1.2 Evaluation Matrix

| Dimension | DynamoDB | Aurora PostgreSQL |
|---|---|---|
| Access pattern | Key-value, known PK/SK | Ad-hoc SQL, complex joins |
| Query flexibility | Low (GSIs only) | High (arbitrary WHERE clauses) |
| Operational cost | Pay-per-request or provisioned RCU/WCU | Instance + storage cost |
| Read at scale | O(1) per item, parallel reads | Limited by instance CPU/connections |
| Scan cost | Very high — avoid | Low via indexes |
| Consistency | Eventual (default) / Strong (same region) | ACID |
| Multi-region active-active | Native Global Tables | Aurora Global DB (write-once per region) |
| Schema evolution | Schemaless | DDL migration required |
| Large attribute sets | 400KB per item limit | Unlimited row width |
| Batch processing suitability | Poor for full-corpus scans | Excellent |
| Projection support | GSI projections (150 fields) | SELECT column list |
| Aggregations | None native | SQL GROUP BY, window functions |
| Conflict resolution | Last-writer-wins (configurable) | Single-writer per region |

### 1.3 KYC Domain Analysis

**Write path (UI + events):**  
- KYC record creation, status transitions, document uploads, workflow state changes  
- Driven by React UI → API Gateway → Lambda  
- Access pattern: known KYC ID → single record CRUD  
- **DynamoDB is ideal**

**Read path (daily processing of 300k records):**  
- Must enumerate all KYCs due for processing today  
- Apply 150-field business logic across each record  
- Depends on review date, risk tier, geography, entity type  
- Requires time-window queries, filtering, sorted traversal  
- **DynamoDB is problematic without careful design**

### 1.4 The 150-Field Processing Subset

With 2500+ fields in the domain but only ~150 needed for processing logic:

- A DynamoDB projection table holding only those 150 fields is viable
- Each item is well within the 400KB limit (150 fields × avg 200 bytes = ~30KB)
- A GSI keyed on `reviewDueDate` (partition) + `entityType` (sort) enables time-window reads
- Eliminates full-table scans

### 1.5 Hot Partition Analysis

**Naive design risk:**  
If `reviewDueDate` alone is the partition key, all 300k records processed on a given day share one partition → hot partition → throttling.

**Mitigation:**  
Use composite partition key: `PROCESS#YYYY-MM-DD#SHARD-{0..N}` where N = 10–20 shards.  
Distribute records across shards at write time (hash(kycId) % N).  
At read time, query all shards in parallel using parallel Query calls.

### 1.6 RCU Cost Model for 300k Daily Reads

Assumptions:
- Item size after 150-field projection = ~8KB (rounds up to 8 read units per strongly consistent read)
- 300,000 records × 8 KB = 2,400,000 KB total
- Eventual consistency = 0.5 RCU per 4KB → 300,000 × 2 = 600,000 RCUs
- At $0.00013/RCU on-demand → **$0.078/day** for reads alone

This is negligible in isolation. The issue is not cost — it is query design and fan-out complexity.

### 1.7 Decision: Single-Table DynamoDB vs Multi-Table vs Aurora Replica

| Option | Verdict |
|---|---|
| Single-table DynamoDB for all access | Reject — GSI fan-out complexity, hot partitions, no aggregations |
| DynamoDB for UI/workflow + Aurora read replica for batch | **Recommended** — cleanest separation |
| DynamoDB for UI/workflow + DynamoDB projection table for batch | **Also viable** if team avoids relational operational overhead |
| All-in Aurora | Reject — loses active-active, schema rigidity, loses EDA fit |

### 1.8 Final Recommendation

**Primary Store: DynamoDB (Global Tables)**
- All UI CRUD, workflow state, real-time KYC record management
- Single-table design with well-defined PK/SK pattern
- Global Tables for multi-region active-active

**Batch Processing Store: DynamoDB Projection Table (KYC-Processing)**
- Separate DynamoDB table holding only the 150 processing fields
- Keyed for time-window access: `PK = PROCESS#DATE#SHARD`, `SK = KYCIID`
- Populated via DynamoDB Streams → Lambda fan-out at write time
- 100% avoids scans on the main table

**Data Lake (S3/Parquet):**
- Authoritative source for analytics, historical queries, external API consumers
- Glue Catalog + Athena for SQL access over Parquet

**Aurora PostgreSQL (Optional but Recommended for Complex Rules):**
- If stored procedure logic requires JOIN-based rule evaluation
- Read-only replica fed by DynamoDB Streams → Aurora sync Lambda
- Isolated from production write path

---

## 2. Designing Queries Against DynamoDB for Scheduled Processing

### 2.1 Main Table: Single-Table Design

```
Table: KYC-MAIN

PK                        | SK                         | Attributes
--------------------------|----------------------------|----------------------------------
KYC#{kycId}               | METADATA                   | entityType, riskTier, status, createdAt, ...
KYC#{kycId}               | DOC#{docId}                | docType, s3Uri, uploadedAt, ...
KYC#{kycId}               | WORKFLOW#CURRENT           | stepName, assignee, dueDate, ...
KYC#{kycId}               | REVIEW#{reviewId}          | reviewDate, outcome, reviewerId, ...
ENTITY#{entityId}         | KYC#{kycId}                | kycId, createdAt, status, ...
```

### 2.2 Processing Projection Table

```
Table: KYC-PROCESSING

PK                               | SK          | Attributes (150 fields only)
---------------------------------|-------------|----------------------------
PROCESS#2024-01-15#SHARD-03      | KYC#abc123  | entityType, riskTier, jurisdiction, reviewDueDate, [148 more...]
PROCESS#2024-01-15#SHARD-07      | KYC#def456  | ...
```

**Shard assignment at write time:**
```java
int shard = Math.abs(kycId.hashCode()) % NUM_SHARDS; // NUM_SHARDS = 20
String pk = "PROCESS#" + reviewDueDate + "#SHARD-" + String.format("%02d", shard);
```

**Read at processing time (parallel fan-out):**
```java
List<CompletableFuture<List<KycRecord>>> futures = IntStream.range(0, NUM_SHARDS)
    .mapToObj(shard -> {
        String pk = "PROCESS#" + today + "#SHARD-" + String.format("%02d", shard);
        return dynamoDbClient.query(QueryRequest.builder()
            .tableName("KYC-PROCESSING")
            .keyConditionExpression("PK = :pk")
            .expressionAttributeValues(Map.of(":pk", AttributeValue.fromS(pk)))
            .build())
            .thenApply(this::mapToKycRecords);
    })
    .collect(Collectors.toList());

List<KycRecord> allRecords = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream().flatMap(f -> f.join().stream()).collect(Collectors.toList()))
    .join();
```

### 2.3 GSI Design

#### GSI-1: Review Due Date Index (on KYC-MAIN)
```
GSI Name: ReviewDue-EntityType-Index
GSI PK:   reviewDueDatePartition   (value: "2024-01-15#SHARD-03")
GSI SK:   entityType
Projection: INCLUDE [150 specific processing attributes]
```

#### GSI-2: Status + Risk Tier (on KYC-MAIN)
```
GSI Name: Status-RiskTier-Index
GSI PK:   status        (PENDING_REVIEW, IN_REVIEW, APPROVED, ...)
GSI SK:   riskTier      (HIGH, MEDIUM, LOW)
Projection: KEYS_ONLY   (for enumeration; then batch-get full item)
```

#### GSI-3: Periodic Review Index
```
GSI Name: PeriodicReview-Index
GSI PK:   nextReviewYearMonth    (value: "2024-01" — sparse: only set when review scheduled)
GSI SK:   nextReviewDate
Projection: INCLUDE [kycId, entityId, riskTier, jurisdiction]
```

**Sparse Index Pattern:**  
Only records with `nextReviewYearMonth` set appear in GSI-3. Records not due for review have no value → not indexed → free. This replaces a full-table scan with a targeted GSI query.

### 2.4 Pagination Strategy

DynamoDB returns 1MB max per Query call. For 300k records:

```java
QueryRequest request = QueryRequest.builder()
    .tableName("KYC-PROCESSING")
    .keyConditionExpression("PK = :pk")
    .expressionAttributeValues(Map.of(":pk", AttributeValue.fromS(pk)))
    .limit(500) // page size
    .build();

String lastEvaluatedKey = null;
do {
    if (lastEvaluatedKey != null) {
        request = request.toBuilder()
            .exclusiveStartKey(Map.of("PK", ..., "SK", ...)) // from lastEvaluatedKey
            .build();
    }
    QueryResponse response = dynamoClient.query(request);
    processPage(response.items());
    lastEvaluatedKey = response.lastEvaluatedKey().isEmpty() ? null : serialize(response.lastEvaluatedKey());
} while (lastEvaluatedKey != null);
```

**For Step Functions orchestration:** Store `lastEvaluatedKey` in Step Functions state between iterations. Each Lambda invocation processes one page, passes cursor forward.

### 2.5 Attribute Projection — Only 150 Fields

Never use `ProjectionExpression` with 150 attributes inline (expression length limit).  
Instead: store only those 150 fields in the KYC-PROCESSING table at write time. The table IS the projection.

For GSI projections, use `INCLUDE` with the critical subset (max 20 GSI projected attributes for cost efficiency), then batch-get full items from KYC-PROCESSING for the remaining fields.

### 2.6 DynamoDB Streams → Processing Table Sync

```
KYC-MAIN Write
     │
     ▼
DynamoDB Streams (NEW_AND_OLD_IMAGES)
     │
     ▼
Lambda: StreamProcessor
     │
     ├── Extract 150 processing fields from new image
     ├── Compute shard: hash(kycId) % 20
     ├── Compute PK: PROCESS#{reviewDueDate}#{shard}
     └── PutItem to KYC-PROCESSING (conditional: only if review-relevant)
```

### 2.7 TTL-Based Cleanup

Set `ttl` attribute on KYC-PROCESSING items = Unix timestamp of (reviewDueDate + 7 days).  
DynamoDB auto-expires processed records. No manual cleanup Lambda needed.

### 2.8 Summary: Best Pattern for 300k Daily KYC Processing

```
Step 1: Nightly scheduled trigger (EventBridge Scheduler, cron)
Step 2: Step Functions execution begins
Step 3: Fan-out — 20 parallel Lambda invocations, one per shard
Step 4: Each Lambda queries KYC-PROCESSING table: PK = PROCESS#{today}#{shard}
Step 5: Paginate through results (500 items/page)
Step 6: Process each KYC record using domain logic
Step 7: Write results back to KYC-MAIN + publish events to EventBridge
Step 8: Step Functions aggregates completion; publishes summary metric
```

---

## 3. Architecture Options for Daily + Event-Driven Processing

### Option A: Read from DynamoDB Directly

**What:** Step Functions → Lambda → Query KYC-PROCESSING (projection table) → process → write results

**Pros:**
- No ETL pipeline required
- Freshest data (milliseconds old)
- Simplest operational model

**Cons:**
- Sharding complexity for 300k records
- RCU consumption (600k RCUs/day)
- Lambda 15-min timeout — must paginate via Step Functions

**Cost Model:**  
~600k RCUs × $0.00013 = **$0.08/day** reads. Lambda: 300k × 2s × 512MB = **~$3/day**. Total: **~$3.08/day**.

**Performance:** 300k records in 20 parallel streams × ~15k records each = ~30 min with 500ms/record processing.

**Multi-region:** KYC-PROCESSING is a Global Table — each region runs independently against its local replica.

**Suitability for 300k/day:** ✅ Viable with projection table + sharding design.

**Verdict: Recommended as primary pattern.**

---

### Option B: Extract DynamoDB → S3 (Columnar) → Process

**What:** DynamoDB Streams → Lambda → write Parquet to S3 → EMR/Glue job processes S3 files → results written back

**Pros:**
- SQL-friendly processing via Spark/Glue
- Unlimited parallelism in Spark
- No RCU consumption during processing (reads from S3)
- Natural audit trail in S3

**Cons:**
- ETL lag (minutes to hours)
- Complex pipeline (Stream → Lambda → S3 → Glue → results → DynamoDB)
- Glue cluster startup time (3–5 min)
- Team must learn Spark (Python/Scala — conflicts with Java-first mandate)
- Higher operational complexity

**Cost Model:**  
Glue DPU: 10 DPUs × 30 min = **$0.44/run**. S3 storage: negligible. Lambda Stream processor: **~$0.10/day**.

**Performance:** Spark parallelism handles 300k records in <10 min once cluster starts. Total: ~15 min.

**Multi-region:** S3 cross-region replication adds latency. Each region maintains own Glue jobs.

**Suitability:** ✅ Good for complex analytics processing, but over-engineered for 300k/day with Java lambdas.

**Verdict: Use for Data Lake analytics; not recommended as primary processing path.**

---

### Option C: Aurora/Postgres Replica for Batch Reads

**What:** DynamoDB Streams → Lambda → sync to Aurora PostgreSQL → SQL-based batch processing → results back to DynamoDB

**Pros:**
- Full SQL: JOINs, aggregations, window functions
- Stored procedure logic maps directly
- Familiar to SQL-experienced teams
- No scan/RCU costs (Aurora uses storage I/O billing)

**Cons:**
- Dual-write complexity (DynamoDB + Aurora sync)
- Sync lag (seconds to minutes)
- Aurora instance cost (~$0.10/hr for r5.large = **$73/month minimum**)
- Active-active is complex (Aurora Global DB has one primary write region)
- Contradicts serverless-first mandate

**Cost Model:**  
Aurora r5.large: **$73/month**. Storage: **$0.10/GB/month**. Total: **~$100/month baseline**.

**Multi-region:** Aurora Global DB adds $0.20/million replicated I/Os. Not truly active-active for writes.

**Suitability:** ✅ Excellent for rule complexity; ❌ conflicts with multi-region active-active requirement.

**Verdict: Use as optional read-replica for complex rule evaluation; not as primary store.**

---

### Option D: Event-Sourced KYC Pipeline with Materialized Views

**What:** All KYC mutations are events. Events stream to Kinesis. Consumers materialize views into DynamoDB or Aurora. Processing reads from materialized views.

**Pros:**
- Complete audit trail
- Perfect for perpetual KYC (event replay)
- Decoupled consumers
- Time-travel queries

**Cons:**
- High architectural complexity
- Event schema versioning is hard
- Eventual consistency in views
- Team learning curve

**Cost Model:**  
Kinesis: $0.015/shard-hr. With 300k events/day → 1 shard → **$10.80/month**. Lambda consumers add ~$5/month. Total: **~$16/month overhead**.

**Multi-region:** Kinesis cross-region replication via EventBridge Pipes or custom replicator.

**Suitability:** ✅ Best for long-term platform; high upfront cost.

**Verdict: Adopt event sourcing as the foundational pattern, but phase in over time.**

---

### Option E: Dual-Write into DynamoDB + S3

**What:** Every KYC write goes to DynamoDB (UI/workflow) AND S3/Parquet (analytics/processing). Outbox pattern or Streams fan-out.

**Pros:**
- Each store optimized for its consumers
- Processing reads from S3 (no DynamoDB scan)
- Data Lake naturally maintained
- Decoupled write and read paths

**Cons:**
- Dual-write consistency: risk of partial failures
- Must use outbox/Streams pattern (not synchronous dual-write)
- S3 eventual consistency for processing (seconds)

**Cost Model:**  
S3 storage: 300k records × 8KB/day × 365 days = ~876GB/yr → **~$20/yr**. Lambda fan-out: **$1–2/month**.

**Multi-region:** S3 cross-region replication (SRR/CRR) for Data Lake sync.

**Suitability:** ✅ Excellent — this IS the recommended pattern when combined with DynamoDB Streams.

**Verdict: Adopt as the write-path pattern. DynamoDB Streams → Lambda → S3 (Parquet).**

---

### Option F: Data Lake–Driven Processing using Glue/EMR/Lambda

**What:** Data Lake (S3/Parquet) is the primary record store for batch processing. Glue ETL jobs run against Parquet files. Results written via Lambda to DynamoDB.

**Pros:**
- Columnar format optimal for 150-field projection reads
- Athena/Glue SQL for complex queries
- Cost-effective at scale (no RCU)
- Decoupled from operational DynamoDB load

**Cons:**
- Data Lake is derivative (fed by DynamoDB Streams) — inherently lagged
- Complex orchestration (Glue + Lambda + DynamoDB)
- Glue job startup overhead

**Suitability:** ✅ Best for external consumer APIs and analytics; complement to Option A.

**Verdict: Use for external API queries and analytics. Not for real-time processing.**

---

### Recommended Combination

| Processing Type | Recommended Option |
|---|---|
| Daily scheduled batch (300k KYCs) | Option A (DynamoDB projection table + Step Functions) |
| Complex rule evaluation needing SQL | Option C (Aurora read replica, optional) |
| Data Lake + external consumer APIs | Option F |
| Write path (all mutations) | Option E (Streams → S3 fan-out) |
| Perpetual KYC + event replay | Option D (phased event sourcing) |

---

## 4. Full End-to-End Target Architecture

### 4.1 High-Level System Topology

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        REGION: us-east-1 (Primary)                      │
│                                                                         │
│  ┌───────────┐    ┌──────────────┐    ┌─────────────────────────────┐  │
│  │ React UI  │───▶│ API Gateway  │───▶│ Lambda (Java)               │  │
│  └───────────┘    │ (REST/WebSocket)   │ KYC Domain Services         │  │
│                   └──────────────┘    └──────────┬──────────────────┘  │
│                                                  │                      │
│                        ┌─────────────────────────▼──────────────┐      │
│                        │         KYC-MAIN DynamoDB Table         │      │
│                        │         (Global Table — multi-region)   │      │
│                        └──────────┬─────────────────────────────┘      │
│                                   │ DynamoDB Streams                    │
│                        ┌──────────▼──────────────┐                      │
│                        │  Stream Processor Lambda │                      │
│                        └──────┬───────┬───────────┘                     │
│                               │       │                                  │
│               ┌───────────────▼─┐  ┌──▼────────────────────┐           │
│               │  KYC-PROCESSING  │  │  S3 Data Lake          │           │
│               │  DynamoDB Table  │  │  (Parquet/partitioned) │           │
│               └───────┬─────────┘  └──────────┬────────────┘           │
│                       │                        │                         │
│        ┌──────────────▼──────────────┐  ┌──────▼───────────────────┐   │
│        │  Step Functions             │  │  Glue Catalog             │   │
│        │  (Daily Batch Orchestrator) │  │  Athena / External APIs   │   │
│        └──────────────┬──────────────┘  └──────────────────────────┘   │
│                       │                                                  │
│        ┌──────────────▼──────────────┐                                  │
│        │  Processing Lambda (Java)   │                                   │
│        │  KYC Business Logic         │                                   │
│        └──────────────┬──────────────┘                                  │
│                       │                                                  │
│        ┌──────────────▼──────────────┐                                  │
│        │  EventBridge                │                                   │
│        │  (Domain Events Bus)        │                                   │
│        └──────────────┬──────────────┘                                  │
│                       │ routes to downstream consumers                   │
└───────────────────────┼─────────────────────────────────────────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │  REGION: eu-west-1  │  (identical topology — active-active)
              └─────────────────────┘
```

### 4.2 Component Inventory

| Component | Technology | Purpose |
|---|---|---|
| API Layer | API Gateway (REST + WebSocket) | UI CRUD, workflow transitions, external APIs |
| Domain Logic | Lambda (Java 21, GraalVM native) | KYC rules, transformations, orchestration |
| Primary Store | DynamoDB Global Tables | UI state, KYC records, workflow state |
| Processing Store | DynamoDB KYC-PROCESSING | Batch read-optimized projection |
| Orchestration | Step Functions Express + Standard | Batch workflows, complex KYC flows |
| Event Bus | EventBridge (custom bus) | Domain events, cross-service decoupling |
| Scheduler | EventBridge Scheduler | Cron triggers replacing Autosys |
| Data Lake | S3 + Glue + Athena | Analytics, historical, external consumers |
| Stream Processor | Lambda (DynamoDB Streams trigger) | Fan-out: Streams → Processing table + S3 |
| Secrets | Secrets Manager | Credentials, API keys |
| Observability | CloudWatch + X-Ray + Embedded Metrics | Metrics, traces, structured logs |
| Auth | Cognito + IAM | UI auth, service-to-service auth |
| CDN | CloudFront | React UI static assets |
| Container Option | ECS Fargate (Java) | Long-running batch if Lambda timeout insufficient |

### 4.3 DDD Domain Services (Hexagonal Architecture)

```
┌─────────────────────────────────────────────────┐
│              KYC Domain Service                  │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │         Application Layer               │   │
│  │  KycOnboardingUseCase                   │   │
│  │  KycReviewUseCase                       │   │
│  │  KycRiskAssessmentUseCase               │   │
│  │  KycWorkflowTransitionUseCase           │   │
│  └──────────────┬──────────────────────────┘   │
│                 │                               │
│  ┌──────────────▼──────────────────────────┐   │
│  │         Domain Layer                    │   │
│  │  KycAggregate (root entity)             │   │
│  │  RiskProfile (value object)             │   │
│  │  ReviewDecision (value object)          │   │
│  │  KycDomainService                       │   │
│  │  KycRules (domain rule interfaces)      │   │
│  └──────────────┬──────────────────────────┘   │
│                 │                               │
│  ┌──────────────▼──────────────────────────┐   │
│  │    Infrastructure (Adapters)            │   │
│  │  DynamoDbKycRepository                  │   │
│  │  EventBridgeKycPublisher                │   │
│  │  S3DocumentRepository                   │   │
│  │  ExternalRiskApiAdapter                 │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Lambda Handler → Application Layer (no domain logic in handler):**
```java
public class KycReviewHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    private final KycReviewUseCase useCase;

    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent event, Context context) {
        ReviewRequest request = objectMapper.readValue(event.getBody(), ReviewRequest.class);
        ReviewResult result = useCase.execute(request);
        return successResponse(result);
    }
}
```

### 4.4 Step Functions: Daily Batch Orchestration

```json
{
  "Comment": "KYC Daily Batch Processing",
  "StartAt": "InitializeBatch",
  "States": {
    "InitializeBatch": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:kyc-batch-initializer",
      "Next": "FanOutToShards"
    },
    "FanOutToShards": {
      "Type": "Map",
      "ItemsPath": "$.shards",
      "MaxConcurrency": 20,
      "Iterator": {
        "StartAt": "ProcessShard",
        "States": {
          "ProcessShard": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:...:kyc-shard-processor",
            "Retry": [{ "ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 3, "IntervalSeconds": 30 }],
            "End": true
          }
        }
      },
      "Next": "AggregateBatch"
    },
    "AggregateBatch": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:kyc-batch-aggregator",
      "Next": "PublishCompletionEvent"
    },
    "PublishCompletionEvent": {
      "Type": "Task",
      "Resource": "arn:aws:states:::events:putEvents",
      "Parameters": {
        "Entries": [{
          "Source": "kyc.batch",
          "DetailType": "BatchProcessingCompleted",
          "Detail.$": "$.batchSummary"
        }]
      },
      "End": true
    }
  }
}
```

### 4.5 React UI Integration

**CRUD Pattern:**
```
React UI
  → PUT /kyc/{id}/documents          → Lambda → DynamoDB PutItem
  → POST /kyc/{id}/workflow/advance  → Lambda → Step Functions StartExecution
  → GET /kyc/{id}                    → Lambda → DynamoDB GetItem
  → POST /kyc                        → Lambda → DynamoDB TransactWriteItems (creates KYC + enqueues)
```

**WebSocket (real-time workflow updates):**
```
Step Functions Task completes
  → EventBridge rule
  → Lambda WebSocket Notifier
  → API Gateway WebSocket → React UI
```

### 4.6 Data Lake Integration

```
DynamoDB Stream Event (KYC-MAIN)
         │
         ▼
Stream Processor Lambda
         │
         ├── Serialize record to Parquet (Apache Arrow Java lib)
         ├── S3 PutObject: s3://kyc-datalake/kyc/year=2024/month=01/day=15/shard=03/{uuid}.parquet
         └── Optional: Glue Catalog partition update (or use Glue Crawler on schedule)

S3 Data Lake Structure:
kyc-datalake/
  kyc/
    year=2024/month=01/day=15/
      *.parquet             ← raw KYC records (all 2500 fields)
  kyc-processing/
    year=2024/month=01/day=15/
      *.parquet             ← 150-field projection
  kyc-results/
    year=2024/month=01/day=15/
      *.parquet             ← batch processing outcomes
  kyc-events/
    year=2024/month=01/day=15/
      *.parquet             ← domain events (event sourcing log)
```

### 4.7 External Consumer API (Data Lake-Backed)

```
External System
  → API Gateway (external-facing)
  → Lambda: ExternalKycApiHandler
      → Check auth (API key / OAuth)
      → Query Athena (async): SELECT fields FROM kyc WHERE entity_id = ?
      → Wait for Athena query completion (polling or async callback)
      → Return paginated results
```

For low-latency external access, maintain a read-only DynamoDB table (KYC-EXTERNAL) or ElastiCache layer backed by Data Lake snapshots.

---

## 5. Business Rules Rewrite Strategy

### 5.1 Stored Procedure Analysis & Extraction

**Phase 1: Catalog all SPs**

| Category | Example SPs | Target |
|---|---|---|
| Validation | sp_validate_kyc_fields | KycValidator (domain service) |
| Risk scoring | sp_calculate_risk_score | RiskScoringService (domain service) |
| Status transitions | sp_advance_workflow | KycWorkflowService + Step Functions |
| Review scheduling | sp_schedule_periodic_review | EventBridge Scheduler + Lambda |
| Aggregations | sp_generate_risk_report | Athena query / Glue job |
| Notifications | sp_notify_reviewer | EventBridge → SES/SNS Lambda |

**Phase 2: Classify logic**

- **Pure domain rules** → Move to Java domain service classes (no infrastructure dependencies)
- **Data transformations** → Move to Java value objects
- **Orchestration** → Move to Step Functions state machines
- **Reporting/aggregations** → Move to Athena or Glue

### 5.2 DDD + Hexagonal Rule Implementation

```java
// Domain rule interface (port)
public interface RiskScoringRule {
    RiskScore evaluate(KycProcessingContext context);
    int priority();
    String ruleId();
    String ruleVersion();
}

// Implementation (adapter)
@Component
public class JurisdictionRiskRule implements RiskScoringRule {
    @Override
    public RiskScore evaluate(KycProcessingContext ctx) {
        if (HIGH_RISK_JURISDICTIONS.contains(ctx.getJurisdiction())) {
            return RiskScore.HIGH;
        }
        return RiskScore.STANDARD;
    }
    @Override public int priority() { return 10; }
    @Override public String ruleId() { return "JURISDICTION_RISK_001"; }
    @Override public String ruleVersion() { return "1.2.0"; }
}

// Rule engine
public class KycRuleEngine {
    private final List<RiskScoringRule> rules;

    public RiskAssessmentResult evaluate(KycProcessingContext ctx) {
        return rules.stream()
            .sorted(Comparator.comparing(RiskScoringRule::priority))
            .map(rule -> rule.evaluate(ctx))
            .reduce(RiskScore.LOW, RiskScore::max);
    }
}
```

### 5.3 Step Functions for Workflow Transitions

```
KYC Onboarding Workflow:
  INITIATED → DOCUMENTS_PENDING → DOCUMENTS_RECEIVED → RISK_ASSESSMENT
           → REVIEW_ASSIGNED → REVIEW_IN_PROGRESS
           → [APPROVED | REJECTED | ESCALATED]
           → ACTIVE (if approved)

Each transition:
  1. Lambda validates preconditions
  2. Lambda updates DynamoDB (conditional write: status must match expected)
  3. Lambda publishes domain event to EventBridge
  4. Step Functions advances to next state
  5. Optional: notify via WebSocket + email
```

### 5.4 Idempotency

All Lambda handlers:
```java
public class IdempotentKycHandler {
    private final DynamoDbClient dynamoDb;
    private final String IDEMPOTENCY_TABLE = "KYC-IDEMPOTENCY";

    public void handle(String idempotencyKey, Runnable operation) {
        try {
            dynamoDb.putItem(PutItemRequest.builder()
                .tableName(IDEMPOTENCY_TABLE)
                .item(Map.of(
                    "idempotencyKey", AttributeValue.fromS(idempotencyKey),
                    "ttl", AttributeValue.fromN(String.valueOf(Instant.now().plusSeconds(86400).getEpochSecond()))
                ))
                .conditionExpression("attribute_not_exists(idempotencyKey)")
                .build());
            operation.run();
        } catch (ConditionalCheckFailedException e) {
            log.info("Duplicate request detected: {}", idempotencyKey);
            // Return cached result or no-op
        }
    }
}
```

### 5.5 Rule Versioning

```java
public class RuleVersion {
    private final String ruleId;
    private final String version;       // semver: "1.2.0"
    private final LocalDate effectiveFrom;
    private final LocalDate effectiveTo; // null = current

    public boolean isApplicable(LocalDate processingDate) {
        return !processingDate.isBefore(effectiveFrom)
            && (effectiveTo == null || !processingDate.isAfter(effectiveTo));
    }
}
```

Rule versions stored in DynamoDB `KYC-RULE-REGISTRY` table.  
Processing Lambda looks up applicable rule version at runtime based on KYC creation date or processing date.

### 5.6 Processing 150 Fields Out of 2500

```java
public class KycProcessingContext {
    // Only the 150 fields needed for processing — enforced at compile time
    private final String kycId;
    private final String entityType;
    private final String jurisdiction;
    private final RiskTier currentRiskTier;
    private final LocalDate reviewDueDate;
    // ... 145 more fields

    // Factory method: extract from full KYC record
    public static KycProcessingContext from(Map<String, AttributeValue> dynamoItem) {
        return new KycProcessingContext(
            extractString(dynamoItem, "kycId"),
            extractString(dynamoItem, "entityType"),
            // ... only the 150 fields
        );
    }
}
```

The full 2500-field domain object is NEVER instantiated in the processing path.

### 5.7 Perpetual KYC Support

Perpetual KYC requires continuous monitoring rather than periodic review:

```
Event triggers review:
  - Sanctions list update → EventBridge event → Lambda → re-score affected KYCs
  - Adverse media hit → External webhook → API Gateway → Lambda → trigger review
  - Transaction anomaly → Risk system event → Lambda → elevate risk tier
  - Time-based trigger → EventBridge Scheduler (cron) → Step Functions

Each trigger:
  1. Identify affected KYC(s) via GSI query
  2. Enqueue for re-processing via SQS (with dedup)
  3. Processing Lambda re-evaluates using current rules
  4. If result differs from current status → trigger review workflow
```

---

## 6. Data Flow & Sequence Designs

### 6.1 Daily KYC Batch Processing

```
06:00 UTC: EventBridge Scheduler fires
  │
  ▼
Step Functions: DailyKycBatchExecution
  │
  ├── State: InitializeBatch
  │   └── Lambda: determine today's date, generate shard list [0..19]
  │
  ├── State: FanOutToShards (Map, MaxConcurrency=20)
  │   └── For each shard (0..19):
  │       └── Lambda: KycShardProcessor
  │           ├── Query KYC-PROCESSING: PK = PROCESS#{today}#{shard}
  │           ├── Paginate (500 records/page)
  │           └── For each record:
  │               ├── Deserialize KycProcessingContext (150 fields)
  │               ├── Apply rule engine (versioned rules)
  │               ├── Compute new risk score, review outcome
  │               ├── Conditional update KYC-MAIN (optimistic lock)
  │               ├── Publish KycProcessed event to EventBridge
  │               └── Batch write results to S3 (accumulate, flush per 10MB)
  │
  ├── State: AggregateBatch
  │   └── Lambda: collect shard stats → build batch summary
  │
  └── State: PublishCompletion
      └── EventBridge: BatchProcessingCompleted event

On error in any shard:
  → Retry (3x, exponential backoff)
  → If all retries fail: DLQ (SQS) + CloudWatch alarm
  → SNS alert to on-call engineer
```

### 6.2 File Ingestion Flow

```
External Data Provider
  │
  ▼
S3 Bucket: kyc-inbound/{provider}/{date}/file.csv
  │
  ▼
S3 Event Notification → EventBridge
  │
  ▼
Step Functions: FileIngestionWorkflow
  │
  ├── State: ValidateFile
  │   └── Lambda: checksum, schema validation, provider auth
  │
  ├── State: ParseAndChunk
  │   └── Lambda: split CSV into chunks (1000 records/chunk) → SQS
  │
  ├── State: ProcessChunks (Map over SQS messages)
  │   └── Lambda: KycIngestionProcessor
  │       ├── Parse each record → KycIngestCommand
  │       ├── Validate required fields
  │       ├── Enrich from reference data
  │       ├── TransactWriteItems to KYC-MAIN (idempotent: conditionExpression)
  │       └── Streams → fan-out to KYC-PROCESSING + S3
  │
  └── State: PublishIngestionComplete
      └── EventBridge: FileIngestionCompleted event
```

### 6.3 DynamoDB → S3 Projection (Stream Processor)

```
DynamoDB Streams (KYC-MAIN)
  │
  ▼
Lambda: KycStreamProcessor (batch size: 100, parallelization factor: 3)
  │
  ├── Parse stream records (MODIFY / INSERT events)
  ├── Filter: only process records relevant to workflow state
  │
  ├── Path A: KYC-PROCESSING update
  │   ├── Extract 150 fields from NEW_IMAGE
  │   ├── Compute shard
  │   └── DynamoDB PutItem to KYC-PROCESSING
  │
  ├── Path B: S3 Data Lake write
  │   ├── Serialize to Parquet (Arrow Java)
  │   ├── Buffer until 10MB or 60s timeout
  │   └── S3 PutObject to partitioned path
  │
  └── Path C: EventBridge event
      └── PutEvents: KycUpdated domain event
```

### 6.4 External Consumer API Request

```
External System
  │
  └── GET /external/kyc/{entityId}
      Authorization: Bearer {oauth_token}
  │
  ▼
API Gateway (external-api.kyc.company.com)
  │
  ▼
Lambda: ExternalKycApiHandler
  │
  ├── Validate OAuth token → Cognito
  ├── Authorize: check consumer's data contract scope
  │
  ├── Check ElastiCache (TTL: 5 min):
  │   ├── Cache HIT → return cached response
  │   └── Cache MISS:
  │       ├── Execute Athena query:
  │       │   SELECT {contracted_fields} FROM kyc
  │       │   WHERE entity_id = '{entityId}'
  │       │   LIMIT 1
  │       ├── Wait for Athena (async polling, max 10s)
  │       ├── Cache result in ElastiCache
  │       └── Return response
  │
  └── Return paginated JSON response
      (with data contract version header)
```

### 6.5 Multi-Region Conflict Resolution

```
SCENARIO: Same KYC record updated simultaneously in us-east-1 and eu-west-1

us-east-1:                              eu-west-1:
PUT KYC#abc123, version=5               PUT KYC#abc123, version=5
status=APPROVED                         status=REJECTED (different reviewer)

DynamoDB Global Tables replication:
  → Last writer wins (by timestamp)
  → One update overwrites the other

SOLUTION: Use conditional writes with optimistic locking:
  ConditionExpression: "version = :expectedVersion"
  → Losing region's write fails with ConditionalCheckFailedException
  → Lambda catches → publishes KycConflict event to EventBridge
  → Conflict resolver Lambda:
      1. Reads both versions from each region's table
      2. Applies conflict resolution rule (e.g., most-conservative status wins: REJECTED > APPROVED)
      3. Writes resolved version with incremented version number
      4. Publishes KycConflictResolved event with audit trail
```

### 6.6 Periodic Review KYC Trigger

```
Monthly trigger (EventBridge Scheduler: cron(0 2 1 * ? *))
  │
  ▼
Lambda: PeriodicReviewInitiator
  │
  ├── Query GSI-3 (PeriodicReview-Index):
  │   PK = nextReviewYearMonth (= current month "2024-01")
  │   Sort by nextReviewDate ASC
  │
  ├── Paginate through results
  │
  └── For each KYC due for review:
      ├── Check: not already in active review (status check)
      ├── PutItem to SQS: ReviewTriggerQueue (dedup ID = kycId + yearMonth)
      └── Lambda: ReviewWorkflowInitiator (SQS trigger)
          ├── Start Step Functions: KycReviewWorkflow
          ├── Update DynamoDB: status = REVIEW_TRIGGERED
          └── Notify assigned reviewer via SES
```

---

## 7. Multi-Region Active-Active

### 7.1 DynamoDB Global Tables

**Configuration:**
- KYC-MAIN: Global Table across us-east-1, eu-west-1, ap-southeast-1
- KYC-PROCESSING: Global Table (same regions)
- Replication lag: typically <1 second, designed for eventual consistency

**Write routing:**
- Each region accepts writes from its local React UI / API calls
- Global Tables replicate asynchronously
- Version attribute (optimistic locking) prevents silent overwrites

**Read consistency:**
- UI reads: eventually consistent (default) — acceptable for workflow display
- Processing reads: eventually consistent — processing tables populated via Streams (seconds lag acceptable)
- Critical reads (status before workflow transition): strongly consistent read against local replica

### 7.2 Conflict Resolution Strategy

**Level 1: Prevent conflicts at application layer**
- Assign KYC records to home region based on entity's jurisdiction (routing in API Gateway)
- European entities default-write to eu-west-1
- Conflicts theoretically impossible if routing is respected

**Level 2: Detect via version attribute**
```java
dynamoDb.putItem(PutItemRequest.builder()
    .conditionExpression("version = :currentVersion OR attribute_not_exists(version)")
    .expressionAttributeValues(Map.of(":currentVersion", AttributeValue.fromN(String.valueOf(currentVersion))))
    .build());
```

**Level 3: Resolve via DynamoDB Streams**
- Stream processor in each region detects conflict events
- Applies deterministic conflict resolution (last-write-wins by application timestamp, not DynamoDB timestamp)
- Publishes resolved record

### 7.3 Multi-Region Step Functions

Each region runs its own Step Functions executions against its local DynamoDB replica.  
No cross-region Step Functions calls in normal operation.

For global orchestration (e.g., cross-border KYC involving both EU and US entities):
- Use EventBridge cross-region event buses
- EventBridge rule in us-east-1 forwards events to eu-west-1 bus
- eu-west-1 Step Functions execution picks up and continues

### 7.4 Multi-Region EventBridge

```
EventBridge Architecture:

us-east-1 custom bus: kyc-events-us
  │
  ├── Rules: route to local Lambda consumers
  └── Rule: forward to eu-west-1 and ap-southeast-1 buses

eu-west-1 custom bus: kyc-events-eu
  │
  └── Rules: route to local Lambda consumers (only EU-relevant events)
```

Cross-region event forwarding via EventBridge Pipes or EventBridge cross-region targets (native feature).

### 7.5 S3 Cross-Region Replication

```
kyc-datalake-us-east-1 (primary)
  │
  └── S3 CRR (Cross-Region Replication)
      ├── → kyc-datalake-eu-west-1 (replica)
      └── → kyc-datalake-ap-southeast-1 (replica)

Replication rules:
  - Replicate: kyc/**, kyc-results/**
  - Do NOT replicate: kyc-processing/** (regenerated per region from local Streams)
  - Replication time: SLA of 15 minutes (S3 RTC enabled)
```

### 7.6 Regional Failover

**Active-active means no failover needed for writes** — each region is self-sufficient.

**For reads (external consumers):**
- Route 53 health checks on API Gateway endpoints
- Weighted routing: 50% us-east-1, 50% eu-west-1 (adjust by latency)
- Failover policy: if us-east-1 unhealthy → Route 53 shifts 100% to eu-west-1

**Lambda/Step Functions regional isolation:**
- Each region's Lambda cold pool is independent
- Regional quota limits managed separately
- No cross-region Lambda invocations in processing path

### 7.7 Global API Design

```
api.kyc.company.com (Route 53 GeoDNS)
  │
  ├── EU traffic → eu-west-1 API Gateway
  ├── APAC traffic → ap-southeast-1 API Gateway
  └── Default / Americas → us-east-1 API Gateway

Each API Gateway:
  → hits local Lambda (Java)
  → reads from local DynamoDB replica
  → publishes events to local EventBridge bus
```

---

## 8. Development & Operational Excellence

### 8.1 CI/CD Pipeline

```
Developer pushes to feature branch
  │
  ▼
GitHub Actions / CodePipeline:
  │
  ├── Build: mvn clean package -Pnative (GraalVM native compilation)
  ├── Unit Tests: mvn test (JUnit 5)
  ├── Integration Tests: testcontainers (DynamoDB local, LocalStack)
  ├── Contract Tests: PactFlow consumer-driven contracts
  ├── Static Analysis: SonarQube + Checkstyle
  ├── SAST: Semgrep / Snyk
  │
  ▼
PR merged to main:
  │
  ├── CDK Synth: cdk synth --context env=staging
  ├── CDK Diff: cdk diff (review before deploy)
  ├── Deploy to Staging: cdk deploy --require-approval never
  ├── Integration Tests vs Staging: Newman/Postman + replay tests
  │
  ▼
Tag for production:
  │
  ├── Deploy to us-east-1: cdk deploy --context env=prod --context region=us-east-1
  ├── Smoke Tests: automated health checks
  ├── Deploy to eu-west-1: rolling (if us-east-1 healthy)
  └── Deploy to ap-southeast-1
```

### 8.2 Java Lambda Project Structure

```
kyc-platform/
├── pom.xml (multi-module)
├── kyc-domain/               ← pure domain, no AWS dependencies
│   ├── KycAggregate.java
│   ├── RiskScoringService.java
│   ├── KycRuleEngine.java
│   └── ports/
│       ├── KycRepository.java (interface)
│       └── KycEventPublisher.java (interface)
│
├── kyc-application/          ← use cases, orchestration
│   ├── KycReviewUseCase.java
│   └── KycOnboardingUseCase.java
│
├── kyc-infrastructure/       ← AWS adapters
│   ├── DynamoDbKycRepository.java
│   ├── EventBridgeKycPublisher.java
│   └── S3DocumentRepository.java
│
├── kyc-lambda-review/        ← Lambda handler (thin)
│   ├── KycReviewHandler.java
│   └── src/main/resources/
│       └── bootstrap (native binary entry point)
│
├── kyc-lambda-processing/    ← Batch processing handler
│   └── KycBatchShardProcessor.java
│
└── kyc-cdk/                  ← Infrastructure as Code
    ├── KycStack.java
    ├── DynamoDbConstruct.java
    ├── LambdaConstruct.java
    └── StepFunctionsConstruct.java
```

**GraalVM native compilation for cold start <100ms:**
```xml
<plugin>
  <groupId>org.graalvm.buildtools</groupId>
  <artifactId>native-maven-plugin</artifactId>
  <configuration>
    <buildArgs>
      <buildArg>--no-fallback</buildArg>
      <buildArg>-H:+ReportExceptionStackTraces</buildArg>
    </buildArgs>
  </configuration>
</plugin>
```

### 8.3 Monitoring Dashboard (CloudWatch)

**Key Metrics to alarm on:**

| Metric | Alarm Threshold | Action |
|---|---|---|
| BatchProcessingDuration | > 4 hours | PagerDuty + SNS |
| BatchRecordsFailed | > 0.1% of daily volume | SNS alert |
| LambdaErrorRate | > 1% over 5 min | Auto-scale + alert |
| DynamoDbThrottledRequests | > 100/min | Scale WCU + alert |
| KycProcessingLag | > 15 min behind schedule | Alert |
| DataLakeWriteFailures | > 0 | Alert (data integrity) |
| ConflictResolutionEvents | > threshold | Engineering review |

**Embedded Metrics Format (Java):**
```java
metricsLogger.putMetric("KycProcessed", 1, Unit.COUNT);
metricsLogger.putMetric("ProcessingDurationMs", durationMs, Unit.MILLISECONDS);
metricsLogger.putDimensions(Map.of("Region", region, "RiskTier", riskTier));
metricsLogger.flush();
```

### 8.4 Structured Logging

```java
log.info("KYC processing completed",
    "kycId", kycId,
    "ruleVersion", ruleVersion,
    "inputRiskTier", inputRiskTier,
    "outputRiskTier", outputRiskTier,
    "processingDurationMs", durationMs,
    "region", System.getenv("AWS_REGION"),
    "traceId", MDC.get("AWSXRayTraceId")
);
```

JSON format in Lambda → CloudWatch Logs Insights queries:
```sql
fields kycId, outputRiskTier, processingDurationMs
| filter outputRiskTier = "HIGH"
| stats count() by bin(1h)
```

### 8.5 Retry & DLQ Strategy

```
Lambda SQS trigger:
  maxReceiveCount: 3
  visibilityTimeout: 120s (> Lambda timeout)
  DLQ: KycProcessing-DLQ (SQS)

DLQ handling:
  → CloudWatch alarm: DLQ depth > 0
  → Lambda: DLQ replay handler (manual trigger or auto-replay)
  → Replay with exponential backoff
  → If still failing: DLQ → S3 archive + PagerDuty alert

Step Functions retry policy:
  RetryPolicy:
    - ErrorEquals: [Lambda.ServiceException, Lambda.TooManyRequestsException]
      IntervalSeconds: 30
      MaxAttempts: 5
      BackoffRate: 2.0
    - ErrorEquals: [States.TaskFailed]
      IntervalSeconds: 60
      MaxAttempts: 3
```

### 8.6 Testing Strategy

**Unit Tests (JUnit 5 + Mockito):**
```java
@Test
void shouldElevateRiskForHighRiskJurisdiction() {
    KycProcessingContext ctx = KycProcessingContext.builder()
        .jurisdiction("IR").riskTier(RiskTier.MEDIUM).build();
    RiskScore result = jurisdictionRiskRule.evaluate(ctx);
    assertThat(result).isEqualTo(RiskScore.HIGH);
}
```

**Integration Tests (Testcontainers + DynamoDB Local):**
```java
@Testcontainers
class KycRepositoryIntegrationTest {
    @Container
    static GenericContainer<?> dynamoDbLocal = new GenericContainer<>("amazon/dynamodb-local:latest")
        .withExposedPorts(8000);

    @Test
    void shouldPersistAndRetrieveKycRecord() { ... }
}
```

**Contract Tests (Pact):**
- Consumer (React UI) defines API contract
- Provider (Lambda) verifies contract in CI
- PactBroker stores published contracts

**Replay-Based Testing:**
- Capture production DynamoDB stream events to S3
- Replay against new Lambda version in staging
- Compare outputs: any divergence = regression

---

## 9. Migration Plan

### 9.1 Phase Overview

| Phase | Duration | Goal |
|---|---|---|
| Phase 0: Foundation | 4 weeks | CDK infrastructure, CI/CD, DynamoDB tables, baseline Lambda |
| Phase 1: Data Migration | 8 weeks | MS SQL → DynamoDB + Data Lake; parallel run |
| Phase 2: Processing Rewrite | 12 weeks | SP logic → Java domain services; shadow mode |
| Phase 3: Autosys → EventBridge | 4 weeks | Scheduler cutover |
| Phase 4: UI Cutover | 4 weeks | React UI switches to AWS APIs |
| Phase 5: Decommission | 4 weeks | Remove MS SQL, Python jobs, Autosys |

### 9.2 MS SQL → DynamoDB Migration

**Step 1: Schema analysis**
- Document all tables, columns, indexes, FK relationships
- Map relational schema to DynamoDB single-table design
- Identify which columns map to the 150 processing fields

**Step 2: DMS (Database Migration Service) + custom Lambda**
- AWS DMS: MS SQL → S3 (full load + CDC)
- S3 → Lambda: transform relational rows to DynamoDB items
- Load into DynamoDB in batches (BatchWriteItem: 25 items/call)
- Run DMS CDC continuously during parallel run

**Step 3: Validation**
```java
// Record count check
assertThat(dynamoDbCount("KYC-MAIN")).isEqualTo(sqlServerCount("kyc"));

// Spot-check 1000 random records: field-by-field comparison
// Automated daily reconciliation Lambda during parallel run
```

**Step 4: Cutover**
- Enable DynamoDB as write-primary (dual-write phase: new writes → DynamoDB + read from DynamoDB)
- Stop DMS CDC
- Decommission MS SQL read path

### 9.3 Stored Procedures → Java Rewrite

For each SP:
1. Document business logic in plain English (extract from SQL)
2. Write unit tests against documented behavior (TDD)
3. Implement Java domain service
4. Run shadow mode: execute both SP and Java, compare outputs, log diffs
5. Switch to Java when diff rate < 0.01%

**Shadow mode pattern:**
```java
public RiskScore calculateRisk(KycProcessingContext ctx) {
    RiskScore javaResult = javaRiskEngine.calculate(ctx);
    if (shadowModeEnabled) {
        RiskScore spResult = legacySpAdapter.calculateRisk(ctx); // calls MS SQL
        if (!javaResult.equals(spResult)) {
            log.warn("Shadow divergence detected", "kycId", ctx.getKycId(),
                "javaResult", javaResult, "spResult", spResult);
            metrics.incrementCounter("ShadowDivergence");
        }
    }
    return javaResult; // always use Java result after shadow mode activated
}
```

### 9.4 Autosys → EventBridge Scheduler Migration

| Autosys Job | Cron | EventBridge Scheduler | Target |
|---|---|---|---|
| DAILY_KYC_PROCESS | 0 6 * * * | cron(0 6 * * ? *) | Step Functions: DailyKycBatch |
| MONTHLY_REVIEW_INIT | 0 2 1 * * | cron(0 2 1 * ? *) | Lambda: PeriodicReviewInitiator |
| WEEKLY_REPORT | 0 8 * * 1 | cron(0 8 ? * 2 *) | Lambda: ReportGenerator |
| SANCTIONS_REFRESH | 0 */6 * * * | cron(0 0/6 * * ? *) | Lambda: SanctionsRefresher |

### 9.5 Python → Java Rewrite

Python processing scripts → Java Lambda functions following hexagonal architecture.

Key mapping:
- Python Pandas operations → Java Streams / Apache Arrow
- Python SQLAlchemy queries → DynamoDB SDK calls
- Python business logic → Java domain services (direct port, then refactor)

Parallel run: Python jobs and Java Lambdas run simultaneously for 4 weeks. Output comparison via reconciliation Lambda.

### 9.6 Cutover Checklist

```
Pre-cutover (T-2 weeks):
  □ All integration tests passing in staging
  □ Shadow mode divergence < 0.01%
  □ DynamoDB capacity provisioned (auto-scaling configured)
  □ CloudWatch alarms configured and tested
  □ Rollback procedure documented and rehearsed
  □ Data reconciliation Lambda ready

Cutover Day:
  □ Freeze MS SQL writes (maintenance mode on legacy UI)
  □ Final DMS sync run → verify record counts match
  □ Switch DNS → new React UI (AWS-backed)
  □ Verify API Gateway + Lambda health
  □ Run smoke test suite
  □ Monitor for 2 hours before declaring success

Post-cutover (T+2 weeks):
  □ Monitor metrics daily
  □ Decommission DMS
  □ Decommission MS SQL (snapshot and archive first)
  □ Decommission Autosys jobs
  □ Decommission Python ETL scripts
```

---

## 10. Final Recommendation

### 10.1 Target Architecture Summary

**Primary Data Store:** DynamoDB Global Tables (3 regions: us-east-1, eu-west-1, ap-southeast-1)  
**Batch Processing Store:** DynamoDB KYC-PROCESSING (projection table, 150 fields, sharded by review date)  
**Data Lake:** S3 (Parquet, partitioned) + Glue Catalog + Athena  
**Processing:** Step Functions (orchestration) + Lambda Java 21 GraalVM (business logic)  
**Event Bus:** EventBridge (custom bus per region, cross-region forwarding)  
**Scheduling:** EventBridge Scheduler (replaces Autosys)  
**External APIs:** API Gateway + Lambda + Athena (Data Lake-backed)  
**UI:** React → API Gateway → Lambda → DynamoDB  
**Auth:** Cognito (UI), API keys + OAuth (external)  

### 10.2 Performance Expectations

| Metric | Expected | Design Basis |
|---|---|---|
| Daily batch completion | < 2 hours | 20 parallel shards × 15k records × 200ms/record |
| API response time (P99) | < 100ms | DynamoDB single-item read + GraalVM Lambda |
| Lambda cold start (native) | < 100ms | GraalVM native compilation |
| DynamoDB replication lag | < 1 second | Global Tables typical replication |
| Data Lake freshness | < 5 minutes | Streams → Lambda → S3 pipeline |
| Athena query (external API) | < 3 seconds | Parquet columnar + partition pruning |

### 10.3 Cost Estimate (Monthly, us-east-1)

| Component | Monthly Cost |
|---|---|
| DynamoDB (on-demand, 300k KYCs/day CRUD) | ~$150 |
| DynamoDB Global Tables replication (2 additional regions) | ~$80 |
| Lambda (batch + API, 300k invocations/day avg) | ~$30 |
| Step Functions Express Workflows | ~$20 |
| S3 Data Lake (storage + requests) | ~$40 |
| Glue Catalog + Athena | ~$25 |
| API Gateway | ~$15 |
| EventBridge | ~$5 |
| CloudWatch + X-Ray | ~$30 |
| Secrets Manager, Cognito, misc | ~$20 |
| **Total (single region)** | **~$415/month** |
| **Total (3 regions, multi-region overhead)** | **~$700–800/month** |

*Provisioned capacity + Savings Plans can reduce DynamoDB costs 40–60% at stable load.*

### 10.4 Multi-Region Design Summary

- **Active-active writes:** All 3 regions accept writes; DynamoDB Global Tables replicate asynchronously
- **Conflict resolution:** Optimistic locking (version attribute) + deterministic resolver Lambda
- **Data jurisdiction:** API Gateway GeoDNS routes EU entities to eu-west-1 to minimize conflict probability
- **Batch processing:** Each region processes its own shard of records (no cross-region processing calls)
- **Data Lake:** Primary in us-east-1; CRR to eu-west-1 and ap-southeast-1; external APIs read from local replica
- **Failover:** Route 53 health checks; automatic traffic shift if a regional API Gateway fails

### 10.5 Future-Proofing

| Capability | Path |
|---|---|
| Perpetual KYC | EventBridge rules on external triggers (sanctions, media) → re-scoring pipeline; already designed |
| Data Mesh | Glue Catalog as central registry; domain teams publish their own data products via separate S3 prefixes |
| External API scale | Athena → DynamoDB Accelerator (DAX) or ElastiCache for hot entities; API Gateway usage plans for throttling |
| Event sourcing | Add Kinesis as event log; Streams already publish events; full event sourcing is additive |
| ML risk scoring | SageMaker endpoint callable from Processing Lambda; interface already abstracted as `RiskScoringRule` port |
| New jurisdictions | Data model is jurisdiction-agnostic; rule versioning supports jurisdiction-specific rules |
| Regulatory audit | S3 data lake + event log provides immutable audit trail; Athena for ad-hoc regulatory queries |

### 10.6 Architecture Principles

1. **Domain first:** No AWS service dependency in domain or application layers.
2. **Event-driven by default:** Every state change is an event; consumers are decoupled.
3. **Process only what you need:** 150-field projection enforced at the data model level — not a runtime filter.
4. **Sharding is intentional:** Not a workaround; built into the key design from day one.
5. **Idempotency everywhere:** Every Lambda, every Step Functions task, every DynamoDB write.
6. **Observability is not optional:** Structured logs, metrics, traces, alarms in every component from the start.
7. **Migrations are incremental:** Shadow mode, parallel run, and reconciliation before cutover — never a big-bang switch.

---

*Document version 1.0 — Architecture review recommended before implementation. All cost estimates are approximate and should be validated against AWS Pricing Calculator with actual traffic profiles.*