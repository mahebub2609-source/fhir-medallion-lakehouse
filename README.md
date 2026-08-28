# FHIR Medallion Lakehouse

[![Databricks](https://img.shields.io/badge/Platform-Databricks-FF3621?logo=databricks)](https://databricks.com)
[![FHIR](https://img.shields.io/badge/Standard-FHIR%20R4-blue)](https://hl7.org/fhir/)
[![Python](https://img.shields.io/badge/Python-3.12+-blue.svg)](https://www.python.org/downloads/)

## Overview

A production-ready data lakehouse implementation for healthcare data using the FHIR (Fast Healthcare Interoperability Resources) standard and the Medallion Architecture pattern.

### Architecture: Medallion Pattern

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│             │     │             │     │             │
│    BRONZE   │────▶│   SILVER    │────▶│    GOLD     │
│   (Raw)     │     │ (Cleansed)  │     │ (Curated)   │
│             │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘
  ↑                                            │
  │                                            ↓
FHIR API                               Analytics/ML/BI
```

* **Bronze (Raw)**: Unmodified API responses with full traceability
* **Silver (Cleansed)**: Validated, deduplicated, and normalized data with SCD Type 2 versioning
* **Gold (Curated)**: Business-level aggregates and feature tables

## Key Features

### SCD Type 2 Versioning (Silver Layer)

The silver layer implements **Slowly Changing Dimension Type 2** to maintain complete historical records:

```sql
-- Example: Track patient record changes over time
SELECT 
  resource_id,
  valid_from,
  valid_to,
  is_current,
  row_hash
FROM silver.patient
WHERE resource_id = '137584287'
ORDER BY valid_from;
```

**How it works**:
1. **Hash-based change detection**: MD5 hash of record content identifies modifications
2. **Version tracking**: Each change creates a new version with timestamps
3. **Current flag**: `is_current = true` marks the latest version
4. **Historical preservation**: Old versions remain with `valid_to` timestamp set

**Benefits**:
* Complete audit trail of all changes
* Point-in-time queries ("show data as of date X")
* Change analysis ("when did this patient's status change?")
* Data quality tracking across time

## Project Structure

```
fhir-medallion-lakehouse/
├── config/
│   └── resources.json          # Ingestion configuration
├── notebooks/
│   ├── 01_raw_ingestion.py     # Bronze layer: Fetch from FHIR API
│   ├── 02_bronze_transform.py  # Bronze layer: Load raw data to Delta tables
│   ├── 03_scd2_versioning.py   # Silver layer: SCD Type 2 versioning
│   ├── 04_silver_clean.py      # Silver layer: Cleanse and normalize
│   └── fhir_ingestion_utils.py # Reusable utility functions
├── data/
│   └── raw/                    # Raw FHIR JSON files by date/resource
├── docs/
│   └── [Documentation]         # Additional documentation
└── README.md                   # This file
```

## Features

### Production-Ready Design
* **Error Handling**: Comprehensive exception handling at every layer
* **Retry Logic**: Exponential backoff for API resilience
* **Logging**: Structured logging for observability
* **Validation**: Configuration and data validation

### FHIR Compliance
* Supports FHIR R4 standard
* Handles standard FHIR resources: Patient, Encounter, Observation, Condition
* Preserves referential integrity across resources
* Pagination support for large datasets
* Nested JSON structure preservation with schema inference

### Data Versioning & Quality
* **SCD Type 2 Versioning**: Full historical tracking of record changes
* **Hash-based Change Detection**: Efficient identification of modified records
* **Data Quality Tracking**: `valid_from`, `valid_to`, `is_current` columns
* **Reference Resolution**: Extracts IDs from FHIR reference fields
* **Schema Evolution**: Handles updates to FHIR resource structures

### Best Practices
* **Separation of Concerns**: Clear boundaries between layers
* **Reusable Components**: Utility modules for common operations
* **Type Hints**: Full type annotations for better IDE support
* **Documentation**: Comprehensive docstrings and README
* **Modular Design**: Easy to extend with new resources

## Getting Started

### Prerequisites
* **Databricks workspace** (AWS/Azure/GCP)
* **Serverless compute** (default) or cluster with Python 3.12+
* **Unity Catalog** enabled (for `bronze.*` and `silver.*` schemas)
* **Workspace folder access** for storing raw data files
* **Access to a FHIR server** (default: public HAPI FHIR server at `https://hapi.fhir.org/baseR4`)

### Important Notes
* This project uses **serverless compute** which does not support DBFS paths (`/dbfs/`, `dbfs:/`)
* All file paths use workspace paths with `file:` prefix
* SQL cells execute native SQL; Python code must be in Python cells

### Storage Locations

**Raw Data Files** (JSON from FHIR API):
```
file:/Workspace/Users/{user}/fhir-medallion-lakehouse/data/raw/{date}/{resource}/
```

**Delta Tables**:
* Bronze layer: `bronze.patient`, `bronze.encounter`, `bronze.observation`, `bronze.condition`
* Silver layer (versioned): `silver.patient`, `silver.encounter`, etc.
* Silver layer (clean): `silver.patient_clean`, `silver.encounter_clean`, etc.
* Gold layer: `gold.*` (to be implemented)

### Configuration

1. **Edit** `config/resources.json`:

```json
{
  "base_url": "https://hapi.fhir.org/baseR4",
  "resources": ["Patient", "Encounter", "Observation", "Condition"],
  "page_count": 20,
  "lookback_days": 3
}
```

| Parameter | Description | Default |
|-----------|-------------|----------|
| `base_url` | FHIR server endpoint | Public HAPI server |
| `resources` | List of FHIR resource types | 4 core resources |
| `page_count` | Records per API page | 20 |
| `lookback_days` | Days to look back | 3 |

### Running the Pipeline

#### Option 1: Manual Execution

Run notebooks in sequence:

1. **Bronze Layer - Raw Ingestion**: `notebooks/01_raw_ingestion`
   - Fetches FHIR data from API
   - Saves raw JSON to filesystem

2. **Bronze Layer - Delta Tables**: `notebooks/02_bronze_transform`
   - Loads raw JSON into Delta tables
   - Creates `bronze.patient`, `bronze.encounter`, etc.

3. **Silver Layer - Versioning**: `notebooks/03_scd2_versioning`
   - Implements SCD Type 2 historical tracking
   - Creates versioned `silver.{resource}` tables

4. **Silver Layer - Cleansing**: `notebooks/04_silver_clean`
   - Extracts structured fields from JSON
   - Creates clean `silver.{resource}_clean` tables

#### Option 2: Scheduled Job (Recommended for Production)

```python
# Create a Databricks job with all pipeline tasks
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
job = w.jobs.create(
    name="FHIR Medallion Pipeline",
    tasks=[
        {
            "task_key": "01_raw_ingestion",
            "notebook_task": {
                "notebook_path": "/Users/{user}/fhir-medallion-lakehouse/notebooks/01_raw_ingestion"
            }
        },
        {
            "task_key": "02_bronze_transform",
            "depends_on": [{"task_key": "01_raw_ingestion"}],
            "notebook_task": {
                "notebook_path": "/Users/{user}/fhir-medallion-lakehouse/notebooks/02_bronze_transform"
            }
        },
        {
            "task_key": "03_scd2_versioning",
            "depends_on": [{"task_key": "02_bronze_transform"}],
            "notebook_task": {
                "notebook_path": "/Users/{user}/fhir-medallion-lakehouse/notebooks/03_scd2_versioning"
            }
        },
        {
            "task_key": "04_silver_clean",
            "depends_on": [{"task_key": "03_scd2_versioning"}],
            "notebook_task": {
                "notebook_path": "/Users/{user}/fhir-medallion-lakehouse/notebooks/04_silver_clean"
            }
        }
    ],
    schedule={"quartz_cron_expression": "0 0 */6 * * ?"}  # Every 6 hours
)
```

## Table Schemas

### Bronze Tables: `bronze.{resource}`
Raw FHIR data loaded from JSON files.

| Column | Type | Description |
|--------|------|-------------|
| `resource_type` | string | FHIR resource type (Patient, Encounter, etc.) |
| `resource_json` | struct | Full nested FHIR JSON structure |
| `resource_id` | string | FHIR resource ID |
| `bronze_load_date` | string | Date when data was loaded |
| `ingestion_ts` | timestamp | Timestamp of ingestion |
| `extraction_timestamp` | string | API extraction timestamp |
| `api_url_or_params` | string | Source API URL or parameters |

### Silver Tables: `silver.{resource}`
Versioned tables with SCD Type 2 tracking.

| Column | Type | Description |
|--------|------|-------------|
| All bronze columns | - | Inherited from bronze layer |
| `row_hash` | string | MD5 hash of record for change detection |
| `valid_from` | timestamp | Start of validity period |
| `valid_to` | timestamp | End of validity (null = current) |
| `is_current` | boolean | True for current version, false for history |

### Silver Clean Tables: `silver.{resource}_clean`
Flattened, analytical-ready tables.

**Example**: `silver.encounter_clean`
| Column | Type | Description |
|--------|------|-------------|
| `resource_id` | string | Encounter ID |
| `patient_id` | string | Referenced patient ID (resolved from reference) |
| `status` | string | Encounter status (finished, in-progress, etc.) |
| Additional extracted fields | - | Resource-specific structured data |

## Data Flow

### 1. Bronze Layer (Raw Ingestion)

#### Step 1: API Ingestion
**Notebook**: `01_raw_ingestion`

* Fetches data from FHIR API with pagination
* Handles retry logic with exponential backoff
* Stores raw JSON responses to filesystem
* Captures full metadata (timestamps, URLs, page numbers)
* **Output**: `file:/Workspace/Users/{user}/fhir-medallion-lakehouse/data/raw/{date}/{resource}/`

#### Step 2: Bronze Delta Tables
**Notebook**: `02_bronze_transform`

* Reads raw JSON files from filesystem
* Loads data into Delta tables using Spark's `read_files()` with schema inference
* Preserves complete raw FHIR JSON structure
* Adds metadata columns: `resource_id`, `bronze_load_date`, `ingestion_ts`
* **Output**: Delta tables `bronze.{resource}` (e.g., `bronze.patient`, `bronze.encounter`)

### 2. Silver Layer (Cleansing & Versioning)

#### Step 1: SCD Type 2 Versioning
**Notebook**: `03_scd2_versioning`

* Implements Slowly Changing Dimension Type 2 for historical tracking
* Compares incoming data with existing records using row hash
* Tracks record versions with `valid_from`, `valid_to`, `is_current` columns
* Handles updates by closing old records and creating new versions
* **Output**: Versioned Delta tables `silver.{resource}` with full history

#### Step 2: Data Cleansing
**Notebook**: `04_silver_clean`

* Extracts structured fields from nested FHIR JSON
* Resolves FHIR references (e.g., `patient_id` from encounter references)
* Validates data quality and applies business rules
* Normalizes nested structures for analytical queries
* **Output**: Clean tables `silver.{resource}_clean` (e.g., `silver.encounter_clean`)

### 3. Gold Layer (Business Metrics)
_(To be implemented)_

* Creates business-level aggregations
* Builds feature tables for ML
* Generates reporting views
* **Output**: Delta tables in `gold.*` schema

## Development Guidelines

### Code Organization

1. **Notebooks**: High-level orchestration and documentation
2. **Utilities**: Reusable functions in separate `.py` modules
3. **Configuration**: Externalized in JSON files (no hardcoding)

### Naming Conventions

* **Notebooks**: `##_verb_noun.py` (e.g., `01_raw_ingestion.py`)
* **Functions**: `snake_case` with descriptive names
* **Classes**: `PascalCase` for custom exceptions and models
* **Constants**: `UPPER_SNAKE_CASE` for configuration values

### Error Handling Strategy

```python
# Custom exceptions for domain-specific errors
class FHIRIngestionError(Exception):
    pass

# Fail fast for configuration errors
if not validate_config(config):
    raise ValueError("Invalid configuration")

# Graceful degradation for individual resource failures
for resource in resources:
    try:
        fetch_resource(resource)
    except FHIRIngestionError:
        log_error_and_continue()
```

## Monitoring and Observability

### Logging Levels
* **DEBUG**: Detailed page-by-page progress
* **INFO**: High-level progress and summaries
* **WARNING**: Recoverable errors (e.g., failed resource fetch)
* **ERROR**: Critical failures requiring attention

### Key Metrics to Monitor
* **Ingestion Metrics**:
  - Records ingested per resource per run
  - API response times and pagination performance
  - Error rates by resource type
* **Data Quality Metrics**:
  - Row hash collision rate (change detection accuracy)
  - Percentage of current vs historical records in silver tables
  - Reference resolution success rate (e.g., patient_id extraction)
* **Performance Metrics**:
  - Bronze to silver transformation duration
  - SCD2 merge operation duration
  - Storage growth by layer (bronze/silver/gold)
* **Data Lineage**:
  - Track `bronze_load_date` to identify data freshness
  - Monitor `valid_from`/`valid_to` gaps in versioned tables

### Query Examples for Monitoring

```sql
-- Count current vs historical records
SELECT 
  is_current,
  COUNT(*) as record_count,
  COUNT(DISTINCT resource_id) as unique_resources
FROM silver.patient
GROUP BY is_current;

-- Identify records with multiple versions (indicating changes)
SELECT 
  resource_id, 
  COUNT(*) as version_count
FROM silver.patient
GROUP BY resource_id
HAVING COUNT(*) > 1
ORDER BY version_count DESC;

-- Check data freshness
SELECT 
  bronze_load_date,
  COUNT(*) as record_count
FROM bronze.patient
GROUP BY bronze_load_date
ORDER BY bronze_load_date DESC;
```

## Migration from Fabric Lakehouse

This project was designed to be portable between Microsoft Fabric and Databricks:

| Fabric Lakehouse | Databricks (Serverless) | Notes |
|------------------|------------------------|-------|
| `/lakehouse/default/Files/` | `file:/Workspace/Users/{user}/` | Workspace paths for file storage |
| DBFS paths (`/dbfs/`) | Not supported | Use workspace or Unity Catalog volumes |
| `dbfs:/` protocol | `file:` protocol | Required for serverless compute |
| Notebook utilities | `dbutils` | Available in Databricks |
| Spark runtime | Databricks Runtime | Full Spark API support |
| Delta tables | Delta tables | Same format, Unity Catalog integration |
| SQL cells | SQL cells | Language must match cell type |

### Key Differences

1. **File Paths**: Use `file:/Workspace/...` instead of `/dbfs/` or `dbfs:/`
2. **Cell Language**: SQL cells cannot execute Python code (`spark.sql()` must be in Python cells)
3. **Unity Catalog**: Databricks uses UC for schema management (`catalog.schema.table` format)
4. **Serverless Compute**: Default compute type with some path restrictions

## Roadmap

- [x] Bronze layer: Raw ingestion with retry logic
- [x] Bronze layer: Delta table loading from raw JSON
- [x] Utility module for reusable functions
- [x] Configuration management
- [x] Silver layer: SCD Type 2 versioning for historical tracking
- [x] Silver layer: Data cleansing and reference resolution
- [ ] Gold layer: Business metrics and aggregations
- [ ] Delta Live Tables pipeline migration
- [ ] Unity Catalog integration with proper governance
- [ ] Data quality monitoring and alerting
- [ ] ML feature store integration
- [ ] Automated testing suite

## Contributing

### Adding New FHIR Resources

1. Add resource name to `config/resources.json`
2. Ensure referential integrity (dependencies must be fetched first)
3. Update documentation

### Testing

#### Test Data Setup
```python
# Use the public HAPI FHIR test server
config = {
    "base_url": "https://hapi.fhir.org/baseR4",
    "lookback_days": 1,  # Minimal data for testing
    "page_count": 10
}
```

#### Verify Pipeline Execution
```sql
-- Check bronze layer loaded successfully
SELECT COUNT(*) as total_records FROM bronze.patient;

-- Verify SCD2 versioning is working
SELECT 
  COUNT(DISTINCT resource_id) as unique_patients,
  SUM(CASE WHEN is_current THEN 1 ELSE 0 END) as current_versions,
  SUM(CASE WHEN NOT is_current THEN 1 ELSE 0 END) as historical_versions
FROM silver.patient;

-- Test reference resolution in clean tables
SELECT resource_id, patient_id, status 
FROM silver.encounter_clean 
WHERE patient_id IS NOT NULL 
LIMIT 10;
```

#### Simulate Data Updates for SCD2 Testing
```python
# In notebook 02_bronze_transform, Cell 3
# Modify a sample of records and reload to test versioning
from pyspark.sql import functions as F

fake_date = '2026-08-29'
sample = spark.table('bronze.patient').limit(5)
modified = (
    sample
    .withColumn('resource_json', 
                F.col('resource_json').withField('gender', F.lit('unknown')))
    .withColumn('bronze_load_date', F.lit(fake_date))
    .withColumn('ingestion_ts', F.current_timestamp())
)
modified.write.format('delta').mode('append').saveAsTable('bronze.patient')

# Then run 03_scd2_versioning to see new versions created
```

## Resources

* [FHIR R4 Specification](https://hl7.org/fhir/R4/)
* [Databricks Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
* [HAPI FHIR Server](https://hapi.fhir.org/)
* [Delta Live Tables](https://docs.databricks.com/delta-live-tables/index.html)

## License

This project is provided as-is for educational and development purposes.

## Troubleshooting

### Common Issues and Solutions

#### 1. `CAST_INVALID_INPUT` errors in SQL queries
**Symptom**: Error when filtering on `resource_id` with numeric values
```
[CAST_INVALID_INPUT] The value 'specialist-demo-20260822-patient-002' of the type "STRING" cannot be cast to "BIGINT"
```
**Cause**: `resource_id` is a string column, but numeric literals in `IN` clauses trigger implicit casting.

**Solution**: Quote numeric values in WHERE clauses:
```sql
-- Wrong:
WHERE resource_id IN (137584287, 137584285)

-- Correct:
WHERE resource_id IN ('137584287', '137584285')
```

#### 2. `PARSE_SYNTAX_ERROR` in SQL cells
**Symptom**: Syntax error when cell contains Python code

**Cause**: SQL cells (with `language='sql'`) cannot execute Python code like `spark.sql(...).show()`.

**Solution**: Use native SQL in SQL cells:
```sql
-- Instead of: spark.sql("SELECT ...").show()
-- Use:
SELECT * FROM table_name LIMIT 10
```

#### 3. `IndentationError` in Python cells
**Symptom**: Multiple statements crammed on one line

**Solution**: Properly format Python code with line breaks and consistent indentation:
```python
# Wrong:
fake_date = '2026-08-29' sample = spark.table('bronze.patient').limit(5)

# Correct:
fake_date = '2026-08-29'
sample = spark.table('bronze.patient').limit(5)
```

#### 4. `SyntaxError: invalid character` with em dash
**Symptom**: `invalid character '—' (U+2014)`

**Cause**: Em dash (—) used instead of comment marker (#) or hyphen (-)

**Solution**: Replace em dashes with proper Python comment markers:
```python
# Wrong:
df.show() — display results

# Correct:
df.show()  # display results
```

#### 5. File path errors (DBFS_DISABLED)
**Symptom**: Cannot access paths starting with `/dbfs/` or `dbfs:/`

**Cause**: DBFS is disabled on serverless compute.

**Solution**: Use workspace paths with `file:` prefix:
```python
# Wrong:
path = "/dbfs/data/raw/"

# Correct:
path = "file:/Workspace/Users/{user}/fhir-medallion-lakehouse/data/raw/"
```

## Support

For issues or questions:
1. Check the [Troubleshooting](#troubleshooting) section above
2. Review logs for detailed error messages
3. Verify FHIR server connectivity
4. Ensure configuration is valid
5. Check that file paths use the correct format for your compute type

---

**Built with**  **for healthcare data engineering**