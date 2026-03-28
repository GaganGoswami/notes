Act as seasoned prompt Engineer and principle engineer and software design expert and AWS solutions architect then generate prompt for  "Currently in unix boxes I have python jobs those are currently called via Autosys jobs. These python jobs process data from Files, reading MS SQL DB and storing in MS SQL database after processing.  Currently the batch jobs  run daily some jobs pull data from up stream servers either reading data from DB or processing files and after processing storing in MS SQL DB. Some jobs trigger daily and process tasks Periodic Reviews again core data and process and launch KYC. Think really really hard and deeply and we need to design batch processing to Event driven and AWS side data processing during re write. So idea is when designing new solution for data processing some things will be handled using EDA and some with daily scheduler based processing. <Current-core-app-rewrite>on other side note we are already re writing our core apps for AWS platform using serverless architecture using API Gateway, Lambda, dynamoDB, S3, Event Bridge by following Hexagonal architecture and Domain driven design. Also some features are using Event Driven architecture. so it is hybrid solution. </Current-core-app-rewrite> Considering core App architecture and tech stack in mind as core app is currently writing in Dynamo DB. We also need to read data from Dynamo DB for this processing. Suggest all kinds of popular architectures, Pros and Cons for each design. Why , What and how methodology. This must be comprehensive design document considering all aspects. Dataprocessing also have business rules written in Stored procs that also needs to be considered and rewrite in AWS rewrite. Team has java skillsets not python so consider that also. consider few Things.. Domain objects have 2500+ fields but for data processing we do not need all fields consider 150+ fields. How data will be queried from DynamoDB for scheduled based processing? Do we should be using Relational DB or Dynamo for this processing? consider daily we need to procees 300k + KYCs"

Here is the updated and fully optimized prompt, now incorporating your requirement that MS SQL data will be migrated into AWS, and all processing will run in AWS, with on-prem access only when unavoidable.
This is the final prompt you can copy-paste into Copilot/LLM.
Final Prompt (Updated With MS SQL Migration Requirement)
Act as a principal engineer, senior AWS solutions architect, data engineering expert, and prompt-engineering specialist. Create a complete, deeply detailed, end-to-end architecture and design document for the following scenario:
Context
Current workloads run on Unix boxes where Python jobs are triggered by Autosys.
These jobs:
Read files
Read/write from MS SQL Server
Apply business rules implemented in Stored Procedures
Write processed results into MS SQL Server
Some jobs pull upstream data daily
Some jobs execute Periodic Reviews and trigger downstream KYC workflows
Target State (Mandatory)
We are moving toward a fully AWS-based architecture with:
API Gateway, Lambda, DynamoDB, S3
EventBridge, Step Functions
Hexagonal Architecture
Domain Driven Design
Event-Driven Architecture (where useful)
Some tasks will still rely on daily scheduler-based execution
We also need to read/write from DynamoDB for new processing flows
Key Update:
MS SQL data will be migrated into AWS.
All core processing will run in AWS.
On-prem access will occur only if unavoidable for specific upstream dependencies.
Long-term goal is to decommission MS SQL + Unix + Autosys fully. Consider java in place of python as team have skill set in java not in python. 
Your Task
Produce a comprehensive, enterprise-grade architecture and design document that explains:
1. Architecture Options (Compare All)
Provide all relevant patterns, with deep analysis of Why, What, and How:
Full Event-Driven Architecture
Hybrid EDA + Scheduled Batch Processing
Step Functions–driven orchestration pipelines
S3 event ingestion → Lambda → DynamoDB
Lambda + SQS pipelines
Kinesis streaming (if needed for continuous ingestion)
Glue (ETL or ELT) + Glue Workflows
DynamoDB streams for reactive domain events
ECS Fargate batch containers (only if required)
AWS Data Migration Architecture for MS SQL → AWS (DMS, Glue, custom ingestion)



For each architecture, include:
Why
What
How
Pros
Cons
Cost model
Scaling behavior
Observability model
When it should OR should not be used
2. MS SQL → AWS Data Migration Design
Cover in detail:
Data migration strategy using DMS, Glue, custom ingestion, or one-time bulk load
Migration of transactional tables
Migration of historical data
Migration of large objects/files if stored in DB
Validation strategy
Cutover strategy (blue/green or parallel)
Data lineage & cataloging in AWS
3. Rewrite of Business Rules
Stored procedures must be fully rewritten into AWS-native processing logic.
Provide a detailed strategy:
Extract → modularize → map to domain → implement in:
Lambda functions
Domain services (Hexagonal + DDD)
Step Functions Choice logic
Custom AWS rule engine (if appropriate)
How to externalize configuration vs logic
Patterns to handle 3000+ domain fields and complex enrichments
Idempotency, ordering, consistency handling
Refactoring logic to avoid procedural anti-patterns
4. End-to-End Target Architecture Proposal
Design a complete AWS-native solution:
EventBridge-driven ingestion
S3-triggered file pipelines
Step Functions orchestration for multi-step workflows
Lambda-based transformation layer
DynamoDB CRUD patterns (GSI, LSI, single-table design if appropriate)
Upstream integration with on-prem (only when required)
Daily scheduled jobs replaced by EventBridge Scheduler
Domain services following Hexagonal architecture
Error handling, retries, DLQ
Security, IAM boundaries, encryption
Observability: X-Ray, CloudWatch dashboards, structured JSON logs
5. Data Processing Framework Design
Explain how the new AWS batch/EDA runtime will:
Read from S3, DynamoDB, external on-prem DB (only when required)
Apply business rules via domain modules
Produce output to DynamoDB, S3, or other domain stores
Trigger KYC reviews
Handle high volume, spikes, concurrency
Ensure data consistency and versioning
6. Detailed Data Flow & Sequence Diagrams
Provide diagrams (ASCII/structured), explaining:
Daily batch → event-driven conversion
Inbound file → process → persist
Upstream DB ingestion → transform → publish domain events
KYC periodic review triggering flow
DynamoDB read/write flows
Error and retry pathways
7. Migration and Decommissioning Plan
Provide detailed phases:
Build new AWS batch/EDA pipeline
Migrate MS SQL → AWS
Migrate stored procedures → domain logic
Migrate Autosys → EventBridge Scheduler
Parallel run
Cutover
Decommission MS SQL, Autosys, Unix
8. Final Architecture Recommendation
After evaluating all options, produce:
A recommended architecture
Justification
Cost, scalability, operational excellence view
Phased execution roadmap
Formatting Requirements
Deep, concise, technical content
Use tables, diagrams, comparisons
Enterprise-level rigor
No filler content
Generate the complete architecture and design document now.