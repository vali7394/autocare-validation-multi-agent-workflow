# ACES Fitment Validation — Multi-Agent Orchestration Project Summary

> **Document Purpose**: Quick reference for project scope, objectives, constraints, and technology decisions  
> **Last Updated**: June 2026

---

## 1. Use Case

### Business Context

Suppliers submit ACES-compliant XML files containing automotive fitment data — information about which parts fit which vehicles. These files must be validated against industry-standard reference databases (VCDB, QDB, PCDB) before being accepted into the catalog system.

### Processing Model

**Full File Submissions**: Suppliers always send complete product-level fitment files. Each file contains ALL fitments for the products included — not incremental updates. This requires:

1. **Validation** — Verify each fitment against Autocare reference data
2. **Comparison** — Compare incoming fitments against existing fitments in the catalog
3. **Classification** — Determine which fitments are Add, Update, Delete, or Unchanged
4. **Reporting** — Generate uplift summary for business review

### Typical File Characteristics

| Metric | Value |
|--------|-------|
| Fitments per file | ~100,000 |
| Products per file | Varies (hundreds to thousands) |
| File format | ACES XML (multiple versions supported) |
| Frequency | On-demand from suppliers |

---

## 2. Objectives

### Primary Objectives

| # | Objective | Success Criteria |
|---|-----------|------------------|
| 1 | **Validate fitment data accuracy** | All fitments checked against VCDB, QDB, PCDB |
| 2 | **Detect changes from previous submissions** | Accurate Add/Update/Delete classification |
| 3 | **Generate actionable uplift reports** | Business can review what changed |
| 4 | **Scale to 100K fitments per file** | Process within acceptable time window |

### Secondary Objectives

- **Maintainability** — Clear separation of concerns via agent architecture
- **Extensibility** — Easy to add new validation rules
- **Observability** — Traceable processing for debugging
- **Reusability** — Tools can be reused across different workflows

---

## 3. Constraints

### Technical Constraints

| Constraint | Description | Impact |
|------------|-------------|--------|
| **Snowflake MCP Server Limits** | Max 50 tools per MCP server | May need multiple MCP servers if tools exceed 50 |
| **Response Size Limit** | 250 KB per tool response | Batch sizes must keep results under limit |
| **OAuth Session Role** | Uses user's `DEFAULT_ROLE` only | Must configure roles carefully |
| **No Secondary Roles** | MCP OAuth doesn't support secondary roles | Single role must have all permissions |
| **Non-streaming Only** | Snowflake MCP only supports non-streaming | No real-time streaming of results |

### Business Constraints

| Constraint | Description |
|------------|-------------|
| **Reference Data Versions** | File's declared VCDB/QDB/PCDB versions must match or be compatible with loaded versions |
| **Full File Model** | Cannot process incremental updates — must handle full product-level files |
| **Backward Compatibility** | Must support multiple ACES XML versions |

### Data Constraints

| Data Source | Update Frequency | Size Estimate |
|-------------|------------------|---------------|
| VCDB (vehicles) | Monthly | ~50K base vehicles |
| QDB (qualifiers) | Monthly | ~5K qualifiers |
| PCDB (parts config) | Monthly | ~2K part types, ~500 positions |
| Existing Fitments | Real-time | ~100K per job (varies by product scope) |

---

## 4. Technologies

### Data Platform

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Data Warehouse** | Snowflake | Reference data, existing fitments, staging |
| **MCP Server** | Snowflake Managed MCP | Tool exposure via standard MCP protocol |
| **Authentication** | Snowflake OAuth 2.0 | Secure MCP client authentication |

### Agent Architecture

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Agent Framework** | MCP Client (to be determined) | Orchestration, tool invocation |
| **Tool Protocol** | Model Context Protocol (MCP) | Standard interface for tool discovery and invocation |
| **Tool Implementation** | Snowflake Stored Procedures | Validation logic executed in Snowflake |

### Data Formats

| Format | Usage |
|--------|-------|
| **ACES XML** | Input fitment files |
| **XSD Schemas** | XML validation |
| **JSON/VARIANT** | Tool inputs/outputs, Snowflake data |

### Reference Standards

