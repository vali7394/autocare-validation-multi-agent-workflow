# ACES Fitment Validation — Human-In-The-Loop (HITL) UI Design

> **Document Purpose**: Extension of the multi-agent ACES validation system to incorporate a Human-In-The-Loop (HITL) UI. Defines real-time status communication, conversational querying protocols, and analytical tool interfaces.  
> **Last Updated**: June 2026

---

## 1. Executive Summary & Goals

The ACES Fitment Validation Multi-Agent Orchestrator processes massive automotive fitment files (~100,000 fitment records) through a series of automated nodes in a LangGraph pipeline. To make this powerful backend accessible to catalog managers, supplier representatives, and validation analysts, we require a **Human-In-The-Loop (HITL) User Interface**.

### 1.1 Core Goals
* **Self-Service File Upload**: Allow users to drag-and-drop ACES XML files directly into a browser interface.
* **Real-Time progress Tracking**: Visualize the processing state of the active LangGraph agent node (from schema validation through database comparison).
* **Interactive Chat Analytics**: After processing completes, enable conversational exploration of the results. Users can ask questions about metrics, error patterns, specific validation failure reasons, or brand-level insights without navigating complex SQL tables.
* **Granular Failure Debugging**: Present failed fitments and reference data discrepancies in clean, filterable lists derived from conversational requests.

---

## 2. HITL UI User Experience (UX) Flow

The HITL UI transitions dynamically through three primary phases: Ingestion, Tracking, and Conversational Query.

```
┌─────────────────────────────────┐      ┌─────────────────────────────────┐
│        1. Ingestion Phase       │ ───▶ │       2. Tracking Phase         │
│  - Drag-and-drop XML file       │      │  - Active progress indicators   │
│  - Instant client pre-checks    │      │  - Live node logs streaming     │
└─────────────────────────────────┘      └─────────────────────────────────┘
                                                          │
                                                          ▼
┌─────────────────────────────────┐      ┌─────────────────────────────────┐
│        4. Static Downloads      │ ◀─── │     3. Conversational Phase     │
│  - Export uplift reports        │      │  - Executive summary charts     │
│  - Download validation logs     │      │  - Interactive QA Chatbot       │
└─────────────────────────────────┘      └─────────────────────────────────┘
```

### 2.1 Phase 1: Ingestion
1. **File Dropzone**: User uploads an ACES XML file.
2. **Instant Preflight**: A local web worker performs light client-side schema checks (e.g., matching extension, verifying file size under 150MB) and reads the root element's metadata to display the supplier brand and record estimates immediately.
3. **Submit**: Clicking "Submit" triggers a multipart upload, registers a unique `job_id`, and initiates the background agent workflow.

### 2.2 Phase 2: Progress Tracking
The interface renders a sequential pipeline tracker. Each node corresponds to a LangGraph orchestrator step, accompanied by real-time status badges:
* **Schema Validation**: Parsing file structures, checking XSD conformance.
* **Preflight Check**: Checking filename canonical form and reference version dates.
* **Data Staging**: Parsing XML records and streaming JSON stage data into Snowflake `FITMENT_STAGE`.
* **Validation (Set-Based)**: Bulk execution of VCDB, PCDB, QDB validation queries.
* **Change Comparison**: Classifying fitments as ADD, UPDATE, UNCHANGED, or DELETE.
* **Pipeline Finalization**: Aggregating metrics and archiving the original file.

### 2.3 Phase 3: Interactive Conversational Query
When processing finishes, the UI divides into two sections:
* **Left-Hand Pane: Executive Dashboard**: Displaying summary metrics (total counts, overall error rate, uplift percentages) using visual charts (e.g., donut chart for ADD/UPDATE/DELETE/UNCHANGED splits).
* **Right-Hand Pane: Conversational Assistant**: A ChatGPT-style sidebar where the user interacts with an analytical agent.
  * *Example User Queries:*
    * *"What percentage of fitments failed validation, and what was the main reason?"*
    * *"Show me some examples of VCDB errors where base vehicles didn't resolve."*
    * *"Which brands or part types had the highest rate of validation failure?"*
    * *"How many fitments are we deleting from our catalog with this update?"*

