# ACES Fitment Validation — Human-In-The-Loop (HITL) UI Design

> **Document Purpose**: Complete design specification for the HITL UI — screens, UX flows, ingestion architecture, API specification, conversational agent, and edge cases  
> **Parent Document**: [E2E System Design](./00-e2e-design.md)  
> **Version**: 1.0  
> **Last Updated**: June 2026

---

## 1. Goals & Scope

The HITL UI makes the multi-agent validation backend accessible to catalog managers, supplier representatives, and validation analysts.

### 1.1 Core Goals

- **Self-Service File Upload**: Drag-and-drop ACES XML files into a browser interface
- **Real-Time Progress Tracking**: Visualize the active LangGraph node (schema validation through comparison)
- **Interactive Chat Analytics**: Conversational exploration of results — metrics, error patterns, failure reasons, brand insights
- **Granular Failure Debugging**: Filterable lists of failed fitments and reference data discrepancies

### 1.2 UX Phases

```
+-------------------------------+      +-------------------------------+
|     1. Ingestion Phase        | ---> |     2. Tracking Phase         |
|  - Drag-and-drop XML file     |      |  - Active progress indicators |
|  - Instant client pre-checks  |      |  - Live node logs streaming   |
+-------------------------------+      +-------------------------------+
                                                    |
                                                    v
+-------------------------------+      +-------------------------------+
|     4. Static Downloads       | <--- |   3. Conversational Phase     |
|  - Export uplift reports      |      |  - Executive summary charts   |
|  - Download validation logs   |      |  - Interactive QA Chatbot     |
+-------------------------------+      +-------------------------------+
```

---

## 2. System Architecture (HITL Integration)

The architecture comprises five layers working through event-driven and RESTful communication.

```
 +------------------------------------------------------------------+
 |                          1. FRONTEND                              |
 |            React SPA / Tailwind CSS / Lucide Icons                |
 +--------^---------------------------------+-----------------------+
          | Server-Sent Events              | HTTPS / WebSocket
          | (Real-Time Progress)            | (REST API & Chat)
 +--------+--------------------------------v-----------------------+
 |                       2. BACKEND GATEWAY (BFF)                   |
 |        FastAPI or Node.js Express                                |
 +--------^---------------------------------+-----------------------+
          | Redis Pub/Sub                   | Async Invocation /
          |                                 | Scoped Database Queries
 +--------+-------------------+   +---------v----------------------+
 | 3. LANGGRAPH ORCHESTRATOR  |   | 4. CONVERSATIONAL ENGINE      |
 | Executes validation nodes  |   |    JobQueryAgent               |
 | Writes status to Snowflake |   |    (Read-only Snowflake tools) |
 +--------+-------------------+   +---------+----------------------+
          |                                 |
          v                                 v
 +------------------------------------------------------------------+
 |                        5. SNOWFLAKE                               |
 |  ACES_VALIDATION schema (Staging, Job Runs, Reference data)      |
 +------------------------------------------------------------------+
```

### 2.1 Layer Responsibilities

1. **Frontend (Client)**: React SPA coordinating file upload, state visualization, dashboard rendering, and chat interface.
2. **Backend Gateway (BFF)**: API server managing file uploads, SSE streaming, job queries, and chat routing to the conversational agent.
3. **LangGraph Orchestrator**: Background worker executing pipeline nodes. Writes job status to Snowflake `JOB_RUN` and publishes live step events to Redis.
4. **Conversational Engine**: `JobQueryAgent` instantiated on demand for a specific completed `job_id`. Equipped with read-only Snowflake tools.
5. **Snowflake**: Source of truth for reference data, staging records, and job execution history.

---

## 3. Screen-by-Screen UX Specifications

### Screen 1: Job History / Management Dashboard

The primary landing screen. Historical record of all submissions and starting point for analytical deep-dives.

```
+------------------------------------------------------------------------+
|                          APPLICATION SHELL                               |
|  +------------------------------------------------------------------+  |
|  | SCREEN 1: Job History Dashboard                                   |  |
|  | +---------------+ +---------------+ +---------------+             |  |
|  | | Total Jobs    | | Avg Pass Rate | | Change Ratio  |             |  |
|  | +---------------+ +---------------+ +---------------+             |  |
|  | +----------------------------------------------------------------+|  |
|  | | Job Runs Table                                                  ||  |
|  | | ID  | File Name | Supplier | Status   | Progress | Actions     ||  |
|  | | --- | --------- | -------- | -------- | -------- | ----------- ||  |
|  | | 001 | ACES_1    | Bosch    | SUCCESS  | [======] | [DL] [Chat] ||  |
|  | +----------------------------------------------------------------+|  |
|  +------------------------------------------------------------------+  |
+------------------------------------------------------------------------+
```

