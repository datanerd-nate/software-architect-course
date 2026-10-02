# Architectural Submission: Making the Grade Platform
**System Architecture & Technical Design Deliverable**

---

## 1. Final Architecture Diagram

### End-to-End Choreographed Anthology Saga

```mermaid
flowchart TD
    subgraph Client ["Student & Admin UI Layer"]
        STUI["Student Test Interface"]
        ADUI["Admin & Teacher Review Portal"]
    end

    subgraph Ingestion ["1. Answer Intake Domain"]
        STS["Student Testing Service"]
        AIS["Answer Intake Service"]
        AIDB[("Accepted Answer DB\n(PostgreSQL)")]
        AIOB["Outbox Relay"]
    end

    subgraph EventBus ["Asynchronous Event Broker (Kafka / RabbitMQ)"]
        T_ACCEPTED[["topic: answers.accepted"]]
        T_GRADED[["topic: answers.graded"]]
        T_REVIEW[["topic: grading.review-required"]]
        T_VALIDATED[["topic: grades.validated"]]
        T_FAILED[["topic: grading.failed"]]
    end

    subgraph Grading ["2. Grading Domain"]
        GS["Grading Service Worker Pool"]
        MCP["MCP AI Service (LLM Engine)"]
        GSDB[("Evaluations DB\n(PostgreSQL / Redis Key Store)")]
        GSOB["Outbox Relay"]
    end

    subgraph AdminDomain ["3. Administrative & HITL Domain"]
        ADS["Administrative Service"]
        TRQ["Teacher Review Queue"]
        ADDB[("Admin & Audit DB")]
        ADOB["Outbox Relay"]
    end

    subgraph Consolidation ["4. Result Consolidation Domain"]
        RCS["Result Consolidation Service"]
        PGB["PgBouncer Pooler"]
        RCDB[("Consolidated Transcript DB\n(PostgreSQL Cluster)")]
    end

    %% Ingestion Flow
    STUI -->|HTTP 202 Async| STS
    STS --> AIS
    AIS -->|Local ACID Tx| AIDB
    AIDB --> AIOB
    AIOB -->|Publish| T_ACCEPTED

    %% Grading Flow
    T_ACCEPTED -->|Consume| GS
    GS -->|Deterministic Check| GSDB
    GS -->|Short Answer Payload| MCP
    MCP -- Score & Confidence --> GS
    GS -->|Local ACID Tx| GSDB
    GSDB --> GSOB

    %% Grading Routing
    GSOB -->|Auto-Approved or MC| T_GRADED
    GSOB -->|Confidence < 0.85 OR Roll <= X%| T_REVIEW
    GSOB -->|Corrupt / Missing Key| T_FAILED

    %% Admin & HITL Flow
    T_REVIEW -->|Consume| ADS
    ADS --> TRQ
    ADUI -->|Validate / Override| TRQ
    TRQ -->|Local ACID Tx| ADDB
    ADDB --> ADOB
    ADOB -->|Publish| T_VALIDATED

    T_FAILED -->|Compensate| AIS

    %% Consolidation Flow
    T_GRADED -->|Consume| RCS
    T_VALIDATED -->|Consume| RCS
    RCS --> PGB
    PGB -->|Batch Upsert| RCDB