---

## 3. System Architecture with HITL Integration

The architecture comprises five distinct layers working together through event-driven and RESTful communication.

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                          1. FRONTEND                             │
 │            React SPA / Tailwind CSS / Lucide Icons              │
 └─────────────────▲──────────────────────────────┬─────────────────┘
                   │ Server-Sent Events           │ HTTPS / WebSocket
                   │ (Real-Time Progress)         │ (REST API & Chat)
 ┌─────────────────┴──────────────────────────────▼─────────────────┐
 │                       2. BACKEND GATEWAY                         │
 │        BFF (Backend-for-Frontend) — FastAPI / Node.js Express    │
 └─────────────────▲──────────────────────────────┬─────────────────┘
                   │ Redis Pub/Sub                │ Async Invocation /
                   │                              │ Scoped Database Queries
 ┌─────────────────┴─────────────┐      ┌─────────▼────────────────────────┐
 │   3. LANGGRAPH ORCHESTRATOR   │      │       4. CONVERSATIONAL ENGINE   │
 │   Executes validation nodes   │      │    Analytical Chat Agent         │
 │   and writes status updates   │      │    (JobQueryAgent + SQL Tools)   │
 └──────────────┬────────────────┘      └─────────┬────────────────────────┘
                │                                 │
                ▼                                 ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │                           5. SNOWFLAKE                           │
 │     ACES_VALIDATION schema (Staging, Job Runs, Reference data)   │
 └──────────────────────────────────────────────────────────────────┘
```

### 3.1 Layer Responsibilities

1. **Frontend (Client)**: A single-page application (React, Tailwind CSS, Lucide icons) that coordinates file uploading, state visualization, interactive dashboard rendering, and chat interfaces.
2. **Backend Gateway (BFF)**: An API server that manages file uploads, provides endpoints for job queries, streams real-time updates via Server-Sent Events (SSE), and routes chat queries to the conversational agent.
3. **LangGraph Orchestrator**: The background worker executing the deterministic pipeline nodes. It writes job status directly to Snowflake's `JOB_RUN` table and publishes live step events to Redis.
4. **Conversational Engine**: An LLM agent (`JobQueryAgent`) instantiated on demand to answer questions about a specific completed `job_id`. It is equipped with targeted read-only tools to fetch aggregated data from Snowflake.
5. **Snowflake**: The source of truth for both structural ref-data (VCDB, PCDB, QDB) and transient/historical execution staging records (`FITMENT_STAGE`, `FITMENT_CLASSIFICATION`, `JOB_RUN`).

---

## 4. Communication Protocols & Message Sequences

Efficient and responsive user interaction is achieved by using the appropriate protocol for each flow.

### 4.1 Ingestion & Real-Time Tracking Sequence (SSE-driven)

```
Frontend               Backend BFF           LangGraph Engine          Snowflake
   │                        │                      │                       │
   │─── 1. POST File ───────▶                      │                       │
   │    (/api/jobs/upload)  │                      │                       │
   │                        │─── 2. Write Status ─────────────────────────▶│
   │                        │    (status='STAGED') │                       │
   │◀── 3. Return job_id ───│                      │                       │
   │                        │─── 4. Trigger Job ──▶│                       │
   │                        │                      │                       │
   │─── 5. Connect SSE ────▶│                      │                       │
   │    (/api/jobs/{id}/events)                    │                       │
   │                        │◀── 6. Publish Step ──│                       │
   │                        │    (Redis Pub/Sub)   │                       │
   │◀── 7. SSE Event ───────│                      │                       │
   │    (node_progress)     │                      │                       │
   │                        │                      │─── 8. Run validations▶│
   │                        │                      │    & classifications │
   │                        │                      │                      │
   │                        │                      │─── 9. Save final ───▶│
   │                        │                      │    metrics           │
   │                        │◀── 10. Publish Step ─│                       │
   │                        │    (status='COMPLETED')                      │
   │◀── 11. SSE Event ──────│                      │                       │
   │    (complete)          │                      │                       │