- **Primary Metrics Bar**:
  - **Total Uploaded Files**: Aggregate count of files processed
  - **Average Pass Rate**: Quality index (% valid records across all files)
  - **Active Job Status**: Real-time ticker of currently running jobs
- **Job Runs Table**:
  - Columns: `Job ID` (monospace), `Transfer Date`, `File Name`, `Line Code`, `Record Count`, `Status` (color-coded badges: Success/Warning/Failed/Processing), `Actions`
  - **Chat Icon Action**: Prominent chatbot icon next to each completed/warning row. Tooltip: *"Converse with AI about validation issues and statistics for Job {job_id}"*. Click opens Screen 4.
- **Action Button**: "Upload New Fitments File" triggers Screen 2 Modal

### Screen 2: File Upload / Ingestion Modal

Overlay dialog for capturing ACES XML files with fast browser-side pre-validations.

```
+----------------------------------------------------------------+
|  Upload ACES XML Fitments File                                  |
+----------------------------------------------------------------+
|                                                                  |
|   +----------------------------------------------------------+  |
|   |            [ Drag & Drop ACES XML File ]                  |  |
|   |                  Max limit: 150MB                         |  |
|   +----------------------------------------------------------+  |
|                                                                  |
|   Client-Side Preflight Summary:                                 |
|   - Detected File: ACES_BP_2026.xml (112.4 MB)                  |
|   - Schema Version: 4.1  |  Line Code: BP (Auto-detected)       |
|   - Estimated Records: ~105,000 Apps                             |
|                                                                  |
|   [ Target Catalog Environment: [ Staging (Default) ] ]          |
|                                                                  |
|   [ Cancel ]                                [ Start Ingestion ]  |
+----------------------------------------------------------------+
```

- **Drag-and-Drop Area**: Dashed dropzone supporting `.xml` extensions
- **Instant Preflight Worker**:
  - Lightweight web worker processes the first 50 KB of the file
  - Parses XML header: company name, transfer date, ACES version, line code
  - Estimates record count by scanning block structures without blocking UI thread
  - Alerts user if file doesn't conform to ACES structures (missing `<ACES>` or `<App>` nodes)
- **Configuration Selectors**: Target catalog environment (Sandbox/Staging/Production) and manual Line Code override

### Screen 3: Real-Time Ingestion Progress Tracker

Replaces the upload modal upon submission. Supports full-screen focus view or minimized background card.

- **Orchestrator Progress Stepper** (5 stages matching backend nodes):
  1. **Parsing & Staging**: Upload progress and staging progress (*"Parsed and staged 54,000 / 100,000 records into Snowflake"*)
  2. **Schema Validation**: XSD schema alignment check
  3. **Ref Data Checks (VCDB, PCDB, QDB)**: Live check counters (*"VCDB base vehicle resolution... 1,420 errors found"*)
  4. **Catalog Change Comparison**: Change categorization activity (ADD, UPDATE, DELETE)
  5. **Finalizing**: Report generation, run registration, file archival
- **Live Error Log Ticker**: Scrollable terminal-style window at screen bottom streaming warning/error logs:
  ```
  [17:12:04] WARNING: Part BP-901 has invalid qualifier Q-441
  [17:12:15] ERROR: Vehicle ID 54101 does not exist in VCDB master
  ```

### Screen 4: Split-Screen Interactive Query Workspace

Triggered by chat icon on dashboard or upon pipeline completion.

```
+-------------------------------------------+----------------------------+
| JOB ANALYSIS: ACES_BP_2026_FULL.xml       | Job Query Assistant        |
+-------------------------------------------+----------------------------+
| +-------------------+  +----------------+ | Chatting about Job:        |
| |   98.2% Pass      |  | Uplift Summary | | job_01h9ybc93k018s96abc491 |
| |  (102.4k Valid)   |  |   +1,240 Adds  | +----------------------------+
| +-------------------+  |   -430 Deletes | | [AI] Hello! I've loaded    |
| +-------------------+  +----------------+ | the validation data.       |
| | Error Distribution|  +----------------+ | I detected:                |
| | VCDB: [========]  |  | Top Failure    | | - 3,890 Validation Errors  |
| | PCDB: [===     ]  |  | BP-2120 (420x) | | - 1,240 New additions      |
| | QDB : [=       ]  |  +----------------+ | What would you like to     |
| +-------------------+                     | drill down into?           |
| +---------------------------------------+ |                            |
| | Failed Record Sample Drawer           | | [User] why did BP-2120 fail|
| | Part  | Vehicle | Msg                 | | so many times?             |
| | ----- | ------- | ------------------- | |                            |
| | 2120  | 48202   | Vehicle missing     | | [AI] BP-2120 failed due to |
| +---------------------------------------+ | VCDB Base Vehicle ID       |
|                                           | mismatch on ID 48202.      |
+-------------------------------------------+----------------------------+
```

