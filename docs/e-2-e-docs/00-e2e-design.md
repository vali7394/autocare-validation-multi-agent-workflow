# ACES Fitment Validation — End-to-End System Design

> **Document Purpose**: Master design document for planning and implementation of the ACES Fitment Validation Multi-Agent System  
> **Version**: 1.0  
> **Last Updated**: June 2026

---

## Document Index

This E2E document provides the consolidated system view. Detailed designs are in the following companion documents:

| Document | Scope | Link |
|----------|-------|------|
| **Multi-Agent Orchestration Design** | Agent topology, LangGraph nodes, validation/comparison logic, tool definitions | [01-multi-agent-orchestration-design.md](./01-multi-agent-orchestration-design.md) |
| **HITL UI Design** | Frontend screens, UX flows, API specification, conversational agent, ingestion architecture | [02-hitl-ui-design.md](./02-hitl-ui-design.md) |
| **Snowflake & MCP Design** | Schema DDL, stored procedures, MCP server configuration, OAuth, access control | [03-snowflake-mcp-design.md](./03-snowflake-mcp-design.md) |
| **ACES 4.2 Fitment Attributes** | ACES XML standard reference (fitment rules, vehicle attributes, qualifiers) | [aces-4-2-fitment-attributes.md](./aces-4-2-fitment-attributes.md) |
| **A2A Agent Design** | Externalizing JobQueryAgent via Google A2A protocol (post-MVP) | [05-a2a-agent-design.md](./05-a2a-agent-design.md) |

---

## 1. Executive Summary

This system validates automotive fitment data submitted by suppliers in ACES-compliant XML files (~100,000 fitment records per file) against industry-standard reference databases (VCDB, QDB, PCDB). It uses a **multi-agent orchestration architecture** powered by **LangGraph**, with all heavy data processing pushed into **Snowflake** via set-based stored procedures exposed through a **Snowflake Managed MCP Server**. A **Human-In-The-Loop (HITL) UI** provides file upload, real-time progress tracking, visual dashboards, and a conversational AI assistant for post-processing analytics.

### Key Architectural Principles

1. **Agent manages orchestration, not row-level data** — Agents coordinate pipeline stages; all bulk validation/comparison runs inside Snowflake.
2. **Staging-first design** — Parsed ACES XML is loaded to Snowflake once; all subsequent operations act on staging tables scoped by `job_id`.
3. **No large payloads over MCP** — MCP tools accept `job_id` and return small summary objects. The 250 KB response limit is never stressed.
4. **Set-based SQL** — Validation and comparison use UPDATE/JOIN statements in Snowflake, not row-by-row agent-side processing.
5. **Distinct Fitment Hash (DFH)** — An extended composite key resolving primary-key collisions for accurate change detection.

---

## 2. Business Context

### 2.1 Use Case

Suppliers submit ACES-compliant XML files containing automotive fitment data — information about which parts fit which vehicles. These files must be validated against Autocare reference databases before being accepted into the catalog system.

### 2.2 Processing Model — Full File Submissions

**Suppliers always send complete product-level fitment files.** Each file contains ALL fitments for the products included — not incremental updates. The system must:

1. **Validate** all incoming fitments against Autocare reference data (VCDB, QDB, PCDB)
2. **Compare** validated fitments against existing fitments in the catalog
3. **Classify** each fitment as Add, Update, Delete, or Unchanged
4. **Report** an uplift summary for business review

### 2.3 Typical File Characteristics

| Metric | Value |
|--------|-------|
| Fitments per file | ~100,000 |
| Products per file | Hundreds to thousands |
| File format | ACES XML (versions 3.x, 4.x) |
| Max file size | 150 MB |
| Frequency | On-demand from suppliers |

---

## 3. Objectives & Success Criteria

### 3.1 Primary Objectives

