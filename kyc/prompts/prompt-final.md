Here is the **updated final prompt**, now incorporating **all** your new constraints:

* Domain has **2500+ fields**, but processing uses **~150 fields**
* Need **scheduled processing** for **300k+ KYCs/day**
* System is **multi-region active-active**
* React UI drives CRUD + workflow
* DynamoDB + Data Lake both are sources
* Future: external APIs pulling from Data Lake
* Team skillset = **Java** (not Python)
* Need decision-making on **DynamoDB vs Relational DB** for processing
* Need deep analysis of how to query DynamoDB efficiently
* Must think about partition keys, GSIs, projections, read patterns
* Must be comprehensive and not miss anything

Below is the **final prompt** you can use.

---

# **Final Prompt (All Constraints Included)**

**Act as a principal engineer, AWS solutions architect, event-driven architecture expert, DDD specialist, and senior prompt engineer. Produce an extremely deep, complete architecture and design document for the following scenario:**

---

## **Context**

We are rewriting our entire KYC ecosystem in AWS. Current system uses:

* Unix-based Autosys batch jobs
* Python-based data processing
* MS SQL Server for reading/writing
* Stored procedures for business rules
* Daily jobs that process new inbound data
* Periodic reviews and KYC triggers

---

## **Target State**

We are moving to an AWS-native, hybrid serverless + EDA architecture, using:

* API Gateway
* Lambda (Java-based runtime preferred)
* DynamoDB
* S3
* EventBridge
* Step Functions
* DDD + Hexagonal architecture
* Multi-region **active-active**
* Data Lake + Data Mesh

**Important Changes:**

* MS SQL Server will be **fully migrated to AWS**
* All processing will run **inside AWS**
* On-prem access only if exceptional
* **Java** will be the primary language (team skills)
* Domain has **2500+ fields**, but processing needs **150 fields**
* Daily volume = **300k+ KYCs**
* DynamoDB stores UI workflow state and KYC data
* Data is also pushed to the **Data Lake/Data Mesh**
* React UI performs CRUD + workflow transitions
* External consumers must read via **APIs backed by the Data Lake**

---

## **Your Task**

Produce a complete, extremely detailed design document covering **all** aspects below.

---

## **1. DynamoDB vs Relational DB Decision Framework**

Provide a rigorous evaluation:

* Should daily batch processing of **300k KYCs** run directly from DynamoDB?
* When DynamoDB is appropriate vs when to introduce a relational DB (Aurora, RDS)
* Trade-offs:

  * Query patterns
  * Scan costs
  * Read amplification
  * Hot partitions
  * Consistency model
  * Capacity planning (RCU/WCU)
  * Global Tables for active-active
* Whether to maintain a **processing-optimized relational replica**
* Whether to maintain a **DynamoDB projection table** with only required 150 fields
* Whether to adopt **single-table design vs multiple tables**

Give a **clear final recommendation** with justification.

---

## **2. Designing Queries Against DynamoDB for Scheduled Processing**

Explain exactly how to efficiently fetch 300k+ records daily for processing:

* PK/SK design
* GSI design
* Sparse indexes
* Attribute projection (only required 150 fields)
* Pagination
* Time window–based partitions
* Handling large scans efficiently
* Cross-region replication via Global Tables
* Strategies to avoid full scans (TTL-based partitioning, review-due indexes, event markers)

Provide the best pattern(s) for scheduled processing.

---

## **3. Architecture Options for Daily + Event-Driven Processing**

Design and compare:

### **A. Read from DynamoDB directly**

### **B. Extract DynamoDB → S3 (columnar) → Process**

### **C. Maintain an Aurora/Postgres replica for batch reads**

### **D. Event-sourced KYC pipeline with materialized views**

### **E. Dual-write into DynamoDB + S3**

### **F. Data Lake–driven processing using Glue / EMR / Lambda**

For each, provide:

* Why
* What
* How
* Pros & Cons
* Cost model
* Performance & scalability
* Multi-region behavior
* Suitability for 300k+ daily KYCs

---

## **4. Full End-to-End Target Architecture**

Design all components:

* Event-driven ingestion flows
* Daily scheduled pipelines
* Step Functions orchestration patterns
* S3-based batch pipelines
* Domain services (Java / Lambda / containers)
* Multi-region replication
* Data consistency strategy
* Read/write segregation
* Real-time KYC workflow processing
* React UI CRUD against DynamoDB
* Data Lake integration (Glue Catalog, Parquet, versioning)
* Publishing data to Data Mesh
* API layer for external consumers (Data Lake-backed APIs)

Include diagrams via structured ASCII or stepwise sequences.

---

## **5. Business Rules Rewrite Strategy (Stored Procedures → Domain Logic)**

Explain how to:

* Extract and refactor SP logic
* Map to domain services (DDD + Hexagonal)
* Implement as Java Lambdas or containerized microservices
* Implement workflow transitions using Step Functions
* Ensure idempotency
* Maintain correctness with 2500 fields but process only 150
* Introduce rule versioning
* Support perpetual KYC
* Validate transformations

---

## **6. Data Flow/Sequence Designs**

Provide complete sequences for:

* Daily KYC processing
* File ingestion flows
* DynamoDB → S3 projection
* External consumer API requests
* Multi-region conflict resolution
* S3 → Lambda → Dynamo pipelines
* Step Functions workflow transitions
* Periodic review KYC triggers

---

## **7. Multi-Region Active-Active (Global Scale Architecture)**

Cover:

* DynamoDB Global Tables
* Idempotent writes
* Conflict resolution
* Multi-region Step Functions
* Multi-region EventBridge
* S3 cross-region replication
* Latency considerations
* Regional failover
* Consistent data processing across regions
* Global API design strategy

---

## **8. Development & Operational Excellence**

Describe:

* CICD pipelines
* Code structure for Java Lambdas
* Monitoring dashboards
* CloudWatch/X-Ray integration
* Structured logs
* Retry & DLQ strategy
* Testing strategy

  * Unit
  * Integration
  * Contract
  * Replay-based testing for KYC logic

---

## **9. Migration Plan**

Detail:

* MS SQL → AWS migration
* MS SQL stored procedures rewrite
* Autosys → EventBridge Scheduler migration
* Python → Java rewrite
* Data replication strategy
* Parallel run
* Cutover
* Decommissioning

---

## **10. Final Recommendation**

Produce:

* Final architecture
* Justification
* Performance expectations
* Cost estimate
* Multi-region design summary
* Future-proofing approach (perpetual KYC + Data Lake APIs + EDA)

---

## **Formatting Requirements**

* Use clear structure
* Include tables, diagrams, comparisons
* No verbosity—depth without fluff
* Comprehensive, precise, engineering-grade

---

**Generate the final architecture and design document now.**
