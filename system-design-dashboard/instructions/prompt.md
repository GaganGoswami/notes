You are a senior system‑design engineer and architect. I’m building a “System Design Dashboard” in ReactJS + TailwindCSS with an in‑memory database. The dashboard will collect basic info about any use‑case (e.g., Netflix, Twitter, Facebook, Uber), including parameters like number of users, QPS, data size, etc., and then output a complete end‑to‑end system design following this structured framework:
  
1. 🔍 Problem Definition & Goals  
   • Describe what the system does.  
   • List functional & non‑functional requirements.  
   • State scale assumptions (users/day, req/sec, data volume).  
   • Define constraints, SLAs, latency & availability targets.  

2. 🧱 High‑Level Architecture  
   • Identify key components/modules and their interactions.  
   • Provide a high‑level diagram (textual or ASCII).  
   • Explain sync vs async flows.  
   • Trace User → Frontend → Backend → Data flow.  

3. 🧠 Component Design  
   For each major module, detail:  
   • Web/API layer.  
   • Backend services (microservice decomposition if any).  
   • AuthN/AuthZ.  
   • File/object storage.  
   • Business‑logic orchestration.  

4. 🗃️ Data Modeling & Storage  
   • Define data entities & relationships.  
   • Choose databases (SQL/NoSQL/time‑series/graph).  
   • Show schema examples.  
   • Outline caching strategy (Redis/Memcached).  

5. 📡 APIs & Communication  
   • List internal/external APIs (REST/gRPC/WebSockets).  
   • Explain API gateway usage.  
   • Give example request/response (JSON/YAML).  

6. 🕸️ Scalability & Performance  
   • Plan load balancing, horizontal scaling, sharding.  
   • Design async pipelines with message queues (Kafka/SQS).  
   • Include rate‑limiting, throttling, pagination approaches.  

7. 🔐 Security  
   • Detail auth (OAuth, JWT, SSO).  
   • Detail authorization (RBAC/ABAC).  
   • Cover data encryption, secrets management, OWASP concerns.  

8. 📈 Observability & Monitoring  
   • Define logging strategy.  
   • Specify metrics collection (Prometheus/Grafana).  
   • Propose alerts & dashboard layouts.  

9. 🧪 Testing & CI/CD  
   • Outline unit, integration, contract, load tests.  
   • Sketch CI/CD pipeline (tools, stages, branching model).  

10. ☁️ Infrastructure & Deployment  
    • Choose cloud vs on‑prem vs hybrid.  
    • Describe containerization (Docker) & orchestration (K8s).  
    • Show infra‑as‑code approach (Terraform/CDK).  
    • Define environments (dev/stage/prod).  

11. 🧩 Trade‑Offs & Alternatives  
    • For each major decision, list pros/cons.  
    • Suggest alternative tech stacks or patterns.  
    • Note how design would shift at higher or lower scale.  

Produce your answer as a well‑structured markdown document, with clear headings for each section and any diagrams rendered in ASCII or simple descriptive form.
