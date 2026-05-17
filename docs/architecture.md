# 🏛️ Architecture & Design Decisions

This document captures the architectural decisions and rationale behind the project design.

---

## 📐 Medallion Architecture

The project follows Databricks' **medallion architecture** pattern, organizing data in progressive layers of refinement.

### Layer Responsibilities

| Layer | Purpose | Data Quality | Schema | Audience |
|---|---|---|---|---|
| **🟫 Raw** | Source files preserved as-is | Source-dependent | N/A (files) | Data Engineers |
| **🥉 Bronze** | Lossless ingestion + metadata | Type-preserved (strings) | Source-faithful (PT-BR) | Data Engineers |
| **🥈 Silver** | Cleansed, typed, validated, joined | Quality-enforced | Standardized (EN) | Data Scientists |
| **🥇 Gold** | Aggregated, ML-ready | Business-validated | Use-case oriented | Analysts, BI users |

### Catalog Structure (Unity Catalog)

```
energy_project/                      # Catalog
├── raw/                              # Schema
│   └── aneel_files/                  # Volume (CSV files)
│       ├── samp/                     # 7 yearly CSV files (2020–2026)
│       └── inadimplencia/            # inadimplencia.csv + dominio-indicadores.csv
├── bronze/                           # Schema
│   ├── samp                          # Delta table — 6,808,591 rows, partitioned by year
│   ├── inadimplencia                 # Delta table — ~1.1M rows
│   └── dominio_indicadores           # Delta table — ~100 rows (dimension)
├── silver/                           # Schema
│   ├── samp                          # Delta table — 6,319,615 rows, partitioned by year
│   ├── inadimplencia                 # Delta table — 1,104,153 rows, partitioned by year
│   └── samp_quarantine               # Delta table — rows failing quality rules
├── gold/                             # Schema (planned)
└── information_schema/               # System schema
```

---

## 🎯 Key Design Decisions

### 1. Databricks Free Edition over Community Edition

**Decision:** Use Databricks Free Edition.

**Rationale:**
- Modern Unity Catalog (3-level namespace: catalog → schema → table/volume)
- Serverless compute (no cluster management)
- MLflow integration native
- Production-grade environment with no cost

**Trade-off:** Some advanced features (e.g., custom cluster libraries) are not available, but they're not needed for this project's scope.

---

### 2. Delta Lake over Apache Parquet

**Decision:** Use Delta Lake format for all persisted tables.

**Rationale:**

| Feature | Parquet | Delta Lake |
|---|---|---|
| Columnar storage | ✅ | ✅ (uses Parquet underneath) |
| Compression | ✅ | ✅ (~80% vs raw CSV) |
| ACID transactions | ❌ | ✅ |
| Time travel | ❌ | ✅ |
| Schema evolution | ❌ | ✅ |
| MERGE/UPDATE/DELETE | ❌ | ✅ |
| Concurrent writes | ❌ | ✅ |

---

### 3. Explicit Schema over inferSchema

**Decision:** Define `StructType` schemas explicitly for all readers.

**Rationale:**
During development, `inferSchema=true` produced **non-deterministic results** — the same dataset returned different row counts (14M vs 6.8M) across executions due to inconsistent type inference across cluster restarts.

**Resolution:** Use explicit `StringType` schema in Bronze. Type casting is deferred to Silver.

