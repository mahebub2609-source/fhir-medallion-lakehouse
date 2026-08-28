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
* **Silver (Cleansed)**: Validated, deduplicated, and normalized data
* **Gold (Curated)**: Business-level aggregates and feature tables

## Project Structure

```
fhir-medallion-lakehouse/
├── config/
│   └── resources.json          # Ingestion configuration
├── notebooks/
│   ├── 01_raw_ingestion.py     # Bronze layer: Fetch from FHIR API
│   ├── 02_silver_transform.py  # Silver layer: Cleanse and normalize
│   ├── 03_gold_aggregate.py    # Gold layer: Business metrics
│   └── fhir_ingestion_utils.py # Reusable utility functions
├── pipeline/
│   └── [DLT pipeline configs]  # Delta Live Tables definitions
├── data_samples/
│   └── [Sample FHIR data]      # Test/development data
├── docs/
│   └── [Documentation]         # Additional documentation
└── README.md                   # This file
```

## Features

### ✅ Production-Ready Design
* **Error Handling**: Comprehensive exception handling at every layer
* **Retry Logic**: Exponential backoff for API resilience
* **Logging**: Structured logging for observability
* **Validation**: Configuration and data validation

### ✅ FHIR Compliance
* Supports FHIR R4 standard
* Handles standard FHIR resources: Patient, Encounter, Observation, Condition
* Preserves referential integrity
* Pagination support for large datasets

### ✅ Best Practices
* **Separation of Concerns**: Clear boundaries between layers
* **Reusable Components**: Utility modules for common operations
* **Type Hints**: Full type annotations for better IDE support
* **Documentation**: Comprehensive docstrings and README
* **Modular Design**: Easy to extend with new resources

## Getting Started

### Prerequisites
* Databricks workspace (AWS/Azure/GCP)
* Python 3.12+
* Access to a FHIR server (default: public HAPI FHIR server)

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

1. Open `notebooks/01_raw_ingestion`
2. Run all cells sequentially
3. Review the ingestion summary

#### Option 2: Scheduled Job (Recommended for Production)

```python
# Create a Databricks job
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
job = w.jobs.create(
    name="FHIR Ingestion Pipeline",
    tasks=[
        {
            "task_key": "bronze_ingestion",
            "notebook_task": {
                "notebook_path": "/path/to/01_raw_ingestion"
            }
        }
    ],
    schedule={"quartz_cron_expression": "0 0 * * * ?"}  # Hourly
)
```

## Data Flow

### 1. Bronze Layer (Raw Ingestion)
**Notebook**: `01_raw_ingestion`

* Fetches data from FHIR API
* Handles pagination automatically
* Stores raw JSON responses
* Captures full metadata (timestamps, URLs, page numbers)
* **Output**: `data/raw/{date}/{resource}/page_*.json`

### 2. Silver Layer (Cleansing)
**Notebook**: `02_silver_transform` _(To be implemented)_

* Validates FHIR resources against schema
* Deduplicates records
* Normalizes nested structures
* Applies data quality rules
* **Output**: Delta tables in `silver.{resource}` schema

### 3. Gold Layer (Business Metrics)
**Notebook**: `03_gold_aggregate` _(To be implemented)_

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
* Records ingested per resource
* API response times
* Error rates by resource type
* Storage growth

## Migration from Fabric Lakehouse

This project was designed to be portable between Microsoft Fabric and Databricks:

| Fabric Lakehouse | Databricks |
|------------------|------------|
| `/lakehouse/default/Files/` | `/Workspace/path/` or `/Volumes/` |
| Notebook utilities | `dbutils` |
| Spark runtime | Databricks Runtime |

## Roadmap

- [x] Bronze layer: Raw ingestion with retry logic
- [x] Utility module for reusable functions
- [x] Configuration management
- [ ] Silver layer: Data cleansing and validation
- [ ] Gold layer: Business metrics and aggregations
- [ ] Delta Live Tables pipeline
- [ ] Unity Catalog integration
- [ ] Data quality monitoring
- [ ] ML feature store integration

## Contributing

### Adding New FHIR Resources

1. Add resource name to `config/resources.json`
2. Ensure referential integrity (dependencies must be fetched first)
3. Update documentation

### Testing

```python
# Use the public HAPI FHIR test server
config = {
    "base_url": "https://hapi.fhir.org/baseR4",
    "lookback_days": 1,  # Minimal data for testing
    "page_count": 10
}
```

## Resources

* [FHIR R4 Specification](https://hl7.org/fhir/R4/)
* [Databricks Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
* [HAPI FHIR Server](https://hapi.fhir.org/)
* [Delta Live Tables](https://docs.databricks.com/delta-live-tables/index.html)

## License

This project is provided as-is for educational and development purposes.

## Support

For issues or questions:
1. Check the documentation in `docs/`
2. Review logs for detailed error messages
3. Verify FHIR server connectivity
4. Ensure configuration is valid

---

**Built with** ❤️ **for healthcare data engineering**