# North Star KYC Platform — Design Overview

> **Document Type:** Master Design Index & Cross-Cutting Concerns  
> **Version:** 1.0  
> **Date:** March 2026  
> **Status:** Draft  
> **Traceability:** Vision Document §1–§19

---

## 1. Purpose

This document is the master index for the North Star KYC Platform design suite. It provides:

- A glossary of key terms used across all design documents
- A cross-reference index mapping vision requirements to design documents
- A summary of cross-cutting concerns (security, observability, data lineage)
- A traceability matrix validating coverage of all north-star metrics, open questions, and dependencies

---

## 2. Design Document Index

| # | Document | Scope | Vision Sections |
|---|---|---|---|
| 00 | `00-design-overview.md` *(this file)* | Master index, glossary, traceability | §1–§19 (all) |
| 01 | `01-system-architecture.md` | Platform layers, AWS services, deployment topology | §7, §7.1, §7.2 |
| 02 | `02-orchestration-engine.md` | Workflow sequencing, task routing, event/exception routing | §8.1, §9 |
| 03 | `03-document-intelligence-agent.md` | Document classification, extraction, validation | §8.2 |
| 04 | `04-data-acquisition-agent.md` | Multi-source data ingestion, confidence scoring, ownership graphs | §8.3 |
| 05 | `05-quality-check-agent.md` | Continuous parallel QC, aggregation, STP eligibility | §8.4 |
| 06 | `06-continuous-kyc-agent.md` | Event-driven monitoring, risk re-scoring, auto-clear/escalate | §5.2, §8.5 |
| 07 | `07-audit-intelligence-agent.md` | Audit inquiry automation *(proposed design)* | §8.6 |
| 08 | `08-data-model-and-flows.md` | Core entities, data lineage, traceability, event schemas | §5.4, §6, §7 |
| 09 | `09-integration-architecture.md` | Front-office embedding, Meridian, third-party services | §5.3, §10, §12 |
| 10 | `10-exception-handling.md` | Three-tier model, context-rich views, STP rules | §9 |
| 11 | `11-security-governance-ai-controls.md` | HITL governance, explainability, AI oversight, bias detection | §5.1, §11, §16 |

---

## 3. Glossary

| Term | Definition |
|---|---|
| **AML** | Anti-Money Laundering — regulatory framework to detect and prevent financial crime |
| **BAC** | Becoming a Client — the front-office onboarding program into which KYC embeds |
| **CDD** | Customer Due Diligence — the predecessor KYC platform/initiative |
| **Confidence Score** | A numerical measure (0.0–1.0) representing the reliability of a data point based on its source |
| **Connect** | The Private Bank application framework used to host standalone KYC |
| **Continuous KYC** | Event-driven monitoring model replacing scheduled periodic reviews |
| **First-Time Right** | No rework, callbacks, or re-reviews between client, front office, and operations |
| **GFCC** | Global Financial Crimes Compliance — the compliance body that approves KYC policy |
| **HITL** | Human-in-the-Loop — governance model ensuring human oversight of AI decisions |
| **ID&V / IDV** | Identity and Verification — biometric and identity verification as a KYC prerequisite |
| **IPB** | International Private Bank — a business segment with multi-market presence |
| **JKMA** | A regional business segment now included in rollout scope |
| **KYC** | Know Your Customer — regulatory process for verifying client identity and assessing risk |
| **LOB** | Line of Business — a business segment (e.g., USB, IPB, GPM) |
| **Meridian** | The strategic client data platform — single source for demographic / party data |
| **MFE** | Micro Frontend — independent UI component for capturing specific KYC data types |
| **MLRO** | Money Laundering Reporting Officer — local compliance officer with approval authority |
| **NIGO** | Not In Good Order — submission rejected due to data/document quality issues |
| **N2N** | End-to-end (Node-to-Node) — complete KYC process from initiation to closure |
| **Orchestration Engine** | The central workflow backbone that sequences tasks, routes events, and enforces agent boundaries |
| **Party** | Any natural person or legal entity involved in a KYC case (client, beneficial owner, authorized signer, etc.) |
| **Policy Rules Engine** | The deterministic, compliance-driven engine at the core of all KYC decisions |
| **STP** | Straight-Through Processing — end-to-end automated processing with zero human touchpoints |
| **TEM** | Enterprise document storage / digitization platform |
| **Uno** | The strategic Private Bank front-office onboarding application |
| **USB** | US Bank — the first business segment for North Star rollout |

---

## 4. Cross-Cutting Concerns

### 4.1 Data Lineage & Traceability