See [bronze_layer.md](bronze_layer.md#explicit-schema) for the full investigation.

---

### 4. PT-BR Column Names Preserved in Bronze, Translated in Silver

**Decision:** Keep original ANEEL column names (Portuguese) in Bronze. Translate to English (snake_case) in Silver.

**Rationale:**
- **Traceability:** `VlrMercado` maps directly to ANEEL's official dictionary. `market_value` does not.
- **Separation of concerns:** Bronze preserves source; Silver standardizes for analytics.

See [data_dictionary.md](data_dictionary.md) for the full mapping.

---

### 5. Partitioning by Year

**Decision:** Partition `samp` (Bronze and Silver) by year. Do not partition `inadimplencia` or `dominio_indicadores`.

**Rationale:**
- `samp` (~1.5 GB) benefits significantly from partition pruning for year-filtered queries
- `inadimplencia` (~58 MB) and `dominio_indicadores` (~40 KB) are too small to benefit — partitioning adds overhead without gain

```
bronze.samp/             silver.samp/
├── _partition_year=2020  ├── reference_year=2020
├── _partition_year=2021  ├── reference_year=2021
├── ...                   ├── ...
└── _partition_year=2026  └── reference_year=2026
```

---

### 6. Audit Metadata Columns

**Decision:** Every Bronze table includes technical metadata columns prefixed with `_`:

| Column | Type | Purpose |
|---|---|---|
| `_ingestion_timestamp` | timestamp | When the row was loaded |
| `_source_file` | string | Origin file path |
| `_ingestion_date` | date | Load date |

These are carried through to Silver for full lineage.

See [bronze_layer.md](bronze_layer.md#metadata) for details.

---

### 7. Broadcast Join for dominio_indicadores

**Decision:** Use `F.broadcast()` when joining `inadimplencia` with `dominio_indicadores` in Silver.

**Rationale:**
`dominio_indicadores` is ~40 KB — tiny. Broadcasting it sends a full copy to every executor, avoiding a costly shuffle join. This is the recommended pattern in Spark for fact + small dimension joins.

```python
df_inadimp_enriched = (
    df_inadimp_typed
    .join(F.broadcast(df_dominio), on="indicator_code", how="left")
)
```

---

### 8. Single `.count()` Per Table

**Decision:** Avoid `.count()` mid-pipeline. Consolidate all row count checks into a single validation cell at the end of each notebook section.

**Rationale (from Prof. Helder's lecture):**
> *"Count is an extremely expensive operation because it is an action — Spark has to traverse the entire dataset. Working with big data is juggling to deal with these problems."*

All quality checks (null counts, range validation, sample rows) are batched into a single validation step that triggers the DAG once, at the end.

---

### 9. reference_date Constructed from Year + Month

**Decision:** Build `reference_date` from `AnoIndice` + `NumPeriodoIndice` in the inadimplência Silver transformation.

**Rationale:**
The inadimplência source has no single date column. Reference period is encoded as separate year and month integers. We construct the first day of each month as a proper `DateType`:

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
```

---

### 10. Quarantine Table for Bad Data

**Decision:** Rows failing Silver quality rules are written to `silver.samp_quarantine` instead of being silently dropped.

**Rationale:**
Silent data loss is dangerous in production pipelines — it creates invisible discrepancies between source and destination. Quarantine tables make data quality issues visible and auditable.

In this project's run: **0 rows quarantined** — all SAMP data passed quality rules.

---

## 🛡️ Data Governance

### Unity Catalog Tags

Tags applied at multiple levels:

**Volume (`aneel_files`):**
- `data_source: aneel`
- `data_classification: public`
- `pii: false`
- `environment: dev`
- `owner: zara_louise`

These tags enable filtering in Catalog Explorer, access policies by tag, and cost attribution by project.

---

## 🚀 Setup

### Prerequisites
- Databricks Free Edition account
- Power BI Desktop (Windows, optional)

### Initial Setup

1. **Create the catalog structure:**
   ```sql
   CREATE CATALOG IF NOT EXISTS energy_project;
   CREATE SCHEMA IF NOT EXISTS energy_project.raw;
   CREATE SCHEMA IF NOT EXISTS energy_project.bronze;
   CREATE SCHEMA IF NOT EXISTS energy_project.silver;
   CREATE SCHEMA IF NOT EXISTS energy_project.gold;
   ```

2. **Create the Volume for raw files:**
   ```sql
   CREATE VOLUME IF NOT EXISTS energy_project.raw.aneel_files;
   ```

3. **Upload ANEEL data:**
   - Catalog Explorer → `energy_project.raw.aneel_files`
   - Create subdirectories: `samp/` and `inadimplencia/`
   - Upload CSV files to respective folders

4. **Import notebooks:**
   - Workspace → Import → Select `notebooks/` folder

5. **Run the pipeline in order:**
   ```
   01_bronze_ingestion → 02_silver_transformation → 03_gold_features (planned)
   ```

---

## 📚 References

- [Databricks Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [Delta Lake Documentation](https://delta.io/learn/getting-started/)
- [Unity Catalog Best Practices](https://docs.databricks.com/data-governance/unity-catalog/best-practices.html)
- [ANEEL Resolution Nº 1003/2022](https://www2.aneel.gov.br/cedoc/ren20221003.pdf)