| # | Objective | Success Criteria |
|---|-----------|------------------|
| 1 | Validate fitment data accuracy | All fitments checked against VCDB, QDB, PCDB |
| 2 | Detect changes from previous submissions | Accurate Add/Update/Delete classification using DFH |
| 3 | Generate actionable uplift reports | Business can review what changed and why |
| 4 | Scale to 100K fitments per file | Process within acceptable SLA window |
| 5 | Provide self-service HITL interface | Upload, track, query results via browser |

### 3.2 Secondary Objectives

- **Maintainability** — Clear separation of concerns via agent/node architecture
- **Extensibility** — Easy to add new validation rules as stored procedures
- **Observability** — Job-level tracking, error summaries, retry auditability
- **Reusability** — MCP tools reusable across different workflows

---

## 4. System Architecture Overview

### 4.1 High-Level Architecture (5 Layers)

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
 |        File upload, SSE streaming, chat routing                  |
 +--------^---------------------------------+-----------------------+
          | Redis Pub/Sub                   | Async Invocation /
          |                                 | Scoped DB Queries
 +--------+-------------------+   +---------v----------------------+
 | 3. LANGGRAPH ORCHESTRATOR  |   | 4. CONVERSATIONAL ENGINE      |
 | Executes validation nodes  |   |    JobQueryAgent               |
 | Writes status to Snowflake |   |    (Read-only Snowflake tools) |
 +--------+-------------------+   +---------+----------------------+
          |                                 |
          v                                 v
 +------------------------------------------------------------------+
 |                        5. SNOWFLAKE                               |
 |  ACES_VALIDATION DB: AUTOCARE | CATALOG | STAGING | MCP schemas  |
 |  Managed MCP Server: ACES_FITMENT_SERVER                         |
 +------------------------------------------------------------------+
```

### 4.2 Layer Responsibilities

| Layer | Responsibility | Key Technologies |
|-------|----------------|------------------|
| **Frontend** | File upload, progress visualization, dashboard, chat UI | React, Tailwind CSS, Lucide Icons |
| **Backend Gateway (BFF)** | REST API, file streaming, SAX parsing, SSE, chat routing | FastAPI or Express, Redis |
| **LangGraph Orchestrator** | DAG-based pipeline execution, checkpointing, retry | LangGraph (Python/TS), MCP Client |
| **Conversational Engine** | Post-processing analytics via natural language | LLM + JobQueryAgent + read-only tools |
| **Snowflake** | Reference data, staging, validation procedures, MCP server | Snowflake, OAuth 2.0 |

### 4.3 Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| Agent Framework | **LangGraph** (Python or TypeScript) | DAG/state-machine orchestration, checkpointing, parallel nodes |
| Tool Protocol | **Model Context Protocol (MCP)** | Standard interface for tool discovery and invocation |
| Data Warehouse | **Snowflake** | Reference data, catalog, staging, validation compute |
| MCP Server | **Snowflake Managed MCP** | Tools as stored procedures, no custom infra |
| Authentication | **Snowflake OAuth 2.0** _(deferred)_ | Secure MCP client auth — nice-to-have for MVP |
| Real-time Events | **Redis Pub/Sub + SSE** | Pipeline progress streaming to UI |
| Frontend | **React + Tailwind CSS** | SPA with dashboard and chat interface |
| Backend | **FastAPI or Node.js Express** | BFF API gateway |

---

## 5. End-to-End Processing Pipeline

### 5.1 Pipeline Stages

```
ACES XML File (User Upload via Browser)
  |
  v
[1] Client-Side Preflight (Browser Web Worker)
  |
  v
[2] Stream Upload + SAX Parse + Snowflake Staging (BFF)
  |
  v
[3] Schema Validation (LangGraph: SchemaValidationNode) -- Local, non-MCP
  |
  v
[4] Preflight Checks (LangGraph: PreflightNode) -- MCP tool for version tables
  |
  v
[5] Set-Based Validation (LangGraph: ValidationNode) -- MCP tools, job-scoped
  |   - VCDB: base vehicle, attributes, vehicle resolution
  |   - PCDB: part type, position, part-type-position mapping
  |   - QDB: qualifier IDs
  |   - Product: supplier part number resolution
  |
  v