| Standard | Version | Purpose |
|----------|---------|---------|
| **ACES** | Multiple (3.x, 4.x) | Fitment XML schema |
| **VCDB** | Monthly releases | Vehicle reference data |
| **QDB** | Monthly releases | Qualifier reference data |
| **PCDB** | Monthly releases | Parts configuration reference data |
| **MCP** | 2025-11-25 | Tool protocol specification |

---

## 5. Architecture Summary

### High-Level Flow

```
┌──────────────┐     ┌─────────────────────┐     ┌──────────────────────┐
│  ACES XML    │────▶│  Agent Orchestrator │────▶│  Snowflake MCP Server│
│  File Input  │     │  (MCP Client)       │     │  (Tools/Procedures)  │
└──────────────┘     └─────────────────────┘     └──────────┬───────────┘
                              │                             │
                              ▼                             ▼
                     ┌─────────────────────┐     ┌──────────────────────┐
                     │  Uplift Report      │     │  Snowflake Tables    │
                     │  (Output)           │     │  VCDB/QDB/PCDB/Catalog│
                     └─────────────────────┘     └──────────────────────┘
```

### Agent Topology

| Agent | Responsibility |
|-------|---------------|
| **FitmentJobOrchestrator** | Workflow coordination, state management |
| **SchemaValidationAgent** | XSD conformance, version detection |
| **PreflightAgent** | Version alignment, filename validation |
| **FitmentValidationAgent** | Per-record validation via MCP tools |
| **FitmentComparisonAgent** | Add/Update/Delete classification via MCP tools |
| **StatisticsAgent** | Aggregation, uplift report generation |

### Validation Categories

| Category | Checks | Error Type |
|----------|--------|------------|
| **VCDB** | Base vehicle ID, attribute values, vehicle resolution | VCDB ERROR |
| **QDB** | Qualifier IDs | QDB ERROR |
| **PCDB** | Part type ID, position ID, part-type-position mapping | PCDB ERROR |
| **Brand** | Brand ID, sub-brand ID | BRAND WARNING |
| **Product** | Supplier part number resolution | PRODUCT ERROR |

### Comparison Logic

| Scenario | Classification |
|----------|---------------|
| Key not in existing | **ADD** |
| Key in existing, identical | **UNCHANGED** |
| Key in existing, different | **UPDATE** |
| Key in existing, not in incoming | **DELETE** |

**Composite Key**: `product_id + base_vehicle_id + part_type_id + position_id`

---

## 6. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Snowflake Managed MCP** | No custom server infrastructure; built-in OAuth; tools as stored procedures |
| **Stored Procedures for Tools** | Logic executes close to data; leverages Snowflake compute |
| **Full File Comparison Model** | Suppliers send complete files; must detect deletes by absence |
| **Batch Processing** | 100K fitments processed in batches of ~2K for manageability |
| **Agent Separation** | Each agent has single responsibility; easier testing and maintenance |
| **Post-Validation Comparison** | Only compare valid fitments; reduces comparison workload |

---

## 7. Related Documents

| Document | Purpose |
|----------|---------|
| [accept-aces-multi-agent-orchestration.md](./accept-aces-multi-agent-orchestration.md) | Detailed architecture specification |
| [aces-mcp-snowflake-setup.md](./aces-mcp-snowflake-setup.md) | Snowflake setup scripts (DDL, procedures, MCP server) |
| [accept-aces-business-logic.md](./accept-aces-business-logic.md) | Original business logic requirements |
| [Snowflake-managed-MCP-server -Snowflake.md](./Snowflake-managed-MCP-server%20-Snowflake.md) | Snowflake MCP documentation reference |

---

## 8. Open Items / Future Considerations

| Item | Status | Notes |
|------|--------|-------|
| Agent framework selection | TBD | Need to select MCP client implementation |
| Performance testing | Pending | Validate 100K fitment throughput |
| Error handling strategy | Defined | Retry failed batches up to 3 times |
| Multiple MCP servers | If needed | If tools exceed 50, split across servers |
| Caching strategy | Optional | Pre-load reference IDs for performance |

---

*ACES Fitment Validation — Multi-Agent Orchestration Project Summary*
