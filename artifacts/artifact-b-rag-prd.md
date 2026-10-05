# Product Requirements Document (PRD): OEM Asset Telemetry & Maintenance RAG Pipeline

---

## Document Metadata
* **Product Name:** OEM Asset Telemetry & Maintenance RAG Pipeline
* **Document ID:** PRD-ART-B-001
* **Owner:** Vibha Shukla (AI Product Manager / Technical Product Lead)
* **Target Users:** Enterprise Asset Managers, Reliability Engineers, & Field Service Technicians
* **Status:** Prototype / Technical Specification
* **Target Release Window:** Q4 2026

---

## 1. Executive Summary & Problem Statement

### 1.1 Problem Statement
Field service engineers and enterprise asset managers servicing mission-critical OEM equipment (e.g., heavy machinery, IoT-enabled industrial assets) waste an average of 45 to 90 minutes per incident cross-referencing complex telemetry alerts with 500+ page OEM technical manuals, diagnostic schematics, and service bulletins.

Existing solutions rely on standard keyword or vector search across static PDFs. These systems fail when trying to connect dynamic sensor readings and specific fault codes (e.g., `ERR-4092` indicating hydraulic pressure drops) with exact, step-by-step repair protocols, torque specifications, and safety precautions.

### 1.2 Value Proposition
The **OEM Asset Telemetry & Maintenance RAG Pipeline** is an enterprise-grade retrieval-augmented generation (RAG) system designed to combine dynamic IoT telemetry payloads with semi-structured technical documentation. 

By employing hybrid retrieval (Dense + BM25), hierarchical chunking, and strict Cohere reranking, the pipeline yields cited, hallucination-free, and safety-compliant diagnostic workflows directly actionable by field technicians.

---

## 2. User Personas & Core Use Cases

### 2.1 Target Personas
* **Field Service Technician:** Needs immediate, step-by-step diagnostic and repair instructions in low-latency environments while standing next to active machinery. Zero tolerance for unverified torque or electrical safety steps.
* **Enterprise Asset Manager / Reliability Engineer:** Monitors fleet-wide IoT fault trends and requires rapid access to root-cause guidance and OEM documentation across heterogeneous equipment models.

### 2.2 Core Use Cases
1. **Telemetry-Driven Fault Diagnosis:** User inputs an error log or fault code (e.g., `ERR-4092`) along with natural language symptoms (e.g., *"Hydraulic pressure spiking under 40°C ambient temp"*).
2. **OEM Manual Precision Lookup:** User queries exact part numbers, electrical tolerances, or maintenance schedules across multi-version manuals.
3. **Structured Safety & Workflow Extraction:** System outputs forced JSON schema responses containing required tools, step-by-step resolution, safety warnings, and explicit source page citations.

---

## 3. Technical Architecture & Component Selection

### 3.1 Chunking Strategy
* **Approach:** Hierarchical / Parent-Child Chunking.
* **Parent Chunks:** 1,000 tokens (preserves high-level diagnostic context, multi-step procedures, and table boundaries).
* **Child Chunks:** 200 tokens (indexed for precise, fine-grained semantic vector matching).
* **Separators:** Priority splitting on Markdown headers (`\n## `, `\n### `), table boundaries, and double line breaks.

### 3.2 Indexing & Vector Storage
* **Embedding Model:** `text-embedding-3-small` (OpenAI) or `bge-large-en` (Open Source).
* **Vector Store:**
  * **Prototyping Environment:** Local persistent `ChromaDB`.
  * **Enterprise Target:** `Pinecone` or `pgvector` with metadata filtering on `equipment_model`, `manual_version`, and `section_type`.

### 3.3 Retrieval & Reranking Architecture
* **Hybrid Search Strategy:**
  * **Dense Vector Search:** Cosine similarity over child chunks to capture broad semantic meaning.
  * **Sparse Keyword Search (BM25):** Catch exact part numbers, error codes (`ERR-4092`), and model specs.
  * **Ensemble Weighting:** $0.6$ Dense Similarity + $0.4$ BM25 Keyword Search.
