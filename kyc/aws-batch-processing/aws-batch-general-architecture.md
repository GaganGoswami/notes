---
id: 3003-aws-batch-architecture-general
slug: aws-batch-architecture-general
title: "General Architecture Guide (Why-What-How) - aws-batch-java-kyc-implementation"
category: infrastructure
tags: [aws, batch-processing, scheduling, architecture-guide, why-what-how, eventbridge, step-functions, aws-batch, lambda, ecs]
description: "Comprehensive architectural guide covering scheduling, orchestration, execution, and event-driven batch processing services with why-what-how framework and comparison matrices"
difficulty: advanced
estimatedTime: 25
created_at: 2026-04-03T10:00:00Z
updated_at: 2026-04-03T10:00:00Z
---

# General - Why–What–How Architecture Guide - AWS Batch Processing and Schedulers: 
---
## The Core Mental Model
Before selecting any service, internalize this principle: **scheduling and batch processing are separate concerns that compose together**. A *scheduler* triggers work; a *batch processor* executes it. AWS gives you mix-and-match components for each layer. The best architecture pairs the right scheduler with the right compute/orchestration engine based on five axes: **volume, latency tolerance, runtime duration, statefulness, and operational maturity**.

***
## Part 1 — Service Landscape Overview
### Scheduling Layer (Triggers)
| Service | Mechanism | Min Granularity | Best For |
|---|---|---|---|
| **EventBridge Scheduler** | Cron / rate / one-time | 1 min | Any AWS target, high-volume schedules |
| **EventBridge Rules (CRON)** | Cron / rate | 1 min | Simple rule-based triggers, event routing |
| **Step Functions Wait State** | Per-workflow timer | Seconds | In-workflow delays, callback patterns |
| **CloudWatch Events (legacy)** | Rate / cron | 1 min | Older workloads (same backend as EB Rules) |
### Orchestration / Workflow Layer
| Service | Model | Max Duration | State | Best For |
|---|---|---|---|---|
| **Step Functions Standard** | State machine | 1 year | Full audit trail | Multi-step pipelines, human approvals |
| **Step Functions Express** | State machine | 5 min | No built-in history | High-frequency, event-driven micro-flows |
| **AWS MWAA (Airflow)** | DAG / Python | Unlimited | Rich metadata DB | Complex data pipelines, data engineer teams |
| **Glue Workflows** | DAG (Glue native) | Unlimited | Glue catalog | Spark/ETL, data lake transforms |
### Compute / Execution Layer
| Service | Runtime | Cold Start | Memory/CPU Control | Best For |
|---|---|---|---|---|
| **Lambda** | ≤15 min | ~100ms–1s | 128MB–10GB RAM | Short functions, event handlers |
| **ECS/Fargate** | Unlimited | ~30s | Full container spec | Medium jobs, no infra management |
| **AWS Batch** | Unlimited | ~1–3 min | Full EC2/Fargate | HPC, compute-heavy, large scale |
| **EC2 Spot** | Unlimited | Minutes | Fully customizable | Cost-sensitive, fault-tolerant HPC |
| **Glue Jobs** | Unlimited | ~1 min | DPU-based Spark | Large-scale Spark ETL |
### Event-Driven / Streaming Batch
| Service | Trigger | Batch Window | Best For |
|---|---|---|---|
| **SQS + Lambda/ECS** | Queue depth | Configurable | Async fan-out processing |
| **Kinesis + Lambda** | Shard events | Up to 300s window | Real-time micro-batching |
| **DynamoDB Streams** | Table changes | Near real-time | CDC, downstream pipeline triggers |
| **S3 Event Notifications** | Object creation | Immediate | File-drop batch pipelines |

