# ACES Fitment Validation — Multi-Agent Orchestration Design

> **Document Purpose**: Detailed design for the LangGraph-based multi-agent orchestration pipeline  
> **Parent Document**: [E2E System Design](./00-e2e-design.md)  
> **Version**: 1.0  
> **Last Updated**: June 2026

---

## 1. Framework Selection: LangGraph

### 1.1 Why LangGraph

| Capability | Fit |
|------------|-----|
| DAG/state-machine orchestration | Matches deterministic pipeline stages |
| Native checkpointing | Recovery without reprocessing files |
| Parallel node execution | Independent validation tasks run concurrently |
| MCP client integration | Tool discovery and invocation for Snowflake MCP |
| Language support | Python or TypeScript |

### 1.2 Design Principles

1. **Agent manages orchestration, not row-level data.** Agents coordinate pipeline stages; all bulk work runs inside Snowflake.
2. **No large arrays over MCP.** Tools accept `job_id` and act on staging tables. Responses are small summary objects.
3. **Staging-first.** Parse XML once into Snowflake; reuse for validation and comparison.
4. **Idempotent procedures.** Every stored procedure can be re-run for a `job_id` safely (DELETE + re-INSERT or conditional UPDATE).

---

## 2. Agent Topology (LangGraph Nodes)

### 2.1 Node Overview

| Node | Responsibility | MCP? | Notes |
|------|----------------|------|-------|
| **SchemaValidationNode** | XSD validation + ACES version detection | No | Local processing |
| **PreflightNode** | Version alignment, filename checks | Yes | Reads reference version tables |
| **StagingLoadNode** | Parse XML, load to Snowflake staging | No | COPY INTO via Snowflake connector |
| **ValidationNode** | Invoke set-based validation procedures | Yes | Parallel sub-tasks |
| **ComparisonNode** | Invoke set-based comparison procedures | Yes | Job-scoped |
| **StatisticsNode** | Compute uplift summary, validation summary | Yes | Job-scoped |
| **ReportWriterNode** | Build report artifacts from Snowflake summaries | No | Local file generation |
| **FinalStatusNode** | Update job state + cleanup staging | No | Idempotent |

### 2.2 Pipeline State Machine

```
                    START
                      |
                      v
           SchemaValidationNode
              |             |
           [PASS]        [FAIL] --> Archive file, set FAILED, END
              |
              v
           PreflightNode
              |             |
           [PASS]        [FAIL] --> Generate precheck report, set FAILED, END
              |
              v
           StagingLoadNode
              |             |
           [LOADED]      [FAIL] --> Set FAILED, END
              |
              v
           ValidationNode  (parallel sub-tasks for VCDB, QDB, PCDB, Product)
              |
              v
           ComparisonNode  (only valid fitments)
              |
              v
           StatisticsNode
              |
              v
           ReportWriterNode
              |
              v
           FinalStatusNode --> Set COMPLETED or COMPLETED_WITH_ERRORS, END
```

### 2.3 LangGraph State Schema

The shared state object passed between nodes:

```python
class ACESValidationState(TypedDict):
    job_id: str
    file_path: str
    line_code: str
    aces_version: str
    total_records: int
    status: str  # STAGED, VALIDATING, COMPARING, COMPLETED, FAILED
    schema_valid: bool
    preflight_passed: bool
    staging_loaded: bool
    validation_summary: dict  # { valid_count, error_count, vcdb_errors, ... }
    comparison_summary: dict  # { add_count, update_count, delete_count, unchanged_count }
    uplift_summary: dict
    error_message: Optional[str]
    retry_count: int
```

---

## 3. Node Specifications

### 3.1 SchemaValidationNode

**Purpose**: Validate the ACES XML file against the correct XSD schema.

**Execution**:
1. Extract ACES version from XML root element (`version` attribute)
2. Fetch the matching XSD schema file (local storage, supports 3.x and 4.x)
3. Execute full XSD validation
4. Parse the ACES XML header for metadata (company name, transfer date, submission type, version dates)

**Tools Used** (local, non-MCP):

| Tool | Input | Output |
|------|-------|--------|
| `extractAcesVersionFromXml` | XML content | Version string |
| `fetchXsdSchemaForVersion` | Version string | XSD content |
| `executeXsdValidation` | XML + XSD content | `{ is_valid, errors[] }` |
| `parseAcesXmlHeader` | XML content | Header metadata object |

**Decision Logic**:
- If XSD validation fails: archive file, set job status to `FAILED`, halt pipeline.
- If passes: store version and header metadata in state, proceed to PreflightNode.

### 3.2 PreflightNode

**Purpose**: Verify that file metadata is consistent and reference data versions are compatible.

