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

| Delta Table | Source | Rows | Columns | Partitioned |
|---|---|---|---|---|
| `energy_project.bronze.samp` | `samp/samp-*.csv` (2020–2026) | 6,808,591 | 22 | `_partition_year` |
| `energy_project.bronze.inadimplencia` | `inadimplencia/inadimplencia.csv` | ~1.1M | 10 | No |
| `energy_project.bronze.dominio_indicadores` | `inadimplencia/dominio-indicadores.csv` | ~100 | 6 | No |

> `inadimplencia` and `dominio_indicadores` are not partitioned — their sizes (~58 MB and ~40 KB respectively) don't justify the overhead.

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

### Date Format Inconsistency Between Datasets

ANEEL uses different date formats across datasets — a real-world data quality issue:

| Dataset | Column | Format | Example |
|---|---|---|---|
| SAMP | `DatGeracaoConjuntoDados` | `yyyy-MM-dd` | `2026-04-15` |
| SAMP | `DatCompetencia` | `yyyy-MM-dd` | `2024-01-01` |
| Inadimplência | `DatGeracaoConjuntoDados` | `dd-MM-yyyy` | `05-05-2026` |

This inconsistency is handled in the Silver layer with format-aware `to_date()` calls.

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
├── _partition_year=2021/
├── _partition_year=2022/
├── _partition_year=2023/
├── _partition_year=2024/
├── _partition_year=2025/
└── _partition_year=2026/
```

### Why `inadimplencia` and `dominio_indicadores` Are Not Partitioned

Partitioning adds overhead (more files, more metadata). For small tables, it hurts more than it helps:

| Table | Size | Decision |
|---|---|---|
| `samp` | ~1.5 GB, 6.8M rows | Partitioned by year ✅ |
| `inadimplencia` | ~58 MB | Not partitioned |
| `dominio_indicadores` | ~40 KB | Not partitioned |

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

```python
# ❌ DEPRECATED (raises error in Unity Catalog)
.withColumn("_source_file", F.input_file_name())

# ✅ MODERN
.withColumn("_source_file", F.col("_metadata.file_path"))
```

---

## 📋 Final Schemas (Bronze)

### bronze.samp (22 columns)

```
root
 |-- DatGeracaoConjuntoDados: string
 |-- NumCNPJAgenteDistribuidora: string
 |-- SigAgenteDistribuidora: string
 |-- NomAgenteDistribuidora: string
 |-- NomTipoMercado: string
 |-- DscModalidadeTarifaria: string
 |-- DscSubGrupoTarifario: string
 |-- DscClasseConsumoMercado: string
 |-- DscSubClasseConsumidor: string
 |-- DscDetalheConsumidor: string
 |-- IdeAgenteAcessante: string
 |-- NumCNPJAgenteAcessante: string
 |-- NomAgenteAcessante: string
 |-- DscPostoTarifario: string
 |-- DscOpcaoEnergia: string
 |-- DscDetalheMercado: string
 |-- DatCompetencia: string
 |-- VlrMercado: string
 |-- _ingestion_timestamp: timestamp
 |-- _source_file: string
 |-- _ingestion_date: date
 |-- _partition_year: integer
```

### bronze.inadimplencia (10 columns)

```
root
 |-- DatGeracaoConjuntoDados: string
 |-- SigAgente: string
 |-- NumCNPJ: string
 |-- SigIndicador: string
 |-- AnoIndice: string
 |-- NumPeriodoIndice: string
 |-- VlrIndiceEnviado: string
 |-- _ingestion_timestamp: timestamp
 |-- _source_file: string
 |-- _ingestion_date: date
```

### bronze.dominio_indicadores (6 columns)

```
root
 |-- DatGeracaoConjuntoDados: string
 |-- SigIndicador: string
 |-- DscIndicador: string
 |-- _ingestion_timestamp: timestamp
 |-- _source_file: string
 |-- _ingestion_date: date
```

---

## 🎓 Lessons Learned

### 1. Trust but verify Spark's automatic features
`inferSchema=true` is convenient but **not deterministic**. For production pipelines, always use explicit schemas.

### 2. Brazilian data has unique conventions
Always check separator, encoding, and decimal format when working with Brazilian government data. Don't assume defaults.

### 3. Date formats are not consistent even within the same source
ANEEL uses `yyyy-MM-dd` in SAMP but `dd-MM-yyyy` in Inadimplência. Always inspect a sample row before assuming a format.

### 4. Investigate anomalies — don't hide them
When the row count changed between executions (14M → 6.8M), we investigated, found the root cause, and documented the fix.

### 5. Audit metadata is cheap and invaluable
Three additional columns added minimal storage overhead but unlock critical operational capabilities.

### 6. The Bronze layer should be boring
The most "interesting" Bronze layer transformations are usually mistakes. Bronze should preserve, not transform.

---

## 🔗 Related Documentation

- [Architecture Overview](architecture.md)
- [Silver Layer Deep Dive](silver_layer.md)
- [Data Dictionary (PT-BR → EN)](data_dictionary.md)
- [Bronze Notebook Source](../notebooks/01_bronze_ingestion.py)