#### Left Pane (60% Width): Visual Status Dashboard

- **Header Summary Card**: Overall file health, validation status, pipeline execution time
- **Uplift Pie Chart**: Interactive segments for ADD/UPDATE/UNCHANGED/DELETE; hover shows representative part numbers
- **Validation Error Bar Chart**: VCDB, PCDB, QDB, Brand/Product failure counts
- **Sample Drawer**: Searchable, scrollable spreadsheet of failed records matching `FITMENT_STAGE` schema. Clicking a row shows the raw XML snippet that generated the error.

#### Right Pane (40% Width): Conversational Chat Assistant Sidebar

- **Quick Suggestion Chips**: Pre-built queries:
  - *"Show the top 3 vehicle resolution failures."*
  - *"List the parts that have brand warnings."*
  - *"What was the impact on our Ford fitment catalog?"*
- **AI Chat Area**: Markdown-formatted tables, lists, bold callouts. Animated indicator when agent queries Snowflake (*"Assistant is querying Snowflake tables..."*).
- **Input Box**: Standard typing with prompt history (Arrow Up) and mic input for voice queries.

---

## 4. Communication Protocols

### 4.1 Ingestion & Real-Time Tracking (SSE)

```
Frontend               Backend BFF           LangGraph Engine          Snowflake
   |                        |                      |                       |
   |--- 1. POST File ------>|                      |                       |
   |    (/api/jobs/upload)  |                      |                       |
   |                        |--- 2. Write Status --|---------------------->|
   |                        |    (status='STAGED') |                       |
   |<-- 3. Return job_id ---|                      |                       |
   |                        |--- 4. Trigger Job -->|                       |
   |--- 5. Connect SSE ---->|                      |                       |
   |    (/api/jobs/{id}/events)                    |                       |
   |                        |<-- 6. Publish Step --|                       |
   |                        |    (Redis Pub/Sub)   |                       |
   |<-- 7. SSE Event -------|                      |                       |
   |    (node_progress)     |                      |--- 8. Run validations>|
   |                        |                      |                       |
   |                        |<-- 9. Publish Step --|                       |
   |                        |   (status=COMPLETED) |                       |
   |<-- 10. SSE Event ------|                      |                       |
   |    (complete)          |                      |                       |
```

### 4.2 Interactive Chat (Streaming Response)

```
Frontend               Backend BFF            JobQueryAgent            Snowflake
   |                        |                      |                       |
   |--- 1. POST Message --->|                      |                       |
   |    (/api/jobs/{id}/chat)                      |                       |
   |                        |--- 2. Send context ->|                       |
   |                        |    (message + job_id)|                       |
   |                        |                      |-- 3. Call Tool ------> |
   |                        |                      |   (get_job_metrics)   |
   |                        |                      |<- 4. Return Counts -- |
   |                        |                      |-- 5. Call Tool ------> |
   |                        |                      |   (get_failed_records)|
   |                        |                      |<- 6. Return Details -- |
   |                        |<- 7. Stream tokens --|                       |
   |<-- 8. Stream Tokens ---|                      |                       |
```

---

## 5. Conversational Agent Engine (JobQueryAgent)

### 5.1 Agent Role & Persona

The **JobQueryAgent** is a dedicated analytical partner acting as an automotive data analyst who understands ACES standard, VCDB vehicles, PCDB part configurations, and QDB qualifiers.

- **Scope**: Interrogates database tables for the given `job_id` only
- **Safety**: Never performs write/delete operations. Never hallucinate numbers — if a tool returns empty sets or errors, reports directly.

### 5.2 System Prompt Template

