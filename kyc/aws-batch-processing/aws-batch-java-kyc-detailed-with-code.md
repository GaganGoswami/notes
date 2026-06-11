---
id: 3004-aws-batch-java-kyc-implementation
slug: aws-batch-java-kyc-implementation
title: "Code & Config Guide (KYC Java Implementation) - AWS Batch and Scheduling"
category: infrastructure
tags: [aws, batch-processing, java, kyc, eventbridge, step-functions, distributed-map, sqs, fargate, implementation, code-examples]
description: "Detailed implementation guide with Java code examples, configuration patterns, and KYC-specific architecture for 300k+ daily records processing across multi-region active-active setup"
difficulty: advanced
estimatedTime: 30
created_at: 2026-04-03T10:00:00Z
updated_at: 2026-04-03T10:00:00Z
---

# Code and Config - Architecture Guide - AWS Batch Processing and Scheduling: 
---
## 1. Why This Matters: The Design Philosophy
Before picking tools, understand the core tension: **batch processing is fundamentally about trading latency for throughput**, and your constraints — multi-region active-active, 300k+ daily records, 2500-field domain objects, mixed SLAs — demand a layered architecture where no single service does everything. The WHY-WHAT-HOW framework forces intentionality: every service chosen must earn its place.

***
## 2. AWS Scheduling & Batch Options: WHY–WHAT–HOW
### EventBridge Scheduler
**WHY:** Purpose-built serverless scheduler that decouples "when to fire" from "what to run." It replaced cron-in-rules as the preferred AWS scheduling primitive and is now available in all AWS regions. [aws.amazon](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-eventbridge-scheduler-all-aws-regions/?nc1=h_ls)

**WHAT:** A managed schedule store that can invoke 270+ AWS service targets directly (Step Functions, ECS tasks, SQS, Lambda, Batch) using flexible cron, rate, or one-time schedules, with timezone and DST support. [aws.amazon](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-eventbridge-scheduler-all-aws-regions/?nc1=h_ls)

**HOW:**
```
EventBridge Scheduler
  └─ Schedule Group (per tenant / per domain)
       └─ Cron/Rate Schedule
            └─ Target: Step Functions StartExecution / SQS SendMessage / ECS RunTask
                 └─ Dead Letter Queue (SQS) on delivery failure
                 └─ Retry Policy (max 185 retries, max 24h window)
```
- **Scaling:** Supports billions of schedules — schedule-per-tenant is viable [aws.amazon](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-eventbridge-scheduler-all-aws-regions/?nc1=h_ls)
- **Cost:** ~$1/million schedule invocations (essentially free at your scale)
- **Retries/DLQ:** Built-in retry with exponential backoff + SQS DLQ for failed invocations
- **Regional note:** Deploy per region; use SSM Parameter Store flag to designate which region is "active" to prevent dual-firing in active-active [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/)
- **Avoid when:** You need content-based event routing (use EventBridge Rules instead)

***
### EventBridge Rules (CRON)
**WHY:** Original event-routing service. Schedule is a side-feature here; its real power is pattern matching on events.

**WHAT:** Rules on an event bus with cron/rate expressions as triggers, routing to targets.

**HOW:**
- Max 300 targets per rule per region
- No built-in DLQ or retry (unlike Scheduler)
- **Choose over Scheduler when:** you need to react to AWS service events (e.g., S3 object created, ECS task state change) AND trigger batch processing — event-driven, not time-driven
- **Multi-region:** Use cross-region event buses to replicate events
---
### Step Functions Standard vs. Express
**WHY:** Orchestration layer that externalizes retry, error handling, branching, and audit from your application code. [thecloudsolutions](http://thecloudsolutions.com/blog/aws-architecture-best-practices-2025/)

**WHAT:**

| Dimension | Standard | Express |
|---|---|---|
| Duration | Up to 1 year | Up to 5 minutes |
| Execution model | Exactly-once | At-least-once |
| State persistence | Full history | CloudWatch Logs only |
| Pricing | Per state transition (~$0.025/1k) | Per duration+invocation |
| Audit trail | Built-in execution history | Requires log aggregation |
| Best for | Long-running KYC, ETL orchestration | High-throughput micro-batch |

**HOW (Java integration):**
```java
// AWS SDK v2 - Start Step Functions execution from Java
SfnClient sfn = SfnClient.builder().region(Region.US_EAST_1).build();
StartExecutionRequest req = StartExecutionRequest.builder()
    .stateMachineArn(STATE_MACHINE_ARN)
    .name(idempotencyKey)          // deduplicate by unique name
    .input(objectMapper.writeValueAsString(batchInput))
    .build();
sfn.startExecution(req);
```
- **Retries:** Configure per-state with `IntervalSeconds`, `BackoffRate`, `MaxAttempts`, `JitterStrategy: FULL`
- **Failure isolation:** `Catch` blocks route failures to DLQ-write states or compensating transactions
- **Versioning:** Use Step Functions Versions + Aliases; canary deploy new DAG versions

***
### Step Functions Distributed Map
**WHY:** The most powerful AWS-native solution for massive parallel batch — processes millions of items from S3/DynamoDB/SQS directly, spawning up to 10,000 parallel child executions. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html)

