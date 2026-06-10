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

-- Existing fitments (for comparison, with distinct_fitment_hash resolving collisions)
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.CATALOG.FITMENT (
    fitment_id INTEGER AUTOINCREMENT,
    product_id INTEGER NOT NULL,
    base_vehicle_id INTEGER NOT NULL,
    part_type_id INTEGER NOT NULL,
    position_id INTEGER,
    qualifiers VARIANT,                -- JSON array of qualifier IDs & parameters
    notes VARIANT,                     -- JSON array of free-text notes
    vehicle_conditions VARIANT,        -- JSON object of attribute conditions
    distinct_fitment_hash VARCHAR PRIMARY KEY,  -- Extended Composite Key
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Index for efficient lookup by product
CREATE INDEX IF NOT EXISTS idx_fitment_product ON ACES_VALIDATION.CATALOG.FITMENT(product_id);
```

### 2.5 Staging Tables (Job-scoped)

```sql
-- Tracks individual validation job runs
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.STAGING.JOB_RUN (
    job_id VARCHAR PRIMARY KEY,
    status VARCHAR NOT NULL,            -- STAGED, VALIDATING, COMPARING, COMPLETED, FAILED
    file_path VARCHAR NOT NULL,
    line_code VARCHAR NOT NULL,
    total_records INTEGER DEFAULT 0,
    error_summary VARIANT,             -- Aggregated JSON metrics of failures
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Staging table for raw ACES XML parsed apps loaded via streaming COPY INTO
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.STAGING.FITMENT_STAGE (
    job_id VARCHAR NOT NULL,
    app_id VARCHAR,                    -- XML source element App ID
    line_code VARCHAR NOT NULL,
    supplier_part_number VARCHAR NOT NULL,
    product_id INTEGER,                -- Resolved post-load
    base_vehicle_id INTEGER NOT NULL,
    part_type_id INTEGER NOT NULL,
    position_id INTEGER,
    qualifiers VARIANT,                -- JSON array of qualifier IDs & params
    notes VARIANT,                     -- JSON array of text notes
    vehicle_conditions VARIANT,        -- JSON object of attributes
    distinct_fitment_hash VARCHAR,     -- Extended Composite Key
    is_valid BOOLEAN DEFAULT TRUE,
    error_type VARCHAR,                -- VCDB, QDB, PCDB, BRAND, PRODUCT, SYSTEM
    error_message VARCHAR,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
);

-- Clustered index on job_id and distinct_fitment_hash for set-based performance
ALTER TABLE ACES_VALIDATION.STAGING.FITMENT_STAGE ADD PRIMARY KEY (job_id, distinct_fitment_hash);

-- Holds results from change-detection comparison
CREATE TABLE IF NOT EXISTS ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION (
    job_id VARCHAR NOT NULL,
    distinct_fitment_hash VARCHAR NOT NULL,
    classification VARCHAR NOT NULL,   -- ADD, UPDATE, DELETE, UNCHANGED
    change_details VARCHAR,            -- E.g. 'qualifiers_changed', 'notes_changed', etc.
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP(),
    PRIMARY KEY (job_id, distinct_fitment_hash)
);
```

---

## 3. Stored Procedures

All validation and comparison stored procedures operate on the `STAGING` tables scoped by a `job_id`, performing bulk set-based operations.

### 3.1 VCDB Validation Procedures

```sql
-- Validate base vehicle IDs for a job (checks if they exist in VCDB)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_BASE_VEHICLE_IDS_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    invalid_count INTEGER;
BEGIN
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'VCDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Invalid Base Vehicle ID')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      AND f.base_vehicle_id NOT IN (
          SELECT base_vehicle_id 
          FROM ACES_VALIDATION.AUTOCARE.VCDB_BASE_VEHICLE 
          WHERE is_active = TRUE
      );

    SELECT COUNT(*) INTO :invalid_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND is_valid = FALSE AND error_type = 'VCDB';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'invalid_count', :invalid_count);
END;
$$;

-- Validate attribute values (such as DriveType, FuelType, etc. contained in vehicle_conditions JSON)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_ATTRIBUTE_VALUES_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    invalid_count INTEGER;
BEGIN
    -- We parse the vehicle_conditions VARIANT to check that specific attribute IDs exist in VCDB domains.
    -- Example for DriveType:
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'VCDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Invalid Drive Type ID')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      AND f.vehicle_conditions:DriveType IS NOT NULL
      AND f.vehicle_conditions:DriveType::INTEGER NOT IN (
          SELECT drive_type_id FROM ACES_VALIDATION.AUTOCARE.VCDB_DRIVE_TYPE WHERE is_active = TRUE
      );

    -- Example for FuelType:
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'VCDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Invalid Fuel Type ID')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      AND f.vehicle_conditions:FuelType IS NOT NULL
      AND f.vehicle_conditions:FuelType::INTEGER NOT IN (
          SELECT fuel_type_id FROM ACES_VALIDATION.AUTOCARE.VCDB_FUEL_TYPE WHERE is_active = TRUE
      );

    SELECT COUNT(*) INTO :invalid_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND is_valid = FALSE AND error_type = 'VCDB' AND error_message LIKE '%Type%';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'invalid_count', :invalid_count);