**Execution**:
1. Construct expected filename from header fields per ACES naming convention
2. Compare expected vs actual filename
3. Verify submission type consistency across filename suffix, metadata, and XML header
4. Check if file's declared VCDB/QDB/PCDB version dates are compatible with loaded reference data
5. Detect if Delete actions exist in a FULL submission (warning condition)

**Tools Used** (local + MCP for version check):

| Tool | Input | Output |
|------|-------|--------|
| `constructExpectedAcesFilename` | Header fields | Expected filename |
| `compareAcesFilenames` | Expected + actual | `{ match }` |
| `verifySubmissionTypeConsistency` | Three submission type sources | `{ consistent }` |
| `detectDeleteActionsInFullSubmission` | `job_id` | `{ deletes_found, count }` |

**Decision Logic**:
- If critical preflight failures: generate precheck Excel report, cleanup staging, halt pipeline.
- Version mismatches: proceed with warnings if minor; halt if incompatible.

### 3.3 StagingLoadNode

**Purpose**: Parse the XML file and bulk-load fitment records into Snowflake staging.

**Execution**:
1. Initialize SAX/StAX streaming XML parser
2. For each `<App>` block:
   - Extract: base_vehicle_id, part_type_id, position_id, supplier_part_number, qualifiers, notes, vehicle_conditions
   - Compute Distinct Fitment Hash (DFH)
   - Flatten to JSON record
3. Batch records into 10K-record JSONL files (Gzip compressed)
4. Upload batches to Snowflake internal stage (`@ACES_STAGING_STAGE/job_id/`)
5. Execute `COPY INTO ACES_VALIDATION.STAGING.FITMENT_STAGE`
6. Register job in `JOB_RUN` table with status `STAGED`

**DFH Computation**:
```
MD5(CONCAT_WS('|',
    COALESCE(product_id, ''),
    base_vehicle_id,
    part_type_id,
    COALESCE(position_id, '0'),
    HASH(vehicle_conditions),
    HASH(qualifiers),
    HASH(notes)
))
```

**Performance**: 100K records staged in <10 seconds via parallel COPY INTO.

### 3.4 ValidationNode

**Purpose**: Execute set-based validation procedures against Snowflake reference data.

**Execution Strategy**: Invoke MCP tools that run stored procedures operating on `FITMENT_STAGE` rows for the given `job_id`. Each procedure marks invalid records in-place with `is_valid = FALSE`, `error_type`, and `error_message`.

**Validation Order** (respects dependencies):

```
Phase 1 — ID Existence Checks (parallel):
  +-- validate_base_vehicle_ids_job(job_id)      -- VCDB
  +-- validate_part_type_ids_job(job_id)          -- PCDB
  +-- validate_position_ids_job(job_id)           -- PCDB
  +-- validate_qualifier_ids_job(job_id)          -- QDB
  +-- validate_attribute_values_job(job_id)       -- VCDB

Phase 2 — Relationship Checks (after Phase 1):
  +-- validate_part_type_position_job(job_id)     -- PCDB mapping

Phase 3 — Vehicle Resolution (conditional, after Phase 2):
  +-- resolve_vehicle_configuration_job(job_id)   -- Only for is_valid = TRUE records

Phase 4 — Product Resolution:
  +-- resolve_products_job(job_id, line_code)      -- Resolve supplier parts to product_id
```

**Parallel Execution**: Phase 1 tools are independent and can run as parallel LangGraph sub-tasks. Phases 2-4 are sequential because each depends on prior results.

**Decision Logic**:
- After all validation: update `JOB_RUN.status` to `VALIDATING_COMPLETE`.
- Proceed to ComparisonNode regardless of error count (comparison runs on valid records only).

### 3.5 ComparisonNode

**Purpose**: Compare validated incoming fitments against existing catalog fitments to classify changes.

**Execution**:
1. Invoke `classify_fitment_changes_job(job_id)`:
   - LEFT JOINs `FITMENT_STAGE` (valid only) against `CATALOG.FITMENT` on `distinct_fitment_hash`
   - Classifies each as ADD, UPDATE, or UNCHANGED
   - Writes to `FITMENT_CLASSIFICATION`
2. Invoke `identify_deleted_fitments_job(job_id)`:
   - Anti-joins `CATALOG.FITMENT` against `FITMENT_STAGE` for products in the job
   - Identifies existing fitments not in incoming = DELETE
   - Appends to `FITMENT_CLASSIFICATION`

**Classification Logic**:

| Scenario | Classification | SQL Pattern |
|----------|---------------|-------------|
| DFH not in catalog | ADD | LEFT JOIN miss |
| DFH match, VARIANT columns equal | UNCHANGED | Equality comparison |
| DFH match, VARIANT columns differ | UPDATE | Inequality comparison |
| DFH in catalog, not in staging | DELETE | Anti-join |

**Change Details Tracking**: For UPDATEs, the procedure records what changed (qualifiers_changed, notes_changed, conditions_changed).