**WHAT:**
```
S3 input file (300k records CSV)
  └─ Distributed Map State
       ├─ ItemReader: S3 CSV/JSON/JSONL (reads directly, no Lambda needed)
       ├─ ItemBatcher: groups 100 records per child
       ├─ MaxConcurrency: 300 (controls parallelism budget)
       ├─ ToleratedFailurePercentage: 2 (partial failure allowed)
       └─ Child Express Execution
            └─ Lambda or Fargate Task (process batch of 100)
       └─ ResultWriter: S3 JSONL (audit output)
```

**HOW (Java Lambda worker):**
```java
// Handler processes a batch of up to 100 records
public class BatchWorkerHandler implements RequestHandler<Map<String, Object>, BatchResult> {
    @Override
    public BatchResult handleRequest(Map<String, Object> event, Context context) {
        List<Record> records = parseRecords(event); // only extract ~150 fields
        return processWithIdempotency(records);
    }
    
    private BatchResult processWithIdempotency(List<Record> records) {
        // DynamoDB conditional put for idempotency
        records.forEach(r -> {
            String key = r.getTenantId() + "#" + r.getRecordId() + "#" + r.getBatchDate();
            dynamoDb.putItem(PutItemRequest.builder()
                .tableName(IDEMPOTENCY_TABLE)
                .item(Map.of("pk", AttributeValue.fromS(key), "ttl", AttributeValue.fromN(...)))
                .conditionExpression("attribute_not_exists(pk)")
                .build());
        });
        return processRecords(records);
    }
}
```
- **Best for:** Your 300k daily batch — 300k ÷ 100 per batch = 3,000 child executions, easily within limits [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2)
- **Cost:** Express executions at ~$1 per million steps — processing 300k records costs pennies
- **Partial failures:** `ToleratedFailurePercentage` + `ResultWriter` outputs success/failure manifest to S3

***
### SQS + Lambda Fan-out
**WHY:** SQS provides durable buffering and backpressure; Lambda auto-scales consumers to 1,000 concurrent. Ideal for async CRUD-side operations and micro-batch event processing.

**WHAT:**
```
Producer (Java Spring Boot / API Gateway)
  └─ SQS Standard Queue
       ├─ Lambda Event Source Mapping
       │    ├─ BatchSize: 10–1000 (tune per workload)
       │    ├─ BisectOnFunctionError: true (halves batch on failure)
       │    └─ MaximumConcurrency: 500 (throttle)
       └─ DLQ (after 3 retries)
            └─ EventBridge Pipe → Notification / Replay workflow
```

**HOW:**
- **Idempotency:** Lambda Powertools Idempotency (Java port available) using DynamoDB
- **Cost:** Lambda costs ~$0.20/million invocations + SQS ~$0.40/million messages
- **Field filtering:** Extract only 150 fields at Lambda entry point using Jackson `@JsonIgnoreProperties`

***
### SQS FIFO + Fargate Job Consumers
**WHY:** When ordering matters (e.g., per-tenant KYC state machine), FIFO guarantees order within a `MessageGroupId` (one group = one tenant).

**WHAT:**
```
SQS FIFO Queue (MessageGroupId = tenantId)
  └─ ECS Service (Fargate) - polling consumer
       ├─ Java Spring Batch application
       ├─ Scale: ECS Service Auto Scaling on ApproximateNumberOfMessages
       └─ Graceful shutdown: SIGTERM handler → finish current batch → drain
```

