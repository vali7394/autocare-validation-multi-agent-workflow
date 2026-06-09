# ACES Fitment Validation — Multi-Agent Orchestration Architecture

> **Target Throughput**: ~100,000 fitment records per file  
> **Architecture Pattern**: Agentic orchestration with Tools, MCP Servers, and Agent-Native Logic  
> **Data Platform**: Snowflake (reference data + existing fitments)

---

## Section 1: Functional Overview

This specification defines a multi-agent system for ACES fitment validation and change detection.

### 1.1 Processing Model — Full File Submissions

**Suppliers always send complete product-level fitment files**. Each file contains ALL fitments for the products included — not incremental updates. The system must:

1. **Validate** all incoming fitments against Autocare reference data (VCDB, QDB, PCDB)
2. **Compare** validated fitments against existing fitments in Snowflake
3. **Classify** each fitment as Add, Update, or Delete for the uplift report

| Processing Stage | Input | Output | Data Source |
|------------------|-------|--------|-------------|
| **1. Schema Validation** | Raw XML file | Pass/Fail with errors | XSD schemas |
| **2. Preflight Checks** | XML header metadata | Pass/Fail with report | Reference versions |
| **3. Fitment Validation** | Parsed fitment records | Valid/Invalid per fitment | Snowflake (VCDB, QDB, PCDB) |
| **4. Comparison** | Validated fitments | Add/Update/Delete classification | Snowflake (existing fitments) |
| **5. Uplift Report** | Classified fitments | Summary statistics | Computed from comparison |

### 1.2 What Gets Validated

Each fitment record undergoes these validation checks:

| Validation Category | What It Checks | Error Type |
|---------------------|----------------|------------|
| **Base Vehicle** | Base Vehicle ID exists in VCDB | VCDB ERROR |
| **Part Type** | Part Type ID exists in PCDB | PCDB ERROR |
| **Position** | Position ID exists in PCDB | PCDB ERROR |
| **Part-Type-Position Mapping** | Part Type + Position is valid combination | PCDB ERROR |
| **Qualifiers** | All Qualifier IDs exist in QDB | QDB ERROR |
| **Vehicle Attributes** | Attribute values (DriveType, FuelType, etc.) exist in VCDB domains | VCDB ERROR |
| **Valid Vehicle Resolution** | Base Vehicle + all attributes resolves to actual vehicle | VCDB ERROR |
| **Brand/SubBrand** | Brand and SubBrand IDs exist | BRAND WARNING |
| **Product Resolution** | Supplier part number maps to known product | PRODUCT ERROR |

### 1.3 Comparison Logic (Post-Validation)

Since files are FULL submissions, we compare incoming vs existing to classify changes:

| Scenario | Classification | Description |
|----------|---------------|-------------|
| Key not in existing | **ADD** | New fitment for this product/vehicle/part/position |
| Key in existing, details identical | **UNCHANGED** | No action needed |
| Key in existing, details differ | **UPDATE** | Qualifiers, notes, or vehicle conditions changed |
| Key in existing, not in incoming | **DELETE** | Supplier removed this fitment from product |

**Composite Key**: `product_id + base_vehicle_id + part_type_id + position_id`

### 1.4 Architectural Layers

| Layer | Responsibility | Implementation |
|-------|----------------|----------------|
| **Snowflake Managed MCP Server** | Exposes tools via standard MCP protocol | Single MCP SERVER object with GENERIC tools |
| **Snowflake Stored Procedures** | Validation logic, data queries, comparison | UDFs/procedures invoked as MCP tools |
| **Agent Client** | Orchestration, conditional branching, failure recovery | MCP client that discovers and invokes tools |

> **Key Insight**: Snowflake provides a **managed MCP server** — no custom server infrastructure needed. Tools are Snowflake stored procedures exposed via the `GENERIC` tool type. See [MCP Setup Guide](./aces-mcp-snowflake-setup.md) for configuration.

---

## Section 2: Agent Topology

### 2.1 Primary Coordinator — FitmentJobOrchestrator

The orchestrator coordinates the validation pipeline:
- Receives job requests with fitment file path and line code
- Manages workflow state transitions
- Delegates to specialized agents
- Aggregates results for reporting

### 2.2 Specialized Agents