**Applies to:** All agents, all data flows

Every data point in the platform carries:

| Attribute | Description | Example |
|---|---|---|
| `source_type` | Category of origin | `INTERNAL`, `EXTERNAL`, `ADVISOR_INPUT`, `CLIENT_UPLOAD`, `AGENT_DERIVED` |
| `source_id` | Specific system/provider identifier | `meridian`, `lexisnexis`, `sec_edgar`, `advisor_crm` |
| `confidence_score` | Reliability measure (0.0–1.0) | `0.95` (government registry) vs `0.60` (public web) |
| `timestamp` | When the data was acquired | ISO 8601 datetime |
| `agent_id` | Which agent produced/modified the data | `data-acquisition-agent-v2` |
| `audit_trail_ref` | Link to immutable audit log entry | UUID reference |

See `08-data-model-and-flows.md` §3 for the complete data lineage model.

### 4.2 Observability

**Applies to:** All platform layers

| Signal | Tool | Purpose |
|---|---|---|
| **Metrics** | Amazon CloudWatch Metrics | Agent throughput, latency, error rates, STP conversion rate |
| **Logs** | Amazon CloudWatch Logs + OpenSearch | Structured JSON logs with correlation IDs per KYC case |
| **Traces** | AWS X-Ray / OpenTelemetry | Distributed tracing across agents and orchestration steps |
| **AI Metrics** | AI Oversight Dashboard | Agent accuracy, confidence distribution, human override rate, false positive rate |

Every agent emits standardized telemetry via an `ObservabilityContract` interface (see `02-orchestration-engine.md` §5).

### 4.3 Security

**Applies to:** All platform layers

- **Encryption at rest:** AWS KMS-managed keys for all data stores
- **Encryption in transit:** TLS 1.3 for all inter-service communication
- **Authentication:** OAuth 2.0 / OIDC for user-facing apps; IAM roles for service-to-service
- **Authorization:** RBAC with persona-based entitlements (advisor, operations, controls, audit)
- **Data classification:** PII/sensitive data tagged and handled per data classification policy
- **Secrets management:** AWS Secrets Manager for all credentials and API keys

See `11-security-governance-ai-controls.md` for full security design.

### 4.4 Error Handling Strategy

**Applies to:** All agents, orchestration engine

| Error Class | Strategy | Example |
|---|---|---|
| **Transient** | Retry with exponential backoff (max 3 attempts) | Network timeout to external registry |
| **Data Quality** | Route to QC Agent for validation failure handling | Mismatched dates between document and registry |
| **Agent Failure** | Circuit breaker + fallback to human routing | Document extraction model unavailable |
| **Policy Violation** | Hard stop + audit log + human escalation | Agent attempts action outside defined scope |

---

## 5. North-Star Metrics — Design Traceability

| Metric | Target | Design Support | Primary Documents |
|---|---|---|---|
| **First-Time Right** | 100% | QC Agent continuous checks + Document Intelligence pre-validation + Data Acquisition pre-fill + context-rich exception views | `05`, `03`, `04`, `10` |
| **Zero NIGOs** | 0 | Document Intelligence classification & extraction + QC validation before acceptance + real-time advisor feedback | `03`, `05`, `10` |
| **100% STP** (low/medium risk) | 100% | Exception Handling Tier 0 + STP rules framework + Orchestration Engine auto-approval routing | `10`, `02`, `05` |
| **Data Collected Once** | Enforced | Data Model single source of truth + party deduplication + Meridian real-time integration | `08`, `09` |
| **90% Reduction in Periodic Reviews** | ~90% | Continuous KYC Agent event-driven model + configurable auto-clear rules + risk re-scoring | `06`, `02` |

---

## 6. Open Questions Traceability (Vision §19)