**HOW (Java graceful shutdown):**
```java
@Bean
public DisposableBean gracefulShutdown(SqsPoller poller) {
    return () -> {
        log.info("SIGTERM received - draining SQS batch");
        poller.stop();           // stop polling new messages
        poller.awaitCompletion(Duration.ofMinutes(2)); // finish in-flight
    };
}
```
- **Cost:** Fargate Spot saves up to 70% for fault-tolerant consumers [aws.amazon](https://aws.amazon.com/blogs/containers/cost-optimization-checklist-for-ecs-fargate/)
- **FIFO throughput:** 3,000 messages/sec per queue (300 per message group) — shard by tenant for scale

***
### AWS Batch
**WHY:** Fully managed job scheduler for containerized compute-heavy workloads. Handles job queues, compute environment provisioning, retry, and dependency chains without you managing ECS directly. [aws.amazon](https://aws.amazon.com/batch/)

**WHAT:**
```
AWS Batch
  ├─ Job Definition (Docker image: Java ETL processor)
  ├─ Job Queue
  │    ├─ Priority 100 → Spot Compute Environment (EC2 Spot, c5/r5)
  │    └─ Priority 1   → On-Demand Compute Environment (fallback)
  └─ Scheduling Policy (Fair-share per tenant)
```

**HOW:**
- **Best for:** Long ETL (1–3 hours), ML scoring jobs, Glue-alternative for Java-native processing
- **Array Jobs:** Submit 300k records as Array Job size=3000 → 3000 parallel containers each processing 100 records
- **Cost:** Spot instances save 60–90%; AWS Batch has no additional charge
- **Retry:** Job-level `attempts` + exit code–based retry conditions
- **Java:** Native — job container is any Docker image
---
### ECS/Fargate Scheduled Tasks
**WHY:** Simple, container-native scheduling using EventBridge Rules targeting ECS RunTask. No separate scheduler service.

**WHAT:**
```
EventBridge Rule (cron)
  └─ ECS RunTask (Fargate)
       └─ Task Definition (Java container, 4vCPU/8GB)
            └─ EFS or S3 for shared state
```
- **Choose when:** Single long-running container, no complex orchestration needed
- **Avoid when:** You need parallel fan-out, retries with backoff, or audit history

***
### AWS Glue Jobs / Glue Workflows
**WHY:** Managed Spark/Python ETL service — ideal for schema evolution, data catalog integration, and Spark-scale transformations.

**WHAT:**
- Glue Jobs (Spark): Python/Scala only — **not Java-native** but Spark supports Java APIs
- Glue Workflows: DAG of crawlers, jobs, triggers
- **Best for:** Large-scale structured data transformation with schema discovery
- **Cost driver:** DPU-hours (~$0.44/DPU-hour); minimum 2 DPUs, 10-minute billing minimum
- **Avoid when:** You need Java-first, sub-5-minute micro-batch, or complex orchestration

***
### Amazon MWAA (Managed Airflow)
**WHY:** Fully managed Apache Airflow — ideal for complex DAG dependencies, multi-system orchestration, and teams with existing Airflow expertise.

**WHAT:**
```
MWAA (Airflow DAG in Python)
  └─ Operators:
       ├─ EcsRunTaskOperator (trigger Java Fargate task)
       ├─ StepFunctionStartExecutionOperator
       ├─ BatchOperator (AWS Batch jobs)
       └─ SensorOperator (poll for S3/DynamoDB completion)
```
- **Cost:** $0.49–$5.76/hour depending on environment class — **significant fixed cost**
- **Use when:** You have 50+ interdependent DAGs, need Airflow-native features, or migrating existing Airflow workloads
- **Avoid when:** You have <10 pipelines or need sub-minute scheduling (Airflow minimum ~1 min)

***
### Lambda with DynamoDB Streams / Kinesis / S3 Events
**WHY:** Pure event-driven batch trigger — "batch processing when data arrives" rather than on schedule.

**WHAT:**
```
DynamoDB Streams → Lambda (Java, filter: only changed fields)
S3 Event Notification → SQS → Lambda (process new file)
Kinesis Data Streams → Lambda (micro-batch by shard)
```
- **Field filtering:** DynamoDB Streams supports server-side filtering — only send events with specific attribute changes
- **Best for:** CRUD-side async operations in your workload (post-save scoring, audit logging)
- **Lambda limit:** 15 minutes max — route long work to Fargate/Batch via SQS

***
### EC2 Spot-Based Batch Clusters
**WHY:** Maximum flexibility and lowest cost for sustained, compute-intensive workloads (ML training, genomics, video processing).

- Use AWS Batch on EC2 Spot with `SPOT_CAPACITY_OPTIMIZED` allocation strategy
- **Spot interruption handling:** Save checkpoint to S3 every N records; resume from checkpoint on restart
- **Cost:** 60–90% savings vs. On-Demand

***
### AWS Data Pipeline (Legacy)
**WHY (historical):** Pre-Batch, pre-Step Functions pipeline service. AWS has signaled it is in maintenance mode.

- **Do not use** for new workloads. Migrate to Step Functions + AWS Batch or MWAA.

***
## 3. Side-by-Side Comparison Matrix
| Service | Best For | Max Duration | Parallelism | Java Support | Cost Model | Retry/DLQ | Multi-Region | Avoid When |
|---|---|---|---|---|---|---|---|---|
| **EventBridge Scheduler** | Triggering jobs on schedule | N/A (trigger only) | Billions of schedules | Via target | ~$1/M invocations | Built-in retry + SQS DLQ | Deploy per region, use SSM flag | Need content routing |
| **EventBridge Rules** | Event-driven triggers | N/A | High | Via target | Included in bus pricing | No built-in retry | Cross-region buses | Pure scheduling |
| **Step Functions Standard** | ETL/KYC orchestration | 1 year | Via Map state | SDK v2 | $0.025/1k transitions | Per-state Retry+Catch | Global endpoints | High-freq micro-batch (cost) |
| **Step Functions Express** | Micro-batch orchestration | 5 min | High | SDK v2 | Duration + requests | At-least-once, logs only | N/A | Exactly-once required |
| **Step Functions Dist. Map** | Massive parallel batch | 1 year (parent) | 10,000 child exec | SDK v2 | Express child pricing | Built-in + ToleratedFailure% | Via parent SF | Small (<1k item) batches |
| **SQS + Lambda** | Async CRUD ops, fan-out | 15 min (Lambda) | 1,000 concurrent | Java Lambda | Lambda + SQS pricing | Auto-retry + DLQ | Per-region SQS | Long-running tasks |
| **SQS FIFO + Fargate** | Ordered per-tenant processing | Unlimited | Per-queue throughput | Any JVM image | Fargate + SQS | Manual retry + DLQ | Per-region | High-volume (FIFO throughput limit) |
| **AWS Batch** | Long ETL, scoring jobs | Unlimited | Array jobs up to 10k | Any Docker | EC2/Fargate Spot | Job-level retries | Per-region | Sub-minute latency |
| **ECS Fargate Scheduled** | Simple scheduled containers | Unlimited | Manual scale-out | Any JVM image | Fargate on-demand/spot | Manual | Per-region | Complex orchestration |
| **Glue Jobs** | Spark ETL, schema evolution | 48 hours | Auto-scale DPUs | Scala/Python (not Java) | DPU-hours | Workflow retry | N/A | Java-first, <5 min jobs |
| **MWAA (Airflow)** | Complex multi-system DAGs | DAG-defined | Airflow workers | Via operators | Fixed env cost + workers | Airflow native | Manual replication | Small pipeline count |
| **Lambda + Streams** | Event-driven async processing | 15 min | Shard-based | Java Lambda | Per invocation | Auto-retry, shard-level | Per-region streams | Long compute |
| **EC2 Spot Clusters** | Heavy compute, sustained | Unlimited | Manual/ASG | Any JVM | Spot pricing | Custom checkpoint | Cross-region AMI | Quick bootstrapping needed |
| **Data Pipeline** | Legacy migration only | — | Low | N/A | Legacy | Basic | No | All new workloads |

***
## 4. Architecture Examples
### Architecture 1: Daily 300k+ Record Processing (Multi-Region)
**WHY this design:** Distributed Map handles 300k items natively from S3 without Lambda fan-out plumbing. Multi-region active-active uses EventBridge Scheduler in both regions with an SSM Parameter Store leader flag to prevent dual execution. [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/)

```
REGION-A (primary)                    REGION-B (secondary)
┌─────────────────────────────┐       ┌─────────────────────────────┐
│ EventBridge Scheduler       │       │ EventBridge Scheduler       │
│  (daily 02:00 UTC)          │       │  (daily 02:00 UTC)          │
│  └─ Lambda: LeaderCheck     │       │  └─ Lambda: LeaderCheck     │
│       └─ SSM: is_primary?   │       │       └─ SSM: is_primary?   │
│            YES: proceed      │       │            NO: standby      │
│            └─ Start SF       │       │  (promoted if A unhealthy)  │
│                              │       │                             │
│ Step Functions (Standard)   │       │ Route 53 Health Check       │
│  └─ Validate Input           │       │  └─ Triggers SSM flip       │
│  └─ Distributed Map         │       │  └─ Activates Region B      │
│       ├─ S3 ItemReader       │       └─────────────────────────────┘
│       │   (300k records CSV) │
│       ├─ ItemBatcher: 100    │        DynamoDB Global Table
│       ├─ MaxConcurrency: 300 │        ├─ Idempotency records
│       └─ Lambda Java Worker  │        ├─ Job state / checkpoint
│            └─ Read 150 fields│        └─ Audit log entries
│            └─ DynamoDB idem. │
│            └─ Write results  │        S3 Cross-Region Replication
│  └─ ResultWriter → S3        │        └─ Input files replicated
│  └─ Notify SNS on complete   │
└─────────────────────────────┘
```

**Java Lambda Worker (field projection):**
```java
@JsonIgnoreProperties(ignoreUnknown = true)  // Ignore 2350+ unused fields
public class DomainRecord {
    @JsonProperty("id")          private String id;
    @JsonProperty("tenantId")    private String tenantId;
    @JsonProperty("kycStatus")   private String kycStatus;
    // ... 147 more mapped fields, 2350+ silently dropped
}
```

**Retry:** Step Functions per-state retry with `BackoffRate: 2, MaxAttempts: 3`
**Idempotency:** DynamoDB conditional `attribute_not_exists(pk)` on `tenantId#recordId#batchDate`
**Cost estimate:** 300k records ÷ 100 = 3,000 Express executions × ~150 states × $0.00001 = ~$0.45/day
**Consistency:** DynamoDB Global Tables provide multi-region consistency for idempotency keys

***
### Architecture 2: Nightly ETL Pipeline (Java + Python Mix)
**WHY:** Use Step Functions as the orchestrator to mix Java (Fargate for transformation) and Python (Glue for Spark-scale aggregation), with clear handoff via S3 staging. [aws.amazon](https://aws.amazon.com/blogs/publicsector/efficient-large-scale-serverless-data-processing-for-slow-downstream-systems/)

```
EventBridge Scheduler (23:00 UTC)
  └─ Step Functions Standard (ETL Orchestrator)
       │
       ├─  [aws.amazon](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-eventbridge-scheduler-all-aws-regions/?nc1=h_ls) Extract: Java Fargate Task
       │        └─ JDBC → Source DB (read with connection pool)
       │        └─ Write partitioned Parquet → S3 (year/month/day/tenant)
       │        └─ Write manifest.json → S3
       │
       ├─  [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/) Wait for Fargate completion (ECS Task heartbeat → SF callback)
       │
       ├─  [thecloudsolutions](http://thecloudsolutions.com/blog/aws-architecture-best-practices-2025/) Transform: AWS Glue Job (PySpark)
       │        └─ Read Parquet from S3
       │        └─ Apply business rules (Python UDFs)
       │        └─ Write enriched Parquet → S3 output
       │
       ├─  [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html) Load: Java Fargate Task
       │        └─ Read enriched Parquet
       │        └─ Bulk UPSERT → Aurora Global DB / Redshift
       │
       ├─  [aws.amazon](https://aws.amazon.com/blogs/aws/step-functions-distributed-map-a-serverless-solution-for-large-scale-parallel-data-processing/) Validate: Lambda (Java)
       │        └─ Row count reconciliation
       │        └─ Checksum validation
       │        └─ Write audit record → DynamoDB
       │
       └─  [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2) Notify: SNS → Slack / PagerDuty
            └─ On failure → DLQ + CloudWatch Alarm
```

**SF Callback pattern for Fargate (Java):**
```java
// Fargate task sends heartbeat back to Step Functions
String taskToken = System.getenv("SF_TASK_TOKEN");
sfnClient.sendTaskHeartbeat(r -> r.taskToken(taskToken));
// On completion:
sfnClient.sendTaskSuccess(r -> r.taskToken(taskToken).output("{\"status\":\"done\"}"));
```

**Multi-region:** Aurora Global Database handles write promotion; S3 CRR replicates staging files
**Security:** Fargate task role with least-privilege IAM; VPC private subnets; Secrets Manager for DB credentials

***
### Architecture 3: Real-Time Micro-Batching (Every 5 Minutes)
**WHY:** Use Step Functions Express (sub-5-min duration) triggered by EventBridge Scheduler at rate(5 minutes), with SQS buffering to absorb spikes. [thecloudsolutions](http://thecloudsolutions.com/blog/aws-architecture-best-practices-2025/)

```
Source Events → SQS Standard Queue
                  (buffer: max 5 min visibility)
                         │
EventBridge Scheduler ───┘
  (rate: 5 minutes)
  └─ Step Functions Express
       ├─ Lambda: Drain SQS (up to 1000 msgs, batch receive)
       ├─ Distributed Map (inline mode, 1000 items max)
       │    └─ Lambda Java Worker (per-item processing)
       ├─ Lambda: Write results → DynamoDB / Aurora
       └─ Lambda: Emit metrics → CloudWatch EMF
            └─ Alarm: processing lag > 2 batches
```

**Key design decisions:**
- Use `ReceiveMessage` with `MaxNumberOfMessages=10` in a loop until queue is empty or 900-second limit
- Lambda Powertools (Java) for structured logging with correlation IDs
- Express Step Functions: no execution history stored — emit to CloudWatch Logs with log level ALL
- **Cost:** ~8,640 SF Express executions/month × minimal duration = <$10/month

***
### Architecture 4: Container-Based Long-Running Batch (1–3 Hours)
**WHY:** Lambda's 15-minute limit rules it out; AWS Batch with Spot provides the cost/reliability balance, with checkpointing for Spot interruption resilience. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

```
EventBridge Scheduler
  └─ Lambda: Submit AWS Batch Job (Java SDK)
       └─ AWS Batch Job Queue
            ├─ Spot Compute Env (priority 1, SPOT_CAPACITY_OPTIMIZED)
            └─ On-Demand Compute Env (priority 2, fallback)

AWS Batch Job (Java Spring Batch container)
  ├─ Startup: Read checkpoint from S3 (last processed offset)
  ├─ Process loop:
  │    ├─ Read chunk (1000 records)
  │    ├─ Transform + enrich
  │    ├─ Write to DynamoDB / Aurora
  │    └─ Write checkpoint → S3 (every 10k records)
  ├─ SIGTERM Handler (Spot interruption):
  │    └─ Finish current chunk
  │    └─ Flush checkpoint
  │    └─ Exit code 1 (triggers Batch retry from checkpoint)
  └─ Completion: Write job manifest → DynamoDB audit table
```

**Java Spring Batch checkpoint:**
```java
@Bean
public S3CheckpointRepository checkpointRepository(S3Client s3) {
    return new S3CheckpointRepository(s3, CHECKPOINT_BUCKET, jobId);
}

// SIGTERM handler
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    log.info("SIGTERM: saving checkpoint at offset {}", currentOffset);
    checkpointRepository.save(currentOffset);
    jobExecutor.shutdown();
}));
```

**Cost:** Spot c5.2xlarge ~$0.14/hour; 2-hour job = ~$0.28 vs. $1.40 on-demand (80% savings)

***
### Architecture 5: Lambda-Based Parallel Fan-Out / Fan-In
**WHY:** When you need parallel processing with result aggregation and Step Functions Distributed Map is overkill for smaller datasets (<10k items).

```
Step Functions Standard
  ├─ [Fan-out] Lambda: Split 300k → 300 chunks of 1000
  │                    → Send each chunk ref (S3 key) to SQS
  │
  ├─ [Process] SQS → Lambda (Java, 1000 concurrent)
  │                  → Process chunk
  │                  → Write partial result → S3
  │                  → Decrement DynamoDB counter (atomic)
  │
  ├─ [Wait] Step Functions Wait-for-Callback
  │          └─ Lambda monitors DynamoDB counter = 0
  │               → SendTaskSuccess to SF
  │
  └─ [Fan-in] Lambda: Read all S3 partial results
                      → Aggregate
                      → Write final output
```

**DynamoDB atomic counter (Java):**
```java
dynamoDb.updateItem(UpdateItemRequest.builder()
    .tableName(JOB_TABLE)
    .key(Map.of("jobId", AttributeValue.fromS(jobId)))
    .updateExpression("ADD remaining_chunks :dec")
    .expressionAttributeValues(Map.of(":dec", AttributeValue.fromN("-1")))
    .returnValues(ReturnValue.ALL_NEW)
    .build());
```

***
### Architecture 6: Step Functions Distributed Map for High Parallelism
**WHY:** Native AWS solution for processing millions of S3 records with minimal code. AWS manages parallelism, partial failure tolerance, and result aggregation. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html)

```
S3 Input:
  └─ s3://batch-input/2026/03/29/records.csv (300k rows)

Step Functions Distributed Map:
{
  "Type": "Map",
  "ItemReader": { "Resource": "s3:getObject", "InputType": "CSV" },
  "ItemBatcher": { "MaxItemsPerBatch": 100 },
  "MaxConcurrency": 500,
  "ToleratedFailurePercentage": 1,
  "ItemProcessor": {
    "Mode": "DISTRIBUTED",
    "ExecutionType": "EXPRESS",
    "StartAt": "ProcessBatch",
    "States": {
      "ProcessBatch": {
        "Type": "Task",
        "Resource": "arn:aws:states:::ecs:runTask.sync",  // Fargate for heavy work
        "Retry": [{ "BackoffRate": 2, "MaxAttempts": 3 }]
      }
    }
  },
  "ResultWriter": { "Resource": "s3:putObject", "OutputType": "JSONL" }
}
```
**Failure manifest:** `ResultWriter` outputs `FAILED_0.json` with failed item details → separate Step Function triggered for replay of failed items only.

***
### Architecture 7: KYC Periodic Scoring with Audit, DLQ, Replay
**WHY:** KYC has strict compliance requirements — every decision must be auditable, replayable, and isolated per tenant.

```
EventBridge Scheduler (daily per tenant, scheduled group)
  └─ Step Functions Standard (KYC Scoring Orchestrator)
       │
       ├─  [aws.amazon](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-eventbridge-scheduler-all-aws-regions/?nc1=h_ls) Lock: DynamoDB conditional write (prevent concurrent runs)
       │         └─ Item: { pk: "KYC#tenantId#date", status: "RUNNING", ttl: +2h }
       │
       ├─  [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/) Load: Lambda → Read candidate records from Aurora
       │              → Filter: only records due for scoring
       │              → Write to S3 input manifest
       │
       ├─  [thecloudsolutions](http://thecloudsolutions.com/blog/aws-architecture-best-practices-2025/) Score: Distributed Map
       │        ├─ Lambda Java: Call scoring engine (internal or external API)
       │        ├─ Write score + evidence → DynamoDB (audit table, immutable)
       │        └─ ToleratedFailurePercentage: 0 (KYC must be complete)
       │
       ├─  [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html) Review Gate: Lambda → Check all scores written
       │                   → Any failures? → Route to DLQ path
       │
       ├─  [aws.amazon](https://aws.amazon.com/blogs/aws/step-functions-distributed-map-a-serverless-solution-for-large-scale-parallel-data-processing/) Publish: Lambda → Push approved scores → downstream SNS topic
       │
       ├─  [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2) Unlock: DynamoDB delete lock item
       │
       ├─ CATCH (all errors):
       │    └─ Lambda: Write failure record → DynamoDB audit
       │    └─ SQS DLQ: Push failed job metadata (tenantId, date, errorCode)
       │    └─ SNS: Alert compliance team
       │    └─ Step: UNLOCK (always runs)
       │
       └─ [Replay]: Separate SF triggered from DLQ
                    └─ Lambda: Read DLQ message
                    └─ Restart from Step  [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/) with same tenantId/date
```

**Audit table (DynamoDB, immutable writes):**
```
PK: SCORE#tenantId#recordId#timestamp
SK: VERSION#v1
Attributes: score, evidence, modelVersion, processedBy, jobId, correlationId
TTL: 7 years (compliance requirement)
```

**Multi-region:** DynamoDB Global Tables + S3 CRR ensure audit records replicate cross-region within seconds. Regional score submissions use the local region's Aurora replica for reads, Global primary for writes.

***
### Architecture 8: Active-Active Scheduler Model (Multi-Region EventBridge)
**WHY:** Pure active-active scheduling risks dual execution — you need a leader-election pattern with fast failover, not geographic load balancing. [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/)

```
REGION A (us-east-1)                REGION B (eu-west-1)
┌──────────────────────┐            ┌──────────────────────┐
│ EventBridge Scheduler│            │ EventBridge Scheduler│
│  rate(5 min)         │            │  rate(5 min)         │
│  └─ Lambda: Gatekeeper│            │  └─ Lambda: Gatekeeper│
│       ├─ SSM Param:  │            │       ├─ SSM Param:  │
│       │  active_region│            │       │  active_region│
│       │  = us-east-1 │            │       │  = us-east-1 │
│       │              │            │       │              │
│       ├─ is_active?  │            │       ├─ is_active?  │
│       │  YES → fire  │            │       │  NO → skip   │
│       │              │            │       │              │
│       └─ Heartbeat → │            │       └─ Watch CW    │
│         DynamoDB GT  │            │         Heartbeat    │
│                      │            │         Age > 10min? │
│ Route53 Health Check │            │  → promote self      │
│  monitors Lambda     │            │  → SSM update        │
│  execution metric    │            │  → start processing  │
└──────────────────────┘            └──────────────────────┘
         │                                      │
         └──────── DynamoDB Global Table ───────┘
                   (heartbeat + leader record)
```

**Failover logic (Java):**
```java
public boolean isActiveRegion() {
    String currentRegion = System.getenv("AWS_REGION");
    String activeRegion = ssmClient.getParameter(r -> 
        r.name("/batch/active_region")).parameter().value();
    
    if (!currentRegion.equals(activeRegion)) {
        // Check if primary is healthy
        Instant lastHeartbeat = getLastHeartbeat(); // from DynamoDB GT
        if (Duration.between(lastHeartbeat, Instant.now()).toMinutes() > 10) {
            promoteToActive(currentRegion); // update SSM + log to CloudWatch
        }
        return currentRegion.equals(getActiveRegion()); // re-check after promotion
    }
    return true;
}
```

***
## 5. Well-Architected Patterns Reference
### Idempotency with DynamoDB
Every Lambda and Fargate worker must implement idempotency. The canonical pattern: [aws.amazon](https://aws.amazon.com/blogs/storage/attaching-block-storage-with-aws-fargate-and-amazon-ebs-volumes/)

```java
// Conditional put — fails silently if already processed
try {
    dynamoDb.putItem(PutItemRequest.builder()
        .tableName("IdempotencyTable")
        .item(Map.of(
            "pk", AttributeValue.fromS(correlationId),
            "status", AttributeValue.fromS("PROCESSING"),
            "ttl", AttributeValue.fromN(String.valueOf(Instant.now().plusSeconds(86400).getEpochSecond()))
        ))
        .conditionExpression("attribute_not_exists(pk)")
        .build());
} catch (ConditionalCheckFailedException e) {
    log.info("Already processed: {}", correlationId);
    return previousResult(correlationId); // return cached result
}
```
### Observability Stack
```
Every Lambda/Fargate/Batch Job:
  ├─ Structured JSON logs → CloudWatch Logs
  │    └─ Fields: correlationId, tenantId, jobId, duration, itemCount, region
  ├─ Custom Metrics → CloudWatch EMF (Embedded Metric Format)
  │    └─ Namespace: BatchProcessing/Jobs
  │    └─ Metrics: RecordsProcessed, FailureRate, ProcessingLatency
  ├─ X-Ray Traces (Lambda auto-instrumentation, Fargate via ADOT)
  │    └─ Trace per batch execution → identify slow segments
  └─ CloudTrail → S3 (API-level audit for compliance)

Dashboards:
  ├─ CloudWatch Dashboard: per-region job health
  ├─ Composite Alarms: SLA breach (99.9% = max 8.7h downtime/year)
  └─ EventBridge rule on Step Functions FAILED → PagerDuty
```
### S3 Partitioning for Batch Inputs
```
s3://batch-input/
  └─ tenant={tenantId}/
       └─ year={yyyy}/month={mm}/day={dd}/hour={hh}/
            └─ records_{partitionKey}.csv
```
- Enables Distributed Map `ItemReader` to filter by prefix per tenant
- Supports Athena queries for ad-hoc analysis without data duplication
- Lifecycle rules: move to S3-IA after 30 days, Glacier after 90 days
### DLQ Replay Workflow
```
SQS DLQ
  └─ EventBridge Pipe (polling DLQ)
       └─ Lambda: DLQ Inspector
            ├─ Parse failure reason
            ├─ Classify: TRANSIENT (retry) vs. PERMANENT (alert)
            └─ TRANSIENT → Re-enqueue to main queue with correlation ID
            └─ PERMANENT → Write to DynamoDB failure log + SNS alert
```

***
## 6. Decision Tree: Choosing the Right Architecture
```
START: What is your workload?
│
├─ SCHEDULE-TRIGGERED?
│    ├─ Simple trigger → EventBridge Scheduler → [target below]
│    └─ Complex DAG (50+ pipelines) → MWAA
│
├─ DATA-TRIGGERED (event-driven)?
│    ├─ S3 file arrival → S3 Event → SQS → Lambda/Batch
│    ├─ DB change → DynamoDB Streams → Lambda
│    └─ API event → EventBridge Rule → Step Functions
│
├─ WHAT IS THE COMPUTE UNIT?
│    ├─ <15 min, stateless → Lambda (Java)
│    ├─ >15 min OR stateful → Fargate/Batch (Java container)
│    └─ Spark-scale transform → Glue (Python/Scala)
│
├─ DO YOU NEED ORCHESTRATION?
│    ├─ YES, with audit trail → Step Functions Standard
│    ├─ YES, high-freq micro-batch → Step Functions Express
│    └─ NO, fire-and-forget → SQS + Lambda/Fargate
│
├─ PARALLELISM SCALE?
│    ├─ <1k items → Step Functions inline Map
│    ├─ 1k–10M items → Step Functions Distributed Map
│    └─ Sustained heavy compute → AWS Batch (Array Jobs)
│
└─ ORDERING REQUIRED?
     ├─ YES (per tenant) → SQS FIFO + Fargate (grouped by tenantId)
     └─ NO → SQS Standard (higher throughput)
```

***
## 7. Best Final Architecture: Combined Production Design
This combines all constraints: multi-region, 300k+ records, multi-tenant, Java-first, cost-sensitive, event-driven.

```
══════════════════════════════════════════════════════════════════════
                    INGESTION LAYER
══════════════════════════════════════════════════════════════════════

External Sources / Internal APIs
  └─ API Gateway / Internal Service
       └─ Kinesis Data Streams (multi-region via Global Tables mirror)
            └─ Lambda: Normalizer (Java)
                 ├─ Extract 150 of 2500 fields (Jackson projection)
                 ├─ Validate schema
                 └─ Write partitioned to S3 + DynamoDB change feed

══════════════════════════════════════════════════════════════════════
                    SCHEDULING LAYER (Active-Active)
══════════════════════════════════════════════════════════════════════

REGION A                              REGION B
EventBridge Scheduler                 EventBridge Scheduler
  └─ Lambda: Leader Gatekeeper          └─ Lambda: Leader Gatekeeper
       ├─ DynamoDB GT: heartbeat              ├─ Watch heartbeat
       └─ SSM: is_primary? fire               └─ Promote if primary unhealthy

══════════════════════════════════════════════════════════════════════
                    ORCHESTRATION LAYER
══════════════════════════════════════════════════════════════════════

Step Functions Standard (Master Orchestrator)
  │
  ├─  [aws.amazon](https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-eventbridge-scheduler-all-aws-regions/?nc1=h_ls) JOB INIT
  │    └─ Lambda: Acquire distributed lock (DynamoDB, TTL-based)
  │    └─ Lambda: Build job manifest (tenants × batch date × S3 prefix)
  │    └─ Write job record → DynamoDB (STARTED status)
  │
  ├─  [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/) DAILY BATCH (300k records)
  │    └─ Distributed Map
  │         ├─ S3 ItemReader (partitioned by tenant)
  │         ├─ ItemBatcher: 100 records
  │         ├─ MaxConcurrency: 500
  │         └─ Child Express SF:
  │              └─ Lambda Java: transform + score + idempotency check
  │              └─ Write → DynamoDB Global Table (results)
  │              └─ Write → S3 results (audit JSONL)
  │
  ├─  [thecloudsolutions](http://thecloudsolutions.com/blog/aws-architecture-best-practices-2025/) ETL ENRICHMENT
  │    └─ Fargate Task (Java Spring Batch)
  │         ├─ Read from DynamoDB / Aurora
  │         ├─ Apply enrichment rules
  │         └─ Write to Aurora Global DB
  │
  ├─  [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html) KYC SCORING (if applicable today)
  │    └─ Parallel State → per-tenant sub-SF
  │         └─ [see Architecture 7 above]
  │
  ├─  [aws.amazon](https://aws.amazon.com/blogs/aws/step-functions-distributed-map-a-serverless-solution-for-large-scale-parallel-data-processing/) MICRO-BATCH SIDECAR (continuous, every 5 min)
  │    └─ SQS Standard → Lambda Java (CRUD async ops)
  │    └─ Step Functions Express (5-min cadence ETL)
  │
  ├─  [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2) VALIDATION & AUDIT
  │    └─ Lambda: Reconcile counts + checksums
  │    └─ Write immutable audit record → DynamoDB (7-year TTL)
  │    └─ CloudTrail + X-Ray → CloudWatch
  │
  ├─ CATCH (all):
  │    └─ SQS DLQ → EventBridge Pipe → Replay Lambda
  │    └─ CloudWatch Alarm → SNS → PagerDuty
  │    └─ DynamoDB: Mark job FAILED with error detail
  │
  └─  [aws.amazon](https://aws.amazon.com/blogs/containers/cost-optimization-checklist-for-ecs-fargate/) CLEANUP
       └─ Lambda: Release lock
       └─ Lambda: Emit completion metrics (CloudWatch EMF)
       └─ SNS: Notify downstream consumers (new data available)

══════════════════════════════════════════════════════════════════════
                    DATA LAYER (Zero Duplication)
══════════════════════════════════════════════════════════════════════

S3 (Single source of truth, partitioned)
  ├─ Raw input (projected 150 fields only — never store full 2500)
  ├─ Processed results (JSONL per batch)
  └─ Audit manifests (success/failure per execution)

DynamoDB Global Tables (multi-region sync)
  ├─ Idempotency table (TTL: 48h)
  ├─ Job state / lock table (TTL: 7 days)
  ├─ Scoring results (business data)
  └─ Audit log (immutable, TTL: 7 years)

Aurora Global Database
  └─ Relational results (primary region write, secondary read)

══════════════════════════════════════════════════════════════════════
                    OBSERVABILITY
══════════════════════════════════════════════════════════════════════

CloudWatch: Custom metrics (EMF) per tenant, per job type
X-Ray: Full trace per batch execution
CloudTrail: API audit log → S3
CloudWatch Composite Alarm: SLA breach → PagerDuty
Cost Allocation Tags: tenantId, jobType, region → per-tenant cost reporting
```

***
## 8. Cost Optimization Summary
Your most impactful levers, ranked: [thecloudsolutions](http://thecloudsolutions.com/blog/aws-architecture-best-practices-2025/)

- **Fargate Spot** for all Batch/ECS tasks — 70% cost reduction on compute
- **Distributed Map with Express children** — pennies per 300k execution vs. Standard SF
- **Field projection at ingestion** — storing 150 fields instead of 2,500 reduces S3/DynamoDB costs by ~94%
- **S3 Intelligent-Tiering** — auto-moves infrequently accessed batch input files
- **Reserved Concurrency caps on Lambda** — prevent runaway costs from upstream spikes
- **DynamoDB on-demand** for idempotency/audit tables (spiky batch pattern) — pay per request
- **Spot with checkpointing** for AWS Batch long-running jobs — resume from checkpoint, not restart

***
## 9. Security & Isolation Model
- **IAM Roles per workload** — each Lambda, Fargate task, Batch job has a task execution role with least-privilege; never share roles across tenants
- **KMS encryption** — all S3 buckets, DynamoDB tables, SQS queues encrypted with customer-managed CMK per tenant for isolation
- **VPC private subnets** — Fargate tasks run in private subnets; S3/DynamoDB via VPC endpoints (no internet traversal)
- **Secrets Manager** — DB credentials rotated automatically; no plaintext in environment variables
- **Resource-based policies** — EventBridge cross-region event buses locked to specific AWS account IDs
- **AWS Config Rules** — continuous compliance checking (no public S3, no unencrypted queues)