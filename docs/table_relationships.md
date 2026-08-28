# FHIR Table Relationships

## Overview

This document provides explicit documentation of table relationships and join keys across the FHIR medallion lakehouse. Understanding these relationships is critical for:

- Building queries across related resources
- Maintaining referential integrity
- Constructing Gold layer aggregations
- Troubleshooting data quality issues

---

## Core Entity Relationships

### Primary Entities

1. **Patient** - Central entity representing individuals
2. **Encounter** - Healthcare visits/episodes linked to patients
3. **Observation** - Clinical measurements linked to patients and encounters
4. **Condition** - Diagnoses/problems linked to patients and encounters

---

## Explicit Join Keys

### Patient ← Encounter
**Relationship:** One Patient has many Encounters

**Join Key:**
```
Encounter.patient_id = Patient.resource_id
```

**Example Query:**
```sql
SELECT 
    p.resource_id,
    p.name_given,
    p.name_family,
    e.encounter_id,
    e.encounter_class,
    e.period_start,
    e.period_end
FROM silver_fhir.patient p
INNER JOIN silver_fhir.encounter e
    ON e.patient_id = p.resource_id
WHERE p.is_current = TRUE
  AND e.is_current = TRUE
```

---

### Patient ← Observation
**Relationship:** One Patient has many Observations

**Join Key:**
```
Observation.patient_id = Patient.resource_id
```

**Example Query:**
```sql
SELECT 
    p.resource_id,
    p.name_given,
    p.name_family,
    o.observation_id,
    o.code_coding_code,
    o.code_coding_display,
    o.value_quantity_value,
    o.value_quantity_unit,
    o.effective_datetime
FROM silver_fhir.patient p
INNER JOIN silver_fhir.observation o
    ON o.patient_id = p.resource_id
WHERE p.is_current = TRUE
  AND o.is_current = TRUE
```

---

### Encounter ← Observation
**Relationship:** One Encounter has many Observations

**Join Key:**
```
Observation.encounter_id = Encounter.resource_id
```

**Example Query:**
```sql
SELECT 
    e.encounter_id,
    e.encounter_class,
    e.period_start,
    o.observation_id,
    o.code_coding_display,
    o.value_quantity_value,
    o.value_quantity_unit
FROM silver_fhir.encounter e
INNER JOIN silver_fhir.observation o
    ON o.encounter_id = e.resource_id
WHERE e.is_current = TRUE
  AND o.is_current = TRUE
```

---

### Patient ← Observation → Encounter (Three-way Join)
**Relationship:** Observations are linked to both Patient and Encounter

**Join Keys:**
```
Observation.patient_id = Patient.resource_id
Observation.encounter_id = Encounter.resource_id
Encounter.patient_id = Patient.resource_id  -- (redundant but validates integrity)
```

**Example Query:**
```sql
SELECT 
    p.resource_id AS patient_id,
    p.name_given,
    p.name_family,
    e.encounter_id,
    e.encounter_class,
    e.period_start,
    o.observation_id,
    o.code_coding_display AS observation_name,
    o.value_quantity_value AS value,
    o.value_quantity_unit AS unit,
    o.effective_datetime AS measurement_time
FROM silver_fhir.patient p
INNER JOIN silver_fhir.encounter e
    ON e.patient_id = p.resource_id
INNER JOIN silver_fhir.observation o
    ON o.patient_id = p.resource_id
    AND o.encounter_id = e.resource_id
WHERE p.is_current = TRUE
  AND e.is_current = TRUE
  AND o.is_current = TRUE
```

---

### Patient ← Condition
**Relationship:** One Patient has many Conditions

**Join Key:**
```
Condition.patient_id = Patient.resource_id
```

**Example Query:**
```sql
SELECT 
    p.resource_id,
    p.name_given,
    p.name_family,
    c.condition_id,
    c.code_coding_code,
    c.code_coding_display,
    c.clinical_status,
    c.onset_datetime,
    c.recorded_date
FROM silver_fhir.patient p
INNER JOIN silver_fhir.condition c
    ON c.patient_id = p.resource_id
WHERE p.is_current = TRUE
  AND c.is_current = TRUE
```

---

### Encounter ← Condition
**Relationship:** One Encounter has many Conditions

**Join Key:**
```
Condition.encounter_id = Encounter.resource_id
```

**Example Query:**
```sql
SELECT 
    e.encounter_id,
    e.encounter_class,
    e.period_start,
    c.condition_id,
    c.code_coding_display,
    c.clinical_status,
    c.onset_datetime
FROM silver_fhir.encounter e
INNER JOIN silver_fhir.condition c
    ON c.encounter_id = e.resource_id
WHERE e.is_current = TRUE
  AND c.is_current = TRUE
```

---

### Patient ← Condition → Encounter (Three-way Join)
**Relationship:** Conditions are linked to both Patient and Encounter