```text
You are the ACES Validation Analyst Assistant, an AI agent specialized in inspecting
and explaining ACES fitment validation runs.
You are assisting a user with a specific validation run identified by job_id: "{{JOB_ID}}".

Your instructions are:
1. Every database tool query you invoke MUST include job_id = "{{JOB_ID}}".
   Never query across jobs unless explicitly asked to compare two specific jobs.
2. Translate raw database terms into readable explanations:
   - "VCDB" failures = vehicle reference conflicts (Year, Make, Model inconsistencies)
   - "PCDB" failures = part configuration conflicts (Part Type + Position pairing errors)
   - "QDB" failures = Qualifier reference mismatch
3. Be structured. For failure reasons: first a high-level count table, then 3-5 specific
   examples with supplier part number, vehicle details, and exact error message.
4. Use the 'limit' parameter to keep responses fast (<250 KB Snowflake tool limit).
5. If insufficient data from tools, state so clearly and ask user to refine the question.
```

### 5.3 Read-Only Agent Tools

#### Tool 1: `get_job_summary`

**Purpose**: High-level statistics from `JOB_RUN`, `FITMENT_STAGE`, `FITMENT_CLASSIFICATION`.

| Field | Type | Description |
|-------|------|-------------|
| **Input**: `job_id` | string | Job identifier |
| **Output**: `total_records` | integer | Total fitments processed |
| **Output**: `valid_records` | integer | Passed validation |
| **Output**: `invalid_records` | integer | Failed validation |
| **Output**: `classification_summary` | object | `{ ADD, UPDATE, UNCHANGED, DELETE }` counts |
| **Output**: `error_by_category` | object | `{ VCDB, PCDB, QDB, PRODUCT }` counts |

#### Tool 2: `get_failed_records`

**Purpose**: Fetch validation-failed records for drill-down analysis.

| Field | Type | Description |
|-------|------|-------------|
| **Input**: `job_id` | string | Job identifier |
| **Input**: `error_type` | string | Filter: VCDB, PCDB, QDB, BRAND, PRODUCT, SYSTEM |
| **Input**: `limit` | number | Default 10, max 50 |
| **Input**: `offset` | number | Default 0 |
| **Output**: `records[]` | array | `{ app_id, supplier_part_number, base_vehicle_id, part_type_id, position_id, error_message, vehicle_conditions }` |

#### Tool 3: `get_failed_records_by_cause`

**Purpose**: Group and count failures by error message to reveal root causes.

| Field | Type | Description |
|-------|------|-------------|
| **Input**: `job_id` | string | Job identifier |
| **Input**: `error_type` | string | Filter: VCDB, PCDB, QDB, BRAND, PRODUCT, SYSTEM |
| **Output**: `causes[]` | array | `{ error_message, failure_count }` |

#### Tool 4: `get_uplift_details`

**Purpose**: Retrieve details of modified, added, or deleted fitments.

| Field | Type | Description |
|-------|------|-------------|
| **Input**: `job_id` | string | Job identifier |
| **Input**: `classification` | string | ADD, UPDATE, or DELETE |
| **Input**: `limit` | number | Default 10, max 50 |
| **Input**: `offset` | number | Default 0 |
| **Output**: `records[]` | array | `{ distinct_fitment_hash, supplier_part_number, base_vehicle_id, part_type_id, position_id, classification, change_details }` |

#### Tool 5: `get_brand_or_part_performance`

**Purpose**: Summarize error rates by part number or line code.

| Field | Type | Description |
|-------|------|-------------|
| **Input**: `job_id` | string | Job identifier |
| **Input**: `group_by` | string | `part_number` or `line_code` |
| **Output**: `groups[]` | array | `{ group_key, total_fitments, valid_fitments, failed_fitments, failure_rate_percent }` |

---

## 6. API Specification

### 6.1 Upload File

- **Endpoint**: `POST /api/jobs/upload`
- **Content-Type**: `multipart/form-data`
- **Request Body**: `file` (binary XML), `line_code` (string)
- **Response** (202 Accepted):
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

### 6.2 Get Live Progress Stream

- **Endpoint**: `GET /api/jobs/{job_id}/events`
- **Response Headers**: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
- **Event Types**:

| Event | Payload | When |
|-------|---------|------|
| `progress` | `{ node, status, message, percentage }` | Each pipeline stage transition |
| `metrics_update` | `{ invalid_count, valid_count, current_rate }` | During validation (live counters) |
| `complete` | `{ job_id, status, total_records, valid_records, invalid_records, duration_seconds }` | Pipeline finished |

### 6.3 Send Chat Message

- **Endpoint**: `POST /api/jobs/{job_id}/chat`
- **Content-Type**: `application/json`
- **Request Body**: `{ "message": "string" }`
- **Response Headers**: `Content-Type: text/plain; charset=utf-8`, `Transfer-Encoding: chunked`
- **Response**: Streamed text chunks from the JobQueryAgent