END;
$$;

-- Resolve vehicle configurations (checks if base vehicle + attributes resolve to a valid vehicle configuration)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.RESOLVE_VEHICLE_CONFIGURATION_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    invalid_count INTEGER;
BEGIN
    -- Marks fitments invalid if they have attributes that do not map to any active vehicle configuration
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'VCDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Vehicle configuration mismatch')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      -- Only evaluate records where specific attributes are specified
      AND NOT EXISTS (
          SELECT 1 
          FROM ACES_VALIDATION.AUTOCARE.VCDB_VEHICLE v
          WHERE v.base_vehicle_id = f.base_vehicle_id
            AND v.is_active = TRUE
            AND (f.vehicle_conditions:DriveType IS NULL OR v.drive_type_id = f.vehicle_conditions:DriveType::INTEGER)
            AND (f.vehicle_conditions:FuelType IS NULL OR v.fuel_type_id = f.vehicle_conditions:FuelType::INTEGER)
      );

    SELECT COUNT(*) INTO :invalid_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND is_valid = FALSE AND error_message LIKE '%mismatch%';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'invalid_count', :invalid_count);
END;
$$;
```

### 3.2 QDB Validation Procedures

```sql
-- Validate qualifier IDs (checks if all qualifier IDs specified in the qualifiers array exist in QDB)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_QUALIFIER_IDS_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    invalid_count INTEGER;
BEGIN
    -- Flattens the qualifiers array to identify invalid qualifiers and update staging
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'QDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Invalid Qualifier ID')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      AND EXISTS (
          SELECT 1
          FROM TABLE(FLATTEN(INPUT => f.qualifiers)) q
          LEFT JOIN ACES_VALIDATION.AUTOCARE.QDB_QUALIFIER qdb
              ON q.value:qualifier_id::INTEGER = qdb.qualifier_id AND qdb.is_active = TRUE
          WHERE qdb.qualifier_id IS NULL
      );

    SELECT COUNT(*) INTO :invalid_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND is_valid = FALSE AND error_type = 'QDB';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'invalid_count', :invalid_count);
END;
$$;
```

### 3.3 PCDB Validation Procedures

```sql
-- Validate part type IDs (checks if they exist in PCDB)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_IDS_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    invalid_count INTEGER;
BEGIN
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'PCDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Invalid Part Type ID')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      AND f.part_type_id NOT IN (
          SELECT part_type_id 
          FROM ACES_VALIDATION.AUTOCARE.PCDB_PART_TYPE 
          WHERE is_active = TRUE
      );

    SELECT COUNT(*) INTO :invalid_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND is_valid = FALSE AND error_type = 'PCDB' AND error_message LIKE '%Part%';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'invalid_count', :invalid_count);
END;
$$;

-- Validate position IDs (checks if they exist in PCDB)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_POSITION_IDS_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    invalid_count INTEGER;
BEGIN
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'PCDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Invalid Position ID')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      AND f.position_id IS NOT NULL
      AND f.position_id NOT IN (
          SELECT position_id 
          FROM ACES_VALIDATION.AUTOCARE.PCDB_POSITION 
          WHERE is_active = TRUE
      );

    SELECT COUNT(*) INTO :invalid_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND is_valid = FALSE AND error_type = 'PCDB' AND error_message LIKE '%Position%';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'invalid_count', :invalid_count);
END;
$$;

-- Validate part type to position mapping (checks if the position is valid for the given part type)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_POSITION_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    invalid_count INTEGER;
BEGIN
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'PCDB',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Invalid Part-Type-Position Mapping')
    WHERE f.job_id = :job_id
      AND f.is_valid = TRUE
      AND f.position_id IS NOT NULL
      AND NOT EXISTS (
          SELECT 1 
          FROM ACES_VALIDATION.AUTOCARE.PCDB_PART_TYPE_POSITION ptp
          WHERE ptp.part_type_id = f.part_type_id 
            AND ptp.position_id = f.position_id
            AND ptp.is_active = TRUE
      );

    SELECT COUNT(*) INTO :invalid_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND is_valid = FALSE AND error_message LIKE '%Mapping%';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'invalid_count', :invalid_count);
