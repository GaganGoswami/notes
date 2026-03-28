# GitHub Copilot Agent — Java + AWS Architecture Design Review

**Agent initialized and ready.** I am operating as your autonomous architecture review specialist, aligned to the [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)  across all six pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, and Sustainability. [docs.aws.amazon](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)

***

## 🔍 Active Review Capabilities

**Java + Spring Analysis**
- Layered/hexagonal architecture violations, God classes, circular dependencies
- Data-access anti-patterns (N+1 queries, missing connection pool tuning, improper transaction boundaries)
- Concurrency, thread-safety, and memory leak detection
- Resilience gaps (missing circuit breakers, improper retry logic, no bulkhead patterns)

**AWS Service Design Review**
- **Lambda**: Cold start risks, VPC misconfiguration, missing idempotency, payload inefficiencies
- **ECS/Fargate**: Incorrect CPU/memory sizing, autoscaling gaps, container build anti-patterns
- **Step Functions**: Missing error/retry states, polling anti-patterns, runaway executions
- **SQS/SNS/EventBridge**: Missing DLQs, incorrect visibility timeouts, fanout misconfiguration
- **API Gateway**: Missing throttling, no request validation, poor integration types
- **DynamoDB/Aurora/RDS**: Hot partitions, missing GSIs, wrong consistency models 

**Repository & IaC Analysis**
- Terraform/CDK/CloudFormation misconfiguration detection
- Dockerfile best practices validation
- CI/CD pipeline quality (missing test gates, no artifact versioning, no cache layers) 
- Package structure, outdated dependencies, incomplete logging strategy

***

## 📋 Issue Format I Will Use

For every finding, I deliver:

| Field | Description |
|---|---|
| 🔴 **Issue** | What is wrong, with exact code/config reference |
| 💥 **Impact** | Production consequence (data loss, latency, cost blowout, security breach) |
| ✅ **Solution** | Step-by-step remediation with code/config rewrite |
| 📚 **Reference** | AWS Well-Architected pillar + pattern name |

***

## 🚀 Future Design Mode

When you need proactive guidance, I will generate:
- Text-based architecture diagrams (component + sequence flows)
- AWS service selection with trade-off matrices
- Sample Spring Boot + AWS SDK implementations
- Deployment topology recommendations with IaC snippets 

***

## 🧠 Context Memory

Throughout this session I will maintain awareness of:
- Every component analyzed and issues already flagged
- Inferred architectural intent from your codebase patterns
- Cross-cutting concerns (e.g., if auth logic leaks across services)

***

**Which repository or code component should I analyze first?**

You can share any of the following to get started:
- Java/Spring source files or package structure
- `application.yml` / `application.properties`
- Step Functions state machine JSON
- AWS Lambda handler code
- A description of your current architecture