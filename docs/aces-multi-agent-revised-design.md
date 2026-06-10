# ACES Fitment Validation — Revised Scalable Agentic Design

> **Document Purpose**: Updated design recommendations for scalable, agentic processing under Snowflake MCP constraints
> **Last Updated**: June 2026

---

## 1. Goals and Non-Goals

### Goals
- Process **100K+ fitments per file** reliably within an acceptable SLA.
- Preserve **agentic orchestration** while pushing large-scale data operations into Snowflake.
- Stay within Snowflake MCP constraints (50 tools/server, 250 KB tool response limit).
- Maintain clear auditability, retry behavior, and job-level observability.

### Non-Goals
- Replacing Snowflake MCP with a custom server stack.
- Streaming, near-real-time ingestion; this is batch, full-file processing.

---

## 2. Key Constraints (Revalidated)

- **MCP response limit**: 250 KB per tool response.
- **Tools per MCP server**: 50 maximum.
- **OAuth**: Single role (`DEFAULT_ROLE`) only, no secondary roles.
- **Full file model**: All fitments for a supplier’s products in every submission.

Implication: **do not send fitment payloads over MCP**. All large data movement must occur via **Snowflake staging** and **set-based SQL**.

---

## 3. Revised Architecture Overview

### High-Level Flow (New)

```
ACES XML File
  |
  v
Schema Validation + Preflight (Agent)
  |
  v
Parse + Load to Snowflake Staging (Agent -> Snowflake)
  |
  v
Set-Based Validation in Snowflake
  |
  v
Set-Based Comparison (Add/Update/Delete) in Snowflake
  |
  v
Statistics + Uplift Summary in Snowflake
  |
  v
Report Output + Job Status (Agent)
```

### Architectural Principles (Revised)
1. **Agent manages orchestration, not row-level validation.**
2. **All heavy validation/comparison runs inside Snowflake** using set-based SQL.
3. **No large arrays over MCP**. MCP tools accept `job_id` and act on staging tables.
4. **Staging-first design**: ingest parsed ACES data once, reuse for validation and comparison.

---

## 4. Recommended Agent Framework

### Framework: **LangGraph** (Python or TypeScript)
**Why it fits**:
- DAG/state-machine orchestration matches deterministic pipeline stages.
- Native checkpointing enables recovery without reprocessing files.
- Parallel node execution for independent validation tasks.
- Straightforward MCP client integration for tool discovery/invocation.

---

## 5. Revised Agent Topology (LangGraph Nodes)

| Node | Responsibility | Notes |
|------|----------------|-------|
| **SchemaValidationNode** | XSD validation + ACES version detection | Local, non-MCP |
| **PreflightNode** | Version alignment, filename checks | MCP tool to read reference version tables |
| **StagingLoadNode** | Parse XML, load to Snowflake staging | COPY INTO or Snowflake connector |
| **ValidationNode** | Invoke set-based validation procedures | MCP tools, job-scoped |
| **ComparisonNode** | Invoke set-based comparison procedure | MCP tools, job-scoped |
| **StatisticsNode** | Compute uplift summary, validation summary | MCP tools, job-scoped |
| **ReportWriterNode** | Build report artifacts from Snowflake summary tables | Local |
| **FinalStatusNode** | Update job state + cleanup staging | Idempotent |

---

## 6. Staging-First Data Model (New)

### Composite Key & Collision Resolution
A major risk in ACES validation is primary key collision. Suppliers frequently submit multiple application (`<App>`) elements for the same product, base vehicle, part type, and position, separated only by distinct qualifiers or vehicle attributes (e.g., body style configurations, drive type, or qualifier notes).
*   **Legacy Key (Prone to Collision)**: `product_id + base_vehicle_id + part_type_id + position_id`
*   **Extended Composite Key (ECK) / Distinct Fitment Hash (DFH)**: A unique, deterministic hash representing the complete fitment configuration:
    `MD5(CONCAT_WS('|', COALESCE(product_id::STRING, ''), base_vehicle_id::STRING, part_type_id::STRING, COALESCE(position_id::STRING, '0'), HASH(vehicle_conditions)::STRING, HASH(qualifiers)::STRING, HASH(notes)::STRING))`
    By utilizing the DFH, the system supports multiple distinct fitments on the same vehicle configuration without collisions, enabling accurate change detection (ADD/UPDATE/DELETE).

### Core Staging Tables
All staging tables reside in `ACES_VALIDATION.STAGING` and are scoped by `job_id`.

#### 1. `JOB_RUN`
Tracks individual execution runs.
*   `job_id` VARCHAR PRIMARY KEY
*   `status` VARCHAR (e.g., 'STAGED', 'VALIDATING', 'COMPARING', 'COMPLETED', 'FAILED')
*   `file_path` VARCHAR
*   `line_code` VARCHAR
*   `total_records` INTEGER
*   `error_summary` VARIANT
*   `created_at` TIMESTAMP
*   `updated_at` TIMESTAMP

#### 2. `FITMENT_STAGE`
The primary table where parsed XML data is loaded.
*   `job_id` VARCHAR
*   `app_id` VARCHAR (from the `<App id="...">` XML attribute)
*   `line_code` VARCHAR
*   `supplier_part_number` VARCHAR
*   `product_id` INTEGER (resolved post-load)
*   `base_vehicle_id` INTEGER
*   `part_type_id` INTEGER
*   `position_id` INTEGER
*   `qualifiers` VARIANT (JSON array of qualifier IDs and parameters)
*   `notes` VARIANT (JSON array of free-text note strings)
*   `vehicle_conditions` VARIANT (JSON object of key-value attribute configurations, e.g., `{"DriveType": [3, 4]}`)
*   `distinct_fitment_hash` VARCHAR (Extended Composite Key calculated during load)
*   `is_valid` BOOLEAN DEFAULT TRUE
*   `error_type` VARCHAR (VCDB, QDB, PCDB, BRAND, PRODUCT, SYSTEM)
*   `error_message` VARCHAR
*   `created_at` TIMESTAMP