```

### 4.2 Interactive Query Sequence (WebSocket/SSE-driven Chat)

```
Frontend               Backend BFF            JobQueryAgent            Snowflake
   │                        │                      │                       │
   │─── 1. POST Message ───▶│                      │                       │
   │    (/api/jobs/{id}/chat)                      │                       │
   │                        │─── 2. Send context ─▶│                       │
   │                        │    (message + job_id)│                       │
   │                        │                      │── 3. Call Tool ─────▶ │
   │                        │                      │   (get_job_metrics)   │
   │                        │                      │◀── 4. Return Counts ──│
   │                        │                      │                       │
   │                        │                      │── 5. Call Tool ─────▶ │
   │                        │                      │   (get_failed_records)│
   │                        │                      │◀── 6. Return Details ─│
   │                        │                      │                       │
   │                        │◀── 7. Stream token ──│                       │
   │◀── 8. Stream Token ────│    (chunks)          │                       │
```

---

## 5. Conversational Agent Engine (`JobQueryAgent`)

To prevent performance bottlenecks and safeguard workspace directories under multi-agent constraints, the analytical agent operates with strict structural rules.

### 5.1 Agent Role & Persona
The **`JobQueryAgent`** is a dedicated analytical partner. It acts as an automotive data analyst who understands the ACES standard, VCDB vehicles, PCDB part configurations, and QDB qualifiers. 
* **Focus**: Interrogates the specific database tables for the given `job_id` only.
* **Safety Rules**: Must never perform write/delete operations. Must never hallucinate numbers; if a tool returns empty sets or an error, report it directly.

### 5.2 System Prompt Template
```text
You are the ACES Validation Analyst Assistant, an AI agent specialized in inspecting and explaining ACES fitment validation runs.
You are assisting a user with a specific validation run identified by job_id: "{{JOB_ID}}".

Your instructions are:
1. Every database tool query you invoke MUST explicitly include the parameter job_id = "{{JOB_ID}}". Never query records across jobs unless explicitly asked to compare two specific jobs.
2. Translate raw database terms and codes into readable explanations:
   - Identify "VCDB" failures as vehicle reference conflicts (e.g., Year, Make, Model inconsistencies).
   - Identify "PCDB" failures as part configuration conflicts (e.g., Part Type and Position pairing errors).
   - Identify "QDB" failures as Qualifier reference mismatch.
