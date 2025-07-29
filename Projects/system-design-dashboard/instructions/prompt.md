Act as prompt Engineer and the generate prompt for  "a Vibe coding project named System Design Dashboard. This application will generate System design based on user inputs. User inputs can be Free Form or Intake form with params like DAU, MAU, QPS, Data Size estimates, Traffic Patterns, READ/Write ratio, Geographic distribution, Peak multiplier etc etc and all params you can think of needed for deciding system design of given system either predefined like Netflix, Facebook, Twitter, Youtube, Instagram, AirBnb, Uber, Youtube, Google Maps etc etc or user provided for a new System based on 1 liner system goal or definition. Application will analyze and generate system design based on either Pre defined rules in the application or Gen AI LLM based integration to generate System design for the inputs. Now either Rule Engine or LLM will generate Architecture diagram with MErmaid chart format and display on UI. Content will be generated based on following format. """then output a complete end‑to‑end system design following this structured framework:

🔍 Problem Definition & Goals
• Describe what the system does.
• List functional & non‑functional requirements.
• State scale assumptions (users/day, req/sec, data volume).
• Define constraints, SLAs, latency & availability targets.

🧱 High‑Level Architecture
• Identify key components/modules and their interactions.
• Provide a high‑level diagram (textual or ASCII).
• Explain sync vs async flows.
• Trace User → Frontend → Backend → Data flow.

🧠 Component Design
For each major module, detail:
• Web/API layer.
• Backend services (microservice decomposition if any).
• AuthN/AuthZ.
• File/object storage.
• Business‑logic orchestration.

🗃️ Data Modeling & Storage
• Define data entities & relationships.
• Choose databases (SQL/NoSQL/time‑series/graph).
• Show schema examples.
• Outline caching strategy (Redis/Memcached).

📡 APIs & Communication
• List internal/external APIs (REST/gRPC/WebSockets).
• Explain API gateway usage.
• Give example request/response (JSON/YAML).

🕸️ Scalability & Performance
• Plan load balancing, horizontal scaling, sharding.
• Design async pipelines with message queues (Kafka/SQS).
• Include rate‑limiting, throttling, pagination approaches.

🔐 Security
• Detail auth (OAuth, JWT, SSO).
• Detail authorization (RBAC/ABAC).
• Cover data encryption, secrets management, OWASP concerns.

📈 Observability & Monitoring
• Define logging strategy.
• Specify metrics collection (Prometheus/Grafana).
• Propose alerts & dashboard layouts.

🧪 Testing & CI/CD
• Outline unit, integration, contract, load tests.
• Sketch CI/CD pipeline (tools, stages, branching model).

☁️ Infrastructure & Deployment
• Choose cloud vs on‑prem vs hybrid.
• Describe containerization (Docker) & orchestration (K8s).
• Show infra‑as‑code approach (Terraform/CDK).
• Define environments (dev/stage/prod).

🧩 Trade‑Offs & Alternatives
• For each major decision, list pros/cons.
• Suggest alternative tech stacks or patterns.
• Note how design would shift at higher or lower scale. """  Application stack will be ReactJS + TailwindCSS + Local DB and SupaBase for production. NextJS or NodeJS backend for all logic. User Authentication / Preference and Library or Dasboard to display and load previously Generated System  Designs. Library sections for pre defined and pre generated System design for popular systems those will be generated and stored in DB and loaded in the application. User can also select System design from library and Edit (Generate a new version for USer's edited copy) feature for USer to modify and store in Users's generate system designs.  Also Generate Copilot instrcutions and feature-<>.md files for all features. 