| # | Open Question | Design Status | Document | Notes |
|---|---|---|---|---|
| 1 | GFCC policy for zero-event clients under continuous KYC | **Deferred** — design provides configurable thresholds pending GFCC guidance | `06` §6.1 | Configurable `max_quiescent_period` parameter |
| 2 | Approved set of third-party data sources with confidence scoring | **Partially addressed** — confidence scoring methodology proposed; source list pending GFCC approval | `04` §4, `08` §3 | Framework ready; sources pluggable |
| 3 | Target operations tool of choice | **Deferred** — integration architecture designed for tool-agnostic embedding | `09` §2 | Standalone Connect app serves as fallback |
| 4 | Meridian branch consolidation timeline | **Addressed architecturally** — design supports both consolidated and per-branch Meridian integration | `09` §3 | Adapter pattern for forward compatibility |
| 5 | IDV/ID&V vendor contract (MSA) | **Deferred** — integration design uses abstract IDV interface with pluggable provider | `09` §4 | Contract/interface defined; provider swappable |
| 6 | Formal STP rules and pre-approval from Controls/GFCC | **Partially addressed** — STP rules framework designed; specific rules pending GFCC ratification | `10` §4 | Rules engine is configuration-driven |
| 7 | Shared client strategy for cross-region rollouts | **Deferred** — data model supports multi-region party linking; routing rules pending | `08` §2.3 | Multi-region party resolution designed |
| 8 | Maintenance event subscription model | **Addressed** — event bus design includes maintenance event channels | `09` §5, `06` §3 | EventBridge subscription patterns defined |
| 9 | File review process absorption into N2N KYC | **Deferred** — Phase 4 research item | — | Outside current design scope |
| 10 | Data confidence scoring methodology | **Addressed** — proposed methodology with source-tier model | `04` §4, `08` §3 | Three-tier source reliability framework |

---

## 7. Dependencies Traceability (Vision §15)

| Dependency | Design Coverage | Document |
|---|---|---|
| Becoming a Client (BAC) Program | Front-office embedding architecture | `09` §2 |
| Strategic Client Information (Meridian) | Read-only/write-back integration design | `09` §3 |
| Strategic Global Parties | Party data model with relationship graph | `08` §2 |
| Document Digitization / TEM Storage | Document Intelligence Agent storage interface | `03` §5 |
| GFCC Policy Guidance — Continuous KYC | Configurable policy parameters in Continuous KYC Agent | `06` §6 |
| Third-Party Data Source Approvals | Pluggable data source framework in Data Acquisition Agent | `04` §3 |
| IDV/ID&V Vendor Contract | Abstract IDV provider interface | `09` §4 |
| Operating Model Design | Persona-based exception routing model | `10` §3 |
| Uniform / Regulatory Change (IPB) | Jurisdiction-configurable policy layer | `01` §4, `02` §3 |
| UNIF / AUP Regulatory Changes | Annual policy refresh mechanism | `02` §3 |
| STP Rules Definition | Configuration-driven STP rules framework | `10` §4 |
| Agentic Model & HITL Policy | Progressive autonomy governance model | `11` §2–§4 |

---

## 8. Design Conventions

### 8.1 Diagram Notation
- All diagrams use **Mermaid** syntax for native markdown rendering
- Architecture diagrams: `graph TD` (top-down) or `graph LR` (left-right)
- Sequence diagrams: `sequenceDiagram`
- Entity-relationship diagrams: `erDiagram`
- State diagrams: `stateDiagram-v2`

### 8.2 Document Structure
Every design document follows this structure:
1. **Purpose & Scope** — What this document covers and its boundaries
2. **Requirements Addressed** — Traceable references to vision sections
3. **Design** — Architecture, components, flows with Mermaid diagrams
4. **Interfaces & Contracts** — APIs, events, data contracts
5. **Assumptions & Constraints** — What we assume to be true; hard constraints
6. **Open Items** — Unresolved questions specific to this design area

### 8.3 Naming Conventions
- Agent names: `kebab-case` (e.g., `document-intelligence-agent`)
- Event names: `domain.entity.action` (e.g., `kyc.case.created`, `screening.result.updated`)
- API endpoints: REST conventions with `/api/v1/` prefix
- Data fields: `snake_case`

---

## 9. Phase Mapping

| Design Document | Build Phase(s) | Rollout Dependency |
|---|---|---|
| `01` System Architecture | Phase 1 (foundation) | All regions |
| `02` Orchestration Engine | Phase 1 | All regions |
| `03` Document Intelligence | Phase 1 | All regions |
| `04` Data Acquisition | Phase 1 (basic) → Phase 2 (entity expansion) | Incremental by source approval |
| `05` Quality Check | Phase 1 | All regions |
| `06` Continuous KYC | Phase 2 (foundation) → Phase 3 (advanced) | After periodic review cutover |
| `07` Audit Intelligence | Phase 3–4 (concept) | Independent of region rollout |
| `08` Data Model | Phase 1 (foundation) | All regions |
| `09` Integration Architecture | Phase 1 (core) → Phase 2 (entity) | Per-region integration specifics |
| `10` Exception Handling | Phase 1 | All regions |
| `11` Security & Governance | Phase 1 (foundation) | All regions |

---

*This is a living document. It will be updated as individual design documents are finalized and as open questions are resolved.*