3. Be structured. When asked for failure reasons, first present a high-level table grouping of counts, and then list 3-5 specific examples containing: supplier part number, vehicle details, and exact error message.
4. Limit the query sizes using the 'limit' parameter to ensure fast response times and avoid overloading tool response limits (250 KB max).
5. If you do not have sufficient data from tool responses to answer a question, state so clearly and ask the user to refine their question.
```

### 5.3 Specialized Read-Only Agent Tools

To query Snowflake efficiently without passing raw SQL arrays back and forth, the agent has access to five specific read-only tools:

#### Tool 1: `get_job_summary`
* **Purpose**: Retrieves high-level statistics for the job run from `JOB_RUN`, `FITMENT_STAGE`, and `FITMENT_CLASSIFICATION`.
* **Input schema**:
  ```json
  {
    "job_id": "string"
  }
  ```
* **Output schema**:
  ```json
  {
    "total_records": 102450,
    "valid_records": 98560,
    "invalid_records": 3890,
    "classification_summary": {
      "ADD": 1240,
      "UPDATE": 4500,
      "UNCHANGED": 92820,
      "DELETE": 430
    },
    "error_by_category": {
      "VCDB": 2400,
      "PCDB": 890,
      "QDB": 400,
      "PRODUCT": 200
    }
  }
  ```

#### Tool 2: `get_failed_records`
* **Purpose**: Fetches validation-failed records for detailed drill-down analysis.
* **Input schema**:
  ```json
  {
    "job_id": "string",
    "error_type": "string (VCDB | PCDB | QDB | BRAND | PRODUCT | SYSTEM)",
    "limit": "number (default: 10, max: 50)",
    "offset": "number (default: 0)"
  }
  ```
* **Output schema**:
  ```json
  {
    "records": [
      {
        "app_id": "A40212",
        "supplier_part_number": "BP-2120",
        "base_vehicle_id": 48202,
        "part_type_id": 1502,
        "position_id": 4,
        "error_message": "Base Vehicle ID 48202 does not exist in VCDB",
        "vehicle_conditions": "{\"DriveType\": [3, 4]}"
      }
    ]
  }
  ```

#### Tool 3: `get_failed_records_by_cause`
* **Purpose**: Groups and counts validation failures by their specific error messages to reveal the root cause of issues in the file.
* **Input schema**:
  ```json
  {
    "job_id": "string",
    "error_type": "string (VCDB | PCDB | QDB | BRAND | PRODUCT | SYSTEM)"
  }
  ```
* **Output schema**:
  ```json
  {
    "causes": [
      {
        "error_message": "Base Vehicle ID does not exist in VCDB",
        "failure_count": 2400
      },
      {
        "error_message": "Part-Type-to-Position mapping is invalid in PCDB",
        "failure_count": 890
      }
    ]
  }
  ```

#### Tool 4: `get_uplift_details`
* **Purpose**: Retrieves details of modified, added, or deleted fitments to explain changes to the catalog.
* **Input schema**:
  ```json
  {
    "job_id": "string",
    "classification": "string (ADD | UPDATE | DELETE)",
    "limit": "number (default: 10, max: 50)",
    "offset": "number (default: 0)"
  }
  ```
* **Output schema**:
  ```json
  {
    "records": [
      {
        "distinct_fitment_hash": "2f281ca8c32d",
        "supplier_part_number": "TR-902",
        "base_vehicle_id": 1284,
        "part_type_id": 90,
        "position_id": 1,
        "classification": "UPDATE",
        "change_details": "qualifiers_changed"
      }
    ]
  }
  ```

#### Tool 5: `get_brand_or_part_performance`
* **Purpose**: Summarizes error rates aggregated by individual parts, line codes, or brands within the file.
* **Input schema**:
  ```json
  {
    "job_id": "string",
    "group_by": "string (part_number | line_code)"
  }
  ```
* **Output schema**:
  ```json
  {
    "groups": [
      {
        "group_key": "BP-2120",
        "total_fitments": 540,
        "valid_fitments": 120,
        "failed_fitments": 420,
        "failure_rate_percent": 77.7
      }
    ]
  }
  ```

---

## 6. Detailed API Specification

### 6.1 Upload File (HTTP POST)
Registers the job and streams the uploaded XML file to local storage or cloud stage bucket.

* **Endpoint**: `POST /api/jobs/upload`
* **Content-Type**: `multipart/form-data`
* **Request Body**:
  - `file`: `Binary (XML File)`
  - `line_code`: `string`
* **Response Payload (202 Accepted)**:
  ```json
  {
    "job_id": "job_01h9ybc93k018s96abc491pq",
    "status": "STAGED",
    "filename": "ACES_BP_20260610_FULL.xml",
    "line_code": "BP",
    "message": "File uploaded successfully. Job validation started in background.",
    "created_at": "2026-06-10T20:56:00.000Z"
  }
  ```

### 6.2 Get Live Progress Stream (SSE GET)
Establishes a persistent, unidirectional Server-Sent Events (SSE) connection to push real-time agent execution status back to the UI progress bar.

* **Endpoint**: `GET /api/jobs/{job_id}/events`
* **Response Headers**:
  - `Content-Type: text/event-stream`
  - `Cache-Control: no-cache`
  - `Connection: keep-alive`

* **Event Stream Payloads**:

  * **Event 1: Ingestion Start**
    ```text
    event: progress
    data: {"node": "StagingLoadNode", "status": "IN_PROGRESS", "message": "Parsing ACES XML file and staging records in Snowflake...", "percentage": 15}
    ```

  * **Event 2: Ingestion Complete, Validation Start**
    ```text
    event: progress
    data: {"node": "ValidationNode", "status": "IN_PROGRESS", "message": "Performing set-based validation across VCDB, PCDB, and QDB reference tables...", "percentage": 45}
    ```

  * **Event 3: Validation Failed Record Snapshot (Update UI Metrics Card)**
    ```text
    event: metrics_update
    data: {"invalid_count": 1820, "valid_count": 42000, "current_rate": 4.15}
    ```

  * **Event 4: Stage Comparison Complete**
    ```text
    event: progress
    data: {"node": "ComparisonNode", "status": "IN_PROGRESS", "message": "Comparing incoming fitments with current catalog state to classify changes...", "percentage": 80}
    ```

  * **Event 5: Pipeline Completed**
    ```text
    event: complete
    data: {"job_id": "job_01h9ybc93k018s96abc491pq", "status": "COMPLETED_WITH_ERRORS", "total_records": 102450, "valid_records": 98560, "invalid_records": 3890, "duration_seconds": 142}
    ```

### 6.3 Send Conversational Message (HTTP POST with Streaming Response)
Posts a user question about the job and receives a streamed text chunk response from the analytical assistant.

* **Endpoint**: `POST /api/jobs/{job_id}/chat`
* **Content-Type**: `application/json`
* **Request Body**:
  ```json
  {
    "message": "What were the primary validation failure reasons and can you show me 2 failed parts?"
  }
  ```
* **Response Headers**:
  - `Content-Type: text/plain; charset=utf-8`
  - `Transfer-Encoding: chunked`
* **Response Body (Streamed Chunks)**:
  ```text
  Based on the validation results for job run `job_01h9ybc93k018s96abc491pq`,
  the primary failure causes were associated with **VCDB vehicle resolution errors**.
  
  Out of 3,890 total errors, **2,400 (61.7%)** failed because the supplied Base Vehicle ID does not exist in the VCDB master reference table.
  
  Here are 2 failed parts:
  1. **Part BP-2120** (Base Vehicle 48202): Base Vehicle ID does not exist in VCDB.
  2. **Part BP-3004** (Base Vehicle 51201): Base Vehicle ID does not exist in VCDB.
  ```

---

## 7. Security & Query Guardrails

When allowing natural language queries to traverse business tables containing supplier data and validation results, robust architectural guardrails are required.

### 7.1 Cross-Job & Cross-Tenant Data Leakage Prevention
* **Strict Job-ID Sandboxing**: The BFF API Layer must enforce that any tool invocation is parameterized programmatically with the active `job_id` extracted from the user's secure route.
* **No Raw SQL Inputs**: The `JobQueryAgent` has access *only* to parameterized tool definitions (described in Section 5). It is strictly forbidden from writing or executing raw SQL strings directly. This completely prevents SQL injection attacks.
* **Schema Segregation**: Staging tables reside inside the `ACES_VALIDATION.STAGING` schema. The MCP user role configured for the chatbot has access *only* to staging tables and specific read-only master reference tables, ensuring the production fitment catalog remains isolated.

### 7.2 Tool-Calling Rate Limiting & Sizing Guardrails
* **Row-Count Constraints**: Tools such as `get_failed_records` enforce a hardcoded maximum value for the `limit` parameter (e.g., maximum 50 records per call) directly within the database procedure to prevent high memory usage and buffer overflows on the BFF.
* **Token Sparing**: All database tool responses are structured as highly compressed, field-stripped JSON objects. This keeps payload sizes well below the 250 KB Snowflake MCP tool-call boundary and minimizes token usage in the LLM context window.

---

*ACES Fitment Validation — HITL UI Design Specification*