| Agent | Responsibility | When It Runs |
|-------|---------------|--------------|
| **SchemaValidationAgent** | XSD conformance, ACES version detection | First — gate for all processing |
| **PreflightAgent** | Version alignment, filename validation | After schema validation |
| **FitmentValidationAgent** | Per-record validation using tiered strategy | After preflight |
| **FitmentComparisonAgent** | Compares incoming vs existing in Snowflake, classifies Add/Update/Delete | After validation (only valid fitments) |
| **StatisticsAgent** | Computes aggregations and generates uplift report | Final step |

---

## Section 3: Snowflake Managed MCP Server

Snowflake provides a **managed MCP server** that exposes tools via the standard Model Context Protocol. No custom server infrastructure is required — tools are Snowflake stored procedures/UDFs exposed via the `GENERIC` tool type.

> **Setup**: See [MCP Setup Guide](./aces-mcp-snowflake-setup.md) for complete SQL scripts and OAuth configuration.

### 3.1 Data Platform

All reference data and existing fitments reside in Snowflake:

| Data Category | Snowflake Schema | Purpose |
|---------------|------------------|---------|
| VCDB (vehicles) | `AUTOCARE.VCDB` | Base vehicle IDs, attribute domains, vehicle configurations |
| QDB (qualifiers) | `AUTOCARE.QDB` | Qualifier IDs and metadata |
| PCDB (parts config) | `AUTOCARE.PCDB` | Part types, positions, mappings |
| Existing Fitments | `CATALOG.FITMENTS` | Current fitments for Add/Update/Delete comparison |
| Product Catalog | `CATALOG.PRODUCTS` | Supplier part to product mapping |

### 3.2 MCP Server Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent Client (MCP Client)                │
│         Discovers tools, invokes via tools/call             │
└─────────────────────┬───────────────────────────────────────┘
                      │ OAuth 2.0
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Snowflake Managed MCP Server                   │
│         ACES_VALIDATION.MCP.ACES_FITMENT_SERVER            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   GENERIC Tools                      │   │
│  │  (Backed by Stored Procedures in ACES_VALIDATION)   │   │
│  │                                                      │   │
│  │  • validate_base_vehicle_ids_batch                  │   │
│  │  • validate_qualifier_ids_batch                     │   │
│  │  • validate_part_type_position_batch                │   │
│  │  • resolve_vehicle_configuration_batch              │   │
│  │  • fetch_existing_fitments_by_products              │   │
│  │  • classify_fitment_changes_batch                   │   │
│  │  • ...                                              │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                    Snowflake Tables                         │
│  AUTOCARE.VCDB | AUTOCARE.QDB | AUTOCARE.PCDB | CATALOG.*  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Tool Categories Exposed via MCP

| Category | Tools | Backing Procedure |
|----------|-------|-------------------|
| **VCDB Validation** | `validate_base_vehicle_ids_batch`, `validate_attribute_values_batch`, `resolve_vehicle_configuration_batch` | Queries `AUTOCARE.VCDB` |
| **QDB Validation** | `validate_qualifier_ids_batch` | Queries `AUTOCARE.QDB` |
| **PCDB Validation** | `validate_part_type_ids_batch`, `validate_position_ids_batch`, `validate_part_type_position_batch` | Queries `AUTOCARE.PCDB` |
| **Comparison** | `fetch_existing_fitments_by_products`, `classify_fitment_changes_batch`, `identify_deleted_fitments` | Queries `CATALOG.FITMENTS` |
| **Product Resolution** | `resolve_products_batch`, `validate_brand_subbrand_batch` | Queries `CATALOG.PRODUCTS` |
| **Aggregation** | `compute_validation_summary`, `compute_uplift_summary` | Aggregation queries |

### 3.4 Snowflake MCP Server Constraints

| Constraint | Value | Implication |
|------------|-------|-------------|
| Max tools per server | 50 | Split into multiple MCP servers if needed |
| Response size limit | 250 KB | Batch results must stay under limit |
| Supported tool type | `GENERIC` | All validation logic in stored procedures |
| Authentication | OAuth 2.0 | User's `DEFAULT_ROLE` used for session |

### 3.5 File and Staging Operations

File operations and staging database access are handled **outside the MCP server** by the agent client directly:

