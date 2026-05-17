# 🥉 Bronze Layer — Deep Dive

This document provides a comprehensive technical breakdown of the Bronze layer implementation, including the engineering decisions, challenges encountered, and lessons learned.

---

## 🎯 Purpose

The Bronze layer is the **first persisted stage** of the medallion architecture. It receives raw CSV files from ANEEL and stores them as Delta tables with **minimal transformation** — only what's needed to make the data queryable while preserving its source-faithful nature.

### Bronze Philosophy

> **"Bronze is a mirror of the source."**

The Bronze layer must:
- ✅ Preserve **100% of source data** — no row should be lost during ingestion
- ✅ Keep **column names in their original form** (Portuguese, from ANEEL's dictionary)
- ✅ Treat **all values as strings** — type casting is the Silver layer's responsibility
- ✅ Add **audit metadata** for traceability
- ❌ NOT apply business logic, validations, or transformations

---

## 📊 Datasets

### Tables produced in Bronze

| Delta Table | Source | Rows | Columns |
|---|---|---|---|
| `energy_project.bronze.samp` | `samp/samp-*.csv` (2020–2026) | 6,808,591 | 22 |
| `energy_project.bronze.inadimplencia` | `inadimplencia/inadimplencia.csv` | TBD | TBD |
| `energy_project.bronze.dominio_indicadores` | `inadimplencia/dominio-indicadores.csv` | TBD | TBD |

### SAMP Breakdown by Source File

| File | Rows | Coverage |
|---|---|---|
| samp-2020.csv | 845,362 | Full year |
| samp-2021.csv | 950,318 | Full year |
| samp-2022.csv | 973,685 | Full year |
| samp-2023.csv | 1,107,340 | Full year |
| samp-2024.csv | 1,303,566 | Full year |
| samp-2025.csv | 1,371,594 | Full year |
| samp-2026.csv | 256,726 | Partial (year in progress) |
| **Total** | **6,808,591** | 2020-01 to 2026-03 |

**Observation:** Row volume grows year-over-year — likely due to ANEEL expanding coverage of reporting distributors. This trend is useful context for time-series analyses in downstream layers.

---

## 🏗️ Implementation Highlights

### Reading CSV with Brazilian Conventions

Brazilian CSV files follow conventions that differ from the US/UK standard:

| Setting | Brazilian | US/UK | Our Config |
|---|---|---|---|
| Separator | `;` | `,` | `sep=";"` |
| Encoding | Latin-1 / ISO-8859-1 | UTF-8 | `encoding="ISO-8859-1"` |
| Decimal separator | `,` (comma) | `.` (period) | Handled in Silver |

```python
df_samp_raw = (
    spark.read
    .option("header", "true")
    .option("sep", ";")
    .option("encoding", "ISO-8859-1")
    .schema(SAMP_SCHEMA)
    .csv(f"{PATH_SAMP}/*.csv")
)
```

The wildcard `*.csv` enables **parallel reading** of all 7 yearly files — leveraging Spark's distributed compute.

---

## 🔑 Explicit Schema {#explicit-schema}

### The Problem We Discovered

During initial implementation using `inferSchema=true`, we observed **non-deterministic behavior**:

| Execution | Row count returned |
|---|---|
| First run | 14,070,353 |
| Second run | 6,808,591 |
| Triangulation (file-by-file) | 6,808,591 ✅ |

The true row count is **6,808,591** — but Spark's automatic schema inference was producing inconsistent results between executions.

### Root Cause

`inferSchema=true` works by:
1. Spark samples a portion of the data
2. It infers each column's most likely type
3. It applies that type during the actual read
4. Rows that don't match the inferred type may be silently dropped or coerced to NULL

When sampling produces different results across cluster restarts (due to lazy evaluation and caching), **inferred types change**, and the resulting row count changes too.

### Affected Columns (Examples)

| Column | With inferSchema | Real type | Issue |
|---|---|---|---|
| `NumCNPJAgenteDistribuidora` | `long` | string | Loses leading zeros |
| `NumCNPJAgenteAcessante` | `double` | string | Becomes `3.71216699E10` (scientific notation) |
| `DatGeracaoConjuntoDados` | `date` | string | Drops rows with non-standard dates |
| `VlrMercado` | `double` or `string` (varies!) | string | Comma decimal separator confuses parser |

### Solution: Explicit StringType Schema

```python
SAMP_SCHEMA = StructType([
    StructField("DatGeracaoConjuntoDados",    StringType(), True),
    StructField("NumCNPJAgenteDistribuidora", StringType(), True),
    StructField("SigAgenteDistribuidora",     StringType(), True),
    # ... all 18 columns as StringType
])

df_samp_raw = (
    spark.read
    .option("header", "true")
    .option("sep", ";")
    .option("encoding", "ISO-8859-1")
    .schema(SAMP_SCHEMA)              # ← Explicit, deterministic
    .csv(f"{PATH_SAMP}/*.csv")
)
```

### Benefits

- ✅ **Deterministic** — same data always produces same row count
- ✅ **Lossless** — no rows dropped due to type inference failures
- ✅ **Source-faithful** — CSVs ARE strings; this preserves that
- ✅ **Identifiers preserved** — CNPJs keep their format intact
- ✅ **Reproducible** — pipeline can be re-run with identical results

---

## 🗂️ Partitioning {#partitioning}

### Why Partition?

Following Prof. Helder's lecture guidance:

> *"Operations of partitioning and compression are critically important. Normally I won't want to look at the entire database; if it's partitioned by year, each folder has that year's data and I can access it directly."*

### Implementation

```python
df_samp_bronze_partitioned = df_samp_bronze.withColumn(
    "_partition_year",
    F.year(F.to_date(F.col("DatCompetencia"), "yyyy-MM-dd"))
)

(
    df_samp_bronze_partitioned.write
    .format("delta")
    .mode("overwrite")
    .partitionBy("_partition_year")
    .saveAsTable(TABLE_SAMP)
)
```

### Physical Layout

```
energy_project.bronze.samp/
├── _partition_year=2020/
│   └── part-001.parquet
├── _partition_year=2021/
│   └── part-001.parquet
├── _partition_year=2022/
│   └── part-001.parquet
├── _partition_year=2023/
│   └── part-001.parquet
├── _partition_year=2024/
│   └── part-001.parquet
├── _partition_year=2025/
│   └── part-001.parquet
└── _partition_year=2026/
    └── part-001.parquet
```

### Query Performance Impact

Queries that filter by year benefit from **partition pruning** — Spark reads only the relevant folder(s):

```sql
-- This query reads ONLY the 2024 partition (1 file)
SELECT *
FROM energy_project.bronze.samp
WHERE _partition_year = 2024;

-- Without partitioning, this would scan ALL 6.8M rows
```

Expected speedup for year-filtered queries: **~7x** (with 7 partitions).

### Use of `F.to_date()` over `.cast("date")`

We use `F.to_date(col, "yyyy-MM-dd")` rather than `.cast("date")` because:
- ✅ **Format-aware** — explicit about the expected pattern
- ✅ **Robust** — invalid dates become NULL instead of failing
- ✅ **Better error handling** — easier to debug

---

## 🔍 Audit Metadata {#metadata}

Every Bronze table includes three audit columns prefixed with `_`:

| Column | Type | Source | Purpose |
|---|---|---|---|
| `_ingestion_timestamp` | timestamp | `F.current_timestamp()` | Exact moment of load |
| `_source_file` | string | `F.col("_metadata.file_path")` | Original file path |
| `_ingestion_date` | date | `F.current_date()` | Date of load |

### Why the `_` Prefix?

Industry convention: columns prefixed with `_` are **technical metadata** (not business data). This visual cue helps consumers:
- Ignore metadata in BI reports
- Identify pipeline-injected columns instantly
- Maintain clear separation between source and engineering data

### Modern Unity Catalog Syntax

We use `F.col("_metadata.file_path")` rather than the deprecated `F.input_file_name()`:

```python
# ❌ DEPRECATED (raises error in Unity Catalog)
.withColumn("_source_file", F.input_file_name())

# ✅ MODERN
.withColumn("_source_file", F.col("_metadata.file_path"))
```

The `_metadata` struct also provides additional useful properties:
- `_metadata.file_name`
- `_metadata.file_size`
- `_metadata.file_modification_time`

We currently use only `file_path`, but these are available for future enrichment if needed.

### Real-World Use Cases Enabled

#### 1. Debugging
> *"Values for ELETROPAULO in October/2020 look wrong — where did they come from?"*

```sql
SELECT DISTINCT _source_file
FROM energy_project.bronze.samp
WHERE SigAgenteDistribuidora = 'ELETROPAULO'
  AND DatCompetencia LIKE '2020-10%';
-- Returns: dbfs:/Volumes/.../samp-2020.csv
```

#### 2. Incremental Processing
```sql
-- Process only data loaded today
SELECT *
FROM energy_project.bronze.samp
WHERE _ingestion_date = CURRENT_DATE();
```

#### 3. Drift Detection
```sql
-- Compare value distributions between ingestion batches
SELECT _ingestion_date, AVG(CAST(VlrMercado AS DOUBLE)) AS avg_value
FROM energy_project.bronze.samp
GROUP BY _ingestion_date
ORDER BY _ingestion_date;
```

#### 4. Selective Reprocessing
```sql
-- Reprocess only files from a specific year
DELETE FROM energy_project.bronze.samp
WHERE _source_file LIKE '%samp-2023%';
-- Then re-run ingestion for 2023 only
```

### Connection to MLOps

This metadata pattern is foundational for **MLOps best practices**:

| MLOps Practice | Enabled By |
|---|---|
| Model Reproducibility | Filter training data by `_ingestion_timestamp <= model_creation_date` |
| Distribution Drift Monitoring | Compare value stats across `_ingestion_date` buckets |
| Continuous Training | Trigger retraining based on new `_ingestion_date` values |
| Audit Trail | Trace any prediction back to its source data |

---

## 📋 Final Schema (Bronze)

After all transformations, the `energy_project.bronze.samp` table has the following structure:

```
root
 |-- DatGeracaoConjuntoDados: string (nullable = true)
 |-- NumCNPJAgenteDistribuidora: string (nullable = true)
 |-- SigAgenteDistribuidora: string (nullable = true)
 |-- NomAgenteDistribuidora: string (nullable = true)
 |-- NomTipoMercado: string (nullable = true)
 |-- DscModalidadeTarifaria: string (nullable = true)
 |-- DscSubGrupoTarifario: string (nullable = true)
 |-- DscClasseConsumoMercado: string (nullable = true)
 |-- DscSubClasseConsumidor: string (nullable = true)
 |-- DscDetalheConsumidor: string (nullable = true)
 |-- IdeAgenteAcessante: string (nullable = true)
 |-- NumCNPJAgenteAcessante: string (nullable = true)
 |-- NomAgenteAcessante: string (nullable = true)
 |-- DscPostoTarifario: string (nullable = true)
 |-- DscOpcaoEnergia: string (nullable = true)
 |-- DscDetalheMercado: string (nullable = true)
 |-- DatCompetencia: string (nullable = true)
 |-- VlrMercado: string (nullable = true)
 |-- _ingestion_timestamp: timestamp (nullable = true)
 |-- _source_file: string (nullable = true)
 |-- _ingestion_date: date (nullable = true)
 |-- _partition_year: integer (nullable = true)
```

**Total: 22 columns** (18 source + 3 audit metadata + 1 partition).

---

## 🎓 Lessons Learned

### 1. Trust but verify Spark's automatic features

`inferSchema=true` is convenient but **not deterministic**. For production pipelines, always use explicit schemas.

### 2. Brazilian data has unique conventions

Always check separator, encoding, and decimal format when working with Brazilian government data. Don't assume defaults.

### 3. Investigate anomalies — don't hide them

When the row count changed between executions (14M → 6.8M), we didn't ignore it or apply a workaround. We investigated, found the root cause (non-deterministic inferSchema), and documented the fix. This investigation became one of the most valuable parts of the project.

### 4. Audit metadata is cheap and invaluable

Three additional columns added minimal storage overhead but unlock critical operational capabilities. Always include audit metadata in production data lakes.

### 5. The Bronze layer should be boring

The most "interesting" Bronze layer transformations are usually mistakes. Bronze should preserve, not transform. Save the interesting work for Silver and Gold.

---

## 🔗 Related Documentation

- [Architecture Overview](architecture.md)
- [Data Dictionary (PT-BR → EN)](data_dictionary.md)
- [Bronze Notebook Source](../notebooks/01_bronze_ingestion.py)
