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

    Src[(On-Prem DB)]

    Src -.->|Interconnect | GCS[Cloud Storage<br/>Image & Thumbnail]

    GCS --> IngestJob[Cloud Run / Dataflow]

    IngestJob --> BatchEmbed[Vertex AI Batch Embeddings]

    BatchEmbed --> VecStore[(Vertex AI Vector Search)]


    Src --> MetaStore[(Firestore Batch Write<br/> Product Metadata)]

    MetaStore --> IngestJob

end


subgraph "Query Pipeline"

    User((Web / App))
    
    User <--> CDN[Cloud CDN]

    User --> GLB[Cloud Load Balancing]

    GLB --> API[Cloud Run]
    
    API --> Cache[(Memorystore Redis <br/>
    Query + Result Cache)]

    Cache -->|Cache Hit| User

    Cache -->|Cache Miss| Safety[Vertex AI Safety]
    Safety -->Embeddings[Vertex AI Multimodal Embedding]

    Embeddings --> ANN[(Vertex AI Vector Search with metadata filter)]

    ANN --> Filter[(Firestore Batch Get)]

    Filter --> LLM[Vertex AI LLM Generation + Vertex AI Grounding]

    LLM --> Verify[Vertex AI Content Filter]

    Verify --> API

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

    Src[(On-Prem DB)]
    Src -.->|Interconnect / VPN| GCS[Cloud Storage<br/>Raw Documents]
    GCS --> DocAI[Document AI<br/>OCR + Layout Parsing]
    DocAI --> Chunk[Dataflow<br/>Chunking + Tagging]
    Chunk --> TextEmbed[Vertex AI Batch Text Embeddings]
    Chunk --> MultimodalEmbed[Vertex AI Multimodal Embeddings]
    TextEmbed --> VecStore[(Vertex AI Vector Search)]
    MultimodalEmbed --> VecStore[(Vertex AI Vector Search)]
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
    Hybrid --> MultimodalEmbedd[Vertex AI Multimodal Embedding]
    Hybrid --> TextEmbedd[Vertex AI Text Embeddings]
    MultimodalEmbedd --> ANN[(Vertex AI Vector Search <br\>hybrid search)]
    TextEmbedd --> ANN
    ANN --> MetaFilter[(Firestore<br/>Metadata Filter)]
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
%% ========== Budget Control ==========
%% =========================
subgraph "Budget Control"
direction TB
    Sched[Cloud Scheduler]
    Orchestrator[Workflows<br/>Create run_id + Control Concurrency]
    Budget[Billing Budget Alert]
end

Budget -.->|Pause Trigger| Sched
Sched --> Orchestrator

%% =========================
%% ========== PROCESSING ==========
%% =========================
subgraph "Processing Pipeline"
direction TB

    Src[(On-Prem DB)]
    Src -.->|VPN / Interconnect| GCSRaw[Cloud Storage<br/>Raw Data]

    GCSRaw --> Ingest[Dataflow Batch<br/>Batch ETL + Validation]

    Ingest --> Prep[Chunk + Cleanup<br/>]

    Prep --> BatchLLM[Vertex AI Batch Prediction<br/>]

    BatchLLM --> Report[(BigQuery<br/>Partitioned + MERGE by run_id)]
    BatchLLM --> Artifact[Cloud Storage<br/>Generated Files]

end

Orchestrator --> Ingest
Orchestrator --> BatchLLM

%% =========================
%% ========== ACCESS ==========
%% =========================
User((Internal Tool)) --> Report

%% =========================
%% ========== OBSERVABILITY ==========
%% =========================
Ingest -.-> Mon[Monitoring]
BatchLLM -.-> Mon

```
## 4. Compliance-First Pattern

Use Cases: Healthcare AI Document Assistant

### 🎯 Core Objectives

- 🔒 Data stays in-domain (region / tenant / environment) with strict residency controls
- 🧩 Least privilege access with full auditability (who accessed what, - when, why)
- 🏢 Strong multi-tenant isolation and fine-grained enforcement

TBD