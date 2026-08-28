# FHIR Medallion Lakehouse Architecture

## Overview

This project implements a medallion architecture for FHIR healthcare data, organizing data into four progressive layers: Raw, Bronze, Silver, and Gold. Each layer adds increasing value through structure, quality, and optimization.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FHIR API Sources                             │
│                  (Patient, Observation, Encounter, etc.)             │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           RAW LAYER                                  │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ • Untouched API responses (JSON)                            │    │
│  │ • Complete audit trail                                      │    │
│  │ • Enables replay & reprocessing                             │    │
│  │ • Immutable source of truth                                 │    │
│  │                                                              │    │
│  │ Storage: raw_fhir.{resource_type}_raw                       │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         BRONZE LAYER                                 │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ • Structured Delta tables                                   │    │
│  │ • Flattened FHIR resources                                  │    │
│  │ • Added technical metadata:                                 │    │
│  │   - ingestion_timestamp                                     │    │
│  │   - source_file/API endpoint                                │    │
│  │   - record_hash for deduplication                           │    │
│  │ • Minimal transformations                                   │    │
│  │ • Preserves data lineage                                    │    │
│  │                                                              │    │
│  │ Storage: bronze_fhir.{resource_type}                        │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         SILVER LAYER                                 │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ • Cleaned & validated data                                  │    │
│  │ • SCD Type 2 (Slowly Changing Dimensions):                  │    │
│  │   - valid_from / valid_to timestamps                        │    │
│  │   - is_current flag                                         │    │
│  │   - Complete version history                                │    │
│  │ • Business rules applied:                                   │    │
│  │   - Data type validation                                    │    │
│  │   - Referential integrity checks                            │    │
│  │   - FHIR profile conformance                                │    │
│  │ • Standardized formats & units                              │    │
│  │ • Deduplicated records                                      │    │
│  │                                                              │    │
│  │ Storage: silver_fhir.{resource_type}                        │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          GOLD LAYER                                  │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ • Denormalized reporting tables                             │    │
│  │ • Business-level aggregations                               │    │
│  │ • Optimized for analytics & BI tools                        │    │
│  │ • Pre-joined dimensions                                     │    │
│  │ • KPIs and metrics ready for consumption                    │    │
│  │                                                              │    │
│  │ Examples:                                                    │    │
│  │   - patient_summary (demographics + latest vitals)          │    │
│  │   - encounter_metrics (admission stats, readmissions)       │    │
│  │   - clinical_quality_measures (HEDIS, CMS measures)         │    │
│  │                                                              │    │
│  │ Storage: gold_fhir.{business_entity}                        │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
                    ┌────────────────────────────┐
                    │  BI Tools & Dashboards     │
                    │  • Power BI                │
                    │  • Databricks SQL          │
                    │  • Tableau                 │
                    └────────────────────────────┘
```

## Layer Details

### Raw Layer
**Purpose:** Immutable archive of source data

- **Content:** Exact API responses in JSON format, no modifications
- **Use Cases:**
  - Audit trail for regulatory compliance
  - Ability to replay ingestion if Bronze/Silver logic changes
  - Historical record of what the source system provided
  - Debugging data quality issues at the source

**Key Characteristics:**
- No schema enforcement (schema-on-read)
- No data quality checks
- Complete preservation of source format
- Timestamped for temporal ordering

---

### Bronze Layer
**Purpose:** Structured foundation for data engineering

- **Content:** Structured Delta tables with flattened FHIR resources
- **Metadata Added:**
  - `ingestion_timestamp`: When the record was ingested
  - `source_system`: API endpoint or source identifier
  - `record_hash`: MD5/SHA hash for deduplication
  - `raw_json`: Optional reference back to raw layer
  
**Transformations:**
- JSON flattening (nested structures → columns)
- Basic type casting (strings → dates, numbers)
- Array explosion where needed
- Primary key extraction (FHIR resource IDs)

**Key Characteristics:**
- Delta Lake format (ACID transactions)
- Append-only or upsert patterns
- 1:1 mapping with FHIR resources
- Minimal business logic

---

### Silver Layer
**Purpose:** Clean, versioned, business-ready data

- **Content:** Validated and cleaned data with full history
- **SCD Type 2 Implementation:**
  - `valid_from`: Start of record validity period
  - `valid_to`: End of validity (NULL = current)
  - `is_current`: Boolean flag for latest version
  - `version_number`: Incremental version counter
  - `change_type`: INSERT, UPDATE, DELETE

**Transformations:**
- Data quality rules:
  - Remove duplicates
  - Validate against FHIR profiles
  - Check referential integrity (Patient references exist)
  - Standardize code systems (SNOMED, LOINC)
- Business enrichment:
  - Calculate derived fields
  - Apply business rules
  - Resolve terminology mappings
  
**Key Characteristics:**
- Complete temporal history preserved
- Point-in-time queries supported
- Data quality score/flags
- Conformed dimensions

---

### Gold Layer
**Purpose:** Analytics-optimized business entities

- **Content:** Denormalized tables optimized for specific use cases
- **Examples:**

**`patient_summary`**
- Demographics (name, DOB, gender, address)
- Latest vitals (BP, weight, BMI)
- Active conditions
- Current medications
- Last encounter date
- Risk scores

**`encounter_metrics`**
- Admission/discharge statistics
- Length of stay calculations
- Readmission flags (30/60/90 day)
- Department/facility breakdowns
- Cost/billing summaries

**`clinical_quality_measures`**
- HEDIS measures
- CMS quality metrics
- Patient cohort assignments
- Care gap identification

**Key Characteristics:**
- Pre-aggregated where possible
- Denormalized (minimal joins needed)
- Business-friendly column names
- Optimized indexing/partitioning
- May include calculated KPIs

---

## Data Flow Principles

### 1. **Idempotency**
All pipelines can be re-run safely without duplicating data.

### 2. **Incremental Processing**
Process only new/changed data where possible (via watermarks, checksums).

### 3. **Data Lineage**
Every record traces back through Silver → Bronze → Raw.

### 4. **Quality Gates**
Data quality checks at each layer boundary:
- Bronze → Silver: Schema validation, null checks
- Silver → Gold: Business rule validation, completeness checks

### 5. **Temporal Consistency**
SCD Type 2 in Silver enables point-in-time queries and historical analysis.

---

## Technology Stack

- **Storage:** Unity Catalog (3-level namespace: catalog.schema.table)
- **Processing:** Spark Declarative Pipelines (formerly DLT)
- **Format:** Delta Lake (all layers)
- **Orchestration:** Databricks Jobs
- **Governance:** Unity Catalog (access control, lineage, auditing)

---

## Naming Conventions

### Catalogs
- `raw_fhir` - Raw layer
- `bronze_fhir` - Bronze layer
- `silver_fhir` - Silver layer
- `gold_fhir` - Gold layer

### Tables
- Raw: `{resource_type}_raw` (e.g., `patient_raw`)
- Bronze: `{resource_type}` (e.g., `patient`)
- Silver: `{resource_type}` (e.g., `patient`)
- Gold: `{business_entity}` (e.g., `patient_summary`, `encounter_metrics`)

---