[6] Set-Based Comparison (LangGraph: ComparisonNode) -- MCP tools, job-scoped
  |   - Classify: ADD, UPDATE, UNCHANGED
  |   - Identify: DELETE (anti-join)
  |
  v
[7] Statistics & Uplift Summary (LangGraph: StatisticsNode)
  |
  v
[8] Report Output + Job Status (LangGraph: FinalStatusNode)
  |
  v
[9] Interactive Analysis (HITL UI: Dashboard + Conversational Chat)
```

### 5.2 Stage Details

| Stage | Node | Input | Output | Execution |
|-------|------|-------|--------|-----------|
| 1. Client Preflight | Browser | XML file | Version, line code, record estimate | Client-side web worker |
| 2. Ingest & Stage | BFF | XML stream | `FITMENT_STAGE` rows in Snowflake | SAX parser + COPY INTO |
| 3. Schema Validation | SchemaValidationNode | XML content | Pass/Fail + errors | Local XSD validation |
| 4. Preflight Checks | PreflightNode | Header metadata | Pass/Fail + report | MCP tool for version check |
| 5. Validation | ValidationNode | `job_id` | `is_valid`, `error_type`, `error_message` per fitment | Set-based SQL via MCP |
| 6. Comparison | ComparisonNode | `job_id` | `FITMENT_CLASSIFICATION` rows | Set-based SQL via MCP |
| 7. Statistics | StatisticsNode | `job_id` | Validation + uplift summaries | Aggregation SQL via MCP |
| 8. Finalize | FinalStatusNode | `job_id` | Job status update, staging cleanup | Direct SQL |
| 9. Analytics | HITL Chat | User questions | AI-generated answers with data | JobQueryAgent + tools |

### 5.3 Validation Categories

| Category | What It Checks | Error Type | Source DB |
|----------|---------------|------------|-----------|
| **Base Vehicle** | Base Vehicle ID exists | VCDB ERROR | VCDB |
| **Vehicle Attributes** | DriveType, FuelType, etc. exist in domains | VCDB ERROR | VCDB |
| **Vehicle Resolution** | Base Vehicle + attributes resolve to real vehicle | VCDB ERROR | VCDB |
| **Part Type** | Part Type ID exists | PCDB ERROR | PCDB |
| **Position** | Position ID exists | PCDB ERROR | PCDB |
| **Part-Type-Position** | Mapping is valid combination | PCDB ERROR | PCDB |
| **Qualifiers** | Qualifier IDs exist | QDB ERROR | QDB |
| **Brand/SubBrand** | Brand and SubBrand IDs exist | BRAND WARNING | Catalog |
| **Product Resolution** | Supplier part number maps to known product | PRODUCT ERROR | Catalog |

### 5.4 Comparison Logic (Post-Validation)

Since files are FULL submissions, we compare incoming vs existing to classify changes.

| Scenario | Classification | How Detected |
|----------|---------------|--------------|
| DFH not in existing catalog | **ADD** | LEFT JOIN miss |
| DFH in existing, details identical | **UNCHANGED** | LEFT JOIN hit, VARIANT equality |
| DFH in existing, details differ | **UPDATE** | LEFT JOIN hit, VARIANT inequality |
| DFH in existing, not in incoming | **DELETE** | Anti-join from catalog to staging |

### 5.5 Distinct Fitment Hash (DFH)

The DFH is an extended composite key that resolves primary-key collisions when suppliers submit multiple `<App>` elements for the same product/vehicle/part/position with different qualifiers or attributes.

```
DFH = MD5(CONCAT_WS('|',
    COALESCE(product_id::STRING, ''),
    base_vehicle_id::STRING,
    part_type_id::STRING,
    COALESCE(position_id::STRING, '0'),
    HASH(vehicle_conditions)::STRING,
    HASH(qualifiers)::STRING,
    HASH(notes)::STRING
))
```

This enables the system to support multiple distinct fitments on the same vehicle configuration without collisions.

---

## 6. Data Model

### 6.1 Snowflake Database Structure

```
ACES_VALIDATION (Database)
  |
  +-- AUTOCARE (Schema) -- Reference data
  |     VCDB_BASE_VEHICLE, VCDB_VEHICLE, VCDB_DRIVE_TYPE,
  |     VCDB_FUEL_TYPE, VCDB_ENGINE_BASE, VCDB_VERSION,
  |     QDB_QUALIFIER, QDB_VERSION,
  |     PCDB_PART_TYPE, PCDB_POSITION, PCDB_PART_TYPE_POSITION, PCDB_VERSION
  |
  +-- CATALOG (Schema) -- Product & existing fitment data
  |     PRODUCT, BRAND, BRAND_SUB_BRAND, FITMENT
  |
  +-- STAGING (Schema) -- Job-scoped processing data
  |     JOB_RUN, FITMENT_STAGE, FITMENT_CLASSIFICATION
  |
  +-- MCP (Schema) -- Stored procedures & MCP server
        All validation/comparison/aggregation procedures
        ACES_FITMENT_SERVER (MCP Server object)