**Join Keys:**
```
Condition.patient_id = Patient.resource_id
Condition.encounter_id = Encounter.resource_id
Encounter.patient_id = Patient.resource_id  -- (validates integrity)
```

**Example Query:**
```sql
SELECT 
    p.resource_id AS patient_id,
    p.name_given,
    p.name_family,
    e.encounter_id,
    e.encounter_class,
    e.period_start,
    c.condition_id,
    c.code_coding_display AS diagnosis,
    c.clinical_status,
    c.onset_datetime
FROM silver_fhir.patient p
INNER JOIN silver_fhir.encounter e
    ON e.patient_id = p.resource_id
INNER JOIN silver_fhir.condition c
    ON c.patient_id = p.resource_id
    AND c.encounter_id = e.resource_id
WHERE p.is_current = TRUE
  AND e.is_current = TRUE
  AND c.is_current = TRUE
```

---

## Entity Relationship Diagram

```
                    ┌────────────────────────────────┐
                    │          PATIENT              │
                    │────────────────────────────────│
                    │ PK: resource_id              │
                    │     name_given               │
                    │     name_family              │
                    │     birthdate                │
                    │     gender                   │
                    └────────────────────────────────┘
                             │
                             │ 1
                             │
                 ┌───────────┼───────────┐
                 │           │            │
                 │ *         │ *          │ *
                 ▼           ▼            ▼
  ┌────────────────────────────────┐  ┌────────────────────────────────┐  ┌────────────────────────────────┐
  │         ENCOUNTER           │  │        OBSERVATION         │  │         CONDITION          │
  │────────────────────────────────│  │────────────────────────────────│  │────────────────────────────────│
  │ PK: resource_id             │  │ PK: observation_id          │  │ PK: condition_id            │
  │ FK: patient_id ────────────┴──► FK: patient_id ───────────┴──► FK: patient_id             │
  │     encounter_class         │  │ FK: encounter_id            │  │ FK: encounter_id            │
  │     period_start            │  │     code_coding_code        │  │     code_coding_code        │
  │     period_end              │  │     code_coding_display     │  │     code_coding_display     │
  └────────────────────────────────┘  │     value_quantity_value    │  │     clinical_status         │
                 │           │     value_quantity_unit     │  │     onset_datetime          │
                 │           │     effective_datetime      │  └────────────────────────────────┘
                 │           └────────────────────────────────┘
                 │                       │
                 │ 1                     │ *
                 └─────────────────────────┘


Legend:
  PK = Primary Key
  FK = Foreign Key
  1  = One (parent)
  *  = Many (children)
  ──► = Foreign key relationship
```

---

## Join Key Summary Table

| Child Table   | Child Column    | Parent Table | Parent Column | Cardinality |
|---------------|-----------------|--------------|---------------|-------------|
| Encounter     | patient_id      | Patient      | resource_id   | Many-to-One |
| Observation   | patient_id      | Patient      | resource_id   | Many-to-One |
| Observation   | encounter_id    | Encounter    | resource_id   | Many-to-One |
| Condition     | patient_id      | Patient      | resource_id   | Many-to-One |
| Condition     | encounter_id    | Encounter    | resource_id   | Many-to-One |

---

## Best Practices

### 1. Always Filter by `is_current = TRUE`
When joining Silver layer tables with SCD Type 2, always filter for current records:

```sql
WHERE p.is_current = TRUE
  AND e.is_current = TRUE
  AND o.is_current = TRUE
```

### 2. Use LEFT JOIN for Optional Relationships
Some observations may not have an `encounter_id` (e.g., patient-reported data):

```sql
SELECT 
    p.resource_id,
    o.observation_id,
    e.encounter_id  -- May be NULL
FROM silver_fhir.patient p
INNER JOIN silver_fhir.observation o
    ON o.patient_id = p.resource_id
LEFT JOIN silver_fhir.encounter e
    ON o.encounter_id = e.resource_id
WHERE p.is_current = TRUE
  AND o.is_current = TRUE
  AND (e.is_current = TRUE OR e.resource_id IS NULL)
```

### 3. Validate Referential Integrity
Periodically check for orphaned records:

```sql
-- Find observations without valid patient references
SELECT COUNT(*)
FROM silver_fhir.observation o
LEFT JOIN silver_fhir.patient p
    ON o.patient_id = p.resource_id
    AND p.is_current = TRUE
WHERE o.is_current = TRUE
  AND p.resource_id IS NULL;

-- Find observations with invalid encounter references
SELECT COUNT(*)
FROM silver_fhir.observation o
LEFT JOIN silver_fhir.encounter e
    ON o.encounter_id = e.resource_id
    AND e.is_current = TRUE
WHERE o.is_current = TRUE
  AND o.encounter_id IS NOT NULL
  AND e.resource_id IS NULL;
```

### 4. Use Indexes/Partitioning for Performance
Partition large tables by patient_id or date ranges:

