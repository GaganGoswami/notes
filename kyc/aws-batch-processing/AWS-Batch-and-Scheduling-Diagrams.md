---
id: 3002-aws-batch-diagrams-kyc
slug: aws-batch-diagrams-kyc
title: "Diagrams (KYC Implementation)- AWS Batch and Scheduling"
category: infrastructure
tags: [aws, batch-processing, scheduling, kyc, eventbridge, step-functions, distributed-map, aws-themed-diagrams]
description: "KYC-specific implementation diagrams with AWS-themed mermaid styling for EventBridge Scheduler, Rules, Step Functions, and Distributed Map patterns"
difficulty: intermediate
estimatedTime: 12
created_at: 2026-04-03T10:00:00Z
updated_at: 2026-04-03T10:00:00Z
---

# KYC - Diagrams AWS Batch and Scheduling 

## Mermaid theme

Use this front matter or paste the init block above each diagram so the diagrams render with an AWS-like color palette based on AWS orange and dark slate tones. [aws.amazon](https://aws.amazon.com/architecture/icons/)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF",
    "mainBkg": "#FFF4E5",
    "secondBkg": "#EAF3FB",
    "tertiaryBkg": "#F7F7F7",
    "clusterBkg": "#F8FBFF",
    "clusterBorder": "#146EB4",
    "edgeLabelBackground": "#FFFFFF",
    "fontFamily": "Arial"
  },
  "flowchart": {
    "curve": "basis",
    "htmlLabels": true
  }
}}%%
flowchart LR
  A[AWS theme initialized] --> B[Use same init block for all diagrams]
```

***

## 1) EventBridge Scheduler

EventBridge Scheduler is the best fit for centralized recurring schedules and one-time invocations with retries and DLQ support. It is a trigger service, not a workflow engine, so the diagram ends at the target service. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/using-eventbridge-scheduler.html)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF",
    "mainBkg": "#FFF4E5",
    "secondBkg": "#EAF3FB",
    "clusterBkg": "#F8FBFF",
    "clusterBorder": "#146EB4",
    "edgeLabelBackground": "#FFFFFF"
  }
}}%%
flowchart LR
  subgraph SCHED[EventBridge Scheduler]
    CRON[Cron / Rate / One-time Schedule]
    GROUP[Schedule Group<br/>per tenant or domain]
    RETRY[Retry Policy<br/>Max age / attempts]
    DLQ[DLQ<br/>SQS]
    CRON --> GROUP --> RETRY
    RETRY --> DLQ
  end

  GROUP --> L[Lambda Target]
  GROUP --> SFN[Step Functions StartExecution]
  GROUP --> ECS[ECS RunTask]
  GROUP --> SQS[SQS SendMessage]
  GROUP --> BATCH[AWS Batch SubmitJob]

  L --> CW[CloudWatch Logs/Metrics]
  SFN --> CW
  ECS --> CW
  BATCH --> CW
```

***

## 2) EventBridge Rules (CRON)