### 3.6 StatisticsNode

**Purpose**: Compute aggregated statistics for the validation run and uplift summary.

**Execution**:
1. Invoke `compute_validation_summary(job_id)`:
   - Returns: total_fitments, valid_count, error_count, vcdb_errors, qdb_errors, pcdb_errors, product_errors
2. Invoke `compute_uplift_summary(job_id)`:
   - Returns: add_count, update_count, delete_count, unchanged_count

**Output**: Both summaries stored in LangGraph state and persisted to `JOB_RUN.error_summary`.

### 3.7 ReportWriterNode

**Purpose**: Generate report artifacts from the computed statistics.

**Output Formats** (TBD — one or more of):
- JSON summary document
- Excel uplift report
- CSV of failed records

### 3.8 FinalStatusNode

**Purpose**: Set the final job status and perform cleanup.

**Logic**:
```
if validation_summary.error_count == 0:
    status = "COMPLETED"
elif validation_summary.valid_count > 0:
    status = "COMPLETED_WITH_ERRORS"
else:
    status = "FAILED"
```

**Actions**:
- Update `JOB_RUN.status` and `updated_at`
- Archive original XML file to archive bucket
- Publish final SSE event via Redis

---

## 4. Error Handling & Retry

### 4.1 Retry Strategy

LangGraph's native checkpointing enables node-level retry:

| Failure Type | Strategy |
|--------------|----------|
| MCP tool invocation failure | Retry the specific tool call up to 3 times with exponential backoff |
| Snowflake procedure error | Retry the node; procedures are idempotent per `job_id` |
| Staging load failure | Retry StagingLoadNode; COPY INTO is idempotent |
| Network/timeout | Retry with longer timeout on subsequent attempts |

### 4.2 Failure Modes

| Node | Failure Mode | Recovery |
|------|-------------|----------|
| SchemaValidationNode | Invalid XSD | Archive file, halt, mark FAILED |
| PreflightNode | Version mismatch | Generate warning report, halt or proceed based on severity |
| StagingLoadNode | Parse error | Abort at error point, mark FAILED with line number |
| ValidationNode | Procedure timeout | Retry node; staged data remains intact |
| ComparisonNode | Procedure error | Retry node; classification table is cleared and re-populated |
| FinalStatusNode | Status write failure | Retry; idempotent update |

### 4.3 Idempotency Guarantees

All stored procedures follow patterns that ensure re-execution safety:
- **Validation procedures**: UPDATE with `WHERE is_valid = TRUE` — already-marked records are not re-processed.
- **Classification procedures**: `DELETE FROM FITMENT_CLASSIFICATION WHERE job_id = :job_id` before re-inserting.
- **Aggregation procedures**: Pure read queries; no side effects.

---

## 5. Observability

### 5.1 Status Transitions

```
STAGED --> SCHEMA_VALIDATING --> PREFLIGHT_CHECKING --> LOADING
  --> VALIDATING --> COMPARING --> COMPUTING_STATS --> COMPLETED
                                                   --> COMPLETED_WITH_ERRORS
  (any node) --> FAILED
```

### 5.2 Event Publishing

Each node publishes status events to Redis on entry and exit:

```json
{
  "job_id": "job_01h9ybc93k018s96abc491pq",
  "node": "ValidationNode",
  "status": "IN_PROGRESS",
  "message": "Performing set-based validation across VCDB, PCDB, and QDB...",
  "percentage": 45,
  "timestamp": "2026-06-10T20:58:00.000Z"
}
```

These events are relayed to the HITL frontend via SSE.

---

## 6. Scalability Considerations

### 6.1 Parallel Validation

Phase 1 validation tools (base vehicle, part type, position, qualifier, attribute) are independent and can execute as parallel LangGraph sub-tasks, reducing wall-clock time.

### 6.2 Warehouse Sizing

| Workload | Recommended Size |
|----------|-----------------|
| Standard validation (100K records) | MEDIUM |
| Light validation (<10K records) | SMALL |
| Multiple concurrent jobs | MEDIUM with auto-scaling |

### 6.3 Concurrent Jobs

Multiple jobs can run simultaneously. All staging tables are partitioned by `job_id`, ensuring no cross-job interference. LangGraph manages separate state machines per job.

---

## 7. References

| Document | Relevance |
|----------|-----------|
| [E2E System Design](./00-e2e-design.md) | Master architecture context |
| [Snowflake & MCP Design](./03-snowflake-mcp-design.md) | Stored procedure implementations, MCP server configuration |
| [HITL UI Design](./02-hitl-ui-design.md) | Frontend integration, SSE events, chat agent |
| [ACES 4.2 Fitment Attributes](./aces-4-2-fitment-attributes.md) | ACES XML structure reference |

---

*ACES Fitment Validation — Multi-Agent Orchestration Design v1.0*