* **Reranking Engine:** Cohere Rerank (`rerank-english-v3.0`). Retrieves top 10 candidates from hybrid search and filters down to top 3 ($k=3$) to eliminate retrieval noise and mitigate "lost in the middle" context degradation.

### 3.4 Guardrails & Structured Output Schema
To enforce strict reliability, outputs are parsed against a forced JSON schema:
```json

{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DiagnosticWorkflow",
  "type": "object",
  "properties": {
    "fault_code": { "type": "string" },
    "summary": { "type": "string" },
    "safety_warnings": {
      "type": "array",
      "items": { "type": "string" }
    },
    "diagnostic_steps": {
      "type": "array",
      "items": { "type": "string" }
    },
    "required_tools": {
      "type": "array",
      "items": { "type": "string" }
    },
    "cited_manual_pages": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "manual_id": { "type": "string" },
          "page_number": { "type": "integer" }
        },
        "required": ["manual_id", "page_number"]
      }
    }
  },
  "required": [
    "fault_code",
    "summary",
    "safety_warnings",
    "diagnostic_steps",
    "cited_manual_pages"
  ]
}
```

## 4. Evaluation Strategy & Success Metrics

The pipeline must undergo offline batch evaluation using **LangSmith** and the **`ragas`** framework across a golden evaluation set of 50 ground-truth OEM diagnostic queries before deployment.

### 4.1 Key Quantitative Metrics

| Metric | Target Threshold | Evaluation Framework / Tool | Description & Business Rationale |
| :--- | :--- | :--- | :--- |
| **Context Precision** | $\ge 0.85$ | `ragas` framework | Measures the signal-to-noise ratio of retrieved parent chunks. Ensures irrelevant context is filtered out before LLM generation. |
| **Faithfulness / Grounding** | $\ge 0.95$ | `ragas` + LLM-as-a-Judge | Zero-tolerance policy for ungrounded hallucinated specs (e.g., incorrect torque values or wiring steps). |
| **Answer Relevancy** | $\ge 0.90$ | `ragas` framework | Evaluates whether the diagnostic output directly addresses the technician's telemetry alert. |
| **P95 Latency** | $< 2.5$ seconds | LangSmith Tracing | Measures total execution time from query submission to structured JSON payload delivery. |
| **Unit Economics** | $< \$0.02$ per query | Token / Cost Analytics | Tracks cost efficiency across embedding, reranking, and inference stages. |

---

## 5. Risk Analysis & Edge Cases

| Risk Category | Edge Case Description | Mitigation Strategy |
| :--- | :--- | :--- |
| **Conflicting Manuals** | Multiple versions of the same OEM manual contain differing repair specs. | Strict metadata filtering by `equipment_model` and `serial_number_range` prior to vector search. |
| **Out-of-Domain Query** | User asks a question not covered in the ingested OEM corpus. | System prompt guardrail instructing the LLM to return `INSUFFICIENT_OEM_DATA` when context similarity falls below threshold. |
| **Chunk Boundary Truncation** | Diagnostic procedure tables are cut off midway through a chunk. | Parent-child chunking strategy ensures full 1,000-token parent chunk is retrieved when a child chunk hits match criteria. |

---

## 6. Implementation Roadmap & Next Steps

### Phase 1: Pipeline Assembly (Weeks 5–6)
* Set up LangChain/LlamaIndex pipeline with local ChromaDB.
* Implement Hybrid Retrieval (BM25 + OpenAI Embeddings) and Cohere Reranking.

### Phase 2: Golden Dataset & Evals (Week 6)
* Ingest 3–5 representative OEM PDFs.
* Create 50 golden Q&A pairs with ground-truth citations.
* Run initial `ragas` batch evaluation.

### Phase 3: Production Hardening (Week 7+)
* Integrate JSON Schema validation guardrails.
* Connect trace logging via LangSmith to track cost, latency, and context precision.