EventBridge Rules combine scheduling with event pattern routing and can also route events across Regions through event buses. This makes them useful when cron and event-routing logic belong in the same control plane. [aws.amazon](https://aws.amazon.com/blogs/compute/introducing-cross-region-event-routing-with-amazon-eventbridge/)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  CRON[EventBridge Rule<br/>cron expression] --> BUS[Default / Custom Event Bus]
  EVENTS[Application / AWS Events] --> BUS

  BUS --> RULE1[Pattern Rule]
  BUS --> RULE2[Schedule Rule]
  RULE1 --> TARGET1[Lambda / Step Functions / ECS]
  RULE2 --> TARGET2[SQS / SNS / API Destination]

  BUS --> XREGION[Cross-Region Event Bus Target]
  XREGION --> RBUS[Remote Region Event Bus]
  RBUS --> RPROC[Regional Consumer]

  TARGET1 --> OBS[CloudWatch / CloudTrail]
  TARGET2 --> OBS
  RPROC --> OBS
```

***

## 3) Step Functions Standard

Step Functions Standard is intended for durable, auditable workflows with branching, retries, and long-running coordination. It fits batch orchestration when the job needs workflow state, failure handling, and execution history. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/eventbridge-integration.html)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart TD
  START[EventBridge Scheduler / API] --> SFN[Step Functions Standard]

  SFN --> INIT[Init / Validate Input]
  INIT --> LOAD[Load Manifest from S3]
  LOAD --> CHOICE{Job type?}

  CHOICE -->|Short task| LAMBDA[Lambda Task]
  CHOICE -->|Long task| ECS[ECS/Fargate Task]
  CHOICE -->|Bulk ETL| BATCH[AWS Batch Job]
  CHOICE -->|Data prep| GLUE[Glue Job]

  LAMBDA --> AGG[Aggregate Results]
  ECS --> AGG
  BATCH --> AGG
  GLUE --> AGG

  AGG --> SUCCESS[Publish Success Event]
  SFN --> CATCH[Catch / Retry / Backoff]
  CATCH --> DLQ[SQS DLQ / Failure Queue]
  CATCH --> ALERT[SNS / PagerDuty]
  SUCCESS --> AUDIT[CloudWatch + CloudTrail + Execution History]
```

***

## 4) Step Functions Express

Express workflows are optimized for very high request rates and short-duration orchestration, commonly as micro-batch workers or child workflows. They trade full long-term execution history for lower cost and higher throughput. [dev](https://dev.to/aws-builders/step-functions-distributed-map-best-practices-for-large-scale-batch-workloads-55n2)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  TRIG[SQS / API / Scheduler] --> EXP[Step Functions Express]
  EXP --> VAL[Validate Batch]
  VAL --> ENRICH[Enrich]
  ENRICH --> SCORE[Score / Transform]
  SCORE --> SAVE[Persist to DynamoDB / S3]
  SAVE --> EVT[Publish EventBridge Event]
  EXP --> ERR[Retry / Catch]
  ERR --> DLQ[SQS DLQ]
  SAVE --> LOGS[CloudWatch Logs]
```

***

## 5) Step Functions Distributed Map

Distributed Map supports very large-scale parallel processing and can read input directly from Amazon S3, then spawn large numbers of child executions. This is one of the strongest options for your 300k+ daily records pattern. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/state-map-distributed.html)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart TD
  SCHED[Scheduler / API Trigger] --> PARENT[Step Functions Standard Parent]
  PARENT --> S3IN[S3 Manifest / CSV / JSON / Inventory]
  S3IN --> DMAP[Distributed Map<br/>ItemReader + ItemBatcher]

  DMAP --> C1[Child Exec 1<br/>Lambda / Express]
  DMAP --> C2[Child Exec 2<br/>Lambda / Express]
  DMAP --> C3[Child Exec N<br/>Lambda / Fargate]
  
  C1 --> OUT1[S3 Partial Result]
  C2 --> OUT2[S3 Partial Result]
  C3 --> OUT3[S3 Partial Result]

  OUT1 --> WRITER[ResultWriter to S3]
  OUT2 --> WRITER
  OUT3 --> WRITER

  WRITER --> AGG[Aggregate / Finalize]
  AGG --> AUDIT[Audit + Metrics + Replay List]
  DMAP --> FAIL[Tolerated Failure Threshold]
  FAIL --> DLQ[Failure S3 Prefix / SQS DLQ]
```

***

## 6) SQS + Lambda fan-out

SQS with Lambda event source mapping is a standard AWS pattern for decoupled, elastic fan-out processing with buffering and DLQ isolation. It works especially well for stateless record processing and CRUD-side asynchronous operations. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  PRODUCER[API / Scheduler / Stream Consumer] --> Q[SQS Standard Queue]
  Q --> ESM[Lambda Event Source Mapping]
  ESM --> L1[Lambda Worker 1]
  ESM --> L2[Lambda Worker 2]
  ESM --> L3[Lambda Worker N]

  L1 --> DDB[DynamoDB Idempotency / Status]
  L2 --> DDB
  L3 --> DDB

  L1 --> S3[S3 Results / Audit]
  L2 --> S3
  L3 --> S3

  ESM --> RETRY[Visibility Timeout + Retry]
  RETRY --> DLQ[SQS DLQ]
  DDB --> OBS[CloudWatch / X-Ray / CloudTrail]
  S3 --> OBS
```

***

## 7) SQS FIFO + Fargate consumers

FIFO queues preserve ordered processing per message group, which is useful for tenant-entity ordering or customer-state transitions. ECS/Fargate consumers can long-poll FIFO queues while preserving per-group sequencing. [aws.amazon](https://aws.amazon.com/blogs/containers/cost-optimization-checklist-for-ecs-fargate/)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  APP[Producer / Orchestrator] --> FIFO[SQS FIFO Queue<br/>MessageGroupId=tenant#entity]
  FIFO --> ECS[ECS Service on Fargate]
  ECS --> P1[Consumer Task A]
  ECS --> P2[Consumer Task B]
  ECS --> P3[Consumer Task N]

  P1 --> DB[DynamoDB / Aurora / S3]
  P2 --> DB
  P3 --> DB

  ECS --> SCALE[Service Auto Scaling<br/>queue depth driven]
  P1 --> SIGTERM[Graceful Shutdown Hook]
  P2 --> SIGTERM
  P3 --> SIGTERM

  FIFO --> DLQ[FIFO DLQ]
  DB --> OBS[CloudWatch Logs / Metrics]
```

***

## 8) AWS Batch

AWS Batch provides managed batch queues, job definitions, and compute environments on ECS, EKS, EC2, or Fargate, reducing custom scheduler and capacity-management work. It is particularly suitable for long-running containerized jobs and array-job sharding. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart TD
  TRIG[Scheduler / Step Functions / API] --> SUBMIT[AWS Batch SubmitJob]
  SUBMIT --> JQ[Job Queue]
  JQ --> JD[Job Definition<br/>image + vCPU + memory + retry]
  
  JD --> CE1[Compute Environment<br/>Fargate Spot]
  JD --> CE2[Compute Environment<br/>On-Demand fallback]
  JD --> CE3[Compute Environment<br/>EC2 Spot Cluster]

  CE1 --> JOB1[Container Job / Array Job]
  CE2 --> JOB2[Container Job / Array Job]
  CE3 --> JOB3[Container Job / Array Job]

  JOB1 --> S3[S3 Input / Checkpoints / Output]
  JOB2 --> S3
  JOB3 --> S3

  JOB1 --> LOGS[CloudWatch Logs]
  JOB2 --> LOGS
  JOB3 --> LOGS

  JQ --> FAIL[Retry Strategy / Dependencies]
  FAIL --> ALERT[SNS / DLQ Workflow]
```

***

## 9) ECS/Fargate Scheduled Tasks

Scheduled ECS tasks are a straightforward pattern when a containerized task should run on a schedule without a larger workflow engine. EventBridge Scheduler or rules can invoke ECS `RunTask` directly. [builder.aws](https://builder.aws.com/content/34jTIHSM6RwweiUIMU4GJcgSVRs/how-to-schedule-ecs-tasks-choosing-between-eventbridge-scheduler-eventbridge-rules-and-step-functions)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  SCHED[EventBridge Scheduler / Rule] --> RUNTASK[ECS RunTask]
  RUNTASK --> CLUSTER[ECS Cluster]
  CLUSTER --> TASK[Fargate Task<br/>Java or Python container]
  TASK --> S3[S3 Read / Write]
  TASK --> DB[DynamoDB / Aurora]
  TASK --> LOGS[CloudWatch Logs]
  TASK --> RETRY[App-level Retry / Exit Code]
  RETRY --> ALERT[SNS / EventBridge failure rule]
```

***

## 10) Glue Jobs / Glue Workflows

Glue provides managed Spark ETL and workflow chaining with native ties to the Glue Data Catalog and S3-based data lakes. It is a strong option for large-scale transformation and schema-aware ETL. [community](https://community.aws/content/2n7nKFwWWmJs5Bg9s2UNoQO0Ydn/how-to-architect-a-high-performance-batch-processing-pipeline-with-apache-spark-on-aws)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart TD
  TRIG[Scheduler / S3 arrival / API] --> WF[Glue Workflow]
  WF --> CRAWLER[Glue Crawler]
  CRAWLER --> CATALOG[Glue Data Catalog]

  WF --> EXTRACT[Glue Job Extract]
  EXTRACT --> RAW[S3 Raw Zone]

  RAW --> TRANSFORM[Glue Spark Transform Job]
  TRANSFORM --> CURATED[S3 Curated Zone]

  CURATED --> LOAD[Glue Job Load]
  LOAD --> TARGET[Redshift / Aurora / Athena]

  TRANSFORM --> BOOKMARK[Job Bookmark / Incremental State]
  WF --> FAIL[Retry / Trigger Failure Path]
  FAIL --> ALERT[SNS / CloudWatch Alarm]
```

***

## 11) MWAA (Managed Airflow)

MWAA runs Apache Airflow as a managed service, with DAGs stored in S3 and workers executing task operators. It is most appropriate when the team prefers Airflow’s Python DAG model and operator ecosystem. [community](https://community.aws/content/2n7nKFwWWmJs5Bg9s2UNoQO0Ydn/how-to-architect-a-high-performance-batch-processing-pipeline-with-apache-spark-on-aws)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart TD
  DEV[Data Engineering Team] --> DAGS[S3 DAG Bucket]
  DAGS --> MWAA[Amazon MWAA Environment]
  MWAA --> SCHED[Airflow Scheduler]
  MWAA --> WEB[Airflow Web UI]
  MWAA --> WORKERS[Airflow Workers]

  WORKERS --> OP1[PythonOperator / BashOperator]
  WORKERS --> OP2[ECSOperator / GlueOperator]
  WORKERS --> OP3[EMR / Lambda / SQL Operators]

  OP1 --> DATA[S3 / DB / APIs]
  OP2 --> DATA
  OP3 --> DATA

  MWAA --> LOGS[CloudWatch Logs]
  MWAA --> META[Airflow Metadata DB]
```

***

## 12) Lambda with DynamoDB Streams / Kinesis / S3 Events

These patterns are event-driven rather than schedule-driven, and they are useful when the source data change should immediately trigger downstream processing. They fit CRUD-side async operations and near-real-time enrichment especially well. [dev](https://dev.to/aws-builders/event-driven-batch-processing-on-aws-from-scheduled-tasks-to-auto-scaling-workloads-20a6)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  subgraph Sources
    DDB[DynamoDB Table]
    KIN[Kinesis Data Stream]
    S3E[S3 Object Created Event]
  end

  DDB --> DSTREAM[DynamoDB Stream]
  KIN --> KPROC[Lambda Event Source Mapping]
  S3E --> S3NOTIF[S3 Notification]

  DSTREAM --> L[Lambda Processor]
  KPROC --> L
  S3NOTIF --> L

  L --> SQS[SQS Buffer / Retry]
  L --> SFN[Step Functions]
  L --> DB[Target DB / API]
  SQS --> DLQ[SQS DLQ]
  L --> OBS[CloudWatch / X-Ray]
```

***

## 13) EC2 Spot-based batch clusters

EC2 Spot clusters provide the lowest compute cost for fault-tolerant batch but require interruption-aware design and more operational control. They are often paired with AWS Batch or a self-managed scheduler. [docs.aws.amazon](https://docs.aws.amazon.com/batch/latest/userguide/best-practices.html)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart TD
  TRIG[Scheduler / Batch / Custom Controller] --> ASG[Auto Scaling Group / Spot Fleet]
  ASG --> EC2A[EC2 Spot Worker A]
  ASG --> EC2B[EC2 Spot Worker B]
  ASG --> EC2N[EC2 Spot Worker N]

  EC2A --> WORK[Batch Process]
  EC2B --> WORK
  EC2N --> WORK

  WORK --> CKPT[S3 Checkpoint / Progress State]
  WORK --> OUT[S3 Output]
  WORK --> LOGS[CloudWatch Agent / Logs]

  INT[Spot Interruption Notice] --> SAVE[Save checkpoint + drain]
  SAVE --> CKPT
  CKPT --> RESUME[Resume on replacement node]
```

***

## 14) AWS Data Pipeline (legacy)

AWS Data Pipeline is legacy and generally replaced by newer AWS-native orchestration and ETL services such as Glue and Step Functions. The diagram is included only for comparison and migration context. [community](https://community.aws/content/2n7nKFwWWmJs5Bg9s2UNoQO0Ydn/how-to-architect-a-high-performance-batch-processing-pipeline-with-apache-spark-on-aws)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  DEF[Pipeline Definition] --> SCHED[Data Pipeline Scheduler]
  SCHED --> ACT1[Extract Activity]
  ACT1 --> ACT2[Transform Activity]
  ACT2 --> ACT3[Load Activity]
  ACT3 --> TARGET[S3 / RDS / Redshift]
  SCHED --> RETRY[Retry / Attempt]
  RETRY --> ALERT[SNS Notification]
```

***

## 15) Multi-region active-active scheduler

Cross-Region event routing in EventBridge is supported, and active-active coordination patterns commonly combine regional schedulers with shared control state to avoid split-brain execution. For resilient scheduling, pair regional schedulers with a DynamoDB Global Table lease or idempotent downstream execution. [docs.aws.amazon](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cross-region.html)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF"
  }
}}%%
flowchart LR
  subgraph USE1[Region us-east-1]
    S1[EventBridge Scheduler]
    L1[Lease Check Lambda]
    F1[Step Functions / Batch]
    S1 --> L1 --> F1
  end

  subgraph EUW1[Region eu-west-1]
    S2[EventBridge Scheduler]
    L2[Lease Check Lambda]
    F2[Step Functions / Batch]
    S2 --> L2 --> F2
  end

  GT[DynamoDB Global Table<br/>scheduler lease + idempotency]
  S3CRR[S3 Cross-Region Replication]
  BUSX[Cross-Region EventBridge Bus]

  L1 --> GT
  L2 --> GT
  F1 --> S3CRR
  F2 --> S3CRR
  F1 --> BUSX
  F2 --> BUSX

  GT --> AUD[Audit / Replay / Failover visibility]
```

***

## 16) Best final architecture

This combined pattern aligns most directly with your workload constraints because it uses EventBridge Scheduler for schedules, Step Functions for orchestration, SQS for buffering, Lambda for short parallel work, and Batch or Fargate for long-running jobs. Cross-Region EventBridge routing and DynamoDB Global Tables support the active-active control plane, while S3 and DynamoDB support low-duplication storage, idempotency, and auditability. [docs.aws.amazon](https://docs.aws.amazon.com/step-functions/latest/dg/using-eventbridge-scheduler.html)

```mermaid
%%{init: {
  "theme": "base",
  "themeVariables": {
    "primaryColor": "#FF9900",
    "primaryTextColor": "#232F3E",
    "primaryBorderColor": "#146EB4",
    "lineColor": "#146EB4",
    "secondaryColor": "#EAF3FB",
    "tertiaryColor": "#FFF4E5",
    "background": "#FFFFFF",
    "clusterBkg": "#F8FBFF",
    "clusterBorder": "#146EB4"
  }
}}%%
flowchart TD
  subgraph REG1[Region A]
    EBS1[EventBridge Scheduler]
    LEASE1[Lease Check Lambda]
    SFN1[Step Functions Standard]
    Q1[SQS Standard / FIFO]
    L1[Lambda Workers]
    B1[AWS Batch / ECS Fargate]
    EBS1 --> LEASE1 --> SFN1
    SFN1 --> Q1
    Q1 --> L1
    SFN1 --> B1
  end

  subgraph REG2[Region B]
    EBS2[EventBridge Scheduler]
    LEASE2[Lease Check Lambda]
    SFN2[Step Functions Standard]
    Q2[SQS Standard / FIFO]
    L2[Lambda Workers]
    B2[AWS Batch / ECS Fargate]
    EBS2 --> LEASE2 --> SFN2
    SFN2 --> Q2
    Q2 --> L2
    SFN2 --> B2
  end

  GT[DynamoDB Global Tables<br/>Lease + Idempotency + Audit Index]
  S3[S3 Partitioned Data Lake<br/>input / checkpoint / output / replay]
  DLQ[SQS DLQs]
  OBS[CloudWatch + X-Ray + CloudTrail]
  BUS[EventBridge Cross-Region Bus]

  LEASE1 --> GT
  LEASE2 --> GT
  L1 --> GT
  L2 --> GT
  B1 --> S3
  B2 --> S3
  L1 --> S3
  L2 --> S3
  Q1 --> DLQ
  Q2 --> DLQ
  SFN1 --> BUS
  SFN2 --> BUS
  GT --> OBS
  S3 --> OBS
  DLQ --> OBS
```

## Rendering notes

Mermaid architecture diagrams support grouped services and cloud-style layouts in newer Mermaid versions, but many markdown renderers still support flowcharts more consistently, which is why the document above uses mostly flowchart syntax for portability. If your renderer supports Mermaid icon packs, you can replace labels with AWS logos-based architecture syntax, but icon registration is renderer-dependent in tools like VS Code, GitHub Pages, and Obsidian. [stackoverflow](https://stackoverflow.com/questions/79123430/rendering-icons-in-mermaid-architecture-diagram)

Would you like the next step to be a single consolidated `.md` document with:
1. table of contents,
2. all diagrams grouped by category,
3. reusable AWS Mermaid style classes,
4. and your 8 workload-specific diagrams added in the same format?