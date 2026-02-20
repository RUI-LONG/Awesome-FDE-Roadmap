# Enterprise RAG Architecture Template (GCP)

## 1. Speed-First Pattern

Use Cases: Customer-Facing Chatbot, Vision-Based Product Search

### 🎯 Core Objectives

- 🚀 Speed must be fast (low latency first)
- 💰 Cost must be low (optimize batch + cache)
- 🛡 Must include Guardrail & Answer Verification

```mermaid

graph TB

subgraph "Index Pipeline"

    Src[(OnPrem DB)]

    Src -.->|Interconnect | GCS[Cloud Storage<br/>Image & Thumbnail]

    GCS --> BatchEmbed[Vertex AI Batch Prediction]

    BatchEmbed --> IngestJob[Cloud Run / Dataflow]
    IngestJob --> VecStore[(Vertex AI Vector Search)]


    Src --> MetaStore[(Firestore <br/> Product Metadata)]

end


subgraph "Query Pipeline"

    User((Web / App))

    User --> GLB[Cloud Load Balancing]

    GLB --> API[Cloud Run + Vision AI API]

    API --> Cache[(Memorystore Redis <br/>
    Query + Result Cache)]

    Cache -->|Cache Hit| API

    Cache -->|Cache Miss| EmbedOnline[Vertex AI Online Prediction]

    EmbedOnline --> ANN[(Vertex AI Vector Search)]

    ANN --> Filter[(Firestore Metadata Filter)]

    Filter --> Guardrail[Vertex AI Guardrail & Content Filter]

    Guardrail --> Verify[Vertex AI LLM Generation + Vertex AI Grounding]

    Verify --> API

    API --> Thumb[Cloud Storage <br/> Image & Thumbnail]

    API --> User

end


%% =========================
%% ========== OBSERVABILITY ==========
%% =========================
API -.-> Mon[Cloud Monitoring]
API -.-> Log[Cloud Logging]
```

---

## 2. Accuracy-First Pattern

Use Cases: AI Document Assistant, Co-Pilot (Complex Documents)

### 🎯 Core Objectives

- 🎯 Accuracy first with traceable answers and minimal legal risk
- 📚 Handle complex documents, multiple data source and long context reliably
- 🛡 Strong guardrails, citations, human review, and interpretability

```mermaid
graph TB

subgraph "Index Pipeline"
direction TB

    Src[(OnPrem DB / SharePoint / Drive)]
    Src -.->|Interconnect / VPN| GCS[Cloud Storage<br/>Raw Documents]
    GCS --> DocAI[Document AI<br/>OCR + Layout Parsing]
    DocAI --> Chunk[Dataflow<br/>Chunking + Tagging]
    Chunk --> BatchEmbed[Vertex AI Batch Embedding]
    BatchEmbed --> VecStore[(Vertex AI Vector Search)]
    Chunk --> KeywordIndex[Vertex AI Search<br/>Keyword Index]
    Chunk --> MetaStore[(Firestore<br/>File Metadata)]

end

%% =========================
%% ========== QUERY PIPELINE ==========
%% =========================
subgraph "Query Pipeline"
direction TB

    User((Web / Copilot UI))
    User --> Agent[Google ADK Agent Engine]
    Agent --> Hybrid[Cloud Run<br/>Hybrid Retrieval Service]
    Hybrid --> OnlineEmbed[Vertex AI Online Embedding]
    OnlineEmbed --> ANN[(Vertex AI Vector Search)]
    Hybrid --> Keyword[Vertex AI Search]
    ANN --> Combine[Combine Results<br/>]
    Keyword --> Combine
    Combine --> MetaFilter[(Firestore<br/>Metadata Filter)]
    MetaFilter --> Rerank[Cross-Encoder Rerank<br/>Vertex AI Endpoint]
    Rerank --> LLM[Vertex AI LLM <br/>Generation + Grounding]
    LLM --> ContentFilter[Vertex AI Content Filter]
    ContentFilter -->|High risk / Low confidence| Human[Pub/Sub + Cloud Functions<br/>Human Review]
    Human -->|Approved / Edited| Agent
    ContentFilter -->|OK| Agent
    Agent --> User

end

```

## 3. Cost-First Pattern

Use Cases: Offline Report Generation, Document Processing Automation

### 🎯 Core Objectives

- 💸 Predictable monthly spend and enforce budget caps
- ⏱ Latency-insensitive execution (minutes to hours is acceptable)
- 🚀 Maximize throughput (batch efficiency, parallelism, and pipeline utilization)
- 🧾 Stable, repeatable outputs for reporting, audits, and operations


```mermaid
graph TB

%% =========================
%% ========== INDEX PIPELINE ==========
%% =========================
subgraph "Processing Pipeline"
direction TB

    Src[(OnPrem DB / SaaS / Files)]
    Src -.->|Interconnect / VPN| GCS[Cloud Storage<br/>Raw Data]

    GCS --> Ingest[Dataflow<br/>Batch ETL + Validation]

    Ingest --> BatchLLM[Vertex AI Batch Prediction<br/>Summarization / Classification]

    BatchLLM --> Report[(BigQuery<br/>Report Tables)]
    BatchLLM --> Artifact[Cloud Storage<br/>Generated Files]

end

%% =========================
%% ========== REPORT ACCESS PIPELINE ==========
%% =========================
subgraph "Access Layer"
direction TB

    User((Internal Tool))
    User --> Report

end

%% =========================
%% ========== OBSERVABILITY ==========
%% =========================
Ingest -.-> Mon[Cloud Monitoring]
Ingest -.-> Log[Cloud Logging]

BatchLLM -.-> Mon
BatchLLM -.-> Log
```

## 4. Compliance-First Pattern

Use Cases: Healthcare AI Document Assistant

### 🎯 Core Objectives

- 🔒 Data stays in-domain (region / tenant / environment) with strict residency controls
- 🧩 Least privilege access with full auditability (who accessed what, - when, why)
- 🏢 Strong multi-tenant isolation and fine-grained enforcement

TBD