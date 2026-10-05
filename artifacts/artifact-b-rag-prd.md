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