```

### 6.2 Key Tables

| Table | Schema | Purpose | Key |
|-------|--------|---------|-----|
| `FITMENT_STAGE` | STAGING | Parsed XML records for active job | `(job_id, distinct_fitment_hash)` |
| `FITMENT_CLASSIFICATION` | STAGING | ADD/UPDATE/DELETE/UNCHANGED results | `(job_id, distinct_fitment_hash)` |
| `JOB_RUN` | STAGING | Job execution state and metadata | `job_id` |
| `FITMENT` | CATALOG | Existing catalog fitments | `distinct_fitment_hash` |
| `PRODUCT` | CATALOG | Supplier part to product mapping | `(line_code, supplier_part_number)` |

> Full DDL scripts in [Snowflake & MCP Design](./03-snowflake-mcp-design.md).

---

## 7. MCP Tool Inventory

All MCP tools accept `job_id` and operate on staging tables. Responses are small summary objects (never large arrays).

| Tool | Category | Input | Output |
|------|----------|-------|--------|
| `validate_base_vehicle_ids_job` | VCDB | `job_id` | `{ invalid_count }` |
| `validate_attribute_values_job` | VCDB | `job_id` | `{ invalid_count }` |
| `resolve_vehicle_configuration_job` | VCDB | `job_id` | `{ invalid_count }` |
| `validate_qualifier_ids_job` | QDB | `job_id` | `{ invalid_count }` |
| `validate_part_type_ids_job` | PCDB | `job_id` | `{ invalid_count }` |
| `validate_position_ids_job` | PCDB | `job_id` | `{ invalid_count }` |
| `validate_part_type_position_job` | PCDB | `job_id` | `{ invalid_count }` |
| `resolve_products_job` | Product | `job_id`, `line_code` | `{ resolved_count, unresolved_count }` |
| `classify_fitment_changes_job` | Comparison | `job_id` | `{ add, update, unchanged }` |
| `identify_deleted_fitments_job` | Comparison | `job_id` | `{ delete_count }` |
| `compute_validation_summary` | Aggregation | `job_id` | Validation stats |
| `compute_uplift_summary` | Aggregation | `job_id` | Uplift stats |

> Full tool specifications and backing stored procedures in [Snowflake & MCP Design](./03-snowflake-mcp-design.md).

---

## 8. HITL User Interface Summary

The HITL UI provides four primary screens:

| Screen | Purpose | Key Features |
|--------|---------|--------------|
| **Job History Dashboard** | Landing page, historical job list | Metrics bar, job table with status badges, chat icon |
| **Upload Modal** | File upload with client-side preflight | Drag-and-drop, web worker parsing, line code detection |
| **Progress Tracker** | Real-time pipeline visualization | SSE-driven stepper, live error log ticker |
| **Split-Screen Analysis** | Post-processing dashboard + chat | Left: charts & tables; Right: conversational AI assistant |

### Communication Protocols

| Flow | Protocol | Direction |
|------|----------|-----------|
| File Upload | HTTP POST (multipart) | Client -> BFF |
| Pipeline Progress | Server-Sent Events (SSE) | BFF -> Client |
| Chat Messages | HTTP POST (streaming response) | Client <-> BFF <-> JobQueryAgent |
| Orchestrator Status | Redis Pub/Sub | LangGraph -> BFF |

> Full screen designs, UX flows, API specifications, and edge cases in [HITL UI Design](./02-hitl-ui-design.md).

---

## 9. Ingestion Architecture

### 9.1 Streaming Ingest Pattern

To move 100K+ records from browser to Snowflake without memory exhaustion:

```
User File Drop (Browser)        -- <10 MB browser memory
        |
        v (1) Stream Upload via Multipart POST
