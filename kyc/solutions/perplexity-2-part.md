# AWS Migration & Target Architecture Design Document
### Migrating Unix/Autosys/MS SQL → AWS-Native Event-Driven Platform (Java)
---
## 1. Executive Context & Scope
The current platform consists of Python batch jobs on Unix servers, triggered by Autosys, reading/writing from MS SQL Server with business logic embedded in stored procedures. The target state fully decommissions MS SQL, Autosys, and Unix in favor of a Java-based, AWS-native, event-driven architecture following Hexagonal Architecture (Ports & Adapters) and Domain-Driven Design (DDD). [docs.aws.amazon](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/structure-a-python-project-in-hexagonal-architecture-using-aws-lambda.html)

**Guiding Principles:**
- All core processing moves to AWS; on-prem access only for unavoidable upstream dependencies
- MS SQL data fully migrated to AWS-native stores (DynamoDB + S3)
- Stored procedures rewritten as Java domain services
- Python replaced by Java (team skill alignment)
- Long-term goal: zero on-prem footprint

***
## 2. Architecture Options — Deep Comparative Analysis
### Option A: Full Event-Driven Architecture (EDA)
| Dimension | Detail |
|---|---|
| **Why** | Decouples producers from consumers; enables reactive, independently deployable domain services |
| **What** | EventBridge custom event bus, Lambda consumers, DynamoDB event store, SQS for fan-out |
| **How** | Domain events published to EventBridge → routed to Lambda handlers → write to DynamoDB |
| **Pros** | Maximum decoupling; zero-polling; natural audit trail; scales to zero |
| **Cons** | Complex debugging; eventual consistency; event versioning overhead |
| **Cost Model** | Pay-per-event; EventBridge $1/million events; Lambda per-invocation |
| **Scaling** | Automatic, unbounded; Lambda concurrency limits apply |
| **Observability** | X-Ray traces per invocation; CloudWatch structured JSON logs; EventBridge pipe dead-letters |
| **Use When** | Real-time KYC triggers, downstream workflow notifications, domain state change propagation |
| **Don't Use When** | Strict ordering required across 3000+ fields in a single atomic transaction |
### Option B: Hybrid EDA + Scheduled Batch
| Dimension | Detail |
|---|---|
| **Why** | Not all workloads benefit from pure EDA; daily upstream pulls are inherently batch |
| **What** | EventBridge Scheduler triggers Step Functions for batch; EDA for real-time flows |
| **How** | Cron rules in EventBridge Scheduler → Step Functions state machine → Lambda chain → DynamoDB |
| **Pros** | Pragmatic; replicates Autosys behavior natively; easier team onboarding |
| **Cons** | Two runtime paradigms to maintain |
| **Cost Model** | EventBridge Scheduler $0.00864/execution; Step Functions Standard $0.025/1000 state transitions |
| **Use When** | Daily upstream ingestion jobs, periodic KYC review batch |
### Option C: Step Functions–Driven Orchestration
| Dimension | Detail |
|---|---|
| **Why** | Replaces Autosys job chaining with visual, auditable, retryable workflow graphs  [aws.amazon](https://aws.amazon.com/solutions/guidance/scheduling-batch-jobs-for-aws-mainframe-modernization/) |
| **What** | AWS Step Functions Standard Workflows with Lambda task states |
| **How** | State machine defines job DAG; each state invokes a Java Lambda; Choice states replace IF/ELSE in SPs |
| **Pros** | Built-in retry/backoff; visual debugger; execution history; integrates natively with 200+ AWS services |
| **Cons** | 25k event limit per execution (use Express for high-volume); state payload 256KB limit |
| **Cost Model** | Standard: $0.025/1000 transitions; Express: $1/million + duration |
| **Use When** | Multi-step enrichment pipelines, KYC workflow orchestration |
### Option D: S3 Event → Lambda → DynamoDB
| Dimension | Detail |
|---|---|
| **Why** | Direct replacement for file-reading Python jobs |
| **What** | Files dropped to S3 → S3 EventBridge notification → Lambda → parse → DynamoDB write |
| **How** | S3 bucket with EventBridge notifications enabled; Lambda in Java reads S3 object, applies domain rules, writes to DynamoDB |
| **Pros** | Fully serverless; S3 acts as durable file store and trigger; no polling |
| **Cons** | Lambda 15-min timeout for very large files; cold start latency |
| **Use When** | Daily upstream file ingestion, processed result file generation |
### Option E: Lambda + SQS Pipeline
| Dimension | Detail |
|---|---|
| **Why** | Buffer between producers/consumers; handles spikes and back-pressure |
| **What** | SQS Standard or FIFO queues; Lambda event source mappings |
| **How** | Upstream writes messages to SQS → Lambda polls in batches → processes → writes DynamoDB |
| **Pros** | Decoupled; at-least-once delivery; DLQ for failures; batch window tuning |
| **Cons** | FIFO limited to 300 TPS per message group; SQS Standard has no ordering guarantees |
| **Use When** | High-volume record processing with variable load |
### Option F: Kinesis Data Streams
| Dimension | Detail |
|---|---|
| **Why** | Ordered, replayable streaming for continuous ingestion from upstream |
| **What** | Kinesis Data Streams → Lambda consumers → DynamoDB |
| **Pros** | Strict ordering per shard; replay up to 7 days; fan-out to multiple consumers |
| **Cons** | Shard management overhead; cost scales with shards even at idle |
| **Use When** | Continuous upstream CDC feeds from on-prem (if unavoidable); NOT needed for pure daily batch |
### Option G: AWS Glue ETL + Glue Workflows
| Dimension | Detail |
|---|---|
| **Why** | Best for bulk historical data migration and large-scale transformation |
| **What** | Glue Crawlers catalog S3/DMS-migrated data; Glue Jobs (Spark) transform; write to DynamoDB or S3 |
| **How** | DMS full-load to S3 → Glue Crawler → Glue ETL Job → target DynamoDB table |
| **Pros** | Handles terabytes; built-in Data Catalog; serverless Spark |
| **Cons** | Cold start 60–90 seconds; not suited for sub-second latency; Spark overhead for small data |
| **Use When** | One-time or periodic bulk migration; historical data backfill |
### Option H: DynamoDB Streams for Reactive Events
| Dimension | Detail |
|---|---|
| **Why** | Triggers downstream domain events on data mutations without application coupling |
| **What** | DynamoDB Streams → Lambda → EventBridge or SQS |
| **How** | Enable streams on domain table; Lambda processes INSERT/MODIFY records; publishes domain events |
| **Pros** | Zero-latency change capture; no polling; ordered per partition |
| **Use When** | KYC trigger on data write, audit trail generation, downstream notification |
### Option I: ECS Fargate Batch Containers
| Dimension | Detail |
|---|---|
| **Why** | Required only when Lambda limits (15 min, 10GB memory) are exceeded |
| **What** | Docker containers (Java) on ECS Fargate; AWS Batch for job queuing |
| **Pros** | Unlimited runtime; full JVM control; supports large heap |
| **Cons** | Higher cost; longer startup; more operational overhead |
| **Use When** | Large file processing >10GB; jobs requiring >15 minutes runtime |

***
## 3. MS SQL → AWS Data Migration Design
### Migration Strategy Selection
```
MS SQL Server
    │
    ├── [Full Load Phase]  ──► AWS DMS → S3 (Parquet) → Glue ETL → DynamoDB
    │
    ├── [CDC Phase]        ──► AWS DMS CDC → Kinesis/SQS → Lambda → DynamoDB
    │
    └── [Historical Data]  ──► Glue Spark Job → S3 Data Lake (Parquet/ORC)
```

AWS DMS natively supports MS SQL Server as a source, including CDC via SQL Server transaction logs. DMS does **not** automatically migrate secondary indexes, foreign keys, or stored procedures — these must be handled separately. [docs.aws.amazon](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_BestPractices.html)
### Migration Architecture Detail
**Phase 1 — Schema Conversion:**
- Use **AWS Schema Conversion Tool (SCT)** to assess MS SQL → DynamoDB or Aurora mapping
- For relational tables with complex joins: evaluate Aurora PostgreSQL Serverless v2 as interim target before final DynamoDB migration
- For domain aggregate tables: design DynamoDB single-table model per bounded context

**Phase 2 — Full Load:**
- DMS Replication Instance (r5.xlarge minimum for large tables) performs full load to S3 as Parquet [aws.amazon](https://aws.amazon.com/blogs/database/aws-dms-best-practices-for-moving-large-tables-with-table-parallelism-settings/)
- Enable DMS table-level parallelism for large tables (LOB handling: limited LOB mode, 32KB)
- Glue Crawler catalogs S3 output → Glue Job transforms → loads DynamoDB via `BatchWriteItem`

**Phase 3 — CDC (Change Data Capture):**
```
MS SQL (with supplemental logging enabled)
    └──► DMS CDC Task
             └──► Kinesis Data Stream (ordered)
                      └──► Lambda Consumer
                               └──► DynamoDB (conditional writes for idempotency)
```

**Phase 4 — Validation:**

| Validation Type | Tool/Method |
|---|---|
| Row count parity | DMS built-in table statistics + custom Lambda reconciliation |
| Hash-based row validation | AWS Glue DataBrew or custom Spark job |
| Business rule output parity | Parallel run: run both old SP and new Lambda; compare outputs |
| Referential integrity | Custom validation Lambda post-load |

**Cutover Strategy — Blue/Green:**
```
Week N-2: Full load completes; CDC running; both systems active
Week N-1: Parallel run validation; reconciliation reports green
Day 0:    Quiesce MS SQL writes; final CDC drain; DNS/config cutover to AWS
Day 1:    MS SQL in read-only standby; monitor 48h
Day 7:    Decommission MS SQL if no rollback triggered
```
### Data Lineage & Cataloging
- **AWS Glue Data Catalog** as central metadata store for all migrated tables [logisam](https://logisam.com/7-essential-database-migration-best-practices-for-2025/)
- S3 bucket structure: `s3://domain-data/{bounded-context}/{entity}/{yyyy/mm/dd}/`
- AWS Lake Formation for column-level access control on sensitive KYC data
- DynamoDB tables tagged with `domain`, `bounded-context`, `data-classification` tags

***
## 4. Stored Procedure Rewrite Strategy
### Decomposition Framework
```
Stored Procedure Analysis
         │
         ├── Pure Data Transformation  ──► Lambda Domain Service (Java)
         ├── Conditional Business Rules ──► Step Functions Choice States
         ├── Aggregation/Reporting       ──► Glue Job or Athena Query
         ├── Event Triggers             ──► DynamoDB Streams → Lambda
         └── Cross-domain Lookups       ──► Lambda with DynamoDB GSI reads
```
### Hexagonal Architecture + DDD in Java
Following the ports-and-adapters pattern, each domain service is structured as: [docs.aws.amazon](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/structure-a-python-project-in-hexagonal-architecture-using-aws-lambda.html)

```
com.yourorg.kyc.{domain}/
├── domain/
│   ├── model/          ← Pure Java domain objects (no AWS dependencies)
│   ├── ports/
│   │   ├── inbound/    ← Use case interfaces (e.g., ProcessReviewUseCase)
│   │   └── outbound/   ← Repository/service interfaces (e.g., CustomerRepository)
│   └── service/        ← Domain service implementing inbound ports
├── adapters/
│   ├── inbound/
│   │   ├── LambdaHandler.java     ← AWS Lambda entry point
│   │   └── SqsMessageAdapter.java ← SQS event adapter
│   └── outbound/
│       ├── DynamoDbAdapter.java   ← Implements CustomerRepository
│       ├── S3Adapter.java         ← File I/O
│       └── EventBridgeAdapter.java← Domain event publisher
└── config/
    └── ApplicationContext.java    ← Dependency injection wiring (no Spring needed; use Dagger2 or Guice for Lambda cold-start performance)
```

**Key design rules:**
- Domain model has **zero** AWS SDK imports — fully testable in isolation
- Lambda handler is a thin adapter: deserialize event → call use case → serialize response
- All configuration (thresholds, rule parameters) externalized to **AWS AppConfig** or **Parameter Store** — not hardcoded in SP logic
### Handling 3000+ Domain Fields
| Challenge | Solution |
|---|---|
| Large domain objects | Use **Value Objects** and **Aggregates** in DDD; never pass raw maps |
| Complex enrichment chains | Step Functions parallel states for independent enrichments |
| Rule externalization | Store rule parameters in DynamoDB config table; Lambda reads at init |
| Field-level audit | DynamoDB item versioning with `version` attribute + DynamoDB Streams |
| Idempotency | Conditional writes: `attribute_not_exists(processingId)` or `version = :expected` |
| Ordering | SQS FIFO with `MessageGroupId` = entity ID; DynamoDB optimistic locking |

***
## 5. End-to-End Target Architecture
### High-Level Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD (Primary)                               │
│                                                                          │
│  ┌─────────────┐    ┌──────────────────┐    ┌──────────────────────┐    │
│  │  API Gateway│    │EventBridge       │    │ EventBridge Scheduler│    │
│  │  (REST/HTTP)│    │(Custom Event Bus) │    │ (Replaces Autosys)   │    │
│  └──────┬──────┘    └────────┬─────────┘    └──────────┬───────────┘    │
│         │                    │                          │                │
│         ▼                    ▼                          ▼                │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              AWS Step Functions (Orchestration Layer)            │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐   │   │
│  │  │ Ingest   │ │Enrich/   │ │Validate/ │ │ KYC Review        │   │   │
│  │  │ State    │ │Transform │ │Business  │ │ Trigger State     │   │   │
│  │  │          │ │State     │ │Rules     │ │                   │   │   │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────────┬────────┘   │   │
│  └───────┼────────────┼────────────┼─────────────────┼────────────┘   │
│          │            │            │                  │                 │
│          ▼            ▼            ▼                  ▼                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              Java Lambda Functions (Domain Services)             │   │
│  │  [IngestAdapter] [EnrichmentService] [RuleEngine] [KYCTrigger]  │   │
│  └───────────────────────────┬──────────────────────────────────────┘   │
│                               │                                          │
│         ┌─────────────────────┼──────────────────────┐                  │
│         ▼                     ▼                      ▼                  │
│  ┌─────────────┐    ┌──────────────────┐    ┌──────────────┐           │
│  │  DynamoDB   │    │   S3 Data Lake   │    │ SQS / SNS    │           │
│  │ (Domain     │    │ (Files, Archive, │    │ (Fan-out,    │           │
│  │  Store)     │    │  Audit Logs)     │    │  DLQ)        │           │
│  └──────┬──────┘    └──────────────────┘    └──────────────┘           │
│         │                                                                │
│         │ DynamoDB Streams                                               │
│         ▼                                                                │
│  ┌─────────────┐    ┌──────────────────────────────────────────┐        │
│  │  Lambda     │───►│ EventBridge (Domain Events Published)    │        │
│  │ (Stream     │    │ → downstream KYC workflow / audit / notif│        │
│  │  Processor) │    └──────────────────────────────────────────┘        │
│  └─────────────┘                                                         │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ Observability: X-Ray | CloudWatch Dashboards | Structured Logs  │    │
│  │ Security: IAM Roles per Lambda | KMS Encryption | VPC Endpoints │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
                         │ (Only if unavoidable)
              ┌──────────┴──────────┐
              │  On-Premises        │
              │  Upstream System    │
              │  (via PrivateLink   │
              │   or Site-to-Site   │
              │   VPN)              │
              └─────────────────────┘
```
### DynamoDB Design Principles
- **Single-table design** per bounded context (e.g., `KYCDomain`, `CustomerDomain`) [docs.aws.amazon](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/hexagonal-architecture.html)
- Access patterns drive GSI design — define all access patterns before table design
- Example key structure:

| PK | SK | Attributes |
|---|---|---|
| `CUSTOMER#12345` | `PROFILE#v3` | name, dob, status, version |
| `CUSTOMER#12345` | `REVIEW#2026-03-28` | reviewType, triggeredBy, outcome |
| `REVIEW#RV-99` | `METADATA` | assignedTo, dueDate, priority |

- Use **DynamoDB Transactions** (`TransactWriteItems`) for atomic multi-item writes within a domain
- TTL on transient/processing records to control storage costs
### Security & IAM Design
- Each Lambda function gets a dedicated IAM execution role — principle of least privilege
- Cross-account access via IAM role assumption (if multi-account AWS Organizations setup)
- KMS Customer Managed Keys (CMK) for DynamoDB encryption at rest, S3 SSE-KMS
- VPC endpoints for DynamoDB, S3, SQS — no internet traffic for data plane calls
- Secrets Manager for any remaining on-prem DB credentials (not Parameter Store for secrets)

***
## 6. Detailed Data Flow & Sequence Diagrams
### Flow 1: Daily Upstream File Ingestion (Replaces Python Autosys Job)
```
EventBridge Scheduler (cron: 0 2 * * *)
    │
    ▼
Step Functions: "UpstreamIngestWorkflow"
    │
    ├─[Task: FetchUpstreamData]─► Lambda (Java) ──► On-prem API/SFTP (if unavoidable)
    │                                              OR S3 upstream drop zone
    │
    ├─[Task: ValidateAndParse]──► Lambda (Java) ──► Parse, schema validate
    │
    ├─[Task: ApplyBusinessRules]► Lambda (Java) ──► Domain service (Hexagonal)
    │                                              Read reference data from DynamoDB
    │
    ├─[Task: PersistResults]────► Lambda (Java) ──► DynamoDB BatchWriteItem
    │                                              S3 archive (Parquet)
    │
    └─[Task: PublishDomainEvent]► EventBridge ──────► Downstream consumers notified
```
### Flow 2: KYC Periodic Review Trigger
```
EventBridge Scheduler (cron: 0 6 * * MON)
    │
    ▼
Step Functions: "KYCReviewOrchestrator"
    │
    ├─[Task: LoadEligibleCustomers]─► Lambda ──► DynamoDB Query (GSI: status=PENDING_REVIEW)
    │
    ├─[Map State: Per Customer]
    │   ├─[Task: EnrichProfile]────► Lambda ──► DynamoDB GetItem (Customer aggregate)
    │   ├─[Task: ApplyReviewRules]─► Lambda ──► Domain Rule Engine (Java)
    │   ├─[Choice: Risk Level?]
    │   │   ├─ HIGH ──► Lambda ──► SQS (HighRiskQueue) ──► Downstream KYC System
    │   │   └─ LOW  ──► Lambda ──► DynamoDB update (AUTO_CLEARED)
    │   └─[Task: AuditWrite]───────► DynamoDB + S3 audit log
    │
    └─[Task: PublishReviewComplete]► EventBridge ──► Notification / reporting
```
### Flow 3: Error Handling & Retry
```
Lambda Invocation Fails
    │
    ├─[Retry: 3x with exponential backoff, jitter]
    │
    └─[Max Retries Exceeded]
         │
         ├─ Step Functions: Task fails → ErrorHandler state → CloudWatch alarm
         │
         └─ SQS: Message moves to DLQ
              └─ Lambda DLQ processor → structured error log → SNS alert → ops team
```

***
## 7. Data Processing Framework Design (Java Runtime)
### Runtime Architecture
Each Java Lambda implements the hexagonal pattern: [aws.amazon](https://aws.amazon.com/blogs/compute/developing-evolutionary-architecture-with-aws-lambda/)

```java
// Inbound adapter — thin AWS Lambda handler
public class ReviewTriggerHandler implements RequestHandler<SQSEvent, Void> {
    private final ProcessReviewUseCase useCase; // injected via Dagger2

    public Void handleRequest(SQSEvent event, Context ctx) {
        event.getRecords().forEach(record -> {
            ReviewCommand cmd = deserialize(record.getBody());
            useCase.execute(cmd); // pure domain call
        });
        return null;
    }
}

// Domain — zero AWS dependency
public class ReviewDomainService implements ProcessReviewUseCase {
    public void execute(ReviewCommand cmd) {
        Customer customer = customerRepo.findById(cmd.customerId()); // port call
        RiskAssessment risk = riskEngine.assess(customer);
        if (risk.isHigh()) {
            eventPublisher.publish(new HighRiskDetectedEvent(customer.id())); // port call
        }
        customerRepo.save(customer.withReviewOutcome(risk)); // port call
    }
}
```
### Handling High Volume & Concurrency
| Concern | Solution |
|---|---|
| Lambda concurrency spikes | Reserved concurrency per function + SQS batch window |
| DynamoDB hot partitions | Distribute writes across partition keys; use write sharding for high-volume entities |
| Large batch processing | Step Functions Map state with `maxConcurrency` control |
| Data consistency | DynamoDB conditional writes + optimistic locking via `version` attribute |
| Idempotency | Idempotency key in DynamoDB (`processingId` attribute); conditional `attribute_not_exists` writes |
| Versioning | Each DynamoDB item carries `schemaVersion`; Lambda handles version-aware deserialization |

***
## 8. Migration & Decommissioning Plan
### Phased Roadmap
| Phase | Duration | Activities | Exit Criteria |
|---|---|---|---|
| **Phase 0: Foundation** | 4 weeks | AWS account structure, VPC, IAM baselines, CDK project setup, DynamoDB domain table design, Glue Data Catalog bootstrap | AWS landing zone ready; CDK pipeline deploying to dev |
| **Phase 1: Data Migration** | 6–8 weeks | DMS full load MS SQL → S3; Glue ETL → DynamoDB; validation scripts; historical data to S3 Data Lake | Row count parity >99.99%; business validation green |
| **Phase 2: Core Pipeline Build** | 8–10 weeks | Java Lambda domain services; Step Functions workflows; EventBridge Scheduler replacing Autosys; SQS/DLQ plumbing | All daily batch jobs running in AWS (parallel with on-prem) |
| **Phase 3: SP Rewrite** | 8–12 weeks | Extract all stored procedures; map to domain services; implement in Java Hexagonal; unit + integration tests | 100% SP coverage; parallel run output parity |
| **Phase 4: Parallel Run** | 4 weeks | Both old and new systems run; output reconciliation daily; no production writes to MS SQL from new system yet | Zero critical discrepancies for 10 consecutive business days |
| **Phase 5: Cutover** | 1 week | Quiesce MS SQL writes; final DMS drain; flip EventBridge Scheduler live; route all traffic to AWS | Successful 48h production window; rollback not triggered |
| **Phase 6: Decommission** | 4 weeks | Remove Autosys jobs; decommission Unix batch servers; MS SQL set read-only then terminated; on-prem connectivity removed | All AWS metrics nominal; cost baseline confirmed |
### Autosys → EventBridge Scheduler Migration Map
| Autosys Construct | AWS Equivalent |
|---|---|
| Job definition (JIL) | Step Functions state machine + EventBridge Scheduler rule |
| Job dependency chain | Step Functions task sequencing |
| Cron schedule | EventBridge Scheduler cron expression |
| Job failure alert | CloudWatch Alarm → SNS → ops |
| Job box (grouped jobs) | Step Functions workflow |
| Global variables | AWS AppConfig / Parameter Store |

***
## 9. Observability Design
### Three Pillars Implementation
**Metrics (CloudWatch):**
- Custom metrics per domain: `KYC/ReviewsProcessed`, `Ingest/RecordsLoaded`, `Rules/FailureRate`
- CloudWatch dashboards per bounded context with P50/P95/P99 Lambda duration
- Alarms on DLQ depth > 0, Lambda error rate > 1%, Step Functions execution failures

**Tracing (X-Ray):**
- X-Ray enabled on all Lambda functions and API Gateway [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/eventbridge-integration.html)
- Custom subsegments in Java: `AWSXRay.beginSubsegment("DomainRuleEvaluation")`
- Service Map auto-generates end-to-end call graph

**Logging (CloudWatch Logs):**
- Structured JSON logging from all Java Lambdas:
```json
{
  "level": "INFO",
  "service": "kyc-review-service",
  "traceId": "1-abc-123",
  "customerId": "C-99999",
  "event": "ReviewCompleted",
  "riskLevel": "HIGH",
  "durationMs": 145
}
```
- Log Insights queries pre-built for SLA reporting
- Log retention: 90 days hot, export to S3 Glacier for compliance

***
## 10. Final Architecture Recommendation
### Recommended: Hybrid EDA + Step Functions Orchestration on Java Serverless
After evaluating all options, the optimal architecture combines **Option B (Hybrid EDA + Scheduled Batch)** with **Option C (Step Functions Orchestration)** as the primary runtime, with EDA (Option A) for reactive domain events via DynamoDB Streams → EventBridge. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/eventbridge-integration.html)

**Why this combination:**
- Directly replaces Autosys with EventBridge Scheduler — lowest conceptual friction for the team
- Step Functions gives visual, auditable workflow graphs that map 1:1 to existing Autosys job chains
- DynamoDB Streams + EventBridge handles real-time KYC triggers without polling
- Java on Lambda with Dagger2 DI gives fast cold starts and full team skill alignment
- Glue handles bulk MS SQL migration without custom engineering
### Cost & Scalability View
| Component | Estimated Monthly Cost (mid-scale) |
|---|---|
| Lambda (10M invocations, 512MB, 2s avg) | ~$21 |
| Step Functions Standard (500K transitions) | ~$12.50 |
| DynamoDB (on-demand, 100GB, 10M R/W) | ~$135 |
| EventBridge (5M events) | ~$5 |
| SQS (10M messages) | ~$4 |
| S3 (1TB storage + requests) | ~$25 |
| DMS (r5.large, 2 months migration) | ~$280 one-time |
| **Total steady-state** | **~$200–250/month** |

Scales linearly with load; no idle compute costs unlike Unix batch servers or always-on MS SQL.
### Phased Execution Roadmap Summary
```
Q1 2026: Foundation + Data Migration (Phases 0–1)
Q2 2026: Core Pipeline + SP Rewrite (Phases 2–3)
Q3 2026: Parallel Run + Cutover (Phases 4–5)
Q4 2026: Full Decommission (Phase 6)
```

This architecture achieves **zero on-prem dependencies** by end of Q4 2026, with full auditability, elastic scaling, and operational excellence aligned to AWS Well-Architected Framework pillars — at a fraction of the current infrastructure cost.



This is a critical architectural decision point. The 300k daily KYC volume combined with 2500-field domain objects and batch query patterns fundamentally challenges a pure DynamoDB approach. Here is the detailed analysis and recommendation.

***

# Database Architecture Decision: 300k Daily KYC Processing

## The Core Problem — Why This Changes Everything

Three constraints collide here and must be resolved together:

| Constraint | Impact |
|---|---|
| **2500-field domain objects** | DynamoDB item size limit is **400KB** — a 2500-field JSON object can easily breach this  [aws.amazon](https://aws.amazon.com/blogs/compute/creating-a-single-table-design-with-amazon-dynamodb/) |
| **150-field processing subset** | You don't need full objects at processing time — fetching full items wastes RCU and memory |
| **300k batch records queried by schedule** | DynamoDB is **not designed for full-table batch reads** — Scan on 300k items is expensive and slow  [docs.aws.amazon](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-query-scan.html) |
| **Scheduled batch access pattern** | Relational query (`WHERE status = 'PENDING' AND review_date = TODAY`) is trivial in SQL; painful in DynamoDB |

***

## DynamoDB vs Aurora PostgreSQL Serverless v2 — Decision Matrix

| Dimension | DynamoDB | Aurora PostgreSQL Serverless v2 |
|---|---|---|
| **Primary access pattern** | Key-value, known PK/SK lookup | Arbitrary SQL queries, multi-condition filters |
| **Batch "give me all records due today"** | Requires GSI + expensive Query loop or Scan  [stackoverflow](https://stackoverflow.com/questions/64218855/which-is-correct-method-with-respect-to-performance-for-bulk-data-scan-or-quer) | Single SQL: `SELECT ... WHERE review_date = TODAY LIMIT 300000` |
| **300k record batch read** | ~300k individual GetItem OR expensive parallel Scan consuming massive RCU | Single set-based query, optimized by planner |
| **2500-field object storage** | 400KB limit; must split across items; complex reassembly  [aws.amazon](https://aws.amazon.com/blogs/compute/creating-a-single-table-design-with-amazon-dynamodb/) | JSONB column stores full document; no size limit concern |
| **150-field projection at query time** | Must project to GSI (storage cost) or fetch full item and trim in app | `SELECT field1, field2, ... 150 fields FROM kyc WHERE ...` |
| **Complex WHERE conditions (risk, status, date)** | Requires multiple GSIs or filtering post-fetch | Native SQL WHERE clause |
| **Joins / enrichment across entities** | Application-side joins; multiple round trips | SQL JOIN; single round trip |
| **Scaling to zero (off-hours)** | Always-on (on-demand pricing still charges per request) | Aurora Serverless v2 scales to 0 ACUs  [aws.amazon](https://aws.amazon.com/blogs/database/understanding-how-acu-minimum-and-maximum-range-impacts-scaling-in-amazon-aurora-serverless-v2/) |
| **Cost at 300k records/day** | High RCU cost for batch reads + GSI write amplification | Low — optimized sequential scan; scales down overnight |
| **Operational complexity** | No schema; flexible but no query planner | Schema discipline required; familiar SQL |

**Verdict: Aurora PostgreSQL Serverless v2 is the right choice for the batch processing layer.** DynamoDB alone cannot efficiently serve "give me all KYC records due for today's review" without either a ruinously expensive Scan or a brittle GSI chaining pattern. [stackoverflow](https://stackoverflow.com/questions/64218855/which-is-correct-method-with-respect-to-performance-for-bulk-data-scan-or-quer)

***

## Recommended Dual-Store Architecture (Polyglot Persistence)

The solution is **not "Aurora OR DynamoDB"** — it is using each where it excels:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AWS DATA LAYER                                       │
│                                                                              │
│  ┌──────────────────────────────────┐   ┌──────────────────────────────┐   │
│  │  Aurora PostgreSQL Serverless v2  │   │       DynamoDB               │   │
│  │  (Processing / Operational Store) │   │  (Event Store / API Layer)   │   │
│  │                                   │   │                              │   │
│  │  • KYC processing table           │   │  • Real-time API reads       │   │
│  │  • 150 processing fields only     │   │  • Domain event records      │   │
│  │  • Queryable: status, date,       │   │  • KYC outcome records       │   │
│  │    risk tier, entity type         │   │  • Session/request state     │   │
│  │  • Batch SELECT 300k rows/day     │   │  • Downstream notifications  │   │
│  │  • Full ACID for batch writes     │   │                              │   │
│  │  • Scales to 0 ACUs at night      │   │  Single-table per domain     │   │
│  │                                   │   │  Low-latency pk/sk reads     │   │
│  └──────────────────────────────────┘   └──────────────────────────────┘   │
│                    │                                    ▲                    │
│                    │ Step Functions processes           │                    │
│                    │ Aurora batch → writes outcome      │                    │
│                    └────────────────────────────────────┘                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  S3 (Full Domain Object Archive — All 2500 fields)                   │   │
│  │  • Parquet files partitioned by entity/date                          │   │
│  │  • Queried via Athena when full object needed (audit, replay)        │   │
│  │  • NOT in the hot processing path                                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

***

## Handling the 2500-Field Domain Object

### Problem
Storing 2500 fields in DynamoDB violates the 400KB item limit and wastes RCU on every read. Fetching all 2500 fields into Lambda for processing that needs only 150 is wasteful and slow. [aws.amazon](https://aws.amazon.com/blogs/compute/creating-a-single-table-design-with-amazon-dynamodb/)

### Solution: Tiered Field Segregation

```
2500 Domain Fields
        │
        ├── Tier 1: Processing Fields (150 fields)
        │   └── Aurora PostgreSQL "kyc_processing" table
        │       Columns: customer_id, review_date, risk_tier, status,
        │                source_country, pep_flag, sanctions_hit, ...
        │       → Used by Step Functions batch daily run
        │
        ├── Tier 2: API / Lookup Fields (~50 key fields)
        │   └── DynamoDB "KYCDomain" table
        │       PK: CUSTOMER#id, SK: PROFILE#latest
        │       → Used by real-time API Gateway / Lambda reads
        │
        └── Tier 3: Full Archive (all 2500 fields)
            └── S3 as Parquet (partitioned by date/entity)
                → Queried only via Athena for audit, compliance,
                  full replay, or ML feature generation
```

**Field sync strategy:** When a KYC record is updated, a Lambda writes:
1. The 150 processing fields to Aurora (upsert)
2. The 50 API fields to DynamoDB (conditional write)
3. The full 2500-field object to S3 as a versioned Parquet snapshot

***

## How 300k Daily KYC Records Are Queried

### Aurora Query Pattern (Step Functions Batch Job)

```sql
-- Daily batch: fetch all KYC records due for review
-- Runs at 02:00 via EventBridge Scheduler → Step Functions

SELECT
    customer_id, risk_tier, review_date, status,
    source_country, pep_flag, sanctions_hit,
    last_reviewed_date, assigned_team, entity_type,
    -- ... 150 fields total
FROM kyc_processing
WHERE review_date <= CURRENT_DATE
  AND status IN ('PENDING_REVIEW', 'ESCALATED')
  AND processing_batch IS NULL           -- not yet picked up
ORDER BY risk_tier DESC, review_date ASC
LIMIT 300000;
```

### Batch Chunking Strategy in Step Functions

Fetching 300k rows into a single Lambda is impossible (Lambda 10GB memory, 15-min limit). Use chunked processing:

```
EventBridge Scheduler → Step Functions "KYCDailyBatch"
    │
    ├─ [Task: CountAndPartition]
    │   └─ Lambda → Aurora: SELECT COUNT(*) → returns 300,000
    │              → Divides into 300 chunks of 1,000 records
    │              → Writes chunk metadata to Aurora: chunk_id, offset, limit
    │
    ├─ [Map State: maxConcurrency=20]  ← Process 20 chunks in parallel
    │   └─ Per chunk (1,000 records):
    │       ├─ Lambda: SELECT ... FROM kyc_processing WHERE chunk_id = X
    │       ├─ Lambda: Apply Java domain rule engine (150 fields)
    │       ├─ Lambda: Write outcomes → Aurora (UPDATE kyc_processing)
    │       └─ Lambda: Publish domain events → EventBridge → DynamoDB
    │
    └─ [Task: ReconcileAndReport]
        └─ Lambda: Verify all chunks completed; write batch summary to S3
```

**At 20 parallel chunks × 1000 records, 300k records complete in ~15 batch iterations** — well within Lambda limits and Aurora connection pool capacity.

### Aurora Connection Pooling

Aurora Serverless v2 with Lambda at scale requires **RDS Proxy** to prevent connection exhaustion: [aws.amazon](https://aws.amazon.com/blogs/database/understanding-how-acu-minimum-and-maximum-range-impacts-scaling-in-amazon-aurora-serverless-v2/)

```
Lambda (up to 1000 concurrent) → RDS Proxy (connection pool) → Aurora Serverless v2
                                  (max 100 connections to Aurora)
                                  (multiplexes 1000 Lambda connections)
```

***

## Aurora Schema Design for KYC Processing

```sql
-- Core processing table (150 fields — representative subset shown)
CREATE TABLE kyc_processing (
    customer_id         VARCHAR(36) PRIMARY KEY,
    entity_type         VARCHAR(20) NOT NULL,        -- INDIVIDUAL, CORPORATE
    risk_tier           SMALLINT NOT NULL,            -- 1=LOW, 2=MED, 3=HIGH
    status              VARCHAR(30) NOT NULL,
    review_date         DATE NOT NULL,
    pep_flag            BOOLEAN DEFAULT FALSE,
    sanctions_hit       BOOLEAN DEFAULT FALSE,
    source_country      CHAR(2),
    last_reviewed_date  DATE,
    assigned_team       VARCHAR(50),
    processing_batch    VARCHAR(36),                 -- batch_id when picked up
    batch_status        VARCHAR(20),                 -- PROCESSING, COMPLETE, FAILED
    outcome             VARCHAR(30),
    outcome_reason      TEXT,
    schema_version      SMALLINT NOT NULL DEFAULT 1,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW(),
    version             BIGINT NOT NULL DEFAULT 0    -- optimistic locking
    -- ... remaining 130+ fields
);

-- Indexes that power the batch query
CREATE INDEX idx_kyc_batch_pickup
    ON kyc_processing (review_date, status, processing_batch)
    WHERE processing_batch IS NULL;

CREATE INDEX idx_kyc_risk_status
    ON kyc_processing (risk_tier, status, review_date);
```

**Optimistic locking on batch update:**
```sql
UPDATE kyc_processing
SET outcome = $1, batch_status = 'COMPLETE', version = version + 1, updated_at = NOW()
WHERE customer_id = $2
  AND version = $3;    -- fails if another process modified it
-- Java Lambda checks rows_affected = 1; if 0 → conflict → retry
```

***

## Revised End-to-End Data Flow: 300k KYC Daily Batch

```
02:00 UTC
  EventBridge Scheduler
       │
       ▼
  Step Functions: KYCDailyBatchWorkflow
       │
       ├─[Partition Task]──────────────────────────────────────────────────────┐
       │  Lambda → RDS Proxy → Aurora                                          │
       │  "SELECT customer_id FROM kyc_processing                              │
       │   WHERE review_date <= TODAY AND status = 'PENDING'                   │
       │   AND processing_batch IS NULL"                                        │
       │  → Returns 300k IDs → divides into 300 chunks                         │
       │  → Writes chunk manifest to S3 (chunk-manifest-{date}.json)           │
       │  → UPDATE kyc_processing SET processing_batch = {batchId}            │
       │    WHERE customer_id IN (chunk) [marks as claimed, prevents           │
       │    double-processing]                                                  │
       │                                                                       │
       ├─[Map State — 20 parallel workers]────────────────────────────────────┘
       │  Per chunk Lambda:
       │  1. SELECT * FROM kyc_processing WHERE processing_batch = {batchId}
       │     AND chunk_id = {n}                  ← full 150 fields
       │  2. Java Domain Rule Engine evaluates each record
       │     (RuleEngineService.evaluate(KycRecord record))
       │  3. Determine outcome: AUTO_CLEAR / ESCALATE / REFER
       │  4. UPDATE kyc_processing SET outcome=?, batch_status='COMPLETE',
       │     version=version+1 WHERE customer_id=? AND version=?
       │  5. For ESCALATE: publish to EventBridge
       │     → routes to SQS(HighRiskQueue) → downstream KYC system
       │
       ├─[Reconcile Task]
       │  Lambda: COUNT complete vs total; write summary to S3 + CloudWatch
       │  Alert via SNS if completion rate < 99.9%
       │
       └─[DynamoDB Sync Task]  ← optional, async
          Lambda: Write outcome summary (50 fields) to DynamoDB
          for real-time API consumption by front-end/downstream APIs
```

***

## Cost Model for 300k Daily at Scale

| Component | Configuration | Est. Monthly Cost |
|---|---|---|
| Aurora PostgreSQL Serverless v2 | 2–8 ACU; scales to 0 off-hours; 100GB storage | ~$180–$320 |
| RDS Proxy | 1 proxy endpoint | ~$30 |
| DynamoDB (outcomes + events only) | On-demand, ~5M writes/reads/month | ~$40 |
| S3 (full 2500-field archive, Parquet) | 5TB @ Intelligent Tiering | ~$60 |
| Lambda (300k × 150 field processing) | 300 chunks × 2s @ 512MB | ~$8 |
| Step Functions | ~300 transitions/batch × 30 days | ~$0.25 |
| RCU/WCU for DynamoDB event writes | 10M events/month | ~$25 |
| **Total** | | **~$345–$485/month** |

This processes **9 million KYC records/month** (300k/day) at under $500/month — compared to the cost of always-on MS SQL Server infrastructure.

***

## Summary Recommendation

**Use Aurora PostgreSQL Serverless v2 as the primary batch processing database** for scheduled KYC workloads. DynamoDB is retained for real-time API reads, domain event storage, and downstream notification records. S3 holds the full 2500-field archive, queryable via Athena for audit and compliance — keeping it entirely out of the hot processing path. [reddit](https://www.reddit.com/r/aws/comments/1h4vox5/dynamodb_or_aurora_or_rds/)

The key design decisions are:
- **150-field projection** stored as normalized columns in Aurora — not a blob — enabling index-driven batch queries
- **Chunk-based Map State** in Step Functions processes 300k records safely within Lambda constraints
- **RDS Proxy** prevents Aurora connection exhaustion under high Lambda concurrency
- **Optimistic locking** (`version` column) ensures idempotent batch processing with no double-processing risk
- **DynamoDB** reserved strictly for sub-10ms API reads and event records — not batch scans