---

## 7. Upload & Ingestion Architecture

### 7.1 Streaming Ingest Pattern

```
   User File Drop (Browser)
             |
             | (1) Stream Upload via Multipart POST
             v
      Backend BFF Ingestion
             |
             | (2) Streaming SAX XML Parser
             v
      Batch JSONL Files (10K records per batch)
             |
             | (3) Stream Upload via Cloud SDK
             v
      Snowflake Internal Stage
             |
             | (4) COPY INTO FITMENT_STAGE
             v
     Snowflake Staging Tables
             |
             | (5) Trigger LangGraph Orchestrator (pass job_id)
             v
     LangGraph Node Executions (Set-Based SQL Procedures)
```

### 7.2 Step Details

1. **Client-to-BFF Stream Ingest**: React UI uses `ReadableStream` to POST binary XML to `/api/jobs/upload`. Browser memory stays under 10 MB.

2. **BFF Memory-Efficient Parsing**: BFF routes incoming stream through a SAX parser (Python `iterparse` or Node `saxy`/`sax-js`). Event-driven: triggers on `<App>` open/close tags, discards parsed elements immediately. Memory locked under 100 MB.

3. **Chunking & Gzip Staging**: SAX parser flattens each `<App>` to JSON. Batches of 10,000 records are Gzip-compressed as JSONL and uploaded to Snowflake Internal Stage (`@ACES_STAGING_STAGE/job_id/`).

4. **Bulk Copy Execution**:
```sql
COPY INTO ACES_VALIDATION.STAGING.FITMENT_STAGE
FROM @ACES_STAGING_STAGE/job_id/
FILE_FORMAT = (TYPE = JSON COMPRESSION = GZIP)
MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```
100,000 records staged in <10 seconds.

5. **Agent Activation**: BFF triggers LangGraph Orchestrator asynchronously with `job_id`. All subsequent processing uses set-based SQL on already-staged data.

---

## 8. Edge Cases & Mitigations

### 8.1 File Ingest Edge Cases

| Edge Case | Problem | Mitigation |
|-----------|---------|------------|
| **Connection Drop** | Upload interrupted mid-transfer | tus-protocol or chunked multipart with resume offset in local storage |
| **Large Files (>150MB)** | BFF resource exhaustion | Reject at API gate, or redirect to direct cloud staging bucket via Snowflake external stages |
| **Malformed XML** | Broken tags, missing closing structures | SAX parser captures parse error, aborts upload, logs error with line number, sets job FAILED |
| **Invalid Line Code** | Header line code doesn't match user's workspace | Warnings Modal: user can override line code or abort |

### 8.2 Chat Agent Edge Cases

| Edge Case | Problem | Mitigation |
|-----------|---------|------------|
| **No Failures** | User asks about failures when job was 100% success | Agent reads job summary first; responds: *"This job completed with 100% success! No failures encountered."* |
| **Huge Result Sets** | User asks for all 15,000 failed records | Tools enforce max 50 records per call. Agent paginates and directs user to the spreadsheet drawer for full list. |
| **Ambiguous Queries** | *"Why is everything broken?"* | Agent queries overall metrics, finds top failure cause, structures organized summary with percentages and examples |

---

## 9. Security & Guardrails

### 9.1 Data Leakage Prevention

- **Job-ID Sandboxing**: BFF enforces that all tool invocations are parameterized with the active `job_id` from the user's secure route
- **No Raw SQL**: JobQueryAgent has access only to parameterized tool definitions; cannot write or execute raw SQL
- **Schema Segregation**: Staging in `ACES_VALIDATION.STAGING`; chatbot role has read-only access to staging + reference tables; production catalog isolated

### 9.2 Rate Limiting & Sizing

- **Row-Count Constraints**: Tools enforce hardcoded max `limit` (50 records) in the database procedure
- **Token Efficiency**: Tool responses are compressed, field-stripped JSON objects — under 250 KB MCP limit and minimal LLM context usage

---

## 10. References

| Document | Relevance |
|----------|-----------|
| [E2E System Design](./00-e2e-design.md) | Master architecture context |
| [Multi-Agent Orchestration Design](./01-multi-agent-orchestration-design.md) | Pipeline nodes, validation/comparison logic |
| [Snowflake & MCP Design](./03-snowflake-mcp-design.md) | Database schema, stored procedures |

---

*ACES Fitment Validation — HITL UI Design v1.0*
