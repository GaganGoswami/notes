# AWS-Native KYC Processing Migration from Unix/MS SQL/Autosys

## 1. Context, Constraints, and Goals

The current landscape consists of Autosys-scheduled Python jobs on Unix that read and write to Microsoft SQL Server, invoke stored procedures for business rules, pull upstream data daily, and trigger downstream KYC workflows.
The target state is a fully AWS-based architecture using API Gateway, Lambda (primarily Java runtimes), DynamoDB, S3, EventBridge, and Step Functions, augmented where appropriate with SQS, Kinesis, Glue, DynamoDB Streams, and ECS Fargate.
MS SQL data will be migrated into AWS, with core processing running in AWS and on‑prem access retained only for unavoidable upstream dependencies via secure connectivity.
The long‑term objective is to decommission MS SQL, Unix, and Autosys, while adopting Hexagonal Architecture, Domain‑Driven Design (DDD), and event-driven principles where they add value.

Key non‑functional goals:

- Strong observability (tracing, metrics, logs) and operational excellence.
- Elastic scalability and cost efficiency via serverless and managed services.
- Clear separation of domain logic from infrastructure (Hexagonal + DDD).
- Support for both near–real‑time and scheduled/batch workloads.


## 2. Architecture Options and Patterns

This section analyzes the main architecture patterns relevant to the migration, describing why, what, how, pros, cons, cost model, scaling, observability, and when to use or avoid each.

### 2.1 Full Event-Driven Architecture (EDA)

#### What

A fully event-driven approach pushes all significant state changes through domain events (e.g., `CustomerProfileUpdated`, `KycReviewDue`, `KycCaseOpened`) published to an event bus (Amazon EventBridge) or streams (Kinesis) and consumed by multiple downstream services.
Workflows emerge from event choreography, with each microservice responding to events and emitting follow‑up events.

#### Why / When to Use

- High decoupling between bounded contexts is required.
- Multiple consumers need to react to the same business events (e.g., KYC, reporting, notifications, risk scoring).
- New downstream capabilities should be easy to add without changing upstream producers.

Choreography between bounded contexts and orchestration within a single bounded context is a recommended pattern in serverless systems.[1][2]

#### How (AWS implementation)

- Use EventBridge as the central event bus for business and integration events across domains.[1]
- Services (Lambdas or Fargate tasks) publish domain events after successful domain operations.
- Each bounded context (e.g., Customer, KYC, Risk) owns its own DynamoDB tables and emits events on state changes.
- EventBridge rules route events to subscribers (Lambda, Step Functions, SQS, or Kinesis for fan‑out or buffering).
- API Gateway frontends write‑side operations and publishes initial events via Lambda.

#### Pros

- Loose coupling and high extensibility.
- Natural audit trail via event history.
- Independent scaling per consumer.
- Enables near–real‑time processing where previously batch existed.

#### Cons

- Harder global reasoning about end‑to‑end flows; choreography can be difficult to trace and debug at scale.[2][1]
- Complex error handling and compensation across multiple services.
- Potential event explosion; requires disciplined schema versioning and event governance.

#### Cost Model

- EventBridge charges per million events; typically low unit cost but high‑volume systems must limit noisy events.
- Lambda is pay‑per‑invocation and duration; cost proportional to events processed.
- DynamoDB on‑demand pricing scales with read/write throughput, or provisioned with autoscaling.
- No always‑on compute cost; ideal for spiky workloads.

#### Scaling and Observability

- Horizontal scaling via Lambda concurrency and DynamoDB throughput.
- EventBridge is managed and scales automatically.
- Observability via CloudWatch metrics and logs on Lambda and EventBridge, and X-Ray tracing across services.[3]
- For complex flows, raw EDA requires additional tooling (correlation IDs, custom tracing) to reconstruct flows.

#### When Not to Use

- Highly sequential, long‑running workflows where global visibility, explicit branching, and timeout handling matter (better served by Step Functions orchestration).[4][1]
- Where existing processing is simple daily batch with no benefit from real‑time events.


### 2.2 Hybrid EDA + Scheduled Batch Processing

#### What

A hybrid architecture combines:

- Event‑driven flows for real‑time or near–real‑time use cases (e.g., new customer onboarding, immediate KYC triggers).
- Scheduled batch flows for daily or periodic processing (e.g., bulk periodic KYC reviews, nightly quality checks, regulatory extracts).

#### Why / When to Use

- Current processes are mostly daily batch; not all require real‑time re‑engineering.
- Some upstream systems or regulations naturally operate in daily/periodic cycles.
- Incremental migration strategy: start by porting jobs as scheduled serverless tasks (EventBridge Scheduler → Lambda/Fargate), then introduce more EDA where it adds value.

#### How (AWS implementation)

- Use EventBridge Scheduler as the managed scheduler for periodic jobs (replacing Autosys).[3]
- For batch workloads, trigger:
  - Lambda directly for short‑running jobs.
  - Step Functions for multi‑step orchestrations.
  - AWS Batch/ECS Fargate for heavier containerized workloads.[3]
- Within each scheduled pipeline, publish domain events into EventBridge to keep the wider system event‑aware (e.g., `KycReviewsGenerated`, `DailyKycCompleted`).

#### Pros