#### 3. `FITMENT_CLASSIFICATION`
Holds results from change-detection comparison.
*   `job_id` VARCHAR
*   `distinct_fitment_hash` VARCHAR
*   `classification` VARCHAR (ADD, UPDATE, DELETE, UNCHANGED)
*   `change_details` VARCHAR (e.g., 'qualifiers_changed', 'notes_changed', 'conditions_changed')
*   `created_at` TIMESTAMP

### Ingestion Approach (Streaming SAX + COPY INTO)
*   **SAX-Based Streaming Parser**: The agent local process uses a SAX (or StAX) streaming XML parser to process the 100K+ records. It extracts the raw vehicle attributes, qualifiers, and note collections for each `<App>` block without holding the entire file in-memory.
*   **JSON Stage files**: The parser produces a compressed line-delimited JSON stage file containing the fields corresponding to `FITMENT_STAGE`.
*   **Snowflake Stage Upload**: The client uploads this staging file to a Snowflake internal stage and runs a bulk `COPY INTO ACES_VALIDATION.STAGING.FITMENT_STAGE` command. This maximizes write throughput and minimizes client-side memory usage to <100MB.

---

## 7. MCP Tool Redesign (Set-Based)

### Old Pattern (Not Scalable)
- Tools accept arrays of IDs/fitments and return arrays.

### New Pattern (Scalable)
- Tools accept `job_id` and operate on staging tables.
- Tools return small summary objects only.

### Example Tool Signatures (Recommended)
| Tool | Inputs | Output |
|------|--------|--------|
| `validate_base_vehicle_ids_job` | `job_id` | `{ invalid_count }` |
| `validate_qualifier_ids_job` | `job_id` | `{ invalid_count }` |
| `validate_part_type_position_job` | `job_id` | `{ invalid_count }` |
| `resolve_vehicle_configuration_job` | `job_id` | `{ invalid_count }` |
| `validate_product_resolution_job` | `job_id` | `{ invalid_count }` |
| `classify_fitment_changes_job` | `job_id` | `{ add, update, delete, unchanged }` |
| `compute_job_statistics` | `job_id` | `{ totals... }` |
| `fetch_job_errors` | `job_id` | `{ error_sample, error_counts }` |

---

## 8. Revised Validation Strategy (Set-Based)

**Principle**: Perform validation with UPDATE statements and joins inside Snowflake.

### Example Workflow
1. **VCDB Checks**: Mark invalid base vehicles in `FITMENT_STAGE`.
2. **PCDB Checks**: Mark invalid part types/positions/mappings.
3. **QDB Checks**: Mark invalid qualifier IDs.
4. **Vehicle Resolution**: Only run for records still `is_valid = TRUE`.
5. **Product Resolution**: Resolve supplier part numbers to product_id in bulk.

**All logic remains inside Snowflake**, so MCP responses are small and deterministic.

---

## 9. Revised Comparison Strategy (Set-Based)

### Key Change
- No in-memory indexing in the agent.
- Entire comparison runs in Snowflake using joins and anti-joins.

### Summary
- **ADD/UPDATE/UNCHANGED** via join between staging and catalog.
- **DELETE** via anti-join against existing catalog fitments for the same product set.

---

## 10. Scalability & Throughput Considerations

### Performance Tuning
- Use **dedicated warehouse** for validation (`ACES_VALIDATION_WH`), consider `MEDIUM`.
- Apply **clustering keys** on `job_id`, `product_id`, `composite_key` in staging tables.
- Parse XML once; **avoid repeated batch roundtrips** to MCP.

### MCP Constraints Avoidance
- Tool responses only include aggregated summaries (counts, sampling).
- No record arrays returned over MCP.

### Concurrency
- Multiple jobs can run concurrently by partitioning on `job_id`.
- Use **Snowflake tasks** for long-running validation/compare steps if needed.

---

## 11. Reliability and Observability

### Job Tracking
- `JOB_RUN` table with status, timestamps, retry count, and checkpoint state.

### Retry Strategy
- Retries only re-run failed nodes (LangGraph checkpointing).
- Snowflake procedures must be idempotent for a `job_id`.

### Error Reporting
- Persist error counts and top-N error samples for quick reporting.

---

## 12. Security and Access Control

- Use a dedicated MCP OAuth role (`ACES_MCP_USER`) with least privilege.
- Default role must include access to staging and procedure execution.
- Avoid embedding credentials in reports or logs.

---

## 13. Implementation Plan (Revised)

1. **Staging Schema Extensions**: Add required columns and indexes.
2. **Procedure Rewrite**: Convert validation and comparison to set-based jobs.
3. **LangGraph Orchestrator**: Implement nodes and checkpointing.
4. **Integration Testing**: End-to-end job for 100K fitments.

---

## 14. Open Items

- Finalize XML parsing strategy (streaming SAX vs DOM).
- Determine expected SLA and warehouse sizing.
- Define report delivery (CSV/Excel/JSON).

---

*ACES Fitment Validation — Revised Scalable Agentic Design*