***
## Part 2 — Deep Dive: Why–What–How Per Service
### EventBridge Scheduler
**Why:** Amazon built EventBridge Scheduler to replace the fragmented experience of EventBridge Rules + CloudWatch Events for time-based triggers. Rules were designed for *event routing*, not scheduling; Scheduler is purpose-built for it, supporting millions of schedules, timezone awareness, and flexible invocation targets natively. [cloudthat](https://www.cloudthat.com/resources/blog/comparing-aws-step-functions-and-amazon-eventbridge-scheduler)

**What:** A fully managed, serverless scheduling service. It invokes over 270 AWS service targets directly (Lambda, Step Functions, SQS, ECS, Batch, SNS, and more) via templated API calls. Each schedule is an independent resource, not coupled to a rule bus. Supports one-time, recurring (rate/cron), and timezone-aware schedules with optional flexible time windows (±N minutes of the target time). [aws.amazon](https://aws.amazon.com/solutions/guidance/scheduling-batch-jobs-for-aws-mainframe-modernization/)

**How:**
```
[EventBridge Scheduler]
     |
     |-- Cron: "0 2 * * ? *" (daily 2AM UTC)
     |
     +--> [Target: Step Functions StartExecution]
     |         |
     |         +--> [State Machine: ETL Pipeline]
     |
     +--> [Target: Lambda FunctionArn]
     |
     +--> [Target: SQS SendMessage]
     |
     +--> [Target: AWS Batch SubmitJob]
```

- **Retries:** Built-in retry policy with configurable max attempts and DLQ (SQS or SNS)
- **Failure handling:** Failed invocations go to a DLQ you configure on the schedule itself
- **Monitoring:** CloudWatch metrics per schedule group; EventBridge events on invocation failures
- **Scale:** Supports millions of concurrent schedules per account; no throughput ceiling for schedule creation

**When to choose:** You need time-based triggers at scale, across many independent jobs, with timezone control and no infra management. Starting new greenfield scheduler? Start here.

**When to avoid:** You need complex multi-step workflow logic *within* the scheduled work (use Step Functions), or sub-minute granularity (use Kinesis/SQS).

***
### EventBridge Rules (CRON)
**Why:** EventBridge Rules predate Scheduler and were built for event *routing* — filtering and forwarding events between producers and consumers. CRON-based rules are a secondary use-case capability.

**What:** A rule with a `schedule` source expression triggers on a bus. Unlike Scheduler, rules share the same 300 events/second default bus throughput, are not timezone-aware, and cannot deliver to all 270+ targets natively. But they *also* route non-schedule events, making them ideal for hybrid "event OR schedule" patterns.

**How:**
```
[EventBridge Rule]
  Schedule: cron(0 6 * * ? *)  OR  source: "custom.app"
     |
     +--> Target: Lambda
     +--> Target: ECS Task (via EventBridge Input Transformer)
     +--> Target: Step Functions
```

**When to choose:** You already use EventBridge for event routing and want one rule to fire on *both* a schedule and a business event. Or you need simple rate-based triggers for a handful of jobs.

**When to avoid:** Managing 100+ schedules (use Scheduler), needing timezone-aware cron, or wanting per-job DLQs.

***
### Step Functions (Standard + Express)
**Why:** Step Functions exist to eliminate "glue code" — the error handling, retry logic, state passing, and branching logic that clutters application code in distributed systems. Every workflow concern (retry, catch, wait, parallel, fan-out) becomes a declarative JSON/YAML config rather than application code. [cloudthat](https://www.cloudthat.com/resources/blog/comparing-aws-step-functions-and-amazon-eventbridge-scheduler)

**What:** A visual, serverless state machine service. **Standard Workflows** (≤1 year execution, $0.025/1000 transitions, full execution history in CloudWatch) are for long-running, auditable pipelines. **Express Workflows** (≤5 min, ~$1/million executions, async or sync) are for high-throughput event processing. [cloudthat](https://www.cloudthat.com/resources/blog/comparing-aws-step-functions-and-amazon-eventbridge-scheduler)

**The Distributed Map State** is Step Functions' superpower for batch: it reads directly from S3 (CSV, JSON, S3 inventory), partitions into batches, and spawns up to 10,000 parallel child Express executions. Each child has its own execution history. This is the canonical pattern for fan-out at scale. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html)

**How — Distributed Map for 300K records:**
```
[EventBridge Scheduler]
        |
        v
[Step Functions - Standard Workflow]
        |
        v
[Distributed Map State]
  ItemReader: S3 CSV (300K rows)
  ItemBatcher: MaxItemsPerBatch=100
  MaxConcurrency: 1000
  ExecutionType: EXPRESS
        |
        v (3000 child Express executions)
  [Lambda: ProcessBatch(100 records)]
        |  Retry: 3x, backoff 2s, JitterFull
        |  Catch: -> MarkFailed State
        v
[ResultWriter: S3 JSONL output]
        |
        v
[SNS Notification: Pipeline Complete]
```

- **Retries:** Per-state retry config with `BackoffRate`, `MaxAttempts`, `JitterStrategy`
- **Error handling:** `Catch` block routes failures to compensating states
- **Idempotency:** Pass a `correlationId` through `ItemSelector` to every child; workers check idempotency keys in DynamoDB before processing
- **Monitoring:** X-Ray tracing on state machine + Lambda; CloudWatch Execution Metrics dashboard
- **Cost for 300K records in 100-item batches:** 3,000 Express executions × ~10 transitions = ~30K transitions → ~$0.03 for SF overhead alone [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2)

**When to choose:** Multi-step orchestration, human approval gates, complex error/retry topologies, fan-out/fan-in at scale.

**When to avoid:** Simple single-step fire-and-forget triggers (use EventBridge Scheduler directly), or workflows needing Python DAG expressiveness with data scientist teams (use MWAA).

***
### AWS Batch
**Why:** AWS Batch exists for **compute-intensive, container-based HPC workloads** where you need full control over vCPU, memory, GPU, and runtime — but don't want to manage a fleet of EC2 instances yourself. It handles job queuing, compute environment provisioning, instance selection, and bin-packing automatically. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

**What:** Four compute environment types: EC2 On-Demand, EC2 Spot, Fargate, Fargate Spot. Jobs are Docker containers submitted to Job Queues mapped to Compute Environments. Multiple queues with priority ordering are supported. Array Jobs allow a single `SubmitJob` call to fan out N parallel instances of the same container. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

**How — Architecture for nightly genomics / ML training job:**
```
[EventBridge Scheduler: 11PM]
        |
        v
[Step Functions: Orchestrator]
        |
        v
[AWS Batch: SubmitJob]
  Job Queue → Compute Environment (EC2 Spot, m5.4xlarge, BEST_FIT_PROGRESSIVE)
  Array Size: 500 (one per input file)
  vCPUs: 16 per job, Memory: 64GB
  Retries: 3 (on SPOT_INSTANCE_RECLAIMED)
        |
        v
  [ECR: Container Image]
  [S3: Input/Output Buckets]
  [EFS: Shared scratch space]
        |
        v
[CloudWatch Logs: /aws/batch/job]
[Step Functions: Wait for Batch (`.sync` integration)]
        |
        v
[SNS: Alert on completion/failure]
```

- **Spot handling:** Use `SPOT_CAPACITY_OPTIMIZED` allocation strategy; set `Retries=3` with `Action=RETRY` on `SPOT_INSTANCE_RECLAIMED` [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)
- **Multi-instance jobs:** Use `multiNodeParallelJobs` for MPI-style distributed compute
- **Cost:** EC2 Spot saves up to 90% vs On-Demand; Fargate Spot saves ~70% [mkdev](https://mkdev.me/posts/processing-background-jobs-on-aws-lambda-vs-ecs-vs-ecs-fargate)

**When to choose:** Long-running (minutes to hours) compute jobs, HPC, ML training, video transcoding, genomics, simulations. Workloads where Lambda's 15-minute limit is insufficient.

**When to avoid:** Short jobs (<30 sec) where container startup overhead dominates; simple orchestration (use Step Functions + Lambda).

***
### Lambda + SQS + DLQ
**Why:** This triad is the foundational serverless batch pattern on AWS. SQS provides durable decoupling and automatic backpressure; Lambda's event source mapping handles polling, batching, and concurrency scaling; the DLQ captures poison-pill messages. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)

**What:**
```
[Producer] --> [SQS Standard Queue]
                    |
                    | EventSourceMapping
                    | BatchSize: 10
                    | MaxBatchingWindow: 60s
                    | FunctionResponseTypes: ReportBatchItemFailures
                    v
              [Lambda Function]
              (processes up to 10 msgs/invocation)
                    |
                    +--> Success: messages deleted
                    +--> Partial failure: failed item IDs returned
                    |    SQS returns only failed items to queue
                    v
              [SQS DLQ] (after maxReceiveCount=3)
                    |
                    v
              [Lambda: DLQ Reprocessor / Alerting]
```

Key design decisions: [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)
- Set `VisibilityTimeout = 2.5× average processing time`
- Enable `ReportBatchItemFailures` to avoid reprocessing successful items in a failed batch
- Use `MaxBatchingWindow` (up to 300s) to accumulate cost-efficient larger batches
- For **ordered** processing: use SQS FIFO + MessageGroupId; note FIFO limits Lambda concurrency to one concurrent execution per group

**FIFO for ordered batches:**
```
SQS FIFO Queue
  MessageGroupId: "customer-123"  ← serialized per customer
  MessageGroupId: "customer-456"  ← parallel across customers
```

**When to choose:** Async, event-driven processing of variable-volume work. Excellent for fan-out from S3 uploads, API events, notifications.

**When to avoid:** Sub-second latency requirements; jobs requiring >15 min execution (use ECS/Fargate); strict global ordering across all messages (SQS FIFO limits throughput).

***
### ECS / Fargate Scheduled Tasks
**Why:** When Lambda's 15-minute cap is insufficient, but you don't need AWS Batch's job queue/priority/array-job features. ECS Scheduled Tasks give you container-native runtime with no infrastructure management at all (Fargate), triggered on a schedule via EventBridge. [builder.aws](https://builder.aws.com/content/34jTIHSM6RwweiUIMU4GJcgSVRs/how-to-schedule-ecs-tasks-choosing-between-eventbridge-scheduler-eventbridge-rules-and-step-functions)

**What:** An EventBridge rule (or Scheduler) targets `ECS RunTask`. The task uses the latest container image from ECR, runs until completion, and logs to CloudWatch Logs. Fargate Spot can reduce costs by 70% for fault-tolerant jobs. [mkdev](https://mkdev.me/posts/processing-background-jobs-on-aws-lambda-vs-ecs-vs-ecs-fargate)

**How:**
```
[EventBridge Scheduler: cron]
        |
        v
[ECS RunTask API]
  Cluster: batch-cluster
  Task Definition: nightly-etl:v12
  Launch Type: FARGATE_SPOT
  Overrides: { BATCH_DATE: "2026-03-29" }
  Network: VPC private subnet, security group
        |
        v
[Container: Python ETL]
  - Reads from S3
  - Writes to Aurora / Redshift
  - Runtime: up to 4 hours
        |
        v
[CloudWatch Logs: /ecs/nightly-etl]
[Container Insights: CPU/mem metrics]
```

**Cost vs Lambda:** For a 300ms job running thousands of times/day, Lambda is ~100× cheaper than an always-running ECS task. But for a 2-hour nightly job run once, Fargate is cheaper (Lambda can't run 2 hours, and ECS has 1s billing granularity with 1-min minimum). [mkdev](https://mkdev.me/posts/processing-background-jobs-on-aws-lambda-vs-ecs-vs-ecs-fargate)

**When to choose:** Jobs 15 min–many hours, needing specific OS/runtime, moderate frequency, no HPC-scale array jobs.

**When to avoid:** Very short jobs (Lambda is cheaper), massive parallel array jobs (use AWS Batch).

***
### AWS Glue Workflows & Glue Jobs
**Why:** Glue is purpose-built for **Apache Spark-based ETL on the AWS data lake stack**. Glue Workflows orchestrate Crawlers + Jobs into DAGs. DPU-based (Data Processing Unit) pricing means you pay only for Spark cluster time. [docs.aws.amazon](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/batch-data-processing.html)

**What:** Glue Jobs = managed Spark (or Python shell / Ray). Glue Workflows = DAGs connecting crawlers, jobs, and triggers. Integrates natively with AWS Glue Data Catalog, S3, Redshift, RDS, Lake Formation.

**How — Nightly data lake ETL:**
```
[EventBridge Scheduler 1AM]
        |
        v
[Glue Workflow: nightly-datalake-etl]
        |
        +--> [Glue Crawler: raw-zone] --> updates catalog
        +--> [Glue Job: raw→clean (Spark, 10 DPUs, G.1X)]
              reads: s3://raw/
              writes: s3://clean/ (Parquet, partitioned by date)
        +--> [Glue Job: clean→curated (Spark)]
        +--> [Glue Crawler: curated-zone]
        v
[Glue Data Catalog: updated table metadata]
[CloudWatch: Glue job metrics, byte counts]
```

**DPU cost model:** 1 DPU = $0.44/hour (standard), $0.22/hour (Flex execution). A 10-DPU job running 30 minutes = $2.20.

**When to choose:** Spark ETL, PySpark engineers on the team, AWS-native data lake (S3 + Glue Catalog + Athena), schema evolution.

**When to avoid:** Non-Spark workloads, sub-minute latency, teams unfamiliar with Spark (use Step Functions + Lambda).

***
### AWS MWAA (Managed Airflow)
**Why:** MWAA gives data engineering teams the full Apache Airflow experience — Python DAGs, 800+ providers/operators, rich UI — without managing Airflow infrastructure. The tradeoff is a minimum ~$400/month baseline cost for the smallest environment. [dzone](https://dzone.com/articles/a-comprehensive-comparison-of-aws-step-function-an)

**What:** Managed Airflow scheduler + workers + web server on AWS infrastructure. DAGs are Python files in S3. Workers scale via Fargate auto-scaling. Integrates with all AWS services through Airflow providers.

**How:**
```
[S3: dags/nightly_etl.py] --> synced to MWAA
[MWAA Scheduler] -- triggers DAG at cron
        |
        v
[DAG: nightly_etl]
  Task 1: PythonOperator → validate inputs
  Task 2: GlueJobOperator → run Glue ETL
  Task 3: AWSBatchOperator → run Batch job
  Task 4: RedshiftSQLOperator → update summary tables
  Task 5: SnsOperator → notify downstream
        |
        v
[Airflow UI: DAG graph, task logs, SLAs, retries]
[CloudWatch: Airflow scheduler/worker logs]
```

**MWAA vs Step Functions:** [reddit](https://www.reddit.com/r/dataengineering/comments/1rkjxzx/is_there_any_benefit_of_using_airflow_over_aws/)
- MWAA wins: Python DAGs, data engineer UX, cross-cloud portability, 800+ operators, complex dependency management
- Step Functions wins: No server cost, tighter AWS integration, per-execution billing, better for developers not data engineers

**When to choose:** Data engineering teams already using Airflow, complex DAG dependencies, multi-cloud portability needs, 10+ interconnected pipelines.

**When to avoid:** Simple pipelines (overkill), cost-sensitive small workloads, teams unfamiliar with Airflow.

***
### EC2 Spot-Based Batch Processing
**Why:** For massively parallel, cost-sensitive HPC workloads where interruption tolerance is built in and you want the maximum price/performance. Spot saves up to 90% vs On-Demand.

**What:** EC2 Spot instances provisioned via Auto Scaling Groups or AWS Batch Spot Compute Environments. Use Spot interruption notice (2-min warning via EventBridge) to checkpoint work.

**How — Spot-resilient pattern:**
```
[AWS Batch Spot CE]
  AllocationStrategy: SPOT_CAPACITY_OPTIMIZED
  InstanceTypes: [m5.xlarge, m5.2xlarge, m4.xlarge, c5.2xlarge]
        |
  Job receives SIGTERM (2 min warning)
        |
  Checkpoint handler: writes progress to S3
  Job marked FAILED → Batch retries on new Spot/On-Demand
        |
  Result: at-least-once processing guaranteed
```

**When to choose:** Fault-tolerant compute tasks >30 min, ML training, rendering, genomics, where cost is the #1 constraint.

**When to avoid:** State-dependent jobs with no checkpointing, latency-sensitive workloads.

***
### DynamoDB Streams / Kinesis / S3 Event-Driven Batches
**Why:** Not all batching is time-triggered. Data-driven triggers fire when records accumulate, offering lower latency than scheduled polling.

**DynamoDB Streams + Lambda:**
```
[DynamoDB Table: orders]
  New/Updated item → Stream record
        |
  Lambda ESM: BatchSize=100, BisectOnFunctionError=true
        |
  Lambda: aggregate → S3 micro-batch file → trigger downstream
```

**Kinesis Data Streams + Lambda (micro-batching):**
```
[Producers] → [Kinesis Stream: 10 shards]
  Lambda ESM:
    BatchSize: 1000
    BisectOnFunctionError: true
    TumblingWindowInSeconds: 300 (5-min window)
    StartingPosition: TRIM_HORIZON
        |
  Lambda: aggregate 5-min window → write to S3 → trigger Glue
```

**S3 Event → Batch:**
```
[S3: s3://raw/feeds/2026/03/29/file.csv]  (PutObject)
        |
  S3 Event Notification → SQS
        |
  Lambda ESM OR ECS task → process file
```

**When to choose:** Event-volume-triggered batch (not time-triggered), CDC pipelines, real-time aggregation.

***
### AWS Data Pipeline (Legacy)
Data Pipeline predates all modern alternatives and uses EC2/EMR for execution. **Do not use for new workloads.** Replace with Step Functions + AWS Batch/Glue. AWS has effectively soft-deprecated it in favor of the above services. [docs.aws.amazon](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/batch-data-processing.html)

***
## Part 3 — Architecture Examples
### Example A: Daily 300K-Record Processing Job
**Constraints:** 300K records in S3 CSV, process daily at 2AM, each record needs ~100ms work, total target: <30 min, cost: minimize, retry on failure.

**Architecture: EventBridge Scheduler → Step Functions Distributed Map → Lambda**

```
02:00 UTC
[EventBridge Scheduler]
        |
        v
[Step Functions Standard Workflow: daily-processor]
        |
        v
[Task: Validate Input] → Lambda (check S3 file exists)
        |
        v
[Distributed Map State]
  ItemReader: S3 GetObject (300K row CSV)
  ItemBatcher: MaxItemsPerBatch=100  ← 3,000 batches
  MaxConcurrency: 500
  ToleratedFailurePercentage: 1
  ExecutionType: EXPRESS
        |
        v (500 concurrent Express executions)
[Lambda: record-processor]
  - Idempotency: DynamoDB conditional put on recordId
  - Retry: 3x, 2s backoff, full jitter
  - Catch → write failed batch IDs to S3/errors/
        |
        v
[ResultWriter: s3://results/2026-03-29.jsonl]
        |
        v
[Task: Notify] → SNS → Email/PagerDuty
        |
        v (on failure)
[Task: Alert DLQ Processor] → SQS DLQ → Lambda reprocessor

Observability:
- X-Ray: end-to-end trace
- CloudWatch Dashboard: records/sec, failure rate, duration
- CloudWatch Alarm: >1% failure rate → SNS alert
```

**Estimated cost:** 3,000 Express executions × ~$0.000001 = $0.003 SF cost; 3,000 Lambda × 100ms × 512MB = ~$0.05. Total: **<$0.10/run**. [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2)

***
### Example B: Nightly ETL Pipeline
**Constraints:** Extract from RDS, transform with Spark, load to Redshift, multi-step with dependencies, audit trail required, ~2-hour window.

**Architecture: EventBridge Scheduler → Step Functions Standard → Glue + AWS Batch**

```
[EventBridge Scheduler: 11PM]
        |
        v
[Step Functions Standard: nightly-etl]
  State 1: CheckPreviousRunComplete (DynamoDB state lock)
  State 2: ExtractToS3 → Lambda (RDS → S3 Parquet via JDBC)
  State 3: Parallel {
    Branch A: GlueJob (sales transform, 10 DPU, G.1X)
    Branch B: GlueJob (inventory transform, 5 DPU)
  }
  State 4: BatchJob (data quality checks, Fargate)
  State 5: RedshiftDataAPI (COPY command, Step Functions SDK integration)
  State 6: UpdateDataCatalog → Lambda
  State 7: NotifySuccess → SNS
  
  Catch ALL → NotifyFailure → PagerDuty + store failure context in DDB

Audit:
- CloudTrail: all API calls
- Step Functions execution history: full audit log per run
- Glue job metrics: DPU hours, record counts
```

***
### Example C: Real-Time Micro-Batching (Every 5 Minutes)
**Architecture: Kinesis → Lambda (Tumbling Window) → S3 → Glue**

```
[API Gateway / App] → events → [Kinesis: 5 shards]
        |
  Lambda ESM:
    BatchSize: 500
    TumblingWindowInSeconds: 300
    BisectOnFunctionError: true
        |
  Lambda: aggregate window
    - sum metrics, deduplicate
    - write micro-batch to s3://micro/2026-03-29T14:05:00.parquet
        |
  S3 Event → SQS → Lambda → trigger Glue Job (incremental)
        |
  Glue: MERGE into curated layer (Iceberg/Hudi table)
        |
  Athena / QuickSight: query latest 5-min window

DLQ: Kinesis → Lambda failure → SQS DLQ → alert + replay
```

***
### Example D: Multi-Region Active/Active Batch Workflows
**Pattern:** Primary in us-east-1, secondary in us-west-2. Each region processes a geographic partition of data independently. Cross-region coordination via DynamoDB Global Tables.

```
us-east-1                           us-west-2
[EB Scheduler] ──────────────── [EB Scheduler] (same cron)
      |                                  |
[Step Functions]                 [Step Functions]
  Partition: US-East               Partition: US-West
      |                                  |
[AWS Batch]                       [AWS Batch]
      |                                  |
[DynamoDB Global Table] ←──────→ [DynamoDB Global Table]
  job_id, status, region           (replication <1s)
      |
  Leader election:
  Conditional write: SET status=RUNNING WHERE status=PENDING
  Only one region "wins" each job group
      |
[S3 Cross-Region Replication]
  Results from both regions → s3://global-results/
      |
[Step Functions: Aggregator] (runs after both complete via DDB polling)
```

**Failover:** If us-east-1 EventBridge Scheduler fails to fire (regional outage), us-west-2 schedule detects the job hasn't been claimed in DDB within 15 min and claims it.

***
### Example E: Long-Running Compute-Intensive Jobs
**Architecture: EventBridge Scheduler → AWS Batch (Spot) → S3 Checkpointing**

```
[EventBridge Scheduler]
        |
[Step Functions: submit + monitor]
        |
[AWS Batch: SubmitJob]
  Job: ml-training
  ComputeEnv: SpotOptimized (p3.8xlarge, p2.8xlarge)
  Memory: 64GB, vCPUs: 32, GPU: 4
  Retries: 3 (SPOT_INSTANCE_RECLAIMED → retry)
  Timeout: 12 hours
        |
  Container: pytorch-training:latest (ECR)
  entrypoint: train.py --checkpoint-bucket s3://ml-ckpt/
        |
  SIGTERM handler:
    - flush optimizer state to S3
    - write epoch/step to s3://ml-ckpt/latest.json
    - graceful exit
        |
  On retry: resume from latest checkpoint
        |
[Step Functions: .sync integration waits for Batch job]
        |
[Lambda: post-training eval + model registration (SageMaker Model Registry)]
```

***
### Example F: Serverless vs Container Batch — Side by Side
| Dimension | Serverless (Lambda-centric) | Container (ECS/Batch-centric) |
|---|---|---|
| **Max runtime** | 15 minutes | Hours to days |
| **Cold start** | 100ms–1s | 30s–3 min |
| **Memory** | Up to 10GB | Practically unlimited |
| **GPU support** | No | Yes (AWS Batch) |
| **Cost model** | Per-millisecond | Per-second (min 1 min for Fargate) |
| **Cost for <1s jobs** | Cheapest (Lambda ~100× cheaper)  [mkdev](https://mkdev.me/posts/processing-background-jobs-on-aws-lambda-vs-ecs-vs-ecs-fargate) | Expensive (min charge dominates) |
| **Cost for 2h jobs** | N/A (can't run) | Efficient, especially Spot |
| **Operational overhead** | Minimal | Low (Fargate) to medium (EC2) |
| **Dependency packaging** | Lambda layers, container image | Full Docker image |
| **Network** | VPC optional | VPC required for data |
| **Best for** | Event handlers, fan-out workers, <15 min ETL | HPC, ML, video, long-running ETL |

***
## Part 4 — AWS Well-Architected Patterns
### Fan-Out / Fan-In
```
[Orchestrator: Step Functions]
        |
   [Parallel State or Distributed Map]
   /         |          \          \
[Worker A] [Worker B] [Worker C] [Worker D]   ← fan-out
   \         |          /          /
        [Aggregator State]                     ← fan-in
   (collect ResultWriter output from S3)
        |
   [Final Report Lambda]
```
### Idempotent Batch Design
Every batch worker must be safe to retry. The pattern:

```python
# Worker Lambda: idempotency via DynamoDB conditional write
def handler(event, context):
    for record in event['records']:
        record_id = record['id']
        # Conditional put: only process if not already processed
        try:
            dynamodb.put_item(
                TableName='idempotency-keys',
                Item={'pk': {'S': record_id}, 'ttl': {'N': str(time()+86400)}},
                ConditionExpression='attribute_not_exists(pk)'
            )
        except ConditionalCheckFailedException:
            continue  # already processed, skip
        process(record)
```

Use DynamoDB TTL to expire idempotency keys after your deduplication window (e.g., 24 hours).
### At-Least-Once vs Exactly-Once
| Processing Guarantee | How to Achieve It | Trade-off |
|---|---|---|
| **At-least-once** | SQS + Lambda (default), Kinesis, Batch retries | Possible duplicates; require idempotent workers |
| **Exactly-once** | SQS FIFO + deduplication ID + idempotent workers | Lower throughput, more complex |
| **At-most-once** | SQS with `deleteMessage` before processing | Possible data loss; rarely desirable |

AWS has no native exactly-once processing. Achieve it via idempotency keys + at-least-once delivery.
### Error Handling, Retries, Circuit Breakers
**Step Functions retry pattern:**
```json
"Retry": [{
  "ErrorEquals": ["Lambda.ServiceException", "States.TaskFailed"],
  "IntervalSeconds": 2,
  "BackoffRate": 2.0,
  "MaxAttempts": 3,
  "JitterStrategy": "FULL"
}],
"Catch": [{
  "ErrorEquals": ["States.ALL"],
  "ResultPath": "$.error",
  "Next": "NotifyAndFail"
}]
```

**Circuit breaker with DynamoDB:**
```
Each batch run: check DDB counter
  IF failure_count > threshold (e.g., 5 in 10 min):
    Skip execution, alert SNS → "CIRCUIT OPEN"
    Reset automatically after cool-down window
```
### Observability Stack
```
Execution Tier              Observability Stack
─────────────────           ──────────────────────────────
EventBridge Scheduler  →    CloudWatch Metrics: InvocationCount, FailedInvocationCount
Step Functions         →    X-Ray traces, Execution History, CW Logs
Lambda                 →    X-Ray traces, CW Logs, Lambda Insights (EMF metrics)
AWS Batch              →    CW Logs /aws/batch/job, CW Container Insights
ECS/Fargate            →    Container Insights, CW Logs, X-Ray daemon sidecar
Glue                   →    CW Metrics: glue.driver.*, CW Logs, Glue job bookmarks
SQS                    →    ApproximateNumberOfMessagesVisible (scale metric)
                            NumberOfMessagesSentToDLQ (failure alert metric)

Alerting:
  CW Alarm → SNS → PagerDuty / Slack / OpsGenie
  CW Dashboard: per-pipeline operational view
  AWS Health / Personal Health Dashboard: service disruptions
  CloudTrail: API-level audit for all job submissions/mutations
```
### Regional Failover for Scheduling
Primary pattern: **EventBridge Scheduler in 2 regions + DynamoDB Global Tables for leader election**. Secondary pattern: Route 53 health checks + custom health endpoint that disables/enables EB Scheduler groups via API (using CloudWatch Alarms triggering Lambda to pause the secondary region's schedule group when primary is healthy).

***
## Part 5 — Complete Recommendations Matrix
| Workload Type | Recommended Scheduler | Recommended Executor | Orchestrator | Notes |
|---|---|---|---|---|
| Simple periodic trigger | EventBridge Scheduler | Lambda | None | Lowest overhead |
| Multi-step pipeline (<15 min) | EventBridge Scheduler | Lambda | Step Functions Standard | Visual audit trail |
| Large-scale fan-out (100K+ records) | EventBridge Scheduler | Lambda (Express) | Step Functions Distributed Map | Up to 10K concurrent  [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html) |
| Long-running ETL (hours) | EventBridge Scheduler | ECS Fargate / AWS Batch | Step Functions Standard | Use `.sync` integration |
| HPC / ML / GPU jobs | EventBridge Scheduler | AWS Batch (Spot) | Step Functions Standard | Checkpoint for Spot |
| Complex Spark ETL | EventBridge Scheduler | Glue Jobs | Glue Workflows / MWAA | DPU-based cost |
| Multi-team data pipelines | MWAA (Airflow) | Mixed | Airflow DAGs | Python DAGs, rich UI |
| Event-volume triggered | SQS / Kinesis | Lambda / ECS | Step Functions Express | No fixed schedule |
| Micro-batching (≤5 min) | Kinesis TumblingWindow | Lambda | None / Express | Sub-minute aggregation |
| Multi-region active/active | EB Scheduler (per region) | Batch / Lambda | Step Functions + DDB | Leader election pattern |
| Ordered processing | EventBridge Scheduler | Lambda | Step Functions | SQS FIFO per group |
| CDC / stream-triggered | DDB Streams / Kinesis | Lambda | Express Workflows | Not time-based |

***
## Part 6 — Decision Tree
```
START: Define your batch workload
         |
         v
Is processing triggered by TIME or DATA ARRIVAL?
  |                              |
TIME                           DATA
  |                              |
  v                              v
Use EventBridge Scheduler     S3 upload? → S3 Event → SQS → Lambda/ECS
  |                           Queue depth? → SQS + Lambda ESM
  v                           Stream? → Kinesis/DDB Streams + Lambda
Is workflow multi-step?
  |           |
  YES         NO
  |           |
  v           v
Step Functions  EB Scheduler → Lambda / ECS / Batch directly
  |
  v
Jobs run >15 min?
  |           |
  YES         NO
  |           |
  v           v
Is it HPC/ML?   Use Lambda workers
  |       |
  YES     NO
  |       |
  v       v
AWS Batch  ECS Fargate
(Spot CE)
  |
  v
Do you need Spark/large-scale data transforms?
  |           |
  YES         NO → Step Functions + Batch/Lambda
  |
  v
Do you have data engineering teams using Python DAGs?
  |           |
  YES         NO
  |           |
  v           v
MWAA       Glue Workflows + Glue Jobs
```

***
## Part 7 — Best Practice Architecture (Recommended Default)
For **most enterprise batch workloads** (moderate volume, multi-step, auditable, cost-efficient), the AWS-recommended gold standard architecture is:

**EventBridge Scheduler → Step Functions Standard → Distributed Map (Express) → Lambda/Batch → S3 + DynamoDB**
This combination provides: [aws.amazon](https://aws.amazon.com/solutions/guidance/scheduling-batch-jobs-for-aws-mainframe-modernization/)
- **Scheduling:** EventBridge Scheduler for flexible, managed time triggers with DLQ
- **Orchestration:** Step Functions Standard for audit trail, error handling, and workflow visibility
- **Fan-out:** Distributed Map with Express child executions for high-concurrency record processing
- **Compute:** Lambda for sub-15-min work; AWS Batch for longer or HPC jobs
- **Idempotency:** DynamoDB conditional writes per record
- **Results:** S3 ResultWriter for output consolidation
- **Observability:** X-Ray end-to-end, CloudWatch Logs + Alarms + Dashboard, CloudTrail for audit

The architecture is **fully serverless** (zero servers to manage), **pay-per-use** (no idle cost), **infinitely scalable** (10,000 parallel Distributed Map executions), and **natively resilient** (built-in retries, DLQ, circuit-breaker-ready).

***
## Provide Your Constraints for a Tailored Recommendation
To give you a precise "Best Practice Architecture" tuned to your workload, what are your specific constraints across these dimensions: **processing volume** (records/day), **runtime per job** (seconds/minutes/hours), **frequency** (per minute/hour/day), **regional setup** (single or multi-region), and **team profile** (app developers vs. data engineers)?