| Operation | Implementation |
|-----------|----------------|
| Read XML file | Agent reads from GCS/S3 via SDK |
| Parse/stage fitments | Agent loads to staging table via JDBC/Snowflake connector |
| Archive file | Agent moves file to archive bucket |
| Write report | Agent writes to output location |

---

## Section 4: Callable Tools

Tools are stateless functions that agents invoke for deterministic operations.

### 4.1 ACES Schema Validation Tools

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **extractAcesVersionFromXml** | XML content as string | Version string or error details | Parses ACES root element to find version attribute |
| **fetchXsdSchemaForVersion** | ACES version string | XSD content string or null if unsupported | Retrieves matching XSD schema file for the version |
| **executeXsdValidation** | XML content plus XSD content | Object containing validity boolean and array of validation errors with line number, column number, and message | Performs full schema conformance check |
| **parseAcesXmlHeader** | XML content | Object containing company name, document title, transfer date, submission type, and version dates for VCDB/PCDB/QDB | Extracts header metadata fields |

### 4.2 Preflight Check Tools

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **constructExpectedAcesFilename** | Company name, document title, transfer date, submission type | Expected filename string | Builds canonical filename from header fields per ACES spec |
| **compareAcesFilenames** | Expected name plus actual name | Object with match boolean and both names | Simple string equality check |
| **verifySubmissionTypeConsistency** | Filename suffix, metadata submission type, XML header submission type | Object with consistency boolean and all three source values | Three-source cross-validation |
| **detectDeleteActionsInFullSubmission** | Job identifier | Object with deletes-found boolean, delete count, total fitment count, and delete percentage | Scans staging for Delete actions in FULL submission |
| **generatePrecheckExcelReport** | Array of check results plus version comparison data | Excel file as bytes | Produces formatted Excel report for preflight failures |

### 4.3 Fitment Validation Tools

**VCDB Validation Tools**:

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **validateBaseVehicleIdsBatch** | Array of base vehicle IDs | Array of validity booleans | Checks base vehicle IDs exist in VCDB |
| **validateAttributeValuesBatch** | Array of (domain, value) pairs | Array of validity booleans | Checks attribute values exist in VCDB domains |
| **resolveVehicleConfigurationBatch** | Array of (baseVehicleId, attributeMap) | Array of (isValid, matchingVehicleCount) | Resolves complete vehicle configuration |

**QDB Validation Tools**:

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **validateQualifierIdsBatch** | Array of qualifier IDs | Array of validity booleans | Checks qualifier IDs exist in QDB |

**PCDB Validation Tools**:

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **validatePartTypeIdsBatch** | Array of part type IDs | Array of validity booleans | Checks part type IDs exist in PCDB |
| **validatePositionIdsBatch** | Array of position IDs | Array of validity booleans | Checks position IDs exist in PCDB |
| **validatePartTypePositionBatch** | Array of (partTypeId, positionId) pairs | Array of validity booleans | Checks part-type-to-position mapping is valid |

**Brand/Product Validation Tools**:

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **validateBrandSubBrandBatch** | Array of (brandId, subBrandId) pairs | Array of validity booleans | Checks brand-subbrand mapping |
| **validateProductResolutionBatch** | Line code plus array of fitment records | Array of validation results | Maps supplier parts to products |

**Result Aggregation Tools**:

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **groupValidationResultsByDomain** | All validation results | Object with VCDB/QDB/PCDB issues separated | Categorizes errors by source database |
| **formatFitmentErrorMessage** | Grouped issues object | Formatted error string | Builds error message with domain prefixes |

### 4.4 Fitment Comparison Tools (Post-Validation)

Since suppliers send FULL files containing all fitments for their products, we must compare incoming validated fitments against existing fitments in Snowflake to determine Add/Update/Delete classification.

