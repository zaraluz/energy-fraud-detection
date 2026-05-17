# 🥈 Silver Layer — Deep Dive

This document provides a comprehensive technical breakdown of the Silver layer implementation, including engineering decisions, transformations applied, and lessons learned.

---

## 🎯 Purpose

The Silver layer is the **cleansing and standardization stage** of the medallion architecture. It reads from Bronze Delta tables and produces clean, typed, enriched, and deduplicated tables ready for Gold feature engineering and ML.

### Silver Philosophy

> **"Silver is where raw data becomes trustworthy data."**

The Silver layer must:
- ✅ **Cast** all string columns to proper types (Date, Double, Integer)
- ✅ **Rename** columns from PT-BR to English (following the data dictionary)
- ✅ **Clean** whitespace, normalize nulls, drop exact duplicates
- ✅ **Validate** rows against business rules — quarantine failures
- ✅ **Enrich** by joining fact tables with dimension tables
- ❌ NOT apply business aggregations (that is Gold's responsibility)
- ❌ NOT create ML features (that is Gold's responsibility)

---

## 📊 Tables Produced

| Delta Table | Source (Bronze) | Rows | Columns | Partitioned |
|---|---|---|---|---|
| `energy_project.silver.samp` | `bronze.samp` | 6,319,615 | 23 | `reference_year` |
| `energy_project.silver.inadimplencia` | `bronze.inadimplencia` + `bronze.dominio_indicadores` | 1,104,153 | 12 | `reference_year` |
| `energy_project.silver.samp_quarantine` | rows failing quality rules | 0 | 23 | No |

**Total Silver rows:** 7,423,768

---

## ⚠️ Spark Best Practice Applied: Lazy vs Action Operations

A core principle followed throughout this notebook, based on Prof. Helder's lecture:

> *"Action operations are expensive and take a long time — I have to avoid doing action operations multiple times. Everything I bring back to the driver is an expensive operation. A row count is also an extremely expensive operation, because it has to traverse the entire dataset."*

**Implementation:**
- All transformations (`withColumn`, `filter`, `dropDuplicates`, `join`) are **lazy** — they build a DAG but do not execute
- The DAG executes **once**, at the `.saveAsTable()` call
- `.count()` is called **once per table**, only in the final validation cell — never mid-pipeline

---

## 🔄 Transformation Pipeline

### SAMP

```
bronze.samp (22 cols, all strings)
    │
    ├── to_date()           DatGeracaoConjuntoDados → dataset_generation_date
    ├── to_date()           DatCompetencia          → reference_date
    ├── regexp_replace +    VlrMercado              → market_value (Double)
    │   cast(Double)        (comma → period fix)
    ├── trim()              18 string columns       → renamed EN columns
    ├── year()              reference_date          → reference_year (derived)
    ├── month()             reference_date          → reference_month (derived)
    ├── filter()            quality rules           → split valid / quarantine
    ├── dropDuplicates()    business key            → deduplicated
    │
    ▼
silver.samp (23 cols, typed)          silver.samp_quarantine
```

### Inadimplência

```
bronze.inadimplencia (10 cols)    bronze.dominio_indicadores (6 cols)
    │                                       │
    ├── to_date()  DatGeracaoConjuntoDados   ├── trim() SigIndicador → indicator_code
    ├── concat_ws  AnoIndice +               └── trim() DscIndicador → indicator_description
    │   + to_date  NumPeriodoIndice
    │              → reference_date (constructed)
    ├── cast(int)  AnoIndice        → reference_year
    ├── cast(int)  NumPeriodoIndice → reference_month
    ├── regexp_replace + cast(Double) VlrIndiceEnviado → indicator_value
    ├── trim()     SigAgente  → distributor_code
    ├── trim()     NumCNPJ    → distributor_cnpj
    ├── trim()     SigIndicador → indicator_code
    │
    ├── broadcast join on indicator_code (left)
    │
    ├── filter()        quality rules
    ├── dropDuplicates  business key
    │
    ▼
silver.inadimplencia (12 cols, typed + enriched)
```

---

## 🔑 Key Engineering Decisions

### 1. Decimal Separator Fix

ANEEL CSVs use **comma as decimal separator** (Brazilian convention):
- `8,000000` → should be `8.0`
- `2884,00` → should be `2884.0`

Casting directly to `DoubleType` without fixing this produces `null` for every row. The fix:

```python
F.regexp_replace(F.col("VlrMercado"), ",", ".").cast(DoubleType())
```

This was confirmed from the Bronze sample row: `VlrMercado = 8,000000`.

---

### 2. reference_date Constructed for Inadimplência

The inadimplência source has no single date column — the reference period is encoded as separate year and month integers (`AnoIndice`, `NumPeriodoIndice`). We construct `reference_date` as the first day of that month:

```python
.withColumn("reference_date",
    F.to_date(
        F.concat_ws("-",
            F.col("AnoIndice"),
            F.lpad(F.col("NumPeriodoIndice"), 2, "0"),
            F.lit("01")
        ),
        "yyyy-MM-dd"
    )
)
# Example: AnoIndice=2012, NumPeriodoIndice=1 → reference_date=2012-01-01
```

Result: `reference_date: date (nullable = false)` — Spark guarantees non-null because the input columns are controlled.

---

### 3. Broadcast Join for Dimension Enrichment

`dominio_indicadores` is ~40 KB — a classic small dimension table. We use `F.broadcast()` to send it to all executors, avoiding a costly shuffle join:

```python
df_inadimp_enriched = (
    df_inadimp_typed
    .join(F.broadcast(df_dominio), on="indicator_code", how="left")
)
```

Result: every row in `silver.inadimplencia` has `indicator_description` populated — **0 nulls** on this column after join.

---

### 4. Quarantine Table Instead of Silent Drops

Rows failing quality rules are written to `silver.samp_quarantine` rather than dropped silently. This makes data quality issues visible and auditable:

```python
df_samp_valid = df_samp_typed.filter(
    F.col("reference_date").isNotNull() &
    F.col("market_value").isNotNull()
)

df_samp_quarantine = df_samp_typed.filter(
    F.col("reference_date").isNull() |
    F.col("market_value").isNull()
)
```

In this project's run: **0 rows quarantined** — all SAMP data passed quality rules.

---

### 5. Column Count: Bronze 22 → Silver 23

The SAMP Silver table has one more column than Bronze:

| Bronze (22) | Silver (23) |
|---|---|
| 18 PT-BR business columns | 18 EN business columns (renamed) |
| 4 audit columns (`_ingestion_timestamp`, `_source_file`, `_ingestion_date`, `_partition_year`) | 3 audit columns (dropped `_partition_year`) + 2 derived (`reference_year`, `reference_month`) |

`_partition_year` from Bronze is dropped because Silver uses `reference_year` (same value, new name) as its partition column — no duplication needed.

---

## 📋 Final Schemas (Silver)

### silver.samp (23 columns)

```
root
 |-- dataset_generation_date: date
 |-- distributor_cnpj: string
 |-- distributor_code: string
 |-- distributor_name: string
 |-- market_type: string
 |-- tariff_modality: string
 |-- tariff_subgroup: string
 |-- consumption_class: string
 |-- consumer_subclass: string
 |-- consumer_detail: string
 |-- accessing_agent_id: string
 |-- accessing_agent_cnpj: string
 |-- accessing_agent_name: string
 |-- tariff_period: string
 |-- energy_option: string
 |-- market_detail: string
 |-- reference_date: date
 |-- market_value: double
 |-- reference_year: integer
 |-- reference_month: integer
 |-- _ingestion_timestamp: timestamp
 |-- _source_file: string
 |-- _ingestion_date: date
```

### silver.inadimplencia (12 columns)

```
root
 |-- dataset_generation_date: date
 |-- distributor_cnpj: string
 |-- distributor_code: string
 |-- indicator_code: string
 |-- indicator_description: string
 |-- reference_date: date (nullable = false)
 |-- reference_year: integer
 |-- reference_month: integer
 |-- indicator_value: double
 |-- _ingestion_timestamp: timestamp
 |-- _source_file: string
 |-- _ingestion_date: date
```

---

## ✅ Validation Results

### silver.samp

| Check | Result |
|---|---|
| Row count | 6,319,615 |
| Quarantine rows | 0 |
| null reference_date | 0 |
| null market_value | 0 |
| null distributor_cnpj | 0 |
| Partitions | 2020–2026 (7 years) |
| reference_date range | 2020-01-01 → 2026-03-01 |

### silver.inadimplencia

| Check | Result |
|---|---|
| Row count | 1,104,153 |
| null reference_date | 0 |
| null indicator_value | 0 |
| null indicator_description | 0 |
| Partitions | 2012–2026 (15 years) |
| reference_date range | 2012-01-01 → 2026-03-01 |

---

## 🎓 Lessons Learned

### 1. Always inspect a sample row before writing any cast
The decimal separator issue (`8,000000`) would have produced an entirely null `market_value` column if we hadn't inspected the Bronze sample first. `printSchema()` alone doesn't reveal this — only actual data does.

### 2. Source schemas don't always match documentation
The inadimplência columns (`SigAgente`, `NumCNPJ`, `AnoIndice`) were different from what the ANEEL documentation suggested. Always confirm from the actual Bronze data.

### 3. Date formats vary within the same data provider
ANEEL uses `yyyy-MM-dd` in SAMP but `dd-MM-yyyy` in Inadimplência. Never assume consistent formatting across datasets from the same source.

### 4. Building dates from parts is a valid Silver pattern
When a source encodes dates as separate year/month integers, constructing a proper `DateType` in Silver is the right approach — it enables time-series operations in Gold without workarounds.

### 5. Quarantine tables cost almost nothing and save a lot
Writing a quarantine table adds seconds to the pipeline but makes data quality visible. In this run it was empty — which is itself a valuable piece of information.

---

## 🔗 Related Documentation

- [Architecture Overview](architecture.md)
- [Bronze Layer Deep Dive](bronze_layer.md)
- [Data Dictionary (PT-BR → EN)](data_dictionary.md)
- [Silver Notebook Source](../notebooks/02_silver_transformation.py)
