# FHIR Medallion Lakehouse

[![Databricks](https://img.shields.io/badge/Platform-Databricks-FF3621?logo=databricks)](https://databricks.com)
[![FHIR](https://img.shields.io/badge/Standard-FHIR%20R4-blue)](https://hl7.org/fhir/)
[![Python](https://img.shields.io/badge/Python-3.12+-blue.svg)](https://www.python.org/downloads/)

## Project Summary

This project implements a **medallion architecture data lakehouse** for FHIR (Fast Healthcare Interoperability Resources) R4 healthcare data. It ingests four core FHIR resource types — **Patient**, **Encounter**, **Observation**, and **Condition** — from the public HAPI FHIR server, then progressively refines the data through three layers:

* **Bronze** — Raw FHIR API responses loaded into Delta tables with full traceability metadata.
* **Silver** — SCD Type 2 versioned tables that track every change to a resource over time, plus cleansed/normalized tables with extracted structured fields.
* **Gold** — Business-level joined views for analytics, reporting, and BI consumption.

The pipeline is orchestrated as a Databricks Job with five tasks chained by dependencies, so running the job executes all five notebooks in the correct order automatically.

## Platform

| Component | Detail |
|---|---|
| **Cloud platform** | Databricks (serverless compute) |
| **Storage format** | Delta Lake (ACID transactions, time travel) |
| **Catalog** | Unity Catalog (bronze.\*, silver.\*, gold.\* schemas) |
| **Source API** | HAPI FHIR R4 — https://hapi.fhir.org/baseR4 |
| **Languages** | Python (PySpark) and SQL |
| **Orchestration** | Databricks Jobs (5-task pipeline with dependency chain) |

## How to Run

### Option 1 — Run Notebooks Manually (in order 01 → 05)

| Step | Notebook | What It Does | Output |
|---|---|---|---|
| **01** | `notebooks/01_FHIR Raw Data Ingestion` | Fetches Patient, Encounter, Observation, and Condition resources from the HAPI FHIR API with pagination and retry logic. Saves raw JSON to `data/raw/{date}/{resource}/`. | Raw JSON files |
| **02** | `notebooks/02_bronze_transform` | Reads raw JSON, explodes FHIR Bundle entries, and writes to Delta tables in the `bronze` schema with traceability columns. | `bronze.patient`, `bronze.encounter`, `bronze.observation`, `bronze.condition` |
| **03** | `notebooks/03_scd2_versioning` | Applies SCD Type 2 merge logic: hashes each record, detects changes, expires old versions (`valid_to` + `is_current = false`), and inserts new versions. | `silver.patient`, `silver.encounter`, `silver.observation`, `silver.condition` |
| **04** | `notebooks/04_silver_clean` | Extracts structured fields from nested FHIR JSON (names, dates, codes, reference IDs) and creates flattened clean tables for current records only. | `silver.patient_clean`, `silver.encounter_clean`, `silver.observation_clean`, `silver.condition_clean` |
| **05** | `notebooks/05_gold_views` | Joins silver clean tables to create denormalized business views for analytics. | `gold.patient_encounter_summary`, `gold.encounter_observations`, `gold.patient_conditions` |

> **Important:** Run notebooks strictly in order 01 → 05. Each notebook depends on the tables created by the previous one.

### Option 2 — Run the Orchestrated Pipeline (Recommended)

A Databricks Job definition is stored in `pipeline/pipeline_definition.json`. It chains all five notebooks with `ALL_SUCCESS` dependencies and a daily cron schedule.

```
01_raw_ingestion → 02_bronze_transform → 03_scd2_versioning → 04_silver_clean → 05_gold_views
```

To create the job from the definition file, use the Databricks CLI or Jobs UI, pointing to the notebook paths under `/Workspace/Users/{user}/fhir-medallion-lakehouse/notebooks/`.

## Row Counts by Layer

Row counts below reflect the actual tables at the time of the last pipeline run.

### Bronze Layer (Raw Delta Tables)

| Table | Row Count |
|---|---|
| `bronze.patient` | 4,742 |
| `bronze.encounter` | 758 |
| `bronze.observation` | 8,884 |
| `bronze.condition` | 639 |

> Bronze patient count is higher than silver because the SCD2 demo inserted modified patient rows with a second `bronze_load_date` to simulate an updated data pull.

### Silver Layer (SCD Type 2 Versioned)

| Table | Row Count |
|---|---|
| `silver.patient` | 931 |
| `silver.encounter` | 146 |
| `silver.observation` | 1,748 |
| `silver.condition` | 122 |

### Silver Layer (Clean / Current-Only)

| Table | Row Count |
|---|---|
| `silver.patient_clean` | 921 |
| `silver.encounter_clean` | 146 |
| `silver.observation_clean` | 1,748 |
| `silver.condition_clean` | 122 |

### Gold Layer (Business Views)

| Table | Row Count |
|---|---|
| `gold.patient_encounter_summary` | 116 |
| `gold.encounter_observations` | 212 |
| `gold.patient_conditions` | 113 |

## SCD Type 2 Proof

The SCD2 logic in notebook 03 was validated by simulating a data change:

1. Five patient records were copied from `bronze.patient` with `gender` modified to `'unknown'` and `bronze_load_date` set to `2026-08-29`.
2. `apply_scd2('Patient', '2026-08-29')` detected the hash change and inserted new versions.
3. Old versions were expired: `valid_to` was set to the current timestamp and `is_current` flipped to `false`.
4. Each of the 5 modified patients now has **3 versions** in `silver.patient`.

**Query — expired (old) versions:**

```sql
SELECT resource_id, valid_from, valid_to, is_current
FROM silver.patient
WHERE is_current = false
LIMIT 10;
```

Result shows old versions with `valid_to` populated and `is_current = false`:

| resource_id | valid_from | valid_to | is_current |
|---|---|---|---|
| 137584287 | 2026-08-28 13:22:40 | 2026-08-28 13:44:24 | false |
| 137584285 | 2026-08-28 13:22:40 | 2026-08-28 13:44:24 | false |
| 137584306 | 2026-08-28 13:22:40 | 2026-08-28 13:44:24 | false |
| 137584286 | 2026-08-28 13:22:40 | 2026-08-28 13:44:24 | false |
| 137584300 | 2026-08-28 13:22:40 | 2026-08-28 13:44:24 | false |

**Query — current (new) versions:**

```sql
SELECT resource_id, valid_from, is_current
FROM silver.patient
WHERE is_current = true
  AND resource_id IN ('137584287','137584285','137584306','137584286','137584300');
```

All five show `is_current = true` with a newer `valid_from` timestamp.

**Query — version count per modified patient:**

```sql
SELECT resource_id, COUNT(*) AS version_count
FROM silver.patient
WHERE resource_id IN ('137584287','137584285','137584306','137584286','137584300')
GROUP BY resource_id;
```

| resource_id | version_count |
|---|---|
| 137584285 | 3 |
| 137584300 | 3 |
| 137584286 | 3 |
| 137584306 | 3 |
| 137584287 | 3 |

### SCD2 Proof Screenshots

<!-- Paste your SCD2 proof screenshots below. Replace the placeholder lines with: -->
<!-- ![description](docs/screenshots/scd2_expired_versions.png) -->

**Screenshot 1 — Expired versions (is_current = false, valid_to populated):**
> _Paste screenshot here showing the query result with old versions expired._

**Screenshot 2 — Current versions (is_current = true with new valid_from):**
> _Paste screenshot here showing the query result with new current versions._

**Screenshot 3 — Version count (3 versions per modified patient):**
> _Paste screenshot here showing the GROUP BY query result._

## Orchestration Run Screenshot

The pipeline runs as a Databricks Job with five tasks chained by `ALL_SUCCESS` dependencies:

```
raw_ingestion → 02_bronze_transform → 03_scd2_versioning → 04_silver_clean → 05_gold_views
```

Schedule: daily cron `27 20 21 * * ?` (Asia/Kolkata timezone).

<!-- Paste your Databricks Job run screenshot below. Replace the placeholder line with: -->
<!-- ![orchestration run](docs/screenshots/job_run.png) -->

**Screenshot — Databricks Job orchestration run (all 5 tasks succeeded):**
> _Paste screenshot here showing the Jobs UI with all five tasks green/succeeded._

## Project Structure

```
fhir-medallion-lakehouse/
├── README.md                           # This file
├── config/
│   └── resources.json                  # Ingestion configuration (FHIR URL, resources, page size)
├── notebooks/
│   ├── 01_FHIR Raw Data Ingestion       # Bronze: fetch from FHIR API → raw JSON
│   ├── 02_bronze_transform              # Bronze: raw JSON → Delta tables
│   ├── 03_scd2_versioning               # Silver: SCD Type 2 merge & versioning
│   ├── 04_silver_clean                  # Silver: extract fields → clean tables
│   └── 05_gold_views                    # Gold: join silver clean → business views
├── pipeline/
│   └── pipeline_definition.json        # Databricks Job definition (5-task chain)
├── data/
│   └── raw/                             # Raw FHIR JSON files by date/resource
├── data_samples/                        # Sample raw JSON for reference
└── docs/
    ├── architecture.md                  # Detailed architecture documentation
    └── table_relationships.md           # Table schema & relationship docs
```

## Configuration

The pipeline reads its settings from `config/resources.json`:

```json
{
  "base_url": "https://hapi.fhir.org/baseR4",
  "resources": ["Patient", "Encounter", "Observation", "Condition"],
  "page_count": 20,
  "lookback_days": 3
}
```

| Parameter | Description | Default |
|---|---|---|
| `base_url` | FHIR server endpoint | `https://hapi.fhir.org/baseR4` |
| `resources` | FHIR resource types to fetch | Patient, Encounter, Observation, Condition |
| `page_count` | Records per API page | 20 |
| `lookback_days` | How many days back to fetch updated records | 3 |