**Composite Key**: `product_id + base_vehicle_id + part_type_id + position_id`

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **buildExistingFitmentIndex** | Existing fitments from Snowflake | Indexed map by composite key | Creates O(1) lookup for existing fitments |
| **buildIncomingFitmentIndex** | Validated incoming fitments | Indexed map by composite key | Creates O(1) lookup for incoming fitments |
| **classifyFitmentChange** | Incoming fitment, existing fitment index | ADD, UPDATE, or UNCHANGED | Determines if fitment is new or modified |
| **identifyDeletedFitments** | Existing fitment index, incoming fitment index | Array of deleted composite keys | Fitments in existing but not in incoming = DELETE |
| **compareVehicleConditions** | Conditions object A plus conditions object B | Object with identical boolean and array of differences | Deep comparison of all vehicle condition attributes |
| **compareNoteCollections** | Notes array A plus notes array B | Object with identical boolean and reorder-only boolean | Order-aware note comparison |
| **compareQualifierCollections** | Qualifiers array A plus qualifiers array B | Object with identical boolean and reorder-only boolean | Order-aware qualifier comparison |
| **classifyFitmentBatch** | Array of validated fitments plus existing index | Array of (classification, changeDetails) | Bulk classification for performance |

**Classification Logic**:

| Scenario | Classification | Condition |
|----------|---------------|-----------|
| Key not in existing | **ADD** | New fitment for this product/vehicle/part/position |
| Key in existing, details identical | **UNCHANGED** | No action needed |
| Key in existing, details differ | **UPDATE** | Qualifiers, notes, or vehicle conditions changed |
| Key in existing, not in incoming | **DELETE** | Supplier removed this fitment from product |

### 4.5 Statistics Computation Tools

| Tool Name | Input Parameters | Output | Purpose |
|-----------|------------------|--------|---------|
| **computeFileLevelSummary** | Job identifier | Object containing add count, existing count, error count, delete count, total count, replacement delete count, retained count, additional existing count | Aggregates staging table counts |
| **computeUpliftCategorySummary** | Job identifier | Object containing new part YMM count, existing part YMM count, reordered count, added QDB notes count, exact match count, pre-load count | Aggregates sub-record type counts |
| **buildStatisticsJsonDocument** | File summary plus uplift summary | JSONB document | Combines both summaries into final storage format |
| **persistJobStatistics** | Job identifier plus statistics JSON | Boolean success indicator | Upserts to file_statistics table |

---

## Section 5: Agent-Native Logic — Decision Layer

The following logic resides within the agents because it requires conditional branching, dynamic adaptation, and failure recovery decisions.

### 5.1 FitmentJobOrchestrator — Workflow Coordination

**Step 1 — Request Intake**: Receive job identifier and file path. Check idempotency — skip if already processed.

**Step 2 — Schema Validation**: Delegate to SchemaValidationAgent. On failure, archive file and halt.

**Step 3 — Preflight**: Delegate to PreflightAgent. On failure, generate report, cleanup staging, halt.

**Step 4 — Batch Strategy**: Determine batch configuration based on fitment count (50 batches of 2K for 100K fitments).

**Step 5 — Validation**: Delegate to FitmentValidationAgent with parallel execution. Retry failed batches up to 3 times.

**Step 6 — Comparison** (Post-Validation): Delegate to FitmentComparisonAgent to compare validated fitments against existing fitments in Snowflake. Classify each as Add/Update/Delete.

**Step 7 — Uplift Report**: Delegate to StatisticsAgent to compute summary and generate uplift report.

**Step 8 — Final Status**: Compute SUCCESS, COMPLETED_WITH_ERRORS, or FAILED.

### 5.3 FitmentComparisonAgent — Add/Update/Delete Classification

This agent runs AFTER validation completes. Since suppliers send FULL files, we must compare incoming vs existing to determine changes.

**Step 1 — Extract Product IDs**: Identify all unique products in the validated fitment set.

**Step 2 — Load Existing Fitments**: Query Snowflake for all existing fitments for these products via `fetchExistingFitmentsByProducts`.

**Step 3 — Build Indexes**: 
- Build existing fitment index (composite key → full fitment details)
- Build incoming fitment index (composite key → full fitment details)
- Total memory: ~400MB for 100K fitments on each side

**Step 4 — Classify Incoming Fitments** (parallel batches):
- For each incoming fitment, lookup in existing index
- If not found → **ADD**
- If found, compare details (qualifiers, notes, vehicle conditions)
  - If identical → **UNCHANGED**
  - If different → **UPDATE** (record what changed)

**Step 5 — Identify Deletes**:
- Iterate existing index
- For each existing fitment, check if key exists in incoming index
- If not found → **DELETE** (supplier removed this fitment)

**Step 6 — Persist Classifications**:
- Update staging table with classification for each fitment
- Store change details for UPDATEs (what specifically changed)