Backend BFF                     -- <100 MB server memory
        |
        v (2) Streaming SAX XML Parser
Batch JSONL Files               -- 10K records per batch, Gzip compressed
        |
        v (3) Upload to Snowflake Internal Stage
Snowflake Stage                 -- @ACES_STAGING_STAGE/job_id/
        |
        v (4) COPY INTO FITMENT_STAGE
Snowflake Tables                -- <10 seconds for 100K records
        |
        v (5) Trigger LangGraph Orchestrator
Set-Based SQL Validation        -- All processing inside Snowflake
```

### 9.2 Key Ingestion Details

- **SAX Parsing**: Event-driven XML parser processes `<App>` blocks one at a time; discards parsed elements immediately.
- **DFH Calculation**: The Distinct Fitment Hash is computed during parsing, before staging.
- **Bulk Load**: `COPY INTO` with `FILE_FORMAT = (TYPE = JSON COMPRESSION = GZIP)`.
- **Resumable Uploads**: tus-protocol or chunked multipart for connection drop resilience.

---

## 10. Constraints

### 10.1 Snowflake MCP Constraints

| Constraint | Value | Mitigation |
|------------|-------|------------|
| Max tools per MCP server | 50 | Current design uses ~12 tools; headroom available |
| Response size limit | 250 KB | Tools return summary counts only, never large arrays |
| Supported tool type | GENERIC only | All logic in stored procedures |
| OAuth session role | `DEFAULT_ROLE` only | Dedicated `ACES_MCP_USER` role with full access |
| Non-streaming only | No real-time streaming | Acceptable for batch set-based operations |

### 10.2 Business Constraints

| Constraint | Description |
|------------|-------------|
| Reference data versions | File's declared VCDB/QDB/PCDB versions must be compatible with loaded versions |
| Full file model | Cannot process incremental updates; must handle full product-level files |
| Backward compatibility | Must support multiple ACES XML versions (3.x, 4.x) |

### 10.3 Data Characteristics

| Data Source | Update Frequency | Size Estimate |
|-------------|------------------|---------------|
| VCDB (vehicles) | Monthly | ~50K base vehicles |
| QDB (qualifiers) | Monthly | ~5K qualifiers |
| PCDB (parts config) | Monthly | ~2K part types, ~500 positions |
| Existing Fitments | Real-time | ~100K per job (varies by product scope) |

---

## 11. Security

> **Note**: User authentication, OAuth, and multi-tenant isolation are **deferred for the initial implementation**. The MVP will operate without user login or tenant separation. Security hardening is planned as a future phase (see Phase 9 below).

### 11.1 Data Access Security (MVP)

- **Job-ID Sandboxing**: All queries scoped to active `job_id`; enforced by BFF API layer.
- **No Raw SQL**: JobQueryAgent uses only parameterized tool definitions. No raw SQL execution.
- **Schema Segregation**: Staging tables isolated in `ACES_VALIDATION.STAGING`; chatbot role has read-only access to staging and reference tables.

### 11.2 Tool-Calling Guardrails (MVP)

- **Row-Count Limits**: Read-only tools enforce max 50 records per call.
- **Compressed Responses**: JSON payloads kept under 250 KB MCP limit.

### 11.3 Future Security Hardening (Nice-to-Have)

- **OAuth 2.0**: MCP client authenticates via Snowflake OAuth with dedicated `ACES_MCP_USER` role.
- **User Authentication**: JWT, OAuth 2.0/OIDC, or SSO for HITL UI login.
- **Multi-Tenant Isolation**: `user_id`/`tenant_id` on `JOB_RUN` with BFF enforcement.
- **Rate Limiting**: BFF enforces rate limits on chat and API endpoints.
- **CORS**: Proper cross-origin headers on BFF.

---

## 12. Reliability & Observability

### 12.1 Job Tracking

- `JOB_RUN` table tracks status transitions: `STAGED -> VALIDATING -> COMPARING -> COMPLETED | FAILED`.
- Error summaries persisted as VARIANT JSON for quick reporting.

### 12.2 Retry Strategy

- LangGraph checkpointing enables retry of failed nodes without reprocessing the full pipeline.
- Snowflake procedures are idempotent per `job_id` (DELETE + re-INSERT pattern for classifications).
- Failed nodes retried up to 3 times before marking job as FAILED.

### 12.3 Observability

- Redis Pub/Sub events for real-time pipeline step visibility.
- SSE streaming to frontend for user-facing progress.
- Snowflake query history for procedure execution auditing.

---

## 13. Performance Considerations

| Optimization | Approach |
|--------------|----------|
| **Set-Based SQL** | All validation/comparison via Snowflake JOINs and UPDATEs, not row-by-row |
| **Dedicated Warehouse** | `ACES_VALIDATION_WH` (MEDIUM size recommended) |
| **Clustering Keys** | `job_id`, `product_id`, `distinct_fitment_hash` on staging tables |
| **Single Parse** | XML parsed once during ingest; all subsequent operations on staged data |
| **Parallel Validation** | Independent validation procedures (VCDB, QDB, PCDB) can run in parallel LangGraph nodes |
| **Early Exit** | Vehicle resolution only runs for records still `is_valid = TRUE` |
| **Concurrent Jobs** | Multiple jobs run concurrently partitioned by `job_id` |

---

## 14. Implementation Phases

### Phase 1: Snowflake Foundation
- Create `ACES_VALIDATION` database with `AUTOCARE`, `CATALOG`, `STAGING`, `MCP` schemas
- Create reference data tables (VCDB, QDB, PCDB)
- Create catalog tables (Product, Brand, Fitment)
- Create staging tables (JOB_RUN, FITMENT_STAGE, FITMENT_CLASSIFICATION)
- Load reference data
- **Deliverable**: Populated Snowflake environment
- **Reference**: [Snowflake & MCP Design](./03-snowflake-mcp-design.md) Sections 1-2

### Phase 2: Stored Procedures
- VCDB validation procedures (base vehicle, attributes, vehicle resolution)
- QDB validation procedures (qualifier IDs)
- PCDB validation procedures (part type, position, mapping)
- Product resolution procedure
- Comparison procedures (classify changes, identify deletes)
- Aggregation procedures (validation summary, uplift summary)
- **Deliverable**: All stored procedures tested independently
- **Reference**: [Snowflake & MCP Design](./03-snowflake-mcp-design.md) Section 3

### Phase 3: MCP Server Configuration
- Create MCP SERVER object with GENERIC tools
- Configure basic access control (roles, grants)
- Test MCP tool invocation via REST
- _(OAuth security integration deferred to Phase 9)_
- **Deliverable**: Working MCP server with all tools accessible
- **Reference**: [Snowflake & MCP Design](./03-snowflake-mcp-design.md) Sections 4-7

### Phase 4: LangGraph Orchestrator
- Implement LangGraph state machine with all nodes
- SchemaValidationNode (local XSD)
- PreflightNode (version checks via MCP)
- StagingLoadNode (SAX parse + COPY INTO)
- ValidationNode (invoke MCP validation tools)
- ComparisonNode (invoke MCP comparison tools)
- StatisticsNode (invoke MCP aggregation tools)
- FinalStatusNode (job status + cleanup)
- Implement checkpointing and retry logic
- **Deliverable**: End-to-end pipeline processing test files
- **Reference**: [Multi-Agent Orchestration Design](./01-multi-agent-orchestration-design.md)

### Phase 5: Backend Gateway (BFF)
- File upload API with streaming SAX parser
- SSE endpoint for real-time progress
- Chat API with streaming response
- Redis Pub/Sub integration
- **Deliverable**: Working BFF serving frontend
- **Reference**: [HITL UI Design](./02-hitl-ui-design.md) Sections 3-6

### Phase 6: HITL Frontend
- Job History Dashboard
- Upload Modal with client-side preflight
- Real-Time Progress Tracker
- Split-Screen Analysis (dashboard + chat)
- **Deliverable**: Full HITL UI
- **Reference**: [HITL UI Design](./02-hitl-ui-design.md) Sections 1-2

### Phase 7: Conversational Engine
- JobQueryAgent implementation with system prompt
- 5 read-only tools (get_job_summary, get_failed_records, get_failed_records_by_cause, get_uplift_details, get_brand_or_part_performance)
- Streaming token response
- **Deliverable**: Working chat assistant
- **Reference**: [HITL UI Design](./02-hitl-ui-design.md) Section 5

### Phase 8: Integration Testing & Hardening
- End-to-end pipeline with realistic 100K-record files
- MCP connectivity testing
- Error handling and recovery testing
- Performance benchmarking and warehouse sizing
- **Deliverable**: Functional end-to-end system

### Phase 9: Security Hardening (Nice-to-Have / Future)
- OAuth security integration for Snowflake MCP
- User authentication for HITL UI (JWT/OAuth/SSO)
- Multi-tenant isolation (`user_id`/`tenant_id` on JOB_RUN)
- Rate limiting and CORS configuration on BFF
- Security review (cross-job isolation, SQL injection prevention)
- **Deliverable**: Production-hardened system

### Phase 10: A2A Agent Externalization (Post-MVP)
- Extract JobQueryAgent into standalone A2A-compliant service
- Publish Agent Card at `/.well-known/agent.json`
- Implement `tasks/send` and `tasks/sendSubscribe` (SSE) endpoints
- Add bearer token authentication and job-ID authorization
- Deploy as standalone service, integration test with external A2A clients
- **Deliverable**: Externally accessible query agent via A2A protocol
- **Reference**: [A2A Agent Design](./05-a2a-agent-design.md)

---

## 15. Open Items

| Item | Status | Notes |
|------|--------|-------|
| LangGraph language selection | TBD | Python vs TypeScript — depends on team skills |
| BFF framework selection | TBD | FastAPI (Python) vs Express (Node.js) |
| Performance SLA | TBD | Target processing time for 100K fitments |
| Warehouse sizing | TBD | SMALL vs MEDIUM for `ACES_VALIDATION_WH` |
| Report delivery format | TBD | CSV, Excel, JSON, or all three |
| User authentication for HITL UI | Deferred | Nice-to-have — JWT, OAuth, or SSO (Phase 9) |
| Multi-ACES-version XSD handling | TBD | Strategy for supporting 3.x and 4.x schemas |
| Large file bypass (>150MB) | TBD | Direct-to-cloud-stage upload path |
| Redis infrastructure | TBD | Managed Redis (e.g., ElastiCache) or self-hosted |
| Staging data retention policy | TBD | How long to keep job staging data |

---

*ACES Fitment Validation — End-to-End System Design v1.0*
