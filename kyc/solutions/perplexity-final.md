# KYC Ecosystem: AWS-Native Architecture & Design Document

**Version 1.0 | Principal Engineer Reference | Java + AWS-Native | Multi-Region Active-Active**

***

## 1. DynamoDB vs. Relational DB Decision Framework

### The Core Question

Processing 300k+ KYCs per day against a 2,500-field domain using only 150 fields is fundamentally a **read amplification and query pattern problem**. The right answer is a **hybrid model**, not a binary choice.

### Evaluation Matrix

| Dimension | DynamoDB | Aurora PostgreSQL |
|---|---|---|
| **Latency** | Single-digit ms at any scale  [aws.criticalcloud](https://aws.criticalcloud.ai/checklist-choosing-aurora-serverless-dynamodb/) | 1–10 ms, degrades under complex scans |
| **Query flexibility** | PK/SK/GSI only; no ad-hoc SQL | Full SQL, joins, window functions |
| **Scan cost (300k/day)** | High RCU burn if unindexed | Sequential scan on indexed column = cheap |
| **Global replication** | Native Global Tables, ~1s lag  [aws.amazon](https://aws.amazon.com/blogs/database/how-to-use-amazon-dynamodb-global-tables-to-power-multiregion-architectures/) | Aurora Global DB, ~1s lag, read replicas |
| **Multi-region writes** | Native active-active, last-writer-wins  [brooker.co](https://brooker.co.za/blog/2025/08/15/dynamo-dynamodb-dsql.html) | Active-passive; active-active is complex |
| **ACID transactions** | Single-table or item-group only  [aws.criticalcloud](https://aws.criticalcloud.ai/checklist-choosing-aurora-serverless-dynamodb/) | Full ACID across tables |
| **Storage cost** | ~$0.25/GB/month | ~$0.10/GB/month  [aws.criticalcloud](https://aws.criticalcloud.ai/checklist-choosing-aurora-serverless-dynamodb/) |
| **Uptime SLA** | 99.999% (~27 sec/month)  [aws.criticalcloud](https://aws.criticalcloud.ai/checklist-choosing-aurora-serverless-dynamodb/) | 99.99% (~4 min/month) |
| **Schema flexibility** | Schemaless, evolve freely | Schema migrations required |
| **Operational overhead** | Fully managed, zero DBA | Managed but requires tuning |

### Partition Key / Hot Partition Analysis

With 300k KYCs/day and a naïve PK design (e.g., `customerId`), processing jobs that sweep all records for a given day will cause **full table scans** consuming RCUs proportional to item size × count. A 2,500-field item stored flat in DynamoDB would be hundreds of KB per item — making a daily scan **extremely expensive and slow**. [dev](https://dev.to/rahulladumor/how-to-optimize-dynamodb-costs-a-developers-complete-guide-2e9j)

### Final Recommendation: Dual-Purpose Data Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  DynamoDB (Primary Operational Store)                       │
│  • UI state, workflow transitions, real-time CRUD           │
│  • 2500-field full KYC items                                │
│  • Global Tables for active-active                          │
├─────────────────────────────────────────────────────────────┤
│  DynamoDB Projection Table (Processing-Optimized)           │
│  • 150 fields ONLY                                          │
│  • Separate table, populated via DynamoDB Streams + Lambda  │
│  • GSIs tuned for batch processing access patterns          │
├─────────────────────────────────────────────────────────────┤
│  Aurora PostgreSQL Serverless v2 (Processing Replica)       │
│  • Populated from S3/Parquet snapshot (nightly or near-RT)  │
│  • Used ONLY for complex batch query logic                  │
│  • NOT a primary write target                               │
├─────────────────────────────────────────────────────────────┤
│  S3 Data Lake (Parquet / Iceberg)                           │
│  • Source of truth for analytics, external APIs             │
│  • Glue Catalog, versioned partitions                       │
└─────────────────────────────────────────────────────────────┘
```

**Use DynamoDB for**: real-time UI, workflow state, event-driven triggers, low-latency lookups.
**Use Aurora for**: batch processing queries requiring joins, aggregations, or complex filter predicates against the 150-field projection.
**Use S3/Data Lake for**: external API consumers, analytics, audit trails.

***

## 2. DynamoDB Query Design for 300k+ Daily Processing

### Table Design Philosophy

**Adopt single-table design for the operational KYC table** and a **separate projection table** for processing. Single-table design co-locates entity types under one PK, enabling efficient transactional reads for the UI without cross-table joins. [dev](https://dev.to/rahulladumor/how-to-optimize-dynamodb-costs-a-developers-complete-guide-2e9j)

### Primary KYC Table — PK/SK Design

```
Table: KYC_OPERATIONAL
PK (Partition Key): ENTITY#<customerId>
SK (Sort Key):      KYC#<version>#<timestamp-ISO8601>

Example:
PK: ENTITY#CUST-00123456
SK: KYC#v3#2026-03-28T10:00:00Z
```

**Composite PK rationale**: keeps all versions of a KYC record co-located for transactional reads by the UI, prevents hot partitions because customer IDs distribute across partitions, and enables range queries by version/timestamp using SK begins_with or between.

### Processing Projection Table — 150 Fields Only

```
Table: KYC_PROCESSING_PROJECTION
PK: PROCESSING_DATE#<YYYY-MM-DD>#SHARD#<0-9>   ← date + shard suffix to avoid hot partitions
SK: CUSTOMER#<customerId>

Projected Attributes: [150 fields only — see attribute projection section]

GSI-1: 
  PK: STATUS#<kycStatus>
  SK: REVIEW_DUE_DATE#<ISO8601>
  → Used for periodic review triggers

GSI-2:
  PK: RISK_TIER#<HIGH|MEDIUM|LOW>
  SK: PROCESSING_DATE#<YYYY-MM-DD>
  → Used for risk-stratified processing

GSI-3 (Sparse):
  PK: TRIGGER_TYPE#<PERPETUAL|PERIODIC|EVENT>
  SK: TRIGGER_DATE#<ISO8601>
  → Only populated when a trigger exists — sparse index reduces index size dramatically
```

### Avoiding Hot Partitions at 300k/Day

The date-based shard PK is critical: [dev](https://dev.to/rahulladumor/how-to-optimize-dynamodb-costs-a-developers-complete-guide-2e9j)

```
PK: PROCESSING_DATE#2026-03-28#SHARD#3
```

Use 10 shards (0–9). The batch processor queries all 10 shards in parallel using `BatchGetItem` or parallel `Query` calls. This distributes 300k records across 10 partitions = ~30k records per partition, well within DynamoDB's per-partition throughput limit of 3,000 RCU/s.

### Attribute Projection Strategy

Define explicit ProjectionExpression on all Query calls. Never use `ALL` attributes projection in GSIs for the processing table — only project the 150 required fields. In GSI definition:

```java
// Java SDK v2 example
QueryRequest request = QueryRequest.builder()
    .tableName("KYC_PROCESSING_PROJECTION")
    .keyConditionExpression("PK = :pk")
    .expressionAttributeValues(Map.of(
        ":pk", AttributeValue.fromS("PROCESSING_DATE#2026-03-28#SHARD#3")
    ))
    .projectionExpression("customerId, riskScore, kycStatus, reviewDueDate, ...")
    .build();
```

This eliminates read amplification by never deserializing the 2,350 unused fields. [dev](https://dev.to/rahulladumor/how-to-optimize-dynamodb-costs-a-developers-complete-guide-2e9j)

### Pagination Strategy for 300k Records

DynamoDB `Query` returns at most 1 MB per call. With 150-field items averaging ~2 KB, each page returns ~500 items. Processing 300k items = ~600 paginated Query calls per shard × 10 shards.

```
Pattern: Parallel Paginated Query per Shard
┌─────────────────────────────────────────────────────┐
│  EventBridge Scheduler triggers Step Functions      │
│       ↓                                             │
│  Map State: Fan-out across 10 shards                │
│       ↓ (parallel)                                  │
│  Lambda per shard: paginated Query loop             │
│  while (lastEvaluatedKey != null):                  │
│    page = query(shard, exclusiveStartKey)           │
│    publish to SQS batch queue                       │
│       ↓                                             │
│  Lambda consumers: process SQS batches             │
└─────────────────────────────────────────────────────┘
```

### Time-Window Partitioning for Scheduled Processing

Populate the projection table during ingestion (not at query time). When a KYC record is written to the operational table, a DynamoDB Streams → Lambda fan-out immediately writes the 150-field projection with `PROCESSING_DATE#<today>#SHARD#<hash(customerId) % 10>` as PK. This ensures the projection table is pre-built and queryable at zero scan cost. [aws.amazon](https://aws.amazon.com/blogs/database/build-scalable-event-driven-architectures-with-amazon-dynamodb-and-aws-lambda/)

### Strategies to Avoid Full Table Scans

| Pattern | Implementation | Use Case |
|---|---|---|
| **Date-shard index** | PK = date + shard | Daily batch |
| **Sparse GSI** | Attribute only set when trigger fires | Periodic review, event KYC |
| **TTL-based eviction** | TTL on projection records = processing_date + 7 days | Auto-cleanup |
| **Status index (GSI-1)** | Query by status + date range | Workflow-based processing |
| **Event markers** | Boolean flag `needsProcessing = true` + sparse GSI | Re-processing, error recovery |

***

## 3. Architecture Options for Daily + Event-Driven Processing

### Option A — Read Directly from DynamoDB

**What**: Step Functions triggers Lambda workers that query `KYC_PROCESSING_PROJECTION` via parallel shard queries.

**How**: 10-way fan-out, paginated Query, process in-Lambda, write results back.

| Dimension | Assessment |
|---|---|
| **Cost** | High RCU cost at 300k records; ~600K RCUs/day per shard run |
| **Performance** | Parallel sharding achieves ~30 min for 300k at Lambda concurrency 100 |
| **Complexity** | Medium — needs careful shard design |
| **Multi-region** | Each region processes its own Global Table replica independently |
| **Recommendation** | ✅ Viable with projection table + shard design; not viable with raw operational table |

### Option B — DynamoDB → S3 (Columnar) → Process

**What**: Export DynamoDB projection table to S3 in Parquet via DynamoDB Export to S3 (PITR-based, zero RCU cost), then process with Lambda or Glue.

**How**:
```
DynamoDB Export API → S3 (JSON) → Glue Job (Java/Spark) → Parquet → Lambda/EMR processing
```

| Dimension | Assessment |
|---|---|
| **Cost** | DynamoDB Export = $0.10/GB, no RCU cost; Glue compute cost applies |
| **Performance** | Export is asynchronous (~15 min for large tables); not for near-real-time |
| **Complexity** | Higher — requires Glue catalog, Parquet conversion job |
| **Multi-region** | Export per region; S3 CRR for centralized processing |
| **Recommendation** | ✅ Best for daily bulk batch; poor for intraday or event-driven |

### Option C — Aurora PostgreSQL Replica for Batch Reads

**What**: Maintain an Aurora PostgreSQL Serverless v2 table populated by CDC from DynamoDB Streams. Batch jobs run SQL against Aurora.

**How**:
```
DynamoDB Streams → Lambda → Aurora PostgreSQL (150-field table)
                                   ↓
                         Step Functions → Lambda → SQL query → process
```

| Dimension | Assessment |
|---|---|
| **Cost** | Aurora Serverless v2 ~$0.12/ACU-hr; cost-efficient at scheduled scale-up/scale-down |
| **Performance** | SQL query on indexed date column = sub-second for 300k rows |
| **Complexity** | Medium — CDC pipeline from DynamoDB to Aurora required  [aws.amazon](https://aws.amazon.com/blogs/database/continuously-replicate-amazon-dynamodb-changes-to-amazon-aurora-postgresql-using-aws-lambda/) |
| **Multi-region** | Aurora Global DB for cross-region; adds latency |
| **Recommendation** | ✅ Best pattern for complex batch queries; use Aurora as read-optimized processing layer only |

### Option D — Event-Sourced KYC Pipeline with Materialized Views

**What**: KYC state is built from an immutable event log (EventBridge + S3). Materialized views (DynamoDB or Aurora) are projections.

**How**:
```
KYC Domain Events → EventBridge → S3 Event Store (Parquet/Iceberg)
                                       ↓
                              Glue Streaming / Lambda → Materialized Views
```

| Dimension | Assessment |
|---|---|
| **Cost** | S3 event store is cheap; Glue streaming adds cost |
| **Performance** | Replay latency for corrections; excellent for audit |
| **Complexity** | Highest — requires full event sourcing discipline |
| **Multi-region** | S3 CRR + Iceberg metadata replication |
| **Recommendation** | ✅ Long-term ideal for perpetual KYC; phased adoption recommended |

### Option E — Dual-Write into DynamoDB + S3

**What**: Application writes to both DynamoDB and S3 atomically (via Lambda transactional pattern).

**How**:
```
Java Lambda write handler:
  1. Write full item to DynamoDB
  2. Write 150-field Parquet record to S3 (partitioned by date)
  3. Publish event to EventBridge
```

| Dimension | Assessment |
|---|---|
| **Cost** | Low incremental cost; S3 write is cheap |
| **Performance** | Sub-100ms write path; S3 writes are async via SQS buffer |
| **Complexity** | Low-medium; idempotency requires careful design |
| **Multi-region** | S3 CRR handles cross-region; DynamoDB Global Tables for operational |
| **Recommendation** | ✅ Best operational simplicity for immediate delivery |

### Option F — Data Lake–Driven Processing

**What**: All processing reads from S3 Data Lake (Parquet/Iceberg). Glue or EMR Serverless executes batch jobs.

**How**:
```
S3 (Iceberg KYC table) → Glue Job (Java/Spark) or EMR Serverless → results → DynamoDB write-back
```

| Dimension | Assessment |
|---|---|
| **Cost** | Glue $0.44/DPU-hr; EMR Serverless flexible; 300k records = ~10 DPU × 30 min = ~$2.20/run |
| **Performance** | Excellent for large-scale; 300k Parquet records processed in minutes |
| **Complexity** | Medium — Glue catalog maintenance, Iceberg compaction required |
| **Multi-region** | S3 CRR + Iceberg snapshot replication |
| **Recommendation** | ✅ Best for future-proofing, analytics, external APIs |

### Recommended Combination

```
Day-0 (Now):       Option E (Dual-Write) + Option A (Projection Table Query)
Day-90 (Phase 2):  Option C (Aurora Replica) for complex batch logic
Day-180 (Phase 3): Option F (Data Lake) as primary processing source
Long-term:         Option D (Event Sourcing) for perpetual KYC
```

***

## 4. Full End-to-End Target Architecture

### Component Map

```
┌──────────────────────────────────────────────────────────────────────┐
│                    MULTI-REGION ACTIVE-ACTIVE (us-east-1 / eu-west-1)│
│                                                                      │
│  ┌─────────────────┐    ┌─────────────────────────────────────────┐  │
│  │   React UI       │    │          API Gateway (Regional)         │  │
│  │  (CloudFront)    │───▶│  /kyc  /workflow  /review  /external   │  │
│  └─────────────────┘    └──────────────┬────────────────────────┘  │
│                                         │                            │
│              ┌──────────────────────────▼──────────────────┐        │
│              │         Lambda (Java 21 / SnapStart)        │        │
│              │   KYC-Service  Workflow-Service  Rule-Engine │        │
│              └──────┬─────────────────┬────────────────────┘        │
│                     │                 │                              │
│         ┌───────────▼──┐    ┌─────────▼──────────┐                 │
│         │  DynamoDB     │    │   EventBridge Bus   │                 │
│         │  Global Tables│    │   (default + KYC)   │                 │
│         │  (active-act.)│    └─────────┬───────────┘                │
│         └───────┬───────┘              │                             │
│                 │ Streams               │ Rules                       │
│         ┌───────▼──────────────────────▼──────────┐                 │
│         │         Step Functions (Express)         │                 │
│         │  KYC-Processing  Review-Workflow         │                 │
│         └───────────────────┬─────────────────────┘                 │
│                             │                                        │
│              ┌──────────────▼──────────────────┐                    │
│              │    S3 Data Lake                  │                    │
│              │    /raw  /curated  /processed    │                    │
│              │    Iceberg / Parquet             │                    │
│              └──────────────┬──────────────────┘                    │
│                             │                                        │
│         ┌───────────────────▼──────────────────────────────┐        │
│         │  Glue Catalog + EMR Serverless / Glue Jobs        │        │
│         │  Aurora PostgreSQL Serverless v2 (batch replica)  │        │
│         └───────────────────┬──────────────────────────────┘        │
│                             │                                        │
│              ┌──────────────▼────────────────┐                      │
│              │  External API (API GW + Lambda) │                     │
│              │  Data Lake–backed via Athena    │                     │
│              └───────────────────────────────┘                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Ingestion Flow (File-Based Inbound KYC Data)

```
Step 1: Inbound file → S3 (raw zone) → EventBridge notification
Step 2: EventBridge → Lambda (file validator + schema mapper)
Step 3: Lambda → writes validated records to SQS FIFO queue (per-customer grouping)
Step 4: SQS → Lambda (KYC Ingestor) → DynamoDB write (full 2500-field item)
Step 5: DynamoDB Streams → Lambda fan-out:
         a. Write 150-field projection to KYC_PROCESSING_PROJECTION
         b. Write Parquet record to S3 Data Lake (/raw/kyc/YYYY/MM/DD/)
         c. Publish KYCIngestedEvent to EventBridge
Step 6: EventBridge → Step Functions (if immediate processing required)
```

### Daily Scheduled Processing Flow

```
Step 1:  EventBridge Scheduler (cron: 02:00 UTC) → Step Functions (Standard Workflow)
Step 2:  Step Functions: GetShardConfig (Lambda) → determines today's 10 shard PKs
Step 3:  Map State: parallel execution across 10 shards
Step 4:  Per shard Lambda: paginated Query KYC_PROCESSING_PROJECTION
         → batch publish to SQS (batchSize=25)
Step 5:  SQS → Lambda (KYC Processor): 
         → apply business rules (Rule Engine)
         → call external enrichment APIs (if needed)
         → compute risk score / AML flags
Step 6:  Lambda writes results:
         a. DynamoDB: update KYC status + processing output
         b. S3: append processed record to Data Lake (/processed/kyc/)
         c. EventBridge: publish KYCProcessedEvent
Step 7:  Step Functions: aggregate results, publish metrics to CloudWatch
Step 8:  SNS alert if failure rate > threshold
```

### React UI CRUD + Workflow Flow

```
User action → API Gateway → Lambda (KYC Write Service)
  → DynamoDB TransactWrite (optimistic locking via version attribute)
  → EventBridge: KYCUpdatedEvent
  → Step Functions (if workflow transition): ValidateTransition → ApplyTransition → NotifyDownstream
```

### External Consumer API Flow (Data Lake–Backed)

```
External system → API Gateway → Lambda (Data API Service)
  → Athena query against Glue Catalog (S3 Parquet/Iceberg table)
  → result cached in ElastiCache (5-min TTL for repeated queries)
  → response returned with pagination token
```

***

## 5. Business Rules Rewrite Strategy

### Stored Procedure → Domain Logic Migration

**Hexagonal Architecture** applied to KYC Domain:

```
┌──────────────────────────────────────────────────────┐
│  Driving Adapters (Inbound)                          │
│  API Gateway Handler | SQS Consumer | EventBridge   │
│               ↓                                      │
│  Application Services (Use Cases)                    │
│  ProcessKycUseCase | TriggerReviewUseCase            │
│               ↓                                      │
│  Domain Core                                         │
│  KycAggregate | RiskScoringService | RuleEngine     │
│  KycStatus (value object) | ReviewPolicy            │
│               ↓                                      │
│  Driven Adapters (Outbound)                          │
│  DynamoDbKycRepository | S3EventStore               │
│  EventBridgePublisher | ExternalEnrichmentClient    │
└──────────────────────────────────────────────────────┘
```

### Java Lambda Code Structure

```
kyc-service/
├── application/
│   ├── usecase/ProcessKycUseCase.java
│   └── usecase/TriggerReviewUseCase.java
├── domain/
│   ├── model/KycAggregate.java
│   ├── model/KycStatus.java
│   ├── service/RiskScoringDomainService.java
│   ├── service/RuleEngine.java
│   └── port/
│       ├── inbound/KycProcessingPort.java
│       └── outbound/KycRepository.java
├── infrastructure/
│   ├── adapter/inbound/SqsKycHandler.java
│   ├── adapter/inbound/ApiGatewayKycHandler.java
│   ├── adapter/outbound/DynamoDbKycRepository.java
│   ├── adapter/outbound/S3EventStoreAdapter.java
│   └── adapter/outbound/EventBridgePublisher.java
└── config/
    └── DependencyConfig.java (manual DI — no Spring context overhead in Lambda)
```

### Rule Versioning Strategy

```
Table: KYC_RULE_VERSIONS
PK: RULE#<ruleName>
SK: VERSION#<semver>
Attributes: ruleDefinition (JSON), effectiveDate, deprecatedDate, checksum
```

Rules are loaded at Lambda cold start into static cache. `RuleEngine` selects version by KYC effective date, enabling retroactive replay correctness.

### Idempotency

Every Lambda handler uses a DynamoDB idempotency table keyed on `requestId` (from SQS MessageId or API Gateway request ID). Use the [AWS Lambda Powertools for Java idempotency module](https://docs.powertools.aws.dev/lambda/java/utilities/idempotency/) pattern:

```java
@Idempotent
public KycProcessingResult handleRequest(KycProcessingRequest request, Context context) {
    // Safe to retry — duplicate invocations return cached result
}
```

### Perpetual KYC Support

The domain model carries a `ReviewPolicy` value object that determines next-review triggers (time-based, event-based, risk-tier change). On every KYC write, `ReviewPolicy.computeNextReviewDate()` is evaluated and written to `GSI-1 (STATUS + REVIEW_DUE_DATE)`. A separate EventBridge Scheduler (daily) queries the sparse GSI and triggers review workflows for due records.

***

## 6. Data Flow & Sequence Designs

### Multi-Region Conflict Resolution Sequence

```
T=0ms   User A (us-east-1): PUT KYC#CUST-001, version=5, status=APPROVED
T=10ms  User B (eu-west-1): PUT KYC#CUST-001, version=5, status=REJECTED (conflict!)
T=~1s   DynamoDB Global Tables replicates both writes
         Last-writer-wins (by aws:rep:updatetime) [web:6]
         → eu-west-1 write (T=10ms) wins if it arrives last

Mitigation: Use optimistic locking with a condition expression:
  ConditionExpression: "version = :expectedVersion"
  → Reject concurrent conflicting writes at write time
  → EventBridge publishes KYCConflictDetectedEvent for human review
```

### DynamoDB → S3 Projection Sequence

```
1. KYC item written to DynamoDB Operational Table
2. DynamoDB Stream (NEW_IMAGE) → Lambda (ProjectionBuilder)
3. Lambda selects 150 fields from NEW_IMAGE
4. Writes to KYC_PROCESSING_PROJECTION (DynamoDB)
5. Serializes to Parquet record, writes to S3:
   s3://kyc-datalake/raw/kyc/year=2026/month=03/day=28/shard=3/<uuid>.parquet
6. Glue Crawler (hourly) updates Glue Catalog metadata
```

### Periodic Review KYC Trigger Sequence

```
1. EventBridge Scheduler: daily 06:00 UTC
2. Lambda (ReviewScheduler): Query GSI-1 where STATUS=ACTIVE AND REVIEW_DUE_DATE <= today
3. Batch publish KYCReviewDueEvent per record to SQS FIFO
4. Lambda consumer: start Step Functions KYC-Review workflow per customer
5. Step Functions:
   a. FetchCurrentKYC (Lambda)
   b. CheckExternalSources (Lambda — PEP/Sanctions)
   c. EvaluateRiskChange (Lambda — RuleEngine)
   d. UpdateKycStatus (Lambda — DynamoDB write)
   e. NotifyCompliance (Lambda — SNS/SES)
```

***

## 7. Multi-Region Active-Active Architecture

### Core Principles

DynamoDB Global Tables provides multi-region, multi-active replication with asynchronous eventual consistency and last-writer-wins conflict resolution. All application writes go to the local regional replica; DynamoDB replicates within ~1 second to all other regions. [docs.aws.amazon](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)

### Regional Architecture Per Region

```
Each region (us-east-1, eu-west-1) contains:
  - API Gateway (Regional endpoint)
  - Lambda functions (same code, region-specific config)
  - DynamoDB replica (Global Table)
  - EventBridge Bus (regional, no native cross-region sync — use custom bus forwarding)
  - Step Functions (independent per region, triggered by regional events)
  - S3 bucket (with CRR to partner region for Data Lake)
  - SQS queues (regional)
```

### Multi-Region EventBridge Strategy

EventBridge does not natively replicate across regions. Implement cross-region event forwarding: [aws.amazon](https://aws.amazon.com/blogs/database/how-to-use-amazon-dynamodb-global-tables-to-power-multiregion-architectures/)

```
us-east-1 EventBridge → Lambda (cross-region forwarder) → eu-west-1 EventBridge
```

Use EventBridge Pipes for filter-then-forward patterns. Only forward events that require cross-region coordination (e.g., `KYCConflictDetectedEvent`, `GlobalComplianceAlertEvent`).

### S3 Cross-Region Replication

```
us-east-1 S3 (primary Data Lake)
    ↓ CRR (replication rule: prefix=kyc/)
eu-west-1 S3 (replica Data Lake)

Both regions maintain independent Glue Catalogs pointing to local S3.
Athena queries in each region use local S3 — no cross-region read latency.
```

### Global API Strategy

```
Route 53 Latency-Based Routing:
  api.kyc.company.com → routes to lowest-latency regional API Gateway

Health checks: Route 53 health checks on /health endpoint
Failover: if primary region health check fails, Route 53 routes to secondary
RTO: ~30 seconds (Route 53 TTL + propagation)
RPO: ~1 second (DynamoDB Global Tables replication lag)
```

***

## 8. Development & Operational Excellence

### Java Lambda Best Practices

- **Use Java 21 + SnapStart** for all Lambda functions — reduces cold start from 3–8 sec to ~200ms
- **No Spring Boot** — use manual DI or Dagger 2 for Lambda; Spring context init is too slow
- **GraalVM native** as future optimization target once team is proficient
- **Lambda layers** for shared domain model JARs

### CICD Pipeline

```
GitHub (mono-repo per domain) → GitHub Actions:
  1. Unit Tests (JUnit 5 + Mockito)
  2. Integration Tests (Testcontainers: DynamoDB Local, LocalStack)
  3. Contract Tests (Pact JVM)
  4. SAM build + CDK synth
  5. Deploy to dev → integration test suite
  6. Deploy to staging → load test (Gatling)
  7. Manual approval → deploy to prod (us-east-1 first, then eu-west-1)
```

### Monitoring & Observability

| Signal | Tool | KPIs |
|---|---|---|
| **Metrics** | CloudWatch Custom Metrics | KYCs processed/min, failure rate, SLA breach |
| **Tracing** | X-Ray + OpenTelemetry | End-to-end latency per KYC flow |
| **Logging** | CloudWatch Logs + JSON structured | Correlation ID, customerId, ruleVersion |
| **Alarms** | CloudWatch Alarms → SNS | P99 latency > 5s, DLQ depth > 100 |
| **Dashboards** | CloudWatch Dashboard | Regional processing throughput, error rates |

### Retry & DLQ Strategy

```
SQS Standard Queue:
  maxReceiveCount: 3
  visibilityTimeout: 30s × (attempt number) — exponential backoff
  DLQ: kyc-processing-dlq
  DLQ alarm: CloudWatch → SNS → PagerDuty

Lambda async invocations:
  maximumRetryAttempts: 2
  onFailure destination: SQS DLQ

Step Functions:
  Retry: maxAttempts=3, intervalSeconds=5, backoffRate=2.0, jitterStrategy=FULL
  Catch: → FAILED state → SNS alert + store failure event to S3
```

### Testing Strategy

| Level | Framework | Scope |
|---|---|---|
| **Unit** | JUnit 5 + Mockito | Domain logic, rule engine, pure functions |
| **Integration** | Testcontainers + LocalStack | DynamoDB, S3, SQS, EventBridge locally |
| **Contract** | Pact JVM | API consumers vs. Lambda providers |
| **Load** | Gatling (Java DSL) | 300k KYC simulation, latency under load |
| **Replay** | Custom test harness | Replay historical KYC events against new rule version |

***

## 9. Migration Plan

### Phase 1 — Foundation (Weeks 1–8)

| Action | Tool |
|---|---|
| Set up AWS Landing Zone (multi-region, multi-account) | AWS Control Tower |
| Define CDK infrastructure-as-code baseline | AWS CDK (Java) |
| Deploy DynamoDB Global Tables (empty) | CDK |
| Deploy S3 Data Lake structure + Glue Catalog | CDK |
| Implement dual-write proxy in front of MS SQL | Java Lambda shim |

### Phase 2 — Parallel Run (Weeks 9–20)

| Action | Detail |
|---|---|
| MS SQL → DynamoDB migration | AWS DMS (full load + CDC) |
| Stored procedure extraction | Map each SP to a Java domain service; document rule-by-rule |
| Autosys → EventBridge Scheduler | 1:1 job mapping; validate output equivalence |
| Python → Java rewrite | Port processing logic using TDD; validate with replay tests |
| Shadow mode processing | New AWS pipeline runs in parallel; compare outputs to MS SQL baseline |

### Phase 3 — Cutover (Weeks 21–24)

```
Week 21: Route 10% of write traffic to new system (feature flag)
Week 22: 50% traffic; monitor CloudWatch dashboards; compare results
Week 23: 100% traffic; MS SQL in read-only mode
Week 24: Decommission MS SQL; remove dual-write shim; archive data to S3 Glacier
```

### Autosys → EventBridge Scheduler Migration Map

```
Autosys Job          → EventBridge Scheduler + Step Functions
Daily KYC Batch      → cron(0 2 * * ? *) → KYC Processing State Machine
Weekly Review Check  → cron(0 6 ? * MON *) → Review Trigger State Machine
File Pickup Job      → S3 Event Notification → Lambda (replaced by EDA)
```

***

## 10. Final Recommendation

### Architecture Decision Summary

| Concern | Recommendation |
|---|---|
| **Primary data store** | DynamoDB Global Tables (operational + workflow) |
| **Batch processing source** | KYC_PROCESSING_PROJECTION table (150 fields, sharded by date) |
| **Complex batch queries** | Aurora PostgreSQL Serverless v2 (CDC replica from DynamoDB) |
| **Data Lake** | S3 + Iceberg + Glue Catalog (Parquet, partitioned by date/risk tier) |
| **External APIs** | API Gateway + Lambda → Athena → S3 Data Lake |
| **Orchestration** | Step Functions (Standard for KYC workflows; Express for high-volume pipelines) |
| **Event bus** | EventBridge (regional; cross-region forwarding Lambda for global events) |
| **Language** | Java 21 + SnapStart Lambdas; Hexagonal architecture |
| **Multi-region** | Active-active via Global Tables + Route 53 latency routing |
| **Business rules** | Java domain services in Lambda; rule version table in DynamoDB |

### Performance Expectations

- **300k KYCs/day**: ~3.5 KYCs/second average; shard-based parallel processing handles 10× spikes
- **Projection table query**: 10 parallel Lambda workers complete full 300k record query in ~8–12 minutes
- **Lambda latency (SnapStart)**: P50 < 50ms, P99 < 500ms for CRUD operations
- **DynamoDB write throughput**: 10,000+ WCU/s per region (on-demand mode)
- **Data Lake API (Athena)**: P50 < 2s for standard external consumer queries with partition pruning

### Cost Estimate (Monthly, Single Region)

| Component | Estimate |
|---|---|
| DynamoDB (on-demand, 300k writes + reads/day) | ~$400–800 |
| Lambda (300k invocations × avg 500ms × 512MB) | ~$50–150 |
| Step Functions (Express, 300k executions) | ~$30 |
| S3 Data Lake (1 TB/month ingestion + storage) | ~$50 |
| Glue Jobs (2 DPU × 1hr/day) | ~$30 |
| Aurora Serverless v2 (2 ACU min, batch window) | ~$200 |
| EventBridge, SQS, SNS, API Gateway | ~$100–200 |
| **Total (single region)** | **~$860–1,410/month** |
| **Two-region active-active** | **~$1,800–3,000/month** |

### Future-Proofing: Perpetual KYC + Data Mesh

The architecture natively supports perpetual KYC by treating every data change as a first-class domain event stored immutably in S3. External API consumers are fully decoupled from operational DynamoDB via Data Lake–backed APIs. As the Data Mesh matures, domain teams publish their KYC data products as Iceberg tables with schema contracts enforced via Glue Schema Registry, enabling federated external consumption without coupling to the KYC operational database. [docs.aws.amazon](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html)

The event-sourced pipeline (Option D) should be the architectural north star — adopt it incrementally starting with the highest-churn KYC domains (e.g., periodic reviews and perpetual triggers) while the simpler dual-write pattern handles the remainder during migration.