### 5.4 FitmentValidationAgent — Validation Execution

**Initialization Phase**:
- Load reference data caches from MCP servers (VCDB, QDB, PCDB IDs and mappings)
- Load product catalog for supplier part resolution

**Per-Batch Processing** (2,000 fitments per batch, parallel workers):

**Phase 1 — ID Existence Checks**:
- Invoke `validateBaseVehicleIdsBatch` — check base vehicle IDs exist
- Invoke `validatePartTypeIdsBatch` — check part type IDs exist
- Invoke `validatePositionIdsBatch` — check position IDs exist
- Invoke `validateQualifierIdsBatch` — check qualifier IDs exist
- Invoke `validateAttributeValuesBatch` — check attribute values exist in domains

**Phase 2 — Relationship Checks**:
- Invoke `validatePartTypePositionBatch` — check part-type-to-position mapping
- Invoke `validateBrandSubBrandBatch` — check brand-subbrand mapping

**Phase 3 — Vehicle Resolution** (conditional):
- Only for fitments with NO errors from Phase 1/2
- Invoke `resolveVehicleConfigurationBatch` — verify complete vehicle configuration exists

**Phase 4 — Result Aggregation**:
- Group errors by domain (VCDB, QDB, PCDB)
- Format error messages
- Bulk update staging table

---

## Section 6: Summary — Processing Architecture

### Key Design Principles

**1. Full File Processing Model**
- Suppliers always send FULL files with all product fitments
- System compares incoming vs existing (Snowflake) to determine Add/Update/Delete
- Comparison happens AFTER validation for uplift report

**2. Validation by Domain**
- VCDB: Base vehicle ID, attribute values, vehicle configuration resolution
- QDB: Qualifier ID existence
- PCDB: Part type ID, position ID, part-type-position mapping
- Brand/Product: Brand-subbrand mapping, supplier part resolution

**3. Comparison via Indexing**
- Load existing fitments for products in file from Snowflake
- Index both existing and incoming by composite key
- Classify each fitment as Add/Update/Delete in single pass
- Identify deletes by scanning existing index for missing keys

---

## Section 7: Implementation Phases

### Phase 1 — Snowflake Data Setup
- Create schemas: `AUTOCARE`, `CATALOG`, `ACES_VALIDATION`
- Load reference data: VCDB, QDB, PCDB tables
- Create fitments and products tables in `CATALOG`
- See [MCP Setup Guide](./aces-mcp-snowflake-setup.md) for DDL scripts

### Phase 2 — Stored Procedures
- VCDB validation procedures: base vehicle, attributes, vehicle resolution
- QDB validation procedures: qualifier validation
- PCDB validation procedures: part type, position, mapping
- Comparison procedures: fetch existing, classify changes
- Aggregation procedures: validation summary, uplift summary

### Phase 3 — MCP Server Configuration
- Create OAuth security integration
- Create MCP SERVER object with GENERIC tools
- Configure access control (roles, grants)
- See [MCP Setup Guide](./aces-mcp-snowflake-setup.md) for SQL scripts

### Phase 4 — Agent Client Implementation
- MCP client with OAuth authentication
- FitmentJobOrchestrator — workflow coordination
- SchemaValidationAgent — XSD validation (local, not MCP)
- PreflightAgent — preflight checks
- FitmentValidationAgent — invokes MCP tools for validation
- FitmentComparisonAgent — invokes MCP tools for comparison
- StatisticsAgent — invokes MCP tools for aggregation

### Phase 5 — Integration Testing
- End-to-end pipeline with test files
- MCP connectivity and OAuth testing
- Error handling and recovery testing

---

## Appendix: Performance Considerations (Optional)

For high-throughput scenarios (~100K fitments), consider:

| Optimization | Approach |
|--------------|----------|
| **Reference Data Caching** | Pre-load VCDB/QDB/PCDB IDs into memory at job start |
| **Batch Processing** | Process fitments in batches of 2K with parallel workers |
| **Early Exit** | Skip expensive vehicle resolution for fitments with ID errors |
| **Indexed Comparison** | Index existing/incoming fitments by composite key for O(1) lookup |

---

*ACES Fitment Validation — Multi-Agent Orchestration Architecture*