- Balances modernization cost with business value.
- Easier phased migration from Autosys cron‑like jobs.
- Allows selective real‑time enablement where justified.

#### Cons

- Two modes to operate and monitor (event‑driven and scheduled), increasing operational complexity.
- Might delay full benefits of EDA until more flows are converted.

#### Cost Model

- EventBridge Scheduler charges per invocation; low cost for typical schedules.[3]
- Lambda and Step Functions costs as previously described.
- ECS Fargate adds vCPU/GB‑hour costs when used for heavy jobs.[3]

#### Scaling and Observability

- Schedules are independent per job; concurrency managed on target (e.g., Lambda reserved concurrency, Fargate task counts).
- Observability via CloudWatch metrics and logs for EventBridge, Step Functions, Lambda, and ECS/EKS/Batch.[3]

#### When Not to Use

- When there is a strong requirement to move entirely away from batch semantics and everything must be real‑time (rare in regulatory/KYC domains).
- When batch workflows are simple enough that EDA alone suffices.


### 2.3 Step Functions–Driven Orchestration Pipelines

#### What

AWS Step Functions acts as an orchestrator, modeling workflows as state machines that invoke Lambda functions, make decisions, handle retries, and aggregate results.[4][1]

#### Why / When to Use

- Need explicit, visual workflow definitions for complex, multi‑step KYC pipelines.
- Strong need for centralized visibility, auditing, and error handling for business‑critical flows.[1][4]
- Workflows require branching (choices based on customer risk, geography), parallel execution (multiple checks in parallel), and timeouts.

#### How (AWS implementation)

- Define state machines in ASL (Amazon States Language) or CDK.
- Use `Task` states to invoke Lambda functions that encapsulate domain services (Java) via hexagonal ports.
- Use `Choice` states for business branching logic (e.g., risk band → different sub‑flows).
- Integrate with EventBridge to publish events from `Task` states or final states (e.g., `KycCaseOpened`).[1]

#### Pros

- Clear, visual representation and centralized control of workflows.
- Built‑in retries, catch/compensation, and timeout handling.[4][1]
- Detailed execution history and per‑step metrics; excellent observability.
- Decouples workflow from individual Lambda implementations.

#### Cons

- Additional service to learn and operate.
- Cost per state transition (e.g., around 25 USD per million state transitions in standard mode).[1]
- A failure of Step Functions in a critical region affects all orchestrated workflows.

#### Cost Model

- Cost driven by number of state transitions, which can be optimized via design (grouping work, reusing lambdas).[1]
- For high‑volume but simple flows, direct Lambda + SQS or EventBridge may be cheaper.

#### Scaling and Observability

- Step Functions scales horizontally; Lambdas invoked as needed.
- Provides built‑in visual graphs, execution logs, and CloudWatch metrics, simplifying tracing.[1]

#### When Not to Use

- Simple single‑step jobs where a Lambda or Fargate task is sufficient.
- Very high‑frequency, low‑latency event processing with minimal branching where state‑transition cost is prohibitive.


### 2.4 S3 Event Ingestion → Lambda → DynamoDB

#### What

A common pattern for file‑based ingestion where placing a file into S3 triggers a Lambda function that validates, parses, and persists records into DynamoDB (and possibly emits events).

#### Why / When to Use

- Existing processes rely on upstream batch file drops.
- Replacing Unix jobs that watch directories and load data into SQL Server.
- Need serverless, scalable ingestion with pay‑per‑use cost.

#### How (AWS implementation)

- Create S3 buckets for inbound files (segregated by domain, environment, and sensitivity).
- Configure S3 event notifications (via EventBridge or native S3 → Lambda) for `PUT` events.
- Lambda (Java) performs:
  - Schema validation and deduplication.
  - Transformation to domain models.
  - Writes to DynamoDB (or S3 data lake for raw/curated zones).
  - Publishes domain events (e.g., `UpstreamFileProcessed`).

#### Pros

- No servers to manage; scales with file volume.
- Good fit for existing file‑oriented integrations.
- Natural separation of raw and processed data using distinct S3 prefixes.

#### Cons

- Very large files may exceed Lambda limitations (memory/time); may require splitting or using ECS/EMR.
- Requires careful idempotency design to avoid duplicate processing on retries.

#### Cost Model

- S3 storage and request costs.
- Lambda invocations and duration proportional to files and parsing cost.

#### Scaling and Observability

- Parallelism controlled via Lambda concurrency and S3 event fan‑out.
- Observability via CloudWatch logs/metrics and X‑Ray tracing of Lambda.

#### When Not to Use

- Continuous high‑throughput streaming; Kinesis or MSK is more appropriate.
- Workloads requiring heavy, multi‑GB in‑memory processing (use ECS/EMR/Glue).


### 2.5 Lambda + SQS Pipelines

#### What

Queue‑based decoupling where producers enqueue work items (e.g., one message per customer or per KYC case), and Lambda functions poll SQS to process messages.

#### Why / When to Use

- Need buffering to smooth spikes and protect downstream systems.
- Individual work items are relatively small and independent.
- Need at‑least‑once processing with dead‑letter queues (DLQ) for failures.

#### How (AWS implementation)

