# ACES Fitment Validation — Snowflake MCP Server Setup Guide

> **Purpose**: Configure Snowflake managed MCP server for ACES fitment validation  
> **Prerequisites**: Snowflake account with ACCOUNTADMIN access  
> **Reference**: [Snowflake MCP Server Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp)

---

## Table of Contents

1. [Schema Setup](#1-schema-setup)
2. [Reference Data Tables](#2-reference-data-tables)
3. [Stored Procedures](#3-stored-procedures)
4. [MCP Server Configuration](#4-mcp-server-configuration)
5. [OAuth Authentication](#5-oauth-authentication)
6. [Access Control](#6-access-control)
7. [Testing](#7-testing)

---

## 1. Schema Setup

Create the required schemas for reference data, catalog, and validation logic.

```sql
-- Create database for ACES validation
CREATE DATABASE IF NOT EXISTS ACES_VALIDATION;

-- Schema for Autocare reference data (VCDB, QDB, PCDB)
CREATE SCHEMA IF NOT EXISTS ACES_VALIDATION.AUTOCARE;

-- Schema for catalog data (products, existing fitments)
CREATE SCHEMA IF NOT EXISTS ACES_VALIDATION.CATALOG;

-- Schema for MCP server and stored procedures
CREATE SCHEMA IF NOT EXISTS ACES_VALIDATION.MCP;

-- Schema for staging data during processing
CREATE SCHEMA IF NOT EXISTS ACES_VALIDATION.STAGING;
```

---

## 2. Reference Data Tables

### 2.1 VCDB Tables (Vehicle Configuration Database)

```sql
-- Base vehicles
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.VCDB_BASE_VEHICLE (
    base_vehicle_id INTEGER PRIMARY KEY,
    year_id INTEGER,
    make_id INTEGER,
    model_id INTEGER,
    submodel_id INTEGER,
    region_id INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Vehicle configurations (full vehicle with all attributes)
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.VCDB_VEHICLE (
    vehicle_id INTEGER PRIMARY KEY,
    base_vehicle_id INTEGER,
    submodel_id INTEGER,
    region_id INTEGER,
    engine_base_id INTEGER,
    engine_config_id INTEGER,
    drive_type_id INTEGER,
    fuel_type_id INTEGER,
    transmission_type_id INTEGER,
    brake_system_id INTEGER,
    body_type_id INTEGER,
    bed_type_id INTEGER,
    wheel_base_id INTEGER,
    mfr_body_code_id INTEGER,
    steering_type_id INTEGER,
    steering_system_id INTEGER,
    spring_type_id INTEGER,
    is_active BOOLEAN DEFAULT TRUE
);

-- Attribute domain tables (one per attribute type)
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.VCDB_DRIVE_TYPE (
    drive_type_id INTEGER PRIMARY KEY,
    drive_type_name VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.VCDB_FUEL_TYPE (
    fuel_type_id INTEGER PRIMARY KEY,
    fuel_type_name VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.VCDB_ENGINE_BASE (
    engine_base_id INTEGER PRIMARY KEY,
    liter VARCHAR(10),
    cc VARCHAR(10),
    cid VARCHAR(10),
    cylinders VARCHAR(10),
    block_type VARCHAR(10),
    is_active BOOLEAN DEFAULT TRUE
);

-- Add similar tables for: TransmissionType, BrakeSystem, BodyType, etc.

-- VCDB version metadata
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.VCDB_VERSION (
    version_date DATE PRIMARY KEY,
    loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    is_current BOOLEAN DEFAULT FALSE
);
```

### 2.2 QDB Tables (Qualifier Database)

```sql
-- Qualifiers
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.QDB_QUALIFIER (
    qualifier_id INTEGER PRIMARY KEY,
    qualifier_text VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- QDB version metadata
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.QDB_VERSION (
    version_date DATE PRIMARY KEY,
    loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    is_current BOOLEAN DEFAULT FALSE
);
```

### 2.3 PCDB Tables (Parts Configuration Database)

```sql
-- Part types
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.PCDB_PART_TYPE (
    part_type_id INTEGER PRIMARY KEY,
    part_type_name VARCHAR(200),
    is_active BOOLEAN DEFAULT TRUE
);

-- Positions
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.PCDB_POSITION (
    position_id INTEGER PRIMARY KEY,
    position_name VARCHAR(200),
    is_active BOOLEAN DEFAULT TRUE
);

-- Part type to position mapping (valid combinations)
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.PCDB_PART_TYPE_POSITION (
    part_type_id INTEGER,
    position_id INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (part_type_id, position_id)
);

-- PCDB version metadata
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.AUTOCARE.PCDB_VERSION (
    version_date DATE PRIMARY KEY,
    loaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    is_current BOOLEAN DEFAULT FALSE
);
```

### 2.4 Catalog Tables

```sql
-- Products (supplier part to product mapping)
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.CATALOG.PRODUCT (
    product_id INTEGER PRIMARY KEY,
    line_code VARCHAR(10),
    supplier_part_number VARCHAR(100),
    brand_id INTEGER,
    sub_brand_id INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    UNIQUE (line_code, supplier_part_number)
);

-- Brands
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.CATALOG.BRAND (
    brand_id INTEGER PRIMARY KEY,
    brand_name VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE
);

-- Brand to sub-brand mapping
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.CATALOG.BRAND_SUB_BRAND (
    brand_id INTEGER,
    sub_brand_id INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (brand_id, sub_brand_id)
);

-- Existing fitments (for comparison)
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.CATALOG.FITMENT (
    fitment_id INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id INTEGER NOT NULL,
    base_vehicle_id INTEGER NOT NULL,
    part_type_id INTEGER NOT NULL,
    position_id INTEGER,
    qualifiers VARIANT,           -- JSON array of qualifier IDs
    notes VARIANT,                -- JSON array of note objects
    vehicle_conditions VARIANT,   -- JSON object of attribute conditions
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    UNIQUE (product_id, base_vehicle_id, part_type_id, position_id)
);

-- Index for efficient lookup by product
CREATE INDEX IF NOT EXISTS idx_fitment_product ON ACES_VALIDATION.CATALOG.FITMENT(product_id);
```

---

## 3. Stored Procedures

### 3.1 VCDB Validation Procedures

```sql
-- Validate base vehicle IDs in batch
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_BASE_VEHICLE_IDS_BATCH(
    base_vehicle_ids VARIANT  -- Array of integers
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'base_vehicle_id', bv.id,
        'is_valid', CASE WHEN v.base_vehicle_id IS NOT NULL THEN TRUE ELSE FALSE END
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :base_vehicle_ids)) bv
    LEFT JOIN ACES_VALIDATION.AUTOCARE.VCDB_BASE_VEHICLE v 
        ON bv.value::INTEGER = v.base_vehicle_id AND v.is_active = TRUE;
    
    RETURN result;
END;
$$;

-- Validate attribute values in batch
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_ATTRIBUTE_VALUES_BATCH(
    attributes VARIANT  -- Array of {domain: string, value: integer}
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    -- This procedure validates attributes against their respective domain tables
    -- Implementation depends on specific attribute domains
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'domain', attr.value:domain::STRING,
        'value', attr.value:value::INTEGER,
        'is_valid', TRUE  -- Simplified; actual impl queries domain tables
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :attributes)) attr;
    
    RETURN result;
END;
$$;

-- Resolve vehicle configuration (validates complete vehicle exists)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.RESOLVE_VEHICLE_CONFIGURATION_BATCH(
    configurations VARIANT  -- Array of {base_vehicle_id, attributes: {...}}
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    -- For each configuration, check if a matching vehicle exists in VCDB_VEHICLE
    -- This is a complex join that varies based on which attributes are provided
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'base_vehicle_id', cfg.value:base_vehicle_id::INTEGER,
        'is_valid', CASE WHEN COUNT(v.vehicle_id) > 0 THEN TRUE ELSE FALSE END,
        'matching_count', COUNT(v.vehicle_id)
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :configurations)) cfg
    LEFT JOIN ACES_VALIDATION.AUTOCARE.VCDB_VEHICLE v 
        ON cfg.value:base_vehicle_id::INTEGER = v.base_vehicle_id
    GROUP BY cfg.value:base_vehicle_id::INTEGER;
    
    RETURN result;
END;
$$;
```

### 3.2 QDB Validation Procedures

```sql
-- Validate qualifier IDs in batch
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_QUALIFIER_IDS_BATCH(
    qualifier_ids VARIANT  -- Array of integers
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'qualifier_id', q.id,
        'is_valid', CASE WHEN qdb.qualifier_id IS NOT NULL THEN TRUE ELSE FALSE END
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :qualifier_ids)) q
    LEFT JOIN ACES_VALIDATION.AUTOCARE.QDB_QUALIFIER qdb 
        ON q.value::INTEGER = qdb.qualifier_id AND qdb.is_active = TRUE;
    
    RETURN result;
END;
$$;
```

### 3.3 PCDB Validation Procedures

```sql
-- Validate part type IDs in batch
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_IDS_BATCH(
    part_type_ids VARIANT  -- Array of integers
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'part_type_id', pt.id,
        'is_valid', CASE WHEN pcdb.part_type_id IS NOT NULL THEN TRUE ELSE FALSE END
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :part_type_ids)) pt
    LEFT JOIN ACES_VALIDATION.AUTOCARE.PCDB_PART_TYPE pcdb 
        ON pt.value::INTEGER = pcdb.part_type_id AND pcdb.is_active = TRUE;
    
    RETURN result;
END;
$$;

-- Validate position IDs in batch
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_POSITION_IDS_BATCH(
    position_ids VARIANT  -- Array of integers
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'position_id', pos.id,
        'is_valid', CASE WHEN pcdb.position_id IS NOT NULL THEN TRUE ELSE FALSE END
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :position_ids)) pos
    LEFT JOIN ACES_VALIDATION.AUTOCARE.PCDB_POSITION pcdb 
        ON pos.value::INTEGER = pcdb.position_id AND pcdb.is_active = TRUE;
    
    RETURN result;
END;
$$;

-- Validate part type to position mapping in batch
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_POSITION_BATCH(
    mappings VARIANT  -- Array of {part_type_id, position_id}
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'part_type_id', m.value:part_type_id::INTEGER,
        'position_id', m.value:position_id::INTEGER,
        'is_valid', CASE WHEN ptp.part_type_id IS NOT NULL THEN TRUE ELSE FALSE END
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :mappings)) m
    LEFT JOIN ACES_VALIDATION.AUTOCARE.PCDB_PART_TYPE_POSITION ptp 
        ON m.value:part_type_id::INTEGER = ptp.part_type_id 
        AND m.value:position_id::INTEGER = ptp.position_id
        AND ptp.is_active = TRUE;
    
    RETURN result;
END;
$$;
```

### 3.4 Comparison Procedures

```sql
-- Fetch existing fitments by product IDs
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.FETCH_EXISTING_FITMENTS_BY_PRODUCTS(
    product_ids VARIANT  -- Array of integers
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'fitment_id', f.fitment_id,
        'product_id', f.product_id,
        'base_vehicle_id', f.base_vehicle_id,
        'part_type_id', f.part_type_id,
        'position_id', f.position_id,
        'qualifiers', f.qualifiers,
        'notes', f.notes,
        'vehicle_conditions', f.vehicle_conditions,
        'composite_key', CONCAT(f.product_id, '|', f.base_vehicle_id, '|', f.part_type_id, '|', COALESCE(f.position_id, 0))
    ))
    INTO result
    FROM ACES_VALIDATION.CATALOG.FITMENT f
    WHERE f.product_id IN (SELECT value::INTEGER FROM TABLE(FLATTEN(INPUT => :product_ids)));
    
    RETURN result;
END;
$$;

-- Classify fitment changes (Add/Update/Delete)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.CLASSIFY_FITMENT_CHANGES_BATCH(
    incoming_fitments VARIANT,   -- Array of incoming fitment objects
    existing_fitments VARIANT    -- Array of existing fitment objects (from fetch)
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    -- Build comparison and classify each incoming fitment
    WITH incoming AS (
        SELECT 
            i.value:product_id::INTEGER AS product_id,
            i.value:base_vehicle_id::INTEGER AS base_vehicle_id,
            i.value:part_type_id::INTEGER AS part_type_id,
            i.value:position_id::INTEGER AS position_id,
            i.value:qualifiers AS qualifiers,
            i.value:notes AS notes,
            i.value:vehicle_conditions AS vehicle_conditions,
            CONCAT(i.value:product_id, '|', i.value:base_vehicle_id, '|', i.value:part_type_id, '|', COALESCE(i.value:position_id, 0)) AS composite_key
        FROM TABLE(FLATTEN(INPUT => :incoming_fitments)) i
    ),
    existing AS (
        SELECT 
            e.value:composite_key::STRING AS composite_key,
            e.value:qualifiers AS qualifiers,
            e.value:notes AS notes,
            e.value:vehicle_conditions AS vehicle_conditions
        FROM TABLE(FLATTEN(INPUT => :existing_fitments)) e
    )
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'composite_key', inc.composite_key,
        'classification', CASE 
            WHEN ex.composite_key IS NULL THEN 'ADD'
            WHEN inc.qualifiers = ex.qualifiers 
                AND inc.notes = ex.notes 
                AND inc.vehicle_conditions = ex.vehicle_conditions THEN 'UNCHANGED'
            ELSE 'UPDATE'
        END,
        'details', CASE 
            WHEN ex.composite_key IS NULL THEN NULL
            WHEN inc.qualifiers != ex.qualifiers THEN 'qualifiers_changed'
            WHEN inc.notes != ex.notes THEN 'notes_changed'
            WHEN inc.vehicle_conditions != ex.vehicle_conditions THEN 'conditions_changed'
            ELSE NULL
        END
    ))
    INTO result
    FROM incoming inc
    LEFT JOIN existing ex ON inc.composite_key = ex.composite_key;
    
    RETURN result;
END;
$$;

-- Identify deleted fitments (in existing but not in incoming)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.IDENTIFY_DELETED_FITMENTS(
    incoming_keys VARIANT,   -- Array of composite key strings
    existing_keys VARIANT    -- Array of composite key strings
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(
        'composite_key', ex.value::STRING,
        'classification', 'DELETE'
    ))
    INTO result
    FROM TABLE(FLATTEN(INPUT => :existing_keys)) ex
    WHERE ex.value::STRING NOT IN (
        SELECT inc.value::STRING FROM TABLE(FLATTEN(INPUT => :incoming_keys)) inc
    );
    
    RETURN result;
END;
$$;
```

### 3.5 Aggregation Procedures

```sql
-- Compute validation summary
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.COMPUTE_VALIDATION_SUMMARY(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    -- Aggregate validation results from staging table
    SELECT OBJECT_CONSTRUCT(
        'job_id', :job_id,
        'total_fitments', COUNT(*),
        'valid_count', SUM(CASE WHEN is_valid THEN 1 ELSE 0 END),
        'error_count', SUM(CASE WHEN NOT is_valid THEN 1 ELSE 0 END),
        'vcdb_errors', SUM(CASE WHEN error_type = 'VCDB' THEN 1 ELSE 0 END),
        'qdb_errors', SUM(CASE WHEN error_type = 'QDB' THEN 1 ELSE 0 END),
        'pcdb_errors', SUM(CASE WHEN error_type = 'PCDB' THEN 1 ELSE 0 END)
    )
    INTO result
    FROM ACES_VALIDATION.STAGING.FITMENT_VALIDATION
    WHERE job_id = :job_id;
    
    RETURN result;
END;
$$;

-- Compute uplift summary
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.COMPUTE_UPLIFT_SUMMARY(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    result VARIANT;
BEGIN
    -- Aggregate classification results
    SELECT OBJECT_CONSTRUCT(
        'job_id', :job_id,
        'add_count', SUM(CASE WHEN classification = 'ADD' THEN 1 ELSE 0 END),
        'update_count', SUM(CASE WHEN classification = 'UPDATE' THEN 1 ELSE 0 END),
        'delete_count', SUM(CASE WHEN classification = 'DELETE' THEN 1 ELSE 0 END),
        'unchanged_count', SUM(CASE WHEN classification = 'UNCHANGED' THEN 1 ELSE 0 END)
    )
    INTO result
    FROM ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION
    WHERE job_id = :job_id;
    
    RETURN result;
END;
$$;
```

---

## 4. MCP Server Configuration

Create the Snowflake managed MCP server with all validation tools.

```sql
-- Create MCP Server with GENERIC tools backed by stored procedures
CREATE OR REPLACE MCP SERVER ACES_VALIDATION.MCP.ACES_FITMENT_SERVER
FROM SPECIFICATION $$
tools:
  # VCDB Validation Tools
  - name: "validate_base_vehicle_ids_batch"
    title: "Validate Base Vehicle IDs"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_BASE_VEHICLE_IDS_BATCH"
    description: "Validates an array of base vehicle IDs against VCDB. Returns validity status for each ID."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          base_vehicle_ids:
            description: "Array of base vehicle IDs to validate"
            type: "array"
            items:
              type: "integer"

  - name: "validate_attribute_values_batch"
    title: "Validate Attribute Values"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_ATTRIBUTE_VALUES_BATCH"
    description: "Validates attribute values against their VCDB domain tables."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          attributes:
            description: "Array of {domain, value} objects"
            type: "array"

  - name: "resolve_vehicle_configuration_batch"
    title: "Resolve Vehicle Configuration"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.RESOLVE_VEHICLE_CONFIGURATION_BATCH"
    description: "Validates that base vehicle + attributes resolves to actual vehicle in VCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          configurations:
            description: "Array of {base_vehicle_id, attributes} objects"
            type: "array"

  # QDB Validation Tools
  - name: "validate_qualifier_ids_batch"
    title: "Validate Qualifier IDs"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_QUALIFIER_IDS_BATCH"
    description: "Validates an array of qualifier IDs against QDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          qualifier_ids:
            description: "Array of qualifier IDs to validate"
            type: "array"
            items:
              type: "integer"

  # PCDB Validation Tools
  - name: "validate_part_type_ids_batch"
    title: "Validate Part Type IDs"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_IDS_BATCH"
    description: "Validates an array of part type IDs against PCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          part_type_ids:
            description: "Array of part type IDs to validate"
            type: "array"
            items:
              type: "integer"

  - name: "validate_position_ids_batch"
    title: "Validate Position IDs"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_POSITION_IDS_BATCH"
    description: "Validates an array of position IDs against PCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          position_ids:
            description: "Array of position IDs to validate"
            type: "array"
            items:
              type: "integer"

  - name: "validate_part_type_position_batch"
    title: "Validate Part Type Position Mapping"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_POSITION_BATCH"
    description: "Validates part type to position mappings against PCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          mappings:
            description: "Array of {part_type_id, position_id} objects"
            type: "array"

  # Comparison Tools
  - name: "fetch_existing_fitments_by_products"
    title: "Fetch Existing Fitments"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.FETCH_EXISTING_FITMENTS_BY_PRODUCTS"
    description: "Fetches all existing fitments for given product IDs from catalog."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          product_ids:
            description: "Array of product IDs"
            type: "array"
            items:
              type: "integer"

  - name: "classify_fitment_changes_batch"
    title: "Classify Fitment Changes"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.CLASSIFY_FITMENT_CHANGES_BATCH"
    description: "Classifies incoming fitments as ADD, UPDATE, or UNCHANGED vs existing."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          incoming_fitments:
            description: "Array of incoming fitment objects"
            type: "array"
          existing_fitments:
            description: "Array of existing fitment objects"
            type: "array"

  - name: "identify_deleted_fitments"
    title: "Identify Deleted Fitments"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.IDENTIFY_DELETED_FITMENTS"
    description: "Identifies fitments in existing but not in incoming (DELETEs)."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          incoming_keys:
            description: "Array of composite keys from incoming fitments"
            type: "array"
          existing_keys:
            description: "Array of composite keys from existing fitments"
            type: "array"

  # Aggregation Tools
  - name: "compute_validation_summary"
    title: "Compute Validation Summary"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.COMPUTE_VALIDATION_SUMMARY"
    description: "Computes validation statistics for a job."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Job identifier"
            type: "string"

  - name: "compute_uplift_summary"
    title: "Compute Uplift Summary"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.COMPUTE_UPLIFT_SUMMARY"
    description: "Computes Add/Update/Delete counts for uplift report."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Job identifier"
            type: "string"
$$;
```

---

## 5. OAuth Authentication

Set up OAuth for MCP client authentication.

```sql
-- Create OAuth security integration for MCP clients
CREATE OR REPLACE SECURITY INTEGRATION ACES_MCP_OAUTH
  TYPE = OAUTH
  OAUTH_CLIENT = CUSTOM
  ENABLED = TRUE
  OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
  OAUTH_REDIRECT_URI = 'http://localhost:8080/callback'
  OAUTH_ISSUE_REFRESH_TOKENS = TRUE
  OAUTH_REFRESH_TOKEN_VALIDITY = 86400;

-- Retrieve client ID and secret
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('ACES_MCP_OAUTH');
```

**MCP Server URL Format**:
```
https://<account_url>/api/v2/databases/ACES_VALIDATION/schemas/MCP/mcp-servers/ACES_FITMENT_SERVER
```

---

## 6. Access Control

### 6.1 Create Roles

```sql
-- Role for MCP server access
CREATE ROLE IF NOT EXISTS ACES_MCP_USER;

-- Role for data administration
CREATE ROLE IF NOT EXISTS ACES_DATA_ADMIN;
```

### 6.2 Grant Permissions

```sql
-- Grant usage on database and schemas
GRANT USAGE ON DATABASE ACES_VALIDATION TO ROLE ACES_MCP_USER;
GRANT USAGE ON SCHEMA ACES_VALIDATION.AUTOCARE TO ROLE ACES_MCP_USER;
GRANT USAGE ON SCHEMA ACES_VALIDATION.CATALOG TO ROLE ACES_MCP_USER;
GRANT USAGE ON SCHEMA ACES_VALIDATION.MCP TO ROLE ACES_MCP_USER;
GRANT USAGE ON SCHEMA ACES_VALIDATION.STAGING TO ROLE ACES_MCP_USER;

-- Grant SELECT on reference tables
GRANT SELECT ON ALL TABLES IN SCHEMA ACES_VALIDATION.AUTOCARE TO ROLE ACES_MCP_USER;
GRANT SELECT ON ALL TABLES IN SCHEMA ACES_VALIDATION.CATALOG TO ROLE ACES_MCP_USER;

-- Grant USAGE on MCP server
GRANT USAGE ON MCP SERVER ACES_VALIDATION.MCP.ACES_FITMENT_SERVER TO ROLE ACES_MCP_USER;

-- Grant USAGE on all stored procedures (tools)
GRANT USAGE ON ALL PROCEDURES IN SCHEMA ACES_VALIDATION.MCP TO ROLE ACES_MCP_USER;

-- Grant warehouse usage
GRANT USAGE ON WAREHOUSE ACES_VALIDATION_WH TO ROLE ACES_MCP_USER;

-- Assign role to users
GRANT ROLE ACES_MCP_USER TO USER <your_username>;

-- Set default role for OAuth sessions
ALTER USER <your_username> SET DEFAULT_ROLE = 'ACES_MCP_USER' DEFAULT_WAREHOUSE = 'ACES_VALIDATION_WH';
```

---

## 7. Testing

### 7.1 Verify MCP Server

```sql
-- Show MCP servers
SHOW MCP SERVERS IN SCHEMA ACES_VALIDATION.MCP;

-- Describe MCP server to see tools
DESCRIBE MCP SERVER ACES_VALIDATION.MCP.ACES_FITMENT_SERVER;
```

### 7.2 Test Tool Invocation via REST

```bash
# List available tools
curl -X POST \
  "https://<account_url>/api/v2/databases/ACES_VALIDATION/schemas/MCP/mcp-servers/ACES_FITMENT_SERVER" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'

# Invoke validation tool
curl -X POST \
  "https://<account_url>/api/v2/databases/ACES_VALIDATION/schemas/MCP/mcp-servers/ACES_FITMENT_SERVER" \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "validate_base_vehicle_ids_batch",
      "arguments": {
        "base_vehicle_ids": [1001, 1002, 9999]
      }
    }
  }'
```

### 7.3 Test Stored Procedures Directly

```sql
-- Test base vehicle validation
CALL ACES_VALIDATION.MCP.VALIDATE_BASE_VEHICLE_IDS_BATCH(
    PARSE_JSON('[1001, 1002, 9999]')
);

-- Test part type position validation
CALL ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_POSITION_BATCH(
    PARSE_JSON('[{"part_type_id": 100, "position_id": 1}, {"part_type_id": 200, "position_id": 2}]')
);

-- Test fetch existing fitments
CALL ACES_VALIDATION.MCP.FETCH_EXISTING_FITMENTS_BY_PRODUCTS(
    PARSE_JSON('[12345, 12346]')
);
```

---

## Appendix: Warehouse Setup

```sql
-- Create dedicated warehouse for MCP operations
CREATE WAREHOUSE IF NOT EXISTS ACES_VALIDATION_WH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;
```

---

*ACES Fitment Validation — Snowflake MCP Server Setup Guide*