END;
$$;
```

### 3.4 Product Resolution & Comparison Procedures

```sql
-- Resolve supplier part numbers to actual Product IDs
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.RESOLVE_PRODUCTS_JOB(
    job_id VARCHAR,
    line_code VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    resolved_count INTEGER;
    unresolved_count INTEGER;
BEGIN
    -- Resolve part number using CATALOG.PRODUCT matching on line_code and part number
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.product_id = p.product_id
    FROM ACES_VALIDATION.CATALOG.PRODUCT p
    WHERE f.job_id = :job_id
      AND f.supplier_part_number = p.supplier_part_number
      AND p.line_code = :line_code
      AND p.is_active = TRUE;

    -- Mark unresolved ones as product errors
    UPDATE ACES_VALIDATION.STAGING.FITMENT_STAGE f
    SET f.is_valid = FALSE,
        f.error_type = 'PRODUCT',
        f.error_message = CONCAT(COALESCE(f.error_message, ''), ' | Unresolved Supplier Part Number')
    WHERE f.job_id = :job_id
      AND f.product_id IS NULL;

    SELECT COUNT(*) INTO :resolved_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND product_id IS NOT NULL;

    SELECT COUNT(*) INTO :unresolved_count
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id AND product_id IS NULL;

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'resolved_count', :resolved_count, 'unresolved_count', :unresolved_count);
END;
$$;

-- Classify incoming fitments (ADD, UPDATE, UNCHANGED)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.CLASSIFY_FITMENT_CHANGES_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    add_count INTEGER;
    update_count INTEGER;
    unchanged_count INTEGER;
BEGIN
    -- First clear any existing classifications for this job
    DELETE FROM ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION WHERE job_id = :job_id;

    -- Compare staging against CATALOG.FITMENT
    INSERT INTO ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION (job_id, distinct_fitment_hash, classification, change_details)
    SELECT 
        f.job_id,
        f.distinct_fitment_hash,
        CASE 
            WHEN cat.distinct_fitment_hash IS NULL THEN 'ADD'
            WHEN f.qualifiers = cat.qualifiers 
             AND f.notes = cat.notes 
             AND f.vehicle_conditions = cat.vehicle_conditions THEN 'UNCHANGED'
            ELSE 'UPDATE'
        END,
        CASE 
            WHEN cat.distinct_fitment_hash IS NULL THEN NULL
            WHEN f.qualifiers != cat.qualifiers THEN 'qualifiers_changed'
            WHEN f.notes != cat.notes THEN 'notes_changed'
            WHEN f.vehicle_conditions != cat.vehicle_conditions THEN 'conditions_changed'
            ELSE NULL
        END
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE f
    LEFT JOIN ACES_VALIDATION.CATALOG.FITMENT cat 
        ON f.distinct_fitment_hash = cat.distinct_fitment_hash
    WHERE f.job_id = :job_id 
      AND f.is_valid = TRUE;

    SELECT COUNT(*) INTO :add_count FROM ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION WHERE job_id = :job_id AND classification = 'ADD';
    SELECT COUNT(*) INTO :update_count FROM ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION WHERE job_id = :job_id AND classification = 'UPDATE';
    SELECT COUNT(*) INTO :unchanged_count FROM ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION WHERE job_id = :job_id AND classification = 'UNCHANGED';

    RETURN OBJECT_CONSTRUCT(
        'job_id', :job_id, 
        'add_count', :add_count, 
        'update_count', :update_count, 
        'unchanged_count', :unchanged_count
    );
END;
$$;

-- Identify deleted fitments (existing fitments for these products that are not in incoming staging)
CREATE OR REPLACE PROCEDURE ACES_VALIDATION.MCP.IDENTIFY_DELETED_FITMENTS_JOB(
    job_id VARCHAR
)
RETURNS VARIANT
LANGUAGE SQL
AS
$$
DECLARE
    delete_count INTEGER;
BEGIN
    -- Existing fitments that belong to the products processed in this job, 
    -- but do not exist in STAGING.FITMENT_STAGE (where the distinct_fitment_hash is missing)
    INSERT INTO ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION (job_id, distinct_fitment_hash, classification, change_details)
    SELECT 
        :job_id,
        cat.distinct_fitment_hash,
        'DELETE',
        'fitment_removed_by_supplier'
    FROM ACES_VALIDATION.CATALOG.FITMENT cat
    WHERE cat.product_id IN (
        SELECT DISTINCT product_id 
        FROM ACES_VALIDATION.STAGING.FITMENT_STAGE 
        WHERE job_id = :job_id AND product_id IS NOT NULL
    )
    AND cat.distinct_fitment_hash NOT IN (
        SELECT distinct_fitment_hash 
        FROM ACES_VALIDATION.STAGING.FITMENT_STAGE 
        WHERE job_id = :job_id AND distinct_fitment_hash IS NOT NULL
    );

    SELECT COUNT(*) INTO :delete_count 
    FROM ACES_VALIDATION.STAGING.FITMENT_CLASSIFICATION 
    WHERE job_id = :job_id AND classification = 'DELETE';

    RETURN OBJECT_CONSTRUCT('job_id', :job_id, 'delete_count', :delete_count);