- Producers (S3 ingestion Lambda, API Gateway lambdas, Step Functions) send messages to SQS queues.
- Lambda functions subscribed to SQS queues process messages in batches, apply domain logic, update DynamoDB/S3, and publish domain events.
- Configure DLQs per queue for unprocessable messages, with alerting via CloudWatch Alarms.

#### Pros

- Natural back‑pressure and decoupling between producers and consumers.
- Simple, robust error handling via DLQ and message visibility timeouts.
- Cost effective for moderate to high workloads.

#### Cons

- Inherent at‑least‑once semantics; idempotency is mandatory.
- Additional latency due to queueing; not ideal for strict low‑latency flows.

#### Cost Model

- SQS request charges and Lambda invocations.
- Very predictable and low cost for typical enterprise volumes.

#### Scaling and Observability

- Lambda scales concurrency based on queue depth.
- SQS is fully managed with near‑infinite scalability.
- Observability via SQS metrics (queue depth, age of oldest message) and Lambda metrics.

#### When Not to Use

- Strong ordering requirements across many related messages (Kinesis is better if strict ordering is needed per key).
- Ultra low latency (API → Lambda → DynamoDB may suffice).


### 2.6 Kinesis Streaming (If Needed)

#### What

Managed streaming using Kinesis Data Streams for continuous event ingestion, with ordered partitions (shards) and multiple consumers.

#### Why / When to Use

- High‑throughput, low‑latency, ordered ingest is required (e.g., high‑volume transaction streams feeding KYC risk scoring).
- Need multiple independent stream consumers (analytics, models, KYC engine) with replay capability.

#### How (AWS implementation)

- Producers (upstream apps, on‑prem connectors) send events to Kinesis streams.
- Lambda or Kinesis Data Analytics consume from streams, transform records, update domain stores, and publish domain events.
- Use partition keys (e.g., customer ID) to achieve per‑customer ordering.

#### Pros

- Ordered, replayable stream with multiple consumers.
- Well‑suited to high‑volume real‑time analytics and fraud/KYC scoring.

#### Cons

- Higher operational complexity than pure EventBridge/SQS.
- Requires shard capacity planning (unless using on‑demand mode) and consumer lag monitoring.

#### Cost Model

- Charging per shard‑hour and per MB of data; on‑demand mode charges per MB ingested without shard management.
- Additional cost for consumer Lambdas or analytics.

#### Scaling and Observability

- Scale via shard count or on‑demand mode; autoscaling possible.
- Observability via Kinesis and consumer metrics (iterator age, throughput).

#### When Not to Use

- Workloads that are naturally daily batch or low‑volume event‑driven; EventBridge/SQS are simpler.
- When ordering and replay are not critical.


### 2.7 Glue (ETL/ELT) and Glue Workflows

#### What

AWS Glue provides serverless Spark‑based ETL/ELT, crawlers for schema discovery, and workflows for chaining jobs.[5][6]

#### Why / When to Use

- Migration and processing of large data volumes (historical MS SQL data, aggregates for analytics).
- Need for schema discovery and cataloging in AWS Glue Data Catalog.
- Need Spark‑level transformations for complex data enrichment across large datasets.

#### How (AWS implementation)

- Use DMS to move MS SQL data into S3 in Parquet or CSV.[7][8][6][9]
- Configure Glue Crawlers to infer schemas and register them in Glue Data Catalog.[5]
- Implement Glue ETL jobs (Scala/Python) to transform, filter, and partition data into curated S3 zones or to load into DynamoDB/Athena/Redshift.
- Orchestrate sequences of Glue jobs and related steps with Glue Workflows.

#### Pros

- Designed for large‑scale batch ETL and historical data processing.[6][7]
- Tight integration with DMS, S3, Athena, Redshift, Data Catalog.[6][5]

#### Cons

- Spark‑based; developers require different skill set than Java Lambdas.
- Higher job startup latency; not meant for low‑latency transactional workloads.

#### Cost Model

- Charged per DPU‑hour for Glue jobs; cost scales with job duration and resources.

#### Scaling and Observability

- Glue auto‑scales within job allocations; performance tuned via DPUs and partitioning.
- Observability via Glue job metrics, CloudWatch logs, and Data Catalog auditing.

#### When Not to Use

- Fine‑grained transactional or request/response flows (use Lambda/DynamoDB instead).
- CPU‑light, quickly finishing workloads; Lambda or Fargate cheaper and simpler.


### 2.8 DynamoDB Streams for Reactive Domain Events

#### What

DynamoDB Streams capture ordered item‑level changes, enabling reactive event processing when items are inserted, updated, or deleted.[5]

#### Why / When to Use

- Need to react synchronously to data changes in DynamoDB (e.g., when `KycStatus` changes to `Due`, trigger review flow).
- Want to avoid duplicating event publishing logic in application code for some internal events.

#### How (AWS implementation)

- Enable Streams (NEW_AND_OLD_IMAGES) on domain tables.
- Attach Lambda consumers to streams to translate change records into domain events on EventBridge or to trigger follow‑up actions.

#### Pros

- Guaranteed per‑partition ordering of changes.
- No extra write path logic required in application for those reactive use cases.

#### Cons

- Event semantics tied closely to data model; refactors of the table may ripple into stream consumers.
- Not a replacement for explicit business events when rich context or versioning is needed.

#### Cost Model