```sql
CREATE TABLE silver_fhir.observation (
    observation_id STRING,
    patient_id STRING,
    encounter_id STRING,
    effective_datetime TIMESTAMP,
    ...
)
PARTITIONED BY (DATE(effective_datetime))
CLUSTER BY (patient_id);
```

### 5. Handle Multiple Relationships Carefully
Some FHIR resources support multiple references (e.g., Observation.subject can reference Patient or Group). Document these edge cases:

```sql
-- Ensure you're only joining to Patient references
SELECT *
FROM silver_fhir.observation o
INNER JOIN silver_fhir.patient p
    ON o.patient_id = p.resource_id
WHERE o.subject_reference_type = 'Patient'  -- Explicit filter
  AND o.is_current = TRUE
  AND p.is_current = TRUE;
```

---

## Common Query Patterns

### Patient Timeline (All Events for a Patient)
```sql
SELECT 
    'Encounter' AS event_type,
    e.encounter_id AS event_id,
    e.period_start AS event_datetime,
    e.encounter_class AS detail
FROM silver_fhir.encounter e
WHERE e.patient_id = 'patient-123'
  AND e.is_current = TRUE

UNION ALL

SELECT 
    'Observation' AS event_type,
    o.observation_id AS event_id,
    o.effective_datetime AS event_datetime,
    o.code_coding_display AS detail
FROM silver_fhir.observation o
WHERE o.patient_id = 'patient-123'
  AND o.is_current = TRUE

UNION ALL

SELECT 
    'Condition' AS event_type,
    c.condition_id AS event_id,
    c.onset_datetime AS event_datetime,
    c.code_coding_display AS detail
FROM silver_fhir.condition c
WHERE c.patient_id = 'patient-123'
  AND c.is_current = TRUE

ORDER BY event_datetime DESC;
```

### Encounter Summary (Everything That Happened in a Visit)
```sql
SELECT 
    e.encounter_id,
    e.encounter_class,
    e.period_start,
    e.period_end,
    p.name_given || ' ' || p.name_family AS patient_name,
    COUNT(DISTINCT o.observation_id) AS observation_count,
    COUNT(DISTINCT c.condition_id) AS condition_count
FROM silver_fhir.encounter e
INNER JOIN silver_fhir.patient p
    ON e.patient_id = p.resource_id
LEFT JOIN silver_fhir.observation o
    ON o.encounter_id = e.resource_id
    AND o.is_current = TRUE
LEFT JOIN silver_fhir.condition c
    ON c.encounter_id = e.resource_id
    AND c.is_current = TRUE
WHERE e.encounter_id = 'encounter-456'
  AND e.is_current = TRUE
  AND p.is_current = TRUE
GROUP BY 1, 2, 3, 4, 5;
```

### Patient Cohort Analysis
```sql
SELECT 
    p.resource_id AS patient_id,
    p.name_given,
    p.name_family,
    p.birthdate,
    COUNT(DISTINCT e.encounter_id) AS encounter_count,
    COUNT(DISTINCT o.observation_id) AS observation_count,
    COUNT(DISTINCT c.condition_id) AS condition_count,
    MAX(e.period_start) AS last_encounter_date
FROM silver_fhir.patient p
LEFT JOIN silver_fhir.encounter e
    ON e.patient_id = p.resource_id
    AND e.is_current = TRUE
LEFT JOIN silver_fhir.observation o
    ON o.patient_id = p.resource_id
    AND o.is_current = TRUE
LEFT JOIN silver_fhir.condition c
    ON c.patient_id = p.resource_id
    AND c.is_current = TRUE
WHERE p.is_current = TRUE
GROUP BY 1, 2, 3, 4
ORDER BY last_encounter_date DESC;
```

---

## Troubleshooting

### Issue: Missing Joins
**Symptom:** Queries return fewer rows than expected

**Check:**
```sql
-- Verify foreign key coverage
SELECT 
    'Observation' AS table_name,
    COUNT(*) AS total_records,
    COUNT(patient_id) AS has_patient_ref,
    COUNT(encounter_id) AS has_encounter_ref
FROM silver_fhir.observation
WHERE is_current = TRUE;
```

### Issue: Duplicate Rows
**Symptom:** Joins produce more rows than expected

**Check:** Ensure you're filtering by `is_current = TRUE` on all SCD2 tables:
```sql
-- This will create duplicates if not filtered properly
SELECT COUNT(*) FROM silver_fhir.patient;  -- All versions
SELECT COUNT(*) FROM silver_fhir.patient WHERE is_current = TRUE;  -- Current only
```

---

## References

- [FHIR R4 Specification - Resource References](https://hl7.org/fhir/R4/references.html)
- [Medallion Architecture Documentation](./architecture.md)
- [SCD Type 2 Implementation Guide](../docs/scd_type_2.md) (if available)

---

## Version History

| Date       | Version | Changes                                      |
|------------|---------|----------------------------------------------|
| 2026-08-28 | 1.0     | Initial documentation of table relationships |