END;
$$;
```

### 3.5 Aggregation Procedures

```sql
-- Compute validation summary statistics for a job
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
    SELECT OBJECT_CONSTRUCT(
        'job_id', :job_id,
        'total_fitments', COUNT(*),
        'valid_count', SUM(CASE WHEN is_valid THEN 1 ELSE 0 END),
        'error_count', SUM(CASE WHEN NOT is_valid THEN 1 ELSE 0 END),
        'vcdb_errors', SUM(CASE WHEN error_type = 'VCDB' THEN 1 ELSE 0 END),
        'qdb_errors', SUM(CASE WHEN error_type = 'QDB' THEN 1 ELSE 0 END),
        'pcdb_errors', SUM(CASE WHEN error_type = 'PCDB' THEN 1 ELSE 0 END),
        'product_errors', SUM(CASE WHEN error_type = 'PRODUCT' THEN 1 ELSE 0 END)
    )
    INTO result
    FROM ACES_VALIDATION.STAGING.FITMENT_STAGE
    WHERE job_id = :job_id;
    
    RETURN result;
END;
$$;

-- Compute uplift classification summary statistics for a job
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
  -- VCDB Validation Tools
  - name: "validate_base_vehicle_ids_job"
    title: "Validate Base Vehicle IDs for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_BASE_VEHICLE_IDS_JOB"
    description: "Validates base vehicle IDs inside FITMENT_STAGE for a specific job against VCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  - name: "validate_attribute_values_job"
    title: "Validate Attribute Values for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_ATTRIBUTE_VALUES_JOB"
    description: "Validates attribute values (DriveType, FuelType) in vehicle_conditions JSON for a specific job."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  - name: "resolve_vehicle_configuration_job"
    title: "Resolve Vehicle Configuration for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.RESOLVE_VEHICLE_CONFIGURATION_JOB"
    description: "Validates that base vehicle + attributes resolve to a valid vehicle in VCDB for a specific job."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  -- QDB Validation Tools
  - name: "validate_qualifier_ids_job"
    title: "Validate Qualifier IDs for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_QUALIFIER_IDS_JOB"
    description: "Validates qualifier IDs inside the qualifiers array for a specific job against QDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  -- PCDB Validation Tools
  - name: "validate_part_type_ids_job"
    title: "Validate Part Type IDs for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_IDS_JOB"
    description: "Validates part type IDs for a specific job against PCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  - name: "validate_position_ids_job"
    title: "Validate Position IDs for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_POSITION_IDS_JOB"
    description: "Validates position IDs for a specific job against PCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  - name: "validate_part_type_position_job"
    title: "Validate Part Type Position Mapping for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.VALIDATE_PART_TYPE_POSITION_JOB"
    description: "Validates part-type-to-position mappings for a specific job against PCDB."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  -- Product Resolution & Comparison Tools
  - name: "resolve_products_job"
    title: "Resolve Supplier Parts for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.RESOLVE_PRODUCTS_JOB"
    description: "Resolves supplier part numbers to catalog product IDs for a specific job."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
          line_code:
            description: "Catalog brand line code"
            type: "string"
        required: ["job_id", "line_code"]

  - name: "classify_fitment_changes_job"
    title: "Classify Fitment Changes for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.CLASSIFY_FITMENT_CHANGES_JOB"
    description: "Compares valid staging fitments vs catalog and classifies them as ADD, UPDATE, or UNCHANGED."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  - name: "identify_deleted_fitments_job"
    title: "Identify Deleted Fitments for Job"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.IDENTIFY_DELETED_FITMENTS_JOB"
    description: "Identifies existing fitments for processed products that are missing from staging (DELETEs)."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  -- Aggregation & Reporting Tools
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
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]

  - name: "compute_uplift_summary"
    title: "Compute Uplift Summary"
    type: "GENERIC"
    identifier: "ACES_VALIDATION.MCP.COMPUTE_UPLIFT_SUMMARY"
    description: "Computes Add/Update/Delete counts for a job."
    config:
      type: "procedure"
      warehouse: "ACES_VALIDATION_WH"
      input_schema:
        type: "object"
        properties:
          job_id:
            description: "Unique validation job identifier"
            type: "string"
        required: ["job_id"]
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