- DynamoDB Streams and Lambda consumption add modest additional cost; usually small relative to DynamoDB writes.

#### Scaling and Observability

- Streams scale with table throughput; Lambda concurrency matches shard count.
- Observability via stream metrics (records processed, iterator age) and Lambda metrics.

#### When Not to Use

- Cross‑bounded‑context integration (prefer explicit domain events via EventBridge).
- Use cases requiring enriched semantic events that are not directly tied to a single table row.


### 2.9 ECS Fargate Batch Containers (Only if Required)

#### What

Container‑based batch processing on serverless compute via ECS Fargate, optionally orchestrated by AWS Batch.[3]

#### Why / When to Use

- Jobs require long‑running compute, large memory footprint, custom OS/runtime, or third‑party binaries that are not suitable for Lambda.
- Need to host existing Java services as containers with minimal change.
- Need better control over networking, vCPU/memory, or use GPU (Fargate or EC2‑based ECS/Batch).

#### How (AWS implementation)

- Containerize existing Unix jobs (often Java/C++/Python) and publish images to ECR.
- Use AWS Batch with Fargate as compute environment; define batch job definitions and queues.[3]
- Trigger Batch jobs via EventBridge Scheduler (Autosys replacement) or EventBridge events.[3]
- Jobs read from S3/DynamoDB, process, and write results to AWS stores.

#### Pros

- Lift‑and‑shift path for complex workloads not suited to Lambda.
- Fine‑grained control over runtime and dependencies.

#### Cons

- More operational complexity compared to pure Lambda.
- Longer cold starts and more complex scaling logic.[3]

#### Cost Model

- Pay per vCPU/GB‑hour for Fargate plus Batch control plane overhead.[3]
- Potential cost savings with Spot where interruptions are acceptable.

#### Scaling and Observability

- Scaling via Batch job queues and Fargate task counts.[3]
- Observability via Batch dashboards, CloudWatch Logs, and container metrics.[3]

#### When Not to Use

- Short‑lived jobs that run in seconds (Lambda is more cost‑efficient and simpler).[3]
- Simple ETL; Glue/Lambda may be preferable.


### 2.10 AWS Data Migration Architecture (MS SQL → AWS)

This pattern underpins all others and is covered in detail in Section 3.
High‑level options:

- DMS → S3 (data lake) → Glue → DynamoDB/analytics.[8][9][7][6][5]
- DMS → DynamoDB for online workloads with access pattern–driven table design.[10][11]
- Hybrid of both, especially where historical data and hot transactional data have different requirements.


## 3. MS SQL → AWS Data Migration Design

### 3.1 Migration Objectives and Scope

- Migrate transactional KYC‑related data from MS SQL Server to AWS, primarily into DynamoDB and S3.
- Migrate historical data for reporting and compliance, likely into S3 (data lake) and possibly Athena/Redshift.
- Replace stored procedures used in KYC pipelines with AWS‑native domain services.


### 3.2 Data Migration Strategy Overview

Recommended pattern is a multi‑phase migration using AWS DMS and Glue:

- Use DMS to replicate MS SQL schemas into S3 (Parquet/CSV) and/or DynamoDB.[11][9][10][7][8][6]
- Use Glue to transform, partition, and load data into target schemas (DynamoDB, curated S3 zones) and to catalog data for lineage and discovery.[6][5]
- Perform one‑time full load followed by change data capture (CDC) where near‑continuous synchronization is needed until cutover.[9][7][6]


### 3.3 DMS–Based Migration

#### DMS to S3

- Create DMS replication instance sized according to data volume and throughput requirements.[7][9][6]
- Configure source endpoint for MS SQL (on‑prem or RDS) and target endpoint for S3.[8][7]
- Configure DMS tasks to perform full load and optionally CDC into partitioned S3 prefixes (e.g., `s3://data-landing/sql/<schema>/<table>/ingestion_date=...`).[7][8][6]
- Store output in columnar formats like Parquet for efficiency if using S3 as a data lake.[8][6]

#### DMS to DynamoDB

- Where access patterns are well understood and transactional workloads require NoSQL performance, configure DMS to target DynamoDB tables.[10][11]
- Design DynamoDB table schema and GSIs around access patterns first; then map MS SQL schemas via DMS table mappings and transformation rules.[11][5]
- Use DMS transformation to flatten or denormalize relational joins into single DynamoDB items or nested attributes.[11]


### 3.4 Migration of Transactional Tables

- Identify core operational tables (customers, accounts, KYC cases, documents, risk scores).
- For each bounded context, design DynamoDB single‑table or few‑table models focusing on:
  - Primary access patterns (get/update customer by ID, list KYC cases by customer, list open KYC tasks by status).
  - Secondary access patterns requiring GSIs.
- Use DMS to perform initial load into staging S3, then Glue jobs to reshape into target DynamoDB form, or configure DMS directly to target DynamoDB with transformation on the fly.[10][11][5]


### 3.5 Migration of Historical Data

- Historical tables and large audit/log tables should be migrated into S3 via DMS, ideally in compressed Parquet format and partitioned by time or other relevant dimensions.[7][8][6]
- Use Glue Crawlers to register schemas in Glue Data Catalog for Athena/Redshift Spectrum querying.[5][6]
- Consider aggregated or materialized views for frequently accessed reporting.


