---
id: 3005-aws-batch-complete-architecture
slug: aws-batch-complete-architecture
title: "Complete End-to-End AWS Batch Processing Architecture Guide"
category: infrastructure
tags: [aws, batch-processing, scheduling, architecture-design, multi-region, high-availability, 300k-records, eventbridge, step-functions, distributed-map]
description: "Complete end-to-end reference architecture for processing 300k+ daily records across multi-region active-active deployment with 99.9% SLA using EventBridge Scheduler, Step Functions, DynamoDB idempotency, and S3 staging"
difficulty: advanced
estimatedTime: 35
created_at: 2026-04-03T10:00:00Z
updated_at: 2026-04-03T10:00:00Z
---

# Complete End-to-End Architecture Guide - AWS Batch Processing and Scheduling: 
> **Core answer:** For your constraints — multi-region active-active, 300k+ daily records, mixed serverless/container, SLA 99.9% — the optimal architecture combines **EventBridge Scheduler** (multi-region trigger coordination) → **Step Functions Standard** (orchestration spine) → **SQS** (buffering/fan-out) → **Lambda** (parallel micro-tasks) → **AWS Batch / Fargate** (long-running workloads), with **DynamoDB** as the idempotency store and **S3** as the partitioned batch staging layer. This guide covers every dimension.

***
## 1. WHY–WHAT–HOW: All AWS Scheduling & Batch Options
### EventBridge Scheduler
**WHY:** Dedicated scheduler service purpose-built to replace cron-on-EC2 and EventBridge CRON rules. It decouples schedule definition from target execution and supports at-least-once delivery with a retry policy baked in. [cloudthat](https://www.cloudthat.com/resources/blog/comparing-aws-step-functions-and-amazon-eventbridge-scheduler)

**WHAT:**
```
┌─────────────────────────────────────┐
│        EventBridge Scheduler        │
│  ┌────────┐   ┌──────────────────┐  │
│  │ Cron / │──▶│ Schedule Group   │  │
│  │ Rate   │   │ (per tenant/env) │  │
│  └────────┘   └────────┬─────────┘  │
└────────────────────────┼────────────┘
                         ▼
          ┌──────────────────────────┐
          │ Target (Lambda / SFN /   │
          │ SQS / Batch / Fargate)   │
          └──────────────────────────┘
```

**HOW:**
- **Orchestration:** Point-to-point: one schedule → one target invocation. No chaining.
- **Scaling:** Millions of schedules per account; schedule groups allow tenant isolation.
- **Retries:** Built-in retry with configurable `MaxEventAgeInSeconds` + dead-letter SQS target.
- **DLQ:** Native DLQ on the schedule resource itself — failed invocations route here.
- **Monitoring:** CloudWatch metrics per schedule; CloudTrail for audit.
- **Regional Redundancy:** Deploy identical schedule groups in each region. Use a **leader-election** flag in DynamoDB or Parameter Store to prevent dual-execution. Alternatively, run hot-hot and let the downstream target enforce idempotency. [reddit](https://www.reddit.com/r/aws/comments/zmu9bt/eventbridge_scheduler_multi_region_failover/)
- **Versioning:** Schedules are immutable by ARN; update = new version.

**Cost Drivers:** $1.00 per million schedule invocations (first 14M free/month). [cloudthat](https://www.cloudthat.com/resources/blog/comparing-aws-step-functions-and-amazon-eventbridge-scheduler)

**✅ Choose when:** You need simple, reliable cron for triggering pipelines, no workflow logic needed.
**❌ Avoid when:** You need conditional branching, retries with backoff chains, or complex DAGs.

***
### EventBridge Rules (CRON)
**WHY:** The original "cron on AWS" mechanism. Event rules match patterns or fire on schedule, routing to targets. [aws.amazon](https://aws.amazon.com/blogs/compute/introducing-cross-region-event-routing-with-amazon-eventbridge/)

**WHAT:**
```
EventBridge Rule (cron expression)
        │
        ▼
 Event Bus (default or custom)
        │
        ├──▶ Lambda
        ├──▶ SQS
        ├──▶ Step Functions
        └──▶ Cross-Region Bus ──▶ Target Region
```

**HOW:**
- Cross-region event routing is native: define a rule targeting an event bus ARN in another region. [aws.amazon](https://aws.amazon.com/blogs/compute/introducing-cross-region-event-routing-with-amazon-eventbridge/)
- **Regional Redundancy:** Cross-region routing enables centralized event fan-out. Source region fires → routes to multiple target region buses → triggers regional consumers.
- **Retries:** EventBridge retries for 24 hours with exponential backoff for failed targets.
- **DLQ:** Target-level DLQ (SQS queue on the target).

**Cost Drivers:** $1.00 per million custom events; cross-region data transfer adds cost. [aws.amazon](https://aws.amazon.com/blogs/compute/introducing-cross-region-event-routing-with-amazon-eventbridge/)

**✅ Choose when:** You need event-pattern matching + scheduling in the same rule, or cross-region event fan-out.
**❌ Avoid when:** You need millions of unique per-entity schedules (use EventBridge Scheduler instead).

***
### Step Functions Standard Workflows
**WHY:** Visual state machine orchestrator for long-running, auditable business workflows. Exactly-once semantics per step, full execution history, durable state. [cloudthat](https://www.cloudthat.com/resources/blog/comparing-aws-step-functions-and-amazon-eventbridge-scheduler)

**WHAT:**
```
 EventBridge Scheduler
        │
        ▼
 Step Functions (Standard)
 ┌──────────────────────────────────────────┐
 │  IngestTrigger ──▶ ValidateInput         │
 │       ▼                                  │
 │  PartitionRecords ──▶ DistributedMap     │
 │       │          (fan-out to Lambda/ECS) │
 │       ▼                                  │
 │  AggregateResults ──▶ NotifyDownstream   │
 │       │                                  │
 │  [Catch: ALL] ──▶ DLQ / Alert / Retry   │
 └──────────────────────────────────────────┘
```

**HOW:**
- **Orchestration:** State machine as YAML/JSON ASL. Visual in AWS Console. Per-step retry with `IntervalSeconds`, `BackoffRate`, `MaxAttempts`.
- **Scaling:** Horizontal: use `Distributed Map` for parallel item processing (up to 10,000 concurrent child executions). [oneuptime](https://oneuptime.com/blog/post/2026-02-12-use-step-functions-distributed-map-for-large-scale-processing/view)
- **Retries:** Native per-state retry blocks. `Catch` blocks route failures to compensating states.
- **DLQ:** Via `Catch → SQS SendMessage` state or EventBridge Pipes.
- **Monitoring:** Execution history in console + CloudWatch Logs. X-Ray for distributed tracing across Lambda/Fargate sub-calls.
- **Audit Trail:** Every state transition is logged with timestamp, input, output — full replay capability.
- **Idempotency:** Assign deterministic `name` to `StartExecution` calls; SF deduplicates by name within 90 days.
- **Versioning:** State machine versioning + aliasing allows blue/green DAG updates without killing in-flight executions.
- **Regional Redundancy:** Deploy state machines identically in each region. Use EventBridge cross-region routing to trigger the local region's SFN.

**Cost Drivers:** $0.025 per 1,000 state transitions for Standard. For 300k records × ~10 states = 3M transitions/day ≈ $75/day without optimization. **Use Express Workflows for high-frequency, short-duration child executions** inside Distributed Map.

**✅ Choose when:** KYC pipelines, ETL orchestration, anything needing audit history, retries, and conditional branching.
**❌ Avoid when:** You need sub-second latency or true streaming (use Kinesis/Lambda instead).

***
### Step Functions Express Workflows
**WHY:** High-throughput, short-duration (≤5 min) state machines. At-least-once semantics. Priced by duration + invocations, not state transitions. [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2)

**HOW:**
- Used as **child executors** inside Distributed Map or as micro-batch processors.
- Up to 100,000 executions/second.
- Logs to CloudWatch Logs Insights (not execution history console).

**Cost Drivers:** $1.00 per million executions + $0.00001 per GB-second duration. Dramatically cheaper than Standard for high-volume fan-out.

***
### Step Functions Distributed Map
**WHY:** Transforms Step Functions from a workflow tool into a **large-scale parallel data processing engine** capable of processing millions of S3 objects or CSV rows. [oneuptime](https://oneuptime.com/blog/post/2026-02-12-use-step-functions-distributed-map-for-large-scale-processing/view)

**WHAT:**
```
Step Functions (Standard) - Parent
        │
        ▼
 Distributed Map State
 ┌──────────────────────────────────────────┐
 │ ItemReader: S3 CSV / JSON / Inventory    │
 │ MaxConcurrency: up to 10,000             │
 │ ItemBatcher: 100 items/child execution   │
 │ ToleratedFailurePercentage: 2%           │
 │                                          │
 │  Child Executions (Express Workflow)     │
 │  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
 │  │ Lambda   │ │ Lambda   │ │ Fargate  │ │
 │  │ worker   │ │ worker   │ │ (heavy)  │ │
 │  └──────────┘ └──────────┘ └──────────┘ │
 │                                          │
 │ ResultWriter: S3 JSONL output            │
 └──────────────────────────────────────────┘
```

**HOW:**
- `ItemBatcher` groups N items per child — reduces execution overhead for 300k record jobs. [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2)
- `ToleratedFailurePercentage` enables partial failure isolation — 2% of batches can fail without aborting the full run.
- Per-item retry inside child Express Workflow.
- Results written to S3 with `ResultWriter`; aggregate downstream via Athena or Glue.

**Cost Drivers:** 300k records ÷ 100 items/batch = 3,000 child Express executions. At $1/million executions ≈ **$0.003 per daily run** for orchestration overhead alone — extremely cost-efficient.

**✅ Choose when:** Daily 300k+ record batch processing, ETL fan-out, scoring jobs with high parallelism.

***
### SQS + Lambda Fan-out
**WHY:** The simplest and most cost-effective pattern for parallel, stateless record processing. SQS decouples producers from consumers and provides built-in buffering, retry, and DLQ. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)

**WHAT:**
```
Producer (Batch trigger / upstream service)
        │
        ▼
   SQS Standard Queue
   (visibility timeout = 2.5x avg processing time)
        │
        ▼
   Lambda (Event Source Mapping)
   ┌──────────────────────────────────┐
   │ Batch size: 10 messages          │
   │ Concurrency: up to 1,000         │
   │ bisectBatchOnError: true         │
   └──────────────────────────────────┘
        │
        ▼
   DLQ (SQS) ◄── Failed after maxReceiveCount (3)
```

**HOW:**
- **bisectBatchOnError:** On partial batch failure, Lambda splits batch in half and retries — isolates poison-pill messages. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)
- **Idempotency:** Lambda Powertools idempotency decorator using DynamoDB — each message's `MessageId` is the idempotency key.
- **Scaling:** Lambda scales to match SQS queue depth automatically. For 300k records arriving at once, Lambda can spin up ~1,000 concurrent functions in seconds.
- **DLQ:** After `maxReceiveCount` retries, messages go to SQS DLQ → trigger alerting Lambda or replay workflow.
- **Regional Redundancy:** Deploy SQS + Lambda in each region. EventBridge routes region-specific work to regional SQS.

**Cost Drivers:** SQS: $0.40/million requests. Lambda: $0.20/million invocations + duration. For 300k daily records at 100ms/record ≈ **under $5/day** total.

**✅ Choose when:** Stateless, parallelizable record-level processing. CRUD async operations, score updates.
**❌ Avoid when:** You need strict ordering (use FIFO), or job state tracking (use Step Functions).

***
### SQS FIFO + Fargate Job Consumers
**WHY:** When processing order matters per entity (e.g., per-customer KYC state transitions), FIFO queues with `MessageGroupId = customerId` guarantee ordered delivery. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)

**WHAT:**
```
  SQS FIFO Queue
  MessageGroupId = {tenantId}#{entityId}
        │
        ▼
  ECS Service (Fargate)
  ┌─────────────────────────────────┐
  │ Long-polling consumer           │
  │ Processes one group at a time   │
  │ Graceful shutdown: SIGTERM hook │
  └─────────────────────────────────┘
        │
        ▼
  DLQ (SQS FIFO)
```

**HOW:**
- FIFO queues support up to 3,000 messages/second per API action with batching.
- Fargate tasks scale via ECS Service Auto Scaling driven by `ApproximateNumberOfMessages` CloudWatch metric.
- **Graceful shutdown:** Fargate sends `SIGTERM` 30 seconds before task termination. Implement shutdown hook to finish current message, delete from queue, then exit cleanly.
- **Cost:** Fargate Spot saves up to 70% vs. on-demand for fault-tolerant consumers. [aws.amazon](https://aws.amazon.com/blogs/containers/cost-optimization-checklist-for-ecs-fargate/)

***
### AWS Batch
**WHY:** Fully managed job scheduler for compute-intensive, container-based workloads. Handles job queues, compute environment provisioning, priority, array jobs, and retry — you provide the container. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

**WHAT:**
```
  Job Definition (Docker image + resources)
        │
        ▼
  Job Queue (priority-ordered)
        │
  ┌─────┴──────────────────┐
  │ Compute Environment 1  │◄── Spot (primary)
  │ Compute Environment 2  │◄── On-Demand (fallback)
  └────────────────────────┘
        │
        ▼
  ECS/EKS tasks (Fargate or EC2)
        │
        ▼
  CloudWatch Logs + S3 output
```

**HOW:**
- **Array Jobs:** Submit 300k records as a single array job of size 300,000. Each array child gets `AWS_BATCH_JOB_ARRAY_INDEX` as its shard identifier.
- **Retry Strategy:** Up to 10 attempts per job. Status reasons can trigger conditional retry logic (e.g., retry on `CannotPullContainerError`, not on `OutOfMemory`).
- **Job Dependencies:** `dependsOn` for DAG-style sequencing.
- **Spot Integration:** Define primary CE as Fargate Spot + fallback CE as Fargate On-Demand; AWS Batch selects automatically.
- **Cost Optimization:** Fargate Spot for 70% savings on fault-tolerant batch workloads. [aws.amazon](https://aws.amazon.com/blogs/containers/cost-optimization-checklist-for-ecs-fargate/)

**Cost Drivers:** Pay only for compute time. EC2 Spot array jobs are the most cost-efficient for CPU-intensive ETL. Fargate removes bin-packing overhead.

**✅ Choose when:** Long-running Java/Python ETL jobs, scoring jobs, compute-heavy processing without custom orchestration.
**❌ Avoid when:** You need workflow-level branching/conditions (combine with Step Functions as orchestrator).

***
### ECS/Fargate Scheduled Tasks
**WHY:** Lightweight cron for container workloads — EventBridge Scheduler or CRON rule triggers an ECS `RunTask` API call. [builder.aws](https://builder.aws.com/content/34jTIHSM6RwweiUIMU4GJcgSVRs/how-to-schedule-ecs-tasks-choosing-between-eventbridge-scheduler-eventbridge-rules-and-step-functions)

**HOW:**
- Use **EventBridge Scheduler** → ECS `RunTask` target for reliability and DLQ support.
- No orchestration logic — container is responsible for its own retry.
- Best for periodic cleanup jobs, report generation, low-complexity nightly tasks.

**✅ Choose when:** Simple container jobs that own their own logic with no complex retry DAGs.
**❌ Avoid when:** You need state tracking, partial failure isolation, or dependency chains.

***
### AWS Glue Jobs / Glue Workflows
**WHY:** Managed Spark-based ETL engine for large-scale data transformation, cataloging, and movement. Auto-generates PySpark/Scala code; deep S3, Redshift, RDS integration. [community](https://community.aws/content/2n7nKFwWWmJs5Bg9s2UNoQO0Ydn/how-to-architect-a-high-performance-batch-processing-pipeline-with-apache-spark-on-aws)

**HOW:**
- **Glue Workflows:** DAG of Glue Jobs and Crawlers with trigger conditions.
- **DPU-based pricing:** Glue G.1X = 4 vCPU, 16GB. $0.44/DPU-hour. For a 2-hour 10-DPU job: $8.80/run.
- **Pushdown predicates** on S3 Parquet partitions dramatically reduce scanned data.
- **Bookmarks:** Native job bookmarking for incremental processing — reads only new partitions since last run.

**✅ Choose when:** Large-scale Spark-based ETL, data cataloging, schema evolution, Redshift/S3 lake pipelines.
**❌ Choose when:** Sub-minute latency, custom Java processing logic, or cost-sensitive small-record workloads (Glue minimum 1 DPU-minute billing).

***
### Amazon MWAA (Managed Airflow)
**WHY:** Fully managed Apache Airflow for complex DAG orchestration when you need Python-native workflow definition, rich operators, and a data engineering team familiar with Airflow. [community](https://community.aws/content/2n7nKFwWWmJs5Bg9s2UNoQO0Ydn/how-to-architect-a-high-performance-batch-processing-pipeline-with-apache-spark-on-aws)

**HOW:**
- DAGs are Python files in S3; MWAA polls S3 and loads them automatically.
- Workers scale from `mw1.small` to `mw1.4xlarge`.
- **Multi-region:** MWAA does not natively support active-active. You must deploy separate MWAA environments per region with shared DAG code in replicated S3 buckets and an external leader-election mechanism.
- **Cost:** MWAA environment: ~$0.49–$5.76/hour depending on size — **significant baseline cost**. Not cost-efficient for infrequent jobs.

**✅ Choose when:** Data engineering teams with existing Airflow expertise, complex cross-system DAGs with 50+ tasks, ML pipelines.
**❌ Avoid when:** Cost is a primary constraint, or multi-region active-active is required without heavy operational investment.

***
### Lambda with DynamoDB Streams / Kinesis / S3 Events
**WHY:** Pure event-driven, zero-polling architecture. Changes in data stores directly trigger Lambda — ideal for CRUD-side async operations and micro-batch patterns. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)

**WHAT:**
```
  DynamoDB Table ──▶ DynamoDB Stream ──▶ Lambda
  S3 Bucket     ──▶ S3 Event Notification ──▶ Lambda
  Kinesis Stream ──▶ Lambda (shard-based)
                          │
                          ▼
                   Downstream services
                   (SQS / Step Functions / RDS)
```

**HOW:**
- **DynamoDB Streams:** Up to 2 Lambda consumers per stream shard. NEW_AND_OLD_IMAGES capture for audit.
- **Kinesis:** Per-shard Lambda invocation with `bisectBatchOnError` and enhanced fan-out for multiple consumers.
- **S3 Events:** Object creation → Lambda → process file → write results. Use S3 key prefixes for tenant isolation and S3 Event Notifications with SQS intermediary for reliability.

**✅ Choose when:** CRUD-side async enrichment, real-time KYC triggers, change-data-capture patterns.

***
### EC2 Spot-Based Batch Clusters
**WHY:** Maximum compute cost reduction (~90% vs. on-demand) for massively parallel, fault-tolerant batch workloads. Self-managed but gives full control over instance type, memory, GPU. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

**HOW:**
- Use with AWS Batch (EC2 compute environment) or EMR for Spark workloads.
- **Spot interruption handling:** Spot interruption notice = 2-minute warning. Implement checkpoint logic to save progress to S3 before termination.
- **Diversification:** Specify 5+ instance types to maximize Spot pool availability.

**✅ Choose when:** Compute-intensive ETL (Spark/Hadoop), ML training, genomics, rendering — where checkpoint/restart is viable.
**❌ Avoid when:** SLA-sensitive jobs where interruption is unacceptable without fallback On-Demand CE.

***
### AWS Data Pipeline (Legacy)
**WHY:** Original AWS ETL orchestration service (circa 2012). Now effectively deprecated in favor of Glue, Step Functions, and MWAA. [community](https://community.aws/content/2n7nKFwWWmJs5Bg9s2UNoQO0Ydn/how-to-architect-a-high-performance-batch-processing-pipeline-with-apache-spark-on-aws)

**❌ Avoid:** AWS explicitly recommends migrating to Glue Workflows or Step Functions. No new features, limited regional support.

***
## 2. Side-by-Side Comparison Matrix
| Service | Best For | Scalability | Cost Model | Long-Running? | Multi-Region | Orchestration | Audit | DLQ |
|---|---|---|---|---|---|---|---|---|
| **EventBridge Scheduler** | Cron triggers | Millions of schedules | Per invocation | ❌ Trigger only | Hot-hot per region  [reddit](https://www.reddit.com/r/aws/comments/zmu9bt/eventbridge_scheduler_multi_region_failover/) | None (trigger only) | CloudTrail | Native |
| **EventBridge Rules** | Event patterns + CRON | Very high | Per event | ❌ | Cross-region routing  [aws.amazon](https://aws.amazon.com/blogs/compute/introducing-cross-region-event-routing-with-amazon-eventbridge/) | None | CloudTrail | Target-level |
| **Step Functions Standard** | Complex workflow orchestration | Thousands of executions | Per state transition | ✅ | Per-region deploy | Full DAG | Full execution history | Via Catch |
| **Step Functions Express** | High-volume short tasks | 100k exec/sec | Duration + invocations | ❌ (5 min max) | Per-region deploy | Partial | CloudWatch Logs | Via Catch |
| **SF Distributed Map** | Massive parallel batch | 10,000 concurrent children | Express pricing | ✅ (parent) | Per-region | Full DAG | Full + S3 results | ToleratedFailure% |
| **SQS + Lambda** | Stateless parallel processing | Auto-scales to queue depth | Per message + Lambda | ❌ (15 min max) | Regional SQS + Lambda | None | CloudWatch | Native SQS DLQ |
| **SQS FIFO + Fargate** | Ordered per-entity processing | CE autoscaling | Fargate + SQS | ✅ | Regional deploy | None | CloudWatch | FIFO DLQ |
| **AWS Batch** | Compute-intensive containers | Array jobs, CE autoscale | Compute time only | ✅ | Per-region Batch | Job DAG | CloudWatch Logs | Job retry + alerts |
| **ECS Scheduled Tasks** | Simple periodic containers | Manual scaling | Fargate per-task | ✅ | Per-region | None | CloudWatch | EventBridge DLQ |
| **Glue Jobs/Workflows** | Spark ETL, data catalog | DPU autoscaling | Per DPU-hour (min 1 min) | ✅ | Via S3 replication | Glue Workflow | Glue history | Job retry |
| **MWAA** | Complex data engineering DAGs | Worker autoscaling | Per environment/hour | ✅ | Manual dual-env | Full Airflow DAG | Airflow logs | Task retry |
| **Lambda + Streams** | CRUD async, CDC | Per shard | Per invocation | ❌ | Per-region streams | None | X-Ray + CloudWatch | SQS DLQ on ESM |
| **EC2 Spot Clusters** | Max-compute ETL | Manual ASG | Spot pricing (~90% off) | ✅ | Manual | None (self-managed) | Custom | Checkpoint + retry |
| **Data Pipeline** | Legacy only | Limited | Per activity | ✅ | Limited | Limited DAG | Limited | Limited |

***
## 3. Architecture Examples
### Architecture 1: Daily 300k+ Record Processing (Multi-Region)
```
REGION us-east-1                          REGION eu-west-1
┌───────────────────────────────┐         ┌───────────────────────────────┐
│                               │         │                               │
│  EventBridge Scheduler        │         │  EventBridge Scheduler        │
│  (Daily @ 02:00 UTC)         │         │  (Daily @ 02:00 UTC)         │
│         │                     │         │         │                     │
│         ▼                     │         │         ▼                     │
│  DynamoDB Leader Check        │         │  DynamoDB Leader Check        │
│  (Global Table)               │◄───────▶│  (Global Table)               │
│  if leader=this-region:       │         │  if leader=this-region:       │
│    proceed; else: standby     │         │    proceed; else: standby     │
│         │                     │         │                               │
│         ▼                     │         └───────────────────────────────┘
│  Step Functions (Standard)    │
│  ┌────────────────────────┐   │
│  │ 1. ReadManifest(S3)    │   │
│  │ 2. DistributedMap      │   │
│  │    (3000 child execs)  │   │
│  │    ItemBatcher: 100    │   │
│  │    MaxConcurrency: 500 │   │
│  │    each child →        │   │
│  │    Lambda (150 fields) │   │
│  │ 3. AggregateResults    │   │
│  │ 4. WriteAuditLog(S3)   │   │
│  └────────────────────────┘   │
│         │                     │
│         ▼                     │
│  S3 (results, partitioned     │
│  by date/tenant/batch-id)     │
└───────────────────────────────┘
```

**Flow:**
1. EventBridge Scheduler fires in both regions simultaneously. [reddit](https://www.reddit.com/r/aws/comments/zmu9bt/eventbridge_scheduler_multi_region_failover/)
2. Each region checks a DynamoDB Global Table `leader_election` item (conditional write with TTL). The region that wins the write becomes the leader for this run.
3. Leader region's Step Functions Standard workflow begins.
4. Step 1: Reads S3 manifest (list of input file keys for today's batch).
5. Step 2: Distributed Map reads S3 CSV directly; `ItemBatcher` groups 100 records/child Express Workflow.
6. Each child Lambda extracts only 150 required fields (field projection at read time — no full object deserialization).
7. `ToleratedFailurePercentage: 2%` — up to 6,000 records can fail without aborting the run.
8. Step 3: Aggregate child results from S3 `ResultWriter` output.
9. Step 4: Audit log written to S3 + DynamoDB audit table.

**Retry Strategy:** Per-state retry in parent SFN (3 attempts, 2x backoff). Per-item retry inside Express child (3 attempts). Failed batches written to S3 `failures/` prefix for replay.

**Idempotency Model:** `StartExecution` called with `name = "daily-batch-{date}-{batchId}"` — Step Functions deduplicates within 90 days. Lambda workers check DynamoDB idempotency table with `batchId#itemIndex` key before processing.

**Field Projection (2500 → 150 fields):** Input S3 files use **Parquet with column pruning** — Lambda reads only required columns using S3 Select or AWS SDK's Parquet column projection. Zero unnecessary data deserialization.

**Cost Estimate (daily):**
- SFN Distributed Map: ~$0.003 (3,000 Express executions)
- Lambda: 3,000 × 100 items × 100ms avg × 512MB = ~$1.50
- S3 requests + storage: ~$0.50
- **Total: ~$2–5/day for 300k records**

***
### Architecture 2: Nightly ETL Pipeline (Java + Python Mix)
```
EventBridge Scheduler (nightly 01:00 UTC)
        │
        ▼
Step Functions Standard (Orchestrator)
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Extract (Python Lambda)                            │
│  ┌──────────────────────────────┐                   │
│  │ Read from RDS/Aurora         │                   │
│  │ Write partitioned Parquet    │                   │
│  │ to S3 (date=YYYY/MM/DD/      │                   │
│  │ tenant=X/batch=Y/)           │                   │
│  └──────────────────────────────┘                   │
│         │                                           │
│         ▼                                           │
│  Transform (Java Fargate on AWS Batch)              │
│  ┌──────────────────────────────┐                   │
│  │ Array Job (N shards by       │                   │
│  │ tenant partition)            │                   │
│  │ Java 21 + Spring Batch       │                   │
│  │ Reads S3 Parquet             │                   │
│  │ Applies business rules       │                   │
│  │ Writes transformed Parquet   │                   │
│  │ Checkpoint: S3 state file    │                   │
│  └──────────────────────────────┘                   │
│         │                                           │
│         ▼                                           │
│  Load (Python Glue Job)                             │
│  ┌──────────────────────────────┐                   │
│  │ Read transformed S3 data     │                   │
│  │ Upsert to Redshift / Aurora  │                   │
│  │ Update Glue Catalog          │                   │
│  │ Job bookmark for idempotency │                   │
│  └──────────────────────────────┘                   │
│         │                                           │
│  [Catch: ALL] ──▶ SQS DLQ ──▶ Alert Lambda         │
└─────────────────────────────────────────────────────┘
```

**Java + Python Integration:**
- Python Lambda handles lightweight I/O-bound extract (boto3, pandas, pyarrow).
- Java Fargate (Spring Batch) handles CPU-intensive transformations with domain model richness — `@StepScope` beans for per-tenant partitioning.
- Python Glue handles Spark-based load to analytical stores.
- All exchange data via **S3 Parquet** — the universal lingua franca between runtimes with no schema lock-in.

**Multi-Region:** Step Functions deployed identically in each region. S3 Cross-Region Replication (CRR) propagates input data. Regional Step Functions execute against regional S3 + Batch compute — no cross-region data transfer during processing.

**Checkpointing:** Java Spring Batch writes `JobExecutionContext` to S3 every 1,000 records. On Spot interruption, new Fargate task reads checkpoint and resumes from last committed position. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

***
### Architecture 3: Real-Time Micro-Batching Every 5 Minutes
```
Upstream Events (CRUD ops, score triggers)
        │
        ▼
  SQS Standard Queue
  (visibility timeout: 90s)
        │
        │ (EventBridge Scheduler: every 5 min)
        ▼
  Lambda Orchestrator (micro-batch trigger)
  ┌────────────────────────────────────┐
  │ 1. ReceiveMessage(MaxMessages=300) │
  │ 2. Group by tenant/type            │
  │ 3. StartExecution (Express SFN)   │
  │    one per group                   │
  └────────────────────────────────────┘
        │
        ▼
  Step Functions Express (per micro-batch)
  ┌────────────────────────────────────┐
  │ Validate → Enrich → Score →        │
  │ Persist → PublishEvents            │
  └────────────────────────────────────┘
        │
        ▼
  DynamoDB (results + idempotency)
  EventBridge (downstream events)
```

**Key Design:**
- SQS acts as a **buffer** absorbing bursty upstream writes between 5-minute windows. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)
- Lambda orchestrator drains up to 300 messages per trigger, groups them by tenant for multi-tenant isolation.
- Express Workflows process each group with full state tracking but at Express pricing.
- `visibilityTimeout = 90s` (2× avg processing time of 45s) prevents reprocessing of in-flight batches.
- **Idempotency:** Each SQS `MessageId` stored in DynamoDB with TTL=24h. Lambda checks before processing.

***
### Architecture 4: Container-Based Long-Running Batch (1–3 Hours)
```
EventBridge Scheduler
        │
        ▼
Step Functions Standard
        │
        ▼
  AWS Batch Job (Fargate Spot + On-Demand fallback)
  ┌──────────────────────────────────────────────┐
  │ Job Definition:                              │
  │   Image: ECR (Java 21 Spring Batch)          │
  │   vCPU: 4, Memory: 16GB                      │
  │   Retry: 3 (on SpotInterruption)             │
  │                                              │
  │ Processing Loop:                             │
  │   for shard in assigned_partition:           │
  │     1. Read from S3 (150-field projection)   │
  │     2. Apply business rules                  │
  │     3. Write output to S3                    │
  │     4. Checkpoint: s3://checkpoints/{jobId}  │
  │        /{shardId}/{lastProcessedOffset}      │
  │     5. On SIGTERM: flush + write checkpoint  │
  │        + delete SQS message + exit(0)        │
  └──────────────────────────────────────────────┘
        │
        ▼
  Step Functions waits (`.sync:2` integration)
  → Success: trigger downstream
  → Failure: Catch → DLQ → alert
```

**Graceful Shutdown Pattern:**
```java
// Spring Batch: register JVM shutdown hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    jobLauncher.stop(); // drain current chunk
    checkpointService.save(currentOffset, s3Client); // write to S3
    sqsClient.deleteMessage(receiptHandle); // ack SQS
}));
```

**Cost Model (1-hour job, 4 vCPU/16GB Fargate Spot):**
- Fargate Spot: ~$0.03/vCPU-hour × 4 + $0.003/GB-hour × 16 = ~$0.17/hour → **$0.17–$0.51 per run**
- With On-Demand fallback: ~$0.56–$1.68/run
- Spot saves ~70% [aws.amazon](https://aws.amazon.com/blogs/containers/cost-optimization-checklist-for-ecs-fargate/)

***
### Architecture 5: Lambda-Based Parallel Fan-Out / Fan-In
```
Orchestrator Lambda (fan-out)
        │
        ├──▶ SQS (work queue)
        │         │
        │    Lambda workers (N concurrent)
        │    ┌──────────┐ ┌──────────┐ ┌──────────┐
        │    │ worker 1 │ │ worker 2 │ │ worker N │
        │    └────┬─────┘ └────┬─────┘ └────┬─────┘
        │         │             │             │
        │         └──────┬──────┘             │
        │                └────────────────────┘
        │                         │
        ▼                         ▼
  DynamoDB Counter Table    S3 Results Prefix
  (atomic IncrementCounter  (one object per worker)
   per jobId)
        │
        │ (DynamoDB Streams → Lambda fan-in trigger)
        ▼
  Fan-In Lambda (when counter == total_shards)
  ┌─────────────────────────────────┐
  │ List S3 results/jobId/*         │
  │ Merge/aggregate results         │
  │ Write final output              │
  │ Notify downstream               │
  └─────────────────────────────────┘
```

**Fan-In Detection:** Use DynamoDB atomic counter `UpdateItem` with `ADD completedCount 1`. When `completedCount == totalShards` (checked in conditional expression), trigger fan-in Lambda via DynamoDB Streams. [aws.amazon](https://aws.amazon.com/blogs/storage/attaching-block-storage-with-aws-fargate-and-amazon-ebs-volumes/)

**Failure Isolation:** Each worker Lambda has independent retry. Failed workers go to SQS DLQ. Fan-in Lambda checks DLQ depth before aggregating — if DLQ > 0, marks job as PARTIAL_SUCCESS and triggers alert.

***
### Architecture 6: Step Functions Distributed Map for High Parallelism
```
S3 Input: s3://data-bucket/input/batch-2026-03-29.csv
(300,000 rows, Parquet/CSV)
        │
        ▼
Step Functions Standard (Parent)
┌─────────────────────────────────────────────────┐
│  DistributedMap State:                          │
│    ItemReader: S3 CSV                           │
│    ItemBatcher: MaxItemsPerBatch=100            │
│    MaxConcurrency: 500                          │
│    ToleratedFailurePercentage: 2               │
│                                                 │
│  Child Express Workflow (3,000 executions):     │
│  ┌───────────────────────────────────────────┐  │
│  │ ValidateRecords → TransformFields →       │  │
│  │ EnrichFromCache(ElastiCache) →            │  │
│  │ WriteToS3(results/batch/{jobId}/{i})      │  │
│  └───────────────────────────────────────────┘  │
│                                                 │
│  ResultWriter: s3://results/map-output/         │
│  → JSONL (one line per child execution result)  │
│                                                 │
│  Next: GlueJob (aggregate JSONL → Parquet)      │
└─────────────────────────────────────────────────┘
```

This pattern handles 300k records in under 10 minutes at $0.003 for SF orchestration alone. [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2)

***
### Architecture 7: KYC Periodic Scoring with Audit, DLQ, and Replay
```
EventBridge Scheduler (daily/weekly per-tenant cron)
        │
        ▼
Step Functions Standard (KYC Orchestrator)
┌───────────────────────────────────────────────────┐
│                                                   │
│  LoadEntityBatch                                  │
│  ┌──────────────────────────────────────────┐     │
│  │ DynamoDB: query by tenant + status=PENDING│    │
│  │ Write batch manifest to S3               │     │
│  └──────────────────────────────────────────┘     │
│         │                                         │
│  DistributedMap (per entity):                     │
│  ┌──────────────────────────────────────────┐     │
│  │ 1. FetchEntityData (S3 / DynamoDB)       │     │
│  │ 2. RunKYCScoreModel (Lambda Python)      │     │
│  │    - ML model inference                  │     │
│  │    - 150-field input vector              │     │
│  │ 3. WriteScore (DynamoDB + S3 audit)      │     │
│  │    - Include: scoreValue, modelVersion,  │     │
│  │      inputs snapshot, timestamp, runId   │     │
│  │ 4. EmitKYCEvent (EventBridge)            │     │
│  │    - downstream: alerts, notifications   │     │
│  └──────────────────────────────────────────┘     │
│         │                                         │
│  [Catch: States.ALL]                             │
│  → SQS DLQ (kyc-scoring-dlq.fifo)               │
│    MessageGroupId = entityId                     │
│    MessageBody = {entityId, runId, error,        │
│                   inputSnapshot, timestamp}      │
└───────────────────────────────────────────────────┘

Replay Workflow (manual or triggered):
  SQS DLQ ──▶ Replay Lambda ──▶ Re-enqueue to
              (validates + enriches)  main SFN
```

**Audit Trail Design:**
- Every KYC run writes a DynamoDB item: `PK = entityId, SK = runId#timestamp` with TTL = 7 years (regulatory).
- S3 audit bucket with `Object Lock (COMPLIANCE mode)` — immutable records.
- CloudTrail data events on audit S3 bucket capture all reads.
- X-Ray traces link API calls → SFN execution → Lambda → DynamoDB write in a single trace.

**Idempotency:** `runId = SHA256(entityId + scoreDate + modelVersion)` — duplicate runs for same entity/date/model version are no-ops (DynamoDB conditional write on PK existence).

**Multi-Tenant Isolation:** Each tenant's KYC schedule runs in separate SFN execution with IAM role scoped to tenant's S3 prefix and DynamoDB partition.

***
### Architecture 8: Active-Active Scheduler Model with EventBridge Across Regions
```
┌─────────────────────────────────────────────────────────────────────┐
│                     GLOBAL CONTROL PLANE                            │
│                                                                     │
│  DynamoDB Global Table: scheduler_leases                            │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  PK: job_id  │ leaseOwner: region  │ TTL: 10min │ runId  │       │
│  └──────────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────────┘
          ▲                                           ▲
          │                                           │
┌─────────┴────────────────┐           ┌─────────────┴────────────────┐
│     us-east-1            │           │     eu-west-1                │
│                          │           │                              │
│  EB Scheduler            │           │  EB Scheduler                │
│  (same cron)             │           │  (same cron)                 │
│       │                  │           │       │                      │
│       ▼                  │           │       ▼                      │
│  Lambda: TryAcquireLease │           │  Lambda: TryAcquireLease     │
│  (DynamoDB condWrite)    │           │  (DynamoDB condWrite)        │
│  if SUCCESS → proceed    │           │  if FAIL → skip (standby)    │
│  if FAIL → skip (standby)│           │  if SUCCESS → proceed        │
│       │                  │           │                              │
│       ▼                  │           └──────────────────────────────┘
│  Step Functions          │
│  (executes workload)     │
│       │                  │
│       ▼                  │
│  LeaseRelease Lambda     │
│  (TTL refresh or delete) │
└──────────────────────────┘

FAILOVER: If us-east-1 crashes mid-job, DynamoDB Global Table lease
          TTL expires → eu-west-1 wins next EB Scheduler tick →
          resumes from S3 checkpoint or re-runs with idempotency guard.
```

**Key properties of this pattern:** [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/)
- Both regions fire simultaneously — no primary/secondary configuration lag.
- DynamoDB Global Table with conditional writes implements distributed mutex.
- Lease TTL = max expected job setup time (e.g., 10 minutes). If primary crashes, secondary acquires lease on next tick.
- Downstream jobs are always idempotent — if both regions somehow execute (split-brain edge case), DynamoDB conditional writes at the record level prevent double-processing.

***
## 4. Well-Architected Patterns Deep Dive
### Event-Driven Batch Orchestration
Replace time-triggered polling with **event-driven pipeline advancement**. Each stage completion emits an EventBridge event that triggers the next stage — zero idle polling. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)

```
S3 PutObject → S3 Event → SQS → Lambda (detect batch complete)
    → EventBridge PutEvents → SFN StartExecution
```
### Fan-Out / Fan-In with SQS + Lambda + Step Functions
Use Step Functions `Map` state to generate work items → SQS to buffer → Lambda to process → DynamoDB atomic counter for fan-in detection. This pattern decouples fan-out rate from Lambda concurrency limits. [sls](https://www.sls.guru/blog/batch-processing-with-step-functions-map-states---part-1)
### Checkpointing for Long-Running Jobs
For Fargate/Batch jobs running 1–3 hours:
- Write checkpoint to `s3://checkpoints/{jobId}/{shardId}/offset.json` every N records.
- On restart, read checkpoint → skip already-processed records.
- Use S3 conditional writes (`If-None-Match`) to prevent checkpoint races in multi-instance scenarios.
### SQS Buffering for Micro-Batches
```
Producers ──▶ SQS ──▶ (every 5 min) Lambda drains queue
                       groups by type/tenant
                       starts batch execution
```
Long-polling (`WaitTimeSeconds=20`) reduces empty receives by 99%, cutting SQS costs significantly. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)
### DLQ, Retry, and Replay Workflows
```
Primary Queue ──▶ Processing ──▶ Success → delete message
                      │
                      ▼ (after maxReceiveCount=3)
                  SQS DLQ
                      │
              ┌───────┴────────┐
              │                │
         Alert Lambda    Replay Lambda
         (PagerDuty/     (re-enqueue to
          SNS)            primary after
                          fix)
```

**Replay Pattern:** Replay Lambda reads from DLQ, optionally enriches/transforms, re-enqueues to primary SQS with `MessageDeduplicationId` reset — triggers full reprocessing with fresh retry counters.
### Observability Stack
```
Lambda / Fargate / SFN
        │
        ├──▶ X-Ray (distributed traces across all services)
        ├──▶ CloudWatch Logs (structured JSON logs with correlationId)
        ├──▶ CloudWatch Metrics (custom: items_processed, items_failed,
        │    processing_duration_p99, dlq_depth)
        ├──▶ CloudWatch Alarms → SNS → PagerDuty
        └──▶ CloudTrail (API audit — who started what, when)

EMF (Embedded Metrics Format):
  { "_aws": { "Timestamp": ..., "CloudWatchMetrics": [...] },
    "tenantId": "acme", "jobId": "...", "itemsProcessed": 1000 }
```
### DynamoDB Idempotency Controls
```python
# Lambda Powertools Idempotency
from aws_lambda_powertools.utilities.idempotency import (
    idempotent, DynamoDBPersistenceLayer
)

persistence_layer = DynamoDBPersistenceLayer(
    table_name="IdempotencyTable",
    key_attr="id",
    expiry_attr="expiration",
    status_attr="status",
    data_attr="data"
)

@idempotent(persistence_store=persistence_layer)
def handler(event, context):
    # Safe to retry — Lambda Powertools handles dedup
    return process_record(event)
```

Idempotency key = `SHA256(jobId + recordId + processingDate)`. TTL = 24–48 hours. [aws.amazon](https://aws.amazon.com/blogs/storage/attaching-block-storage-with-aws-fargate-and-amazon-ebs-volumes/)
### Fargate Spot for Cost-Optimized Compute
Configure ECS capacity provider with `FARGATE_SPOT` weight 3, `FARGATE` weight 1. ECS places tasks on Spot first; on interruption, drains and replaces with on-demand. Up to 70% savings on batch workloads. [aws.amazon](https://aws.amazon.com/blogs/containers/cost-optimization-checklist-for-ecs-fargate/)
### S3 Partitioning for Batch Inputs
```
s3://batch-data/
  input/
    year=2026/month=03/day=29/
      tenant=acme/
        batch_001.parquet
        batch_002.parquet
      tenant=globex/
        batch_001.parquet
  processing/
    {jobId}/
      checkpoints/
      results/
  output/
    year=2026/month=03/day=29/
      tenant=acme/
```

Partition by `year/month/day/tenant` enables:
- Athena query cost reduction (partition pruning).
- Per-tenant IAM policies scoped to S3 prefix.
- Distributed Map `ItemReader` to target a specific date partition.
- Glue job bookmark to track last processed partition.

***
## 5. Decision Tree for Architecture Selection
```
START: What is your batch workload?
        │
        ├── Duration < 15 minutes AND stateless?
        │         │
        │         ├── High parallelism needed? → SQS + Lambda fan-out
        │         └── Simple trigger? → EventBridge Scheduler → Lambda
        │
        ├── Duration 15 min – 3 hours?
        │         │
        │         ├── Complex workflow logic? → Step Functions + Fargate/Batch
        │         └── Simple container job? → ECS Scheduled Task or Batch
        │
        ├── Need parallel processing of 10k+ items?
        │         │
        │         └── Step Functions Distributed Map (→ Lambda or Fargate child)
        │
        ├── Spark/large-scale SQL ETL?
        │         │
        │         ├── Existing Airflow expertise → MWAA
        │         └── AWS-native → Glue Workflows
        │
        ├── Event-driven (triggered by data change)?
        │         │
        │         ├── DynamoDB changes → DynamoDB Streams + Lambda
        │         ├── S3 arrivals → S3 Event Notifications + Lambda/SQS
        │         └── Kinesis streams → Lambda ESM + Kinesis
        │
        └── Multi-region active-active scheduler?
                  └── EventBridge Scheduler (hot-hot) + DynamoDB lease election
                      → regional Step Functions execution
```

***
## 6. Best Final Architecture
This is the unified reference architecture for your constraints — multi-region active-active, 300k+ daily records, mixed serverless/container, 99.9% SLA, multi-tenant, cost-sensitive.

```
═══════════════════════════════════════════════════════════════════════
                    BEST FINAL ARCHITECTURE
              (Multi-Region Active-Active Batch Platform)
═══════════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────────────────────────────┐
│                        INGESTION PLANE                               │
│                                                                      │
│  Upstream Systems (APIs, DBs, Streams)                               │
│       │              │               │                               │
│       ▼              ▼               ▼                               │
│  Kinesis Data    DynamoDB        S3 Event                            │
│  Streams         Streams         Notifications                       │
│       │              │               │                               │
│       └──────────────┴───────────────┘                               │
│                       │                                              │
│                       ▼                                              │
│               SQS Standard Queue                                     │
│               (per-tenant, per-workload-type)                        │
│               visibilityTimeout=300s                                 │
│               DLQ: sqs-dlq-{workload}                               │
└───────────────────────┬──────────────────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────────────┐
│                    SCHEDULING & COORDINATION PLANE                   │
│                                                                      │
│  ┌──────────────────┐      ┌──────────────────────────────────────┐  │
│  │ EventBridge      │      │ DynamoDB Global Table                │  │
│  │ Scheduler        │─────▶│ scheduler_leases                     │  │
│  │ (per-region,     │      │ {jobId, leaseOwner, TTL, runId}      │  │
│  │  same cron)      │      │ → leader election                    │  │
│  └──────────────────┘      └──────────────────────────────────────┘  │
│          │                                                           │
│          ▼ (if lease acquired)                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │              STEP FUNCTIONS STANDARD (Orchestrator)          │    │
│  │                                                              │    │
│  │  Init ──▶ LoadManifest(S3) ──▶ DistributedMap               │    │
│  │                │                    │                        │    │
│  │           [Catch:ALL]          (fan-out workers)             │    │
│  │                │                    │                        │    │
│  │           SQS DLQ             ┌─────┴──────────────┐        │    │
│  │           + Alert             │                    │         │    │
│  │                          Lambda               AWS Batch      │    │
│  │                          (short, <15m)        Fargate Spot   │    │
│  │                          150-field            (long, 1-3h)   │    │
│  │                          projection           Java/Python     │    │
│  │                               │                    │         │    │
│  │                               └─────────┬──────────┘        │    │
│  │                                         │                    │    │
│  │                              Aggregate ──▶ Notify            │    │
│  └─────────────────────────────────────────────────────────────-┘    │
└───────────────────────────────────────────────────────────────────---┘
                        │
┌───────────────────────▼──────────────────────────────────────────────┐
│                      PROCESSING PLANE                                │
│                                                                      │
│  Lambda Workers (Python):           Fargate/Batch (Java):            │
│  ┌────────────────────────┐         ┌─────────────────────────────┐  │
│  │ - Field projection     │         │ - Spring Batch (chunked)    │  │
│  │   (150/2500 fields)    │         │ - S3 checkpoint every 1000  │  │
│  │ - Idempotency check    │         │ - SIGTERM graceful shutdown  │  │
│  │   (DynamoDB Powertools)│         │ - Fargate Spot (70% saving) │  │
│  │ - X-Ray active tracing │         │ - ECS capacity provider mix │  │
│  │ - Structured JSON logs │         │ - X-Ray SDK (Java)          │  │
│  └────────────────────────┘         └─────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────---┘
                        │
┌───────────────────────▼──────────────────────────────────────────────┐
│                      DATA PLANE                                      │
│                                                                      │
│  S3 (partitioned Parquet):          DynamoDB (multi-tenant):         │
│  input/year/month/day/tenant/        IdempotencyTable (TTL 48h)      │
│  processing/{jobId}/checkpoints/    AuditTable (TTL 7yr, GSI)        │
│  output/year/month/day/tenant/      SchedulerLeases (Global Table)   │
│                                     ScoreHistory (per entity)        │
│  ElastiCache (enrichment cache):    S3 Audit Bucket:                 │
│  - Redis Cluster Mode               - Object Lock COMPLIANCE mode    │
│  - Per-tenant keyspace prefix       - Immutable audit records        │
└───────────────────────────────────────────────────────────────────---┘
                        │
┌───────────────────────▼──────────────────────────────────────────────┐
│                  OBSERVABILITY + COMPLIANCE PLANE                    │
│                                                                      │
│  CloudWatch:              X-Ray:              CloudTrail:            │
│  - EMF custom metrics     - End-to-end        - API audit            │
│  - Alarms → SNS           - SFN → Lambda      - S3 data events       │
│  - Log Insights queries   - → Fargate         - DynamoDB data events │
│                           - → DynamoDB                               │
│  Dashboards:                                                         │
│  - items_processed/s      DLQ Monitor Lambda:                        │
│  - dlq_depth per tenant   - Alarm on DLQ depth > 0                   │
│  - p99 latency per job    - Pagerduty/Slack notification             │
│  - cost per run           - Replay trigger (manual approval)        │
└──────────────────────────────────────────────────────────────────────┘

MULTI-REGION REPLICATION:
  us-east-1 ◄──── DynamoDB Global Tables ────► eu-west-1
  us-east-1 ◄──── S3 Cross-Region Replication ► eu-west-1
  us-east-1 ◄──── Route53 health-based routing ► eu-west-1
  EventBridge cross-region bus for event fan-out [web:23]
```
### Final Architecture Key Properties
**Zero Data Duplication:**
- S3 is the single source of truth; no data copied between services.
- Lambda reads only 150 fields at projection layer — no full object materialization.
- DynamoDB stores references (S3 keys + metadata), not copies of records.
- Results written once to S3 output; downstream consumers read from S3, not passed through memory chains.

**End-to-End Traceability:**
- `correlationId = runId + tenantId + batchId` propagated in all Lambda contexts, SFN inputs, ECS task environments, and log lines.
- X-Ray traces span: EB Scheduler → SFN → Lambda/Fargate → DynamoDB/S3.
- CloudTrail records every `StartExecution`, `SubmitJob`, and S3 object write.
- DynamoDB audit table: every record-level operation logged with actor, timestamp, input snapshot, output.

**99.9% SLA Mechanics:**
- Active-active multi-region with DynamoDB lease failover (RTO ≈ 5 min = next EB Scheduler tick). [softwaremill](https://softwaremill.com/building-a-multi-regional-highly-available-scheduler-with-aws/)
- `ToleratedFailurePercentage: 2%` in Distributed Map prevents single bad records from aborting 300k runs.
- Fargate Spot + On-Demand fallback ensures compute availability even during Spot capacity events.
- SQS DLQ preserves unprocessed messages for up to 14 days — no data loss on failures.
- Idempotency guards ensure safe replay without duplication.

**Multi-Tenant Isolation:**
- Per-tenant SQS queues and schedule groups in EventBridge Scheduler.
- S3 key prefix = `tenant/{tenantId}/` with bucket policies scoped by prefix.
- DynamoDB partition key = `tenantId#entityId` — physical isolation by partition.
- IAM roles for Lambda/Fargate scoped to tenant-specific resources via tag-based conditions.
- Step Functions execution name = `{tenantId}-{jobType}-{date}-{runId}`.
The architecture above is purposefully modular — you can adopt it incrementally: start with EventBridge Scheduler + SQS + Lambda for micro-batches, add Step Functions Distributed Map for the daily 300k run, and graduate to AWS Batch + Fargate for KYC and ETL long-running workloads as your scale demands evolve. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)