### 3.6 Migration of Large Objects / Files

- If MS SQL currently stores BLOBs (e.g., KYC documents, scanned IDs), migrate them to S3.
- Use DMS to export references (keys/metadata) and custom ingestion scripts to move binary content where DMS limits on BLOB sizes apply.[12]
- Store documents in secure S3 buckets with SSE‑KMS encryption and object‑level access controls.


### 3.7 Validation Strategy

- Use DMS pre‑migration assessment to detect incompatible data types and issues before running the migration.[12][9]
- After load, perform:
  - Row‑count comparisons between MS SQL and S3/DynamoDB datasets.
  - Checksum or hash comparisons for key tables.
  - Sample‑based field‑level validation.
- Use Glue or Athena queries to validate referential integrity where relevant.


### 3.8 Cutover Strategy

Recommended approach: blue/green (parallel run) with time‑boxed CDC.

- Phase 1: Full load using DMS while current system continues to write to MS SQL.[9][6][7]
- Phase 2: Enable CDC in DMS to replicate incremental changes to AWS targets; run for extended period to validate parity.[9][6][7]
- Phase 3: Route new writes to AWS‑based KYC system while still replicating to MS SQL (if needed for rollback).
- Phase 4: Freeze writes to MS SQL, finalize replication lag, validate consistency, and then decommission MS SQL once confidence is achieved.


### 3.9 Data Lineage and Cataloging

- Use Glue Data Catalog as the central catalog for S3‑hosted datasets.[6][5]
- Maintain metadata such as source system, transformation jobs, and target usage in the catalog.
- For DynamoDB, maintain data models, events, and field definitions in internal documentation and optionally in a metadata service.


## 4. Rewrite of Business Rules (Stored Procedures → Domain Services)

### 4.1 Strategy Overview

Stored procedures currently encapsulate KYC business rules tightly coupled to MS SQL.
The migration should:

- Extract business logic into domain modules using Hexagonal Architecture and DDD.
- Implement domain logic in Java domain services invoked by Lambda or Fargate.
- Separate configuration (thresholds, mappings, reference data) from code.
- Avoid procedural anti‑patterns by modeling rich domain objects and invariants.


### 4.2 Extraction and Modularization

Steps:

1. Inventory and classify stored procedures by domain (Customer, KYC, Risk, Integration, Utility).
2. For each procedure, document:
   - Inputs/outputs.
   - Business rules and decisions (not just SQL.
   - Side effects (tables updated, external calls).
3. Identify domain aggregates (e.g., `Customer`, `KycCase`, `KycTask`) and map stored procedures to aggregate methods.
4. Design services per bounded context, each implementing a set of use cases.


### 4.3 Mapping to Hexagonal Architecture

- Ports: Java interfaces that represent domain operations (e.g., `KycReviewService`, `CustomerRepository`, `KycEventPublisher`).
- Adapters:
  - Inbound: API Gateway/Lambda handlers, SQS/Kinesis consumers, Step Functions task handlers.
  - Outbound: DynamoDB repositories, S3 repositories, EventBridge publishers, on‑prem connectors.
- Domain: Pure Java modules (POJOs), aggregates, value objects, domain services implementing core logic.

This isolates domain logic from AWS details, easing testing and long‑term maintenance.


### 4.4 Implementation Targets

#### Lambda Functions

- Primary compute for request/response and short‑running batch steps.
- Implement Java Lambdas using frameworks like Spring Cloud Function, Micronaut, or Quarkus for fast startup.
- Each Lambda should be thin: parse event, call domain service via a port, return result or emit event.

#### Domain Services

- Implement domain services as pure Java modules that can run in both Lambda and Fargate contexts.
- Encapsulate KYC business rules (e.g., risk scoring, periodic review generation) in these modules.

#### Step Functions Choice Logic

- Use Choice states for high‑level branching (e.g., `if risk > HIGH then route to enhanced due diligence`), not fine‑grained rules.
- Keep rule complexity inside domain services to avoid bloated state machines.

#### Custom Rule Engine (Optional)

- If KYC rules change frequently and require non‑developer editing, consider a rule engine layer (e.g., Drools packaged in Lambda/Fargate) or a simple DSL loaded from S3/Parameter Store.
- Carefully govern complexity: many KYC systems succeed with well‑structured domain code and configuration without a full rule engine.


### 4.5 Externalizing Configuration vs Logic

- Store configuration in AWS Systems Manager Parameter Store or Secrets Manager for sensitive values.[3]
- Examples of externalized config:
  - Risk thresholds and scoring weights.
  - Country‑specific rules toggles.
  - Job scheduling parameters.
- Do not externalize core domain logic; preserve invariants and complex flows in typed Java code.


### 4.6 Handling 3000+ Domain Fields and Complex Enrichments

Patterns:

- Define structured domain models with nested value objects instead of flat 3000‑field DTOs.
- Use mapping layers (e.g., MapStruct) to translate between persistence schemas and domain objects.
- Use versioned schemas (e.g., JSON Schema) for inbound payloads and S3‑stored files.
- Group enrichments into modules (Customer Data Enrichment, Risk Data Enrichment, Sanctions Screening Enrichment) and orchestrate via Step Functions.


### 4.7 Idempotency, Ordering, and Consistency

- Idempotency:
  - Use idempotency keys (e.g., upstream batch file ID + record ID) stored in DynamoDB to avoid double processing.
  - Design updates as upserts where feasible.
- Ordering:
  - For event flows requiring order per customer, use Kinesis partitioning by customer ID or ensure single‑threaded processing per key via SQS FIFO queues.
  - Within Step Functions, ordering is explicit via state transitions.
- Consistency:
  - Prefer eventual consistency with compensating events for cross‑aggregate operations.
  - For critical strongly consistent reads in DynamoDB, use strongly consistent read options judiciously.


### 4.8 Avoiding Procedural Anti‑Patterns

- Replace giant stored procedures with smaller domain operations.
- Encapsulate logic in methods of aggregates and domain services, not in orchestration code.
- Use Step Functions as high‑level workflow coordinators, not logic engines.


## 5. End-to-End Target Architecture Proposal

### 5.1 High-Level Architecture Overview

The recommended target architecture is a hybrid event‑driven and scheduled system with Step Functions orchestration, Lambda‑centric compute (Java), DynamoDB as the primary operational store, and S3 as the system of record for historical and file data.

Major components:

- Ingestion layer: API Gateway, S3 event ingestion, on‑prem connectors.
- Orchestration: Step Functions for complex flows; EventBridge for cross‑domain events and scheduling.
- Compute: Lambda (Java) for most domain logic, ECS Fargate/Batch only for special heavy workloads.
- Data stores: DynamoDB for operational data, S3 for documents and historical data, Athena/Redshift for analytics.
- Messaging: EventBridge, SQS, Kinesis (if required) for decoupled communication.


### 5.2 EventBridge-Driven Ingestion

- Use EventBridge as the central event bus for domain and integration events.
- Domains publish events like `CustomerUpdated`, `KycCaseOpened`, `KycReviewCompleted` into EventBridge.
- EventBridge rules route events to interested consumers (e.g., case management, notification service, audit service).[1]


### 5.3 S3-Triggered File Pipelines

- Upstream systems drop files into dedicated S3 buckets/prefixes.
- S3 `PUT` events trigger Lambda ingestion functions that:
  - Validate and parse files.
  - Transform to domain models.
  - Persist to DynamoDB and/or S3 curated zones.
  - Publish `FileIngested` and domain‑specific events.


### 5.4 Step Functions Orchestration for Multi-Step Workflows

- Use Step Functions for KYC case lifecycle:

  - Initial case creation.
  - Data enrichment calls.
  - Risk scoring.
  - Task creation and assignment.
  - Escalation and reminders.

- Choice states decide paths based on risk score, geography, or product.[4][1]
- Parallel states run independent checks concurrently (e.g., sanctions, PEP screening, adverse media).


### 5.5 Lambda-Based Transformation and Domain Layer

- Each Step Functions `Task` state invokes a Lambda that delegates to domain services.
- Lambdas are small, stateless, and concentrate on I/O and exception translation.
- Domain modules are reused across Lambda and ECS Fargate if heavy compute is needed.


### 5.6 DynamoDB CRUD and Table Design

- Prefer single‑table design per bounded context to support multiple access patterns, with GSIs for alternate queries.[11][5]
- Example KYC table (simplified):

  - PK: `CUSTOMER#ustomerId>`
  - SK: `KYC#<kycCaseId>`
  - GSI1: `StatusIndex` with PK `STATUS#<status>` and SK `DUE_DATE#<date>` for listing open cases by status/due date.

- Use DynamoDB Streams to trigger secondary processing (e.g., when `status` changes to `Due`, publish event or schedule tasks).


### 5.7 Upstream Integration with On-Prem

- Use AWS Direct Connect or VPN for secure connectivity to remaining on‑prem systems.
- Access on‑prem databases via specialized connectors (e.g., AWS DMS ongoing tasks, custom connectors in Fargate or Lambda with VPC).
- Cache or snapshot data into S3/DynamoDB when possible to reduce latency and dependency.


### 5.8 Scheduler Replacement with EventBridge Scheduler

- Replace Autosys with EventBridge Scheduler targeting:
  - Step Functions state machines for complex jobs.
  - Lambda functions for simple jobs.
  - AWS Batch/Fargate jobs for heavy container workloads.[3]

- Use cron/rate expressions for daily/monthly KYC reviews.


### 5.9 Error Handling, Retries, and DLQs

- Lambda: enable built‑in retries with exponential backoff; configure DLQs (SQS) or Lambda destinations for failures.
- Step Functions: use `Catch` blocks and fallback states; model compensating actions.
- SQS: use DLQs and CloudWatch alarms for high DLQ traffic.
- DMS: monitor progress and errors via CloudWatch logs and metrics; configure alerts.[9][11]


### 5.10 Security, IAM, and Encryption

- IAM:
  - Principle of least privilege for all roles (Lambda, ECS, DMS, Glue, Step Functions).
  - Separate roles per bounded context and environment.
- Encryption:
  - S3 buckets encrypted with SSE‑KMS.
  - DynamoDB tables encrypted at rest with KMS.
  - In‑transit encryption (TLS) for all connections.
- Secrets:
  - Use Secrets Manager for DB credentials and external API keys.[3]
  - Rotate secrets automatically where supported.


### 5.11 Observability Model

- CloudWatch:
  - Metrics and dashboards for Lambda, Step Functions, SQS, Kinesis, DMS, Glue, and DynamoDB.[11][9][3]
  - Alarms on error rates, throttling, queue depth, and iterator age.
- X-Ray:
  - Distributed tracing across API Gateway, Lambda, and Step Functions.
- Logging:
  - Structured JSON logs with correlation IDs for all services.
  - Store logs in CloudWatch Logs; optionally ship to a central log analytics service.


## 6. Data Processing Framework Design

### 6.1 Framework Objectives

- Provide a reusable runtime for AWS batch and event‑driven jobs in Java.
- Standardize how jobs read from S3/DynamoDB/on‑prem, apply domain rules, and write results.
- Embed cross‑cutting concerns: idempotency, retries, metrics, logging, tracing.


### 6.2 Core Abstractions

- `JobContext` (correlation IDs, configuration, feature flags).
- `Reader` interfaces (S3, DynamoDB, SQS, Kinesis, external DB via repositories).
- `Processor` interface encapsulating domain logic.
- `Writer` interfaces (DynamoDB, S3, EventBridge, SQS).
- `JobRunner` orchestrating Reader → Processor → Writer within a Lambda/Fargate container.


### 6.3 Read Paths

- S3: read file via S3 GetObject; stream and parse in chunks.
- DynamoDB: scan or query by partition keys; use pagination for large datasets.
- External DB (only when required): use JDBC libraries in Lambda/Fargate within VPC; ensure connection pooling and back‑off.


### 6.4 Business Rules Application

- Process records via domain modules:
  - Validate fields and enrich from reference data caches.
  - Compute risk scores.
  - Determine whether KYC review or escalation is required.

- Framework provides utilities for:
  - Schema validation (JSON Schema, CSV schema checks).
  - Configuration resolution (Parameter Store, S3‑hosted config files).


### 6.5 Write Paths

- DynamoDB: write via repositories encapsulating conditional writes, optimistic locking, and idempotency.
- S3: write processed results to curated prefixes (`processed/`, `errors/`).
- EventBridge/SQS: publish domain events or messages for downstream flows.


### 6.6 Triggering KYC Reviews

- Batch path: daily scheduler triggers Step Functions that run domain logic to:
  - Identify customers due for review based on last review date and risk band.
  - Create KYC cases in DynamoDB and emit `KycCaseOpened` events.

- Event‑driven path: events such as `CustomerRiskScoreChanged` or `NewHighRiskCustomer` directly trigger KYC flows.


### 6.7 Handling High Volume, Spikes, and Concurrency

- Use SQS or Kinesis between ingestion and processing to absorb spikes.
- Configure Lambda reserved concurrency and SQS batch sizes.
- For extremely large jobs, use ECS Fargate with AWS Batch and EventBridge Scheduler.[3]


### 6.8 Data Consistency and Versioning

- Version domain schemas and events (e.g., `eventType` + `eventVersion`).
- Keep old consumers compatible with newer events via additive changes.
- Use soft deletes and status flags rather than hard deletes for auditability.


## 7. Detailed Data Flows and Sequence Diagrams (ASCII)

### 7.1 Daily Batch → Event-Driven Conversion (Conceptual)

```text
[EventBridge Scheduler]
        |
        v
[Step Functions: Daily KYC Review]
        |
        v
[Lambda: SelectCustomersDueForReview]
        |
        v
[DynamoDB: Customer/KYC Tables]
        |
        v
[Lambda: CreateKycCases]
        |
        v
[DynamoDB: KycCases]
        |
        v
[EventBridge: KycCaseOpened events] ---> [Downstream KYC UI, Notification, Analytics]
```

### 7.2 Inbound File → Process → Persist

```text
[Upstream System]
    |
    | SFTP/HTTPS file upload
    v
[S3 Inbound Bucket]
    |
    | S3 PUT event
    v
[Lambda: FileIngestion]
    |
    | parse & validate
    v
[Domain Services]
    |
    | write
    v
[DynamoDB Operational Tables] + [S3 Curated Zone]
    |
    | emit events
    v
[EventBridge Bus] --> [Other Consumers]
```

### 7.3 Upstream DB Ingestion → Transform → Publish Domain Events

```text
[On-Prem MS SQL]
    |
    | DMS (full + CDC)
    v
[S3 Landing Bucket]
    |
    | Glue Crawler (schema)
    v
[Glue ETL Job]
    |
    | transform
    v
[S3 Curated] / [DynamoDB]
    |
    | for changed records
    v
[Lambda: ChangeDetector]
    |
    | publish events
    v
[EventBridge Bus]
```

### 7.4 KYC Periodic Review Triggering Flow

```text
[EventBridge Scheduler]
    |
    | cron: daily at 02:00
    v
[Step Functions: PeriodicKyc]
    |
    | Task: FetchDueCustomers (Lambda)
    v
[DynamoDB]
    |
    | customers due
    v
[Lambda: GenerateKycCases]
    |
    | create cases
    v
[DynamoDB: KycCases]
    |
    | emit KycCaseOpened
    v
[EventBridge]
    |
    +--> [Notification Service]
    +--> [KYC UI Case Queue]
```

### 7.5 DynamoDB Read/Write Flows

```text
[API Gateway]
    |
    | REST/JSON
    v
[Lambda: CustomerAPI]
    |
    | via repository
    v
[DynamoDB: CustomerTable]
    |
    | Streams (changes)
    v
[Lambda: CustomerChangeHandler]
    |
    | domain events
    v
[EventBridge Bus]
```

### 7.6 Error and Retry Pathways

```text
[Lambda Processor]
    |
    | exception
    v
[Automatic Retry]
    |
    | still failing
    v
[SQS DLQ]
    |
    | CloudWatch Alarm
    v
[Ops / Support]
```

For Step Functions:

```text
[State Machine]
    |
    | Task state fails
    v
[Catch State]
    |
    | log & notify
    v
[Compensation / Rollback State] or [MarkAsFailed]
```


## 8. Migration and Decommissioning Plan

### 8.1 Phase 1 – Foundations

- Set up AWS landing zone, IAM, networking, and connectivity to on‑prem.
- Establish EventBridge, S3, DynamoDB, Step Functions, and CI/CD pipelines.
- Define initial DDD bounded contexts and hexagonal architecture standards.


### 8.2 Phase 2 – Data Migration

- Stand up DMS replication instances and endpoints to MS SQL and S3/DynamoDB.[10][8][7][6][9][11]
- Run full load into S3 (landing) and DynamoDB (where possible).
- Validate data using Glue/Athena queries and row‑count/checksum checks.
- Enable CDC to keep AWS data in sync.


### 8.3 Phase 3 – Stored Procedure to Domain Logic Migration

- Prioritize most critical KYC stored procedures.
- Implement corresponding domain services and AWS workflows (Lambda/Step Functions).
- Run in parallel with existing MS SQL procedures by dual‑writing or shadow processing.


### 8.4 Phase 4 – Autosys to EventBridge Scheduler Migration

- Inventory Autosys jobs and their schedules/dependencies.
- For each job:
  - Reimplement as EventBridge Scheduler → Lambda/Step Functions/Batch.
  - Preserve dependencies using EventBridge, Step Functions, or SQS.
- Disable corresponding Autosys jobs once AWS orchestrations are stable.


### 8.5 Phase 5 – Parallel Run

- Run MS SQL + Unix + Autosys stack in parallel with AWS stack for an agreed period.
- Compare outputs (e.g., KYC case counts, statuses, risk scores) and investigate discrepancies.
- Gradually route traffic from legacy to AWS (e.g., UI and upstream feeds).


### 8.6 Phase 6 – Cutover

- Switch all writes to AWS services; keep MS SQL read‑only for a short fallback window.
- After additional validation, stop CDC replication.
- Lock down legacy systems to avoid accidental use.


### 8.7 Phase 7 – Decommissioning

- Decommission Autosys schedules and Unix boxes.
- Decommission MS SQL databases after archiving required data to S3.
- Update DR and BCP plans for the new AWS‑native environment.


## 9. Final Architecture Recommendation

### 9.1 Recommended Architecture

- Adopt a hybrid EDA + scheduled batch architecture with Step Functions orchestration within bounded contexts and EventBridge for cross‑context events.[2][4][1]
- Use Lambda (Java) as the default compute for domain logic, with ECS Fargate/Batch reserved for heavy or long‑running jobs.[3]
- Store operational data in DynamoDB, design around access patterns, and use S3 as the data lake and document store.[7][5][6][11]
- Use DMS + Glue as the primary migration and large‑scale ETL toolchain from MS SQL to S3/DynamoDB.[8][10][5][6][9][7][11]


### 9.2 Justification

- Aligns with event‑driven and serverless best practices for microservices and KYC workflows.[2][4][1]
- Minimizes operational overhead with managed services and serverless compute.
- Provides strong observability via Step Functions, CloudWatch, and X‑Ray.[4][1][3]
- Supports incremental migration and parallel run with low risk.


### 9.3 Cost, Scalability, and Operational Excellence

- Cost
  - Serverless services (Lambda, EventBridge, DynamoDB on‑demand) ensure pay‑per‑use consumption.
  - Step Functions cost is manageable for business‑critical KYC workflows; optimize state transitions.[1]
  - Fargate/Batch used sparingly where needed to control compute expenses.[3]
- Scalability
  - All core components (Lambda, Step Functions, DynamoDB, S3, EventBridge, SQS) scale automatically with load.[5][7][11][3]
- Operational Excellence
  - Use Infrastructure as Code (CDK/CloudFormation/Terraform) for reproducible environments.
  - Standardize logging, metrics, and alerting.
  - Enforce DDD and Hexagonal Architecture guidelines in engineering standards.


### 9.4 Phased Execution Roadmap (Summary)

1. Foundations and patterns (DDD, hexagonal, observability, CI/CD).
2. Data migration with DMS/Glue and parallel data validation.
3. Domain logic migration (stored procedures → Java domain services on Lambda/Step Functions).
4. Scheduler migration (Autosys → EventBridge Scheduler/Batch).[3]
5. Parallel run, incremental cutover, and decommissioning of MS SQL, Unix, and Autosys.

This roadmap provides a controlled, evidence‑based path to a modern, AWS‑native KYC processing platform aligned with enterprise architecture best practices.