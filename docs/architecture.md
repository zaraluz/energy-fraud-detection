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
| **🤖 ML** | Anomaly scoring | Model output | Score + label per distributor | Business users |

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
├── gold/                             # Schema
│   ├── features_distributor          # Delta table — 105 rows, 12 ML features
│   └── risk_scores                   # Delta table — 105 rows, anomaly scores + ranking
└── information_schema/               # System schema
```

### End-to-End Pipeline

```
Raw CSVs (Volume)
    │
    ▼
01_bronze_ingestion
    │  explicit StringType schema, audit metadata, partitioned by year
    ▼
Bronze Delta Tables
    │
    ▼
02_silver_transformation
    │  type casting, PT-BR→EN rename, dedup, quality rules, broadcast join
    ▼
Silver Delta Tables
    │
    ▼
03_gold_features
    │  aggregation per distributor, 12 features, Pandas bridge
    ▼
gold.features_distributor
    │
    ▼
04_anomaly_model
    │  Bartlett test → StandardScaler → PCA → Isolation Forest → MLflow
    ▼
gold.risk_scores + MLflow Model Registry
    │
    ▼
Power BI Dashboard (planned)
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

---

### 2. Delta Lake over Apache Parquet

**Decision:** Use Delta Lake format for all persisted tables.

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
During development, `inferSchema=true` produced **non-deterministic results** — the same dataset returned different row counts (14M vs 6.8M) across executions due to inconsistent type inference.

**Resolution:** Use explicit `StringType` schema in Bronze. Type casting is deferred to Silver.

See [bronze_layer.md](bronze_layer.md#explicit-schema) for the full investigation.

---

### 4. PT-BR Column Names Preserved in Bronze, Translated in Silver

**Decision:** Keep original ANEEL column names (Portuguese) in Bronze. Translate to English (snake_case) in Silver.

**Rationale:**
- **Traceability:** `VlrMercado` maps directly to ANEEL's official dictionary
- **Separation of concerns:** Bronze preserves source; Silver standardizes

See [data_dictionary.md](data_dictionary.md) for the full mapping.

---

### 5. Partitioning by Year (Selective)

**Decision:** Partition large tables by year. Do not partition small tables.

| Table | Size | Decision |
|---|---|---|
| `bronze.samp` / `silver.samp` | ~1.5 GB, 6.8M rows | Partitioned by year ✅ |
| `bronze.inadimplencia` | ~58 MB | Not partitioned |
| `gold.features_distributor` | 105 rows | Not partitioned |
| `gold.risk_scores` | 105 rows | Not partitioned |

---

### 6. Audit Metadata Columns

**Decision:** Every Bronze table includes technical metadata columns prefixed with `_`:

| Column | Type | Purpose |
|---|---|---|
| `_ingestion_timestamp` | timestamp | When the row was loaded |
| `_source_file` | string | Origin file path |
| `_ingestion_date` | date | Load date |

These are carried through to Silver for full lineage. See [bronze_layer.md](bronze_layer.md#metadata) for details.

---

### 7. Modern Unity Catalog Patterns

**Decision:** Use `_metadata.file_path` instead of the deprecated `input_file_name()`.

---

### 8. Broadcast Join for dominio_indicadores

**Decision:** Use `F.broadcast()` when joining `inadimplencia` with `dominio_indicadores` in Silver.

`dominio_indicadores` is ~40 KB. Broadcasting avoids a costly shuffle join.

---

### 9. Single `.count()` Per Table

**Decision:** Avoid `.count()` mid-pipeline. Consolidate all checks into a single validation cell.

> *"Count is an extremely expensive operation — Spark has to traverse the entire dataset."*

---

### 10. Quarantine Table for Bad Data

**Decision:** Rows failing Silver quality rules go to `silver.samp_quarantine` instead of being silently dropped.

In this project's run: **0 rows quarantined** — all SAMP data passed quality rules.

---

### 11. Unsupervised ML — PCA + Isolation Forest

**Decision:** Use StandardScaler → PCA → Isolation Forest for anomaly detection.

**Rationale:**
No labeled data exists. This rules out all supervised models.

| Criteria | Justification |
|---|---|
| No labels available | Rules out supervised models (Logistic Regression, etc.) |
| Fraud is rare by definition | Isolation Forest designed for imbalanced, rare-event detection |
| Multiple correlated features | PCA removes multicollinearity before detection |
| Interpretable output needed | 0–100 risk score is business-friendly |

**Pipeline:**
```
StandardScaler → PCA (95% variance) → Isolation Forest (contamination=10%)
     ↓                ↓                          ↓
  Normalize      7 components            anomaly score per distributor
  12 features    from 12 inputs          → normalized to 0–100
```

**Results:**
- Bartlett's test: χ²=1456, p<0.000001 → PCA approved ✅
- PCA: 12 features → 7 components, 95% variance explained
- Anomalies: 11/105 distributors (10.5%)
- Silhouette Score: 0.5206 (strong separation)
- Top anomaly: ENEL CE (score 100.0)

---

### 12. MLflow for Experiment Tracking

**Decision:** Log all model parameters, metrics, and artifacts to MLflow. Register in Databricks Model Registry.

**Logged artifacts:**

| Type | Content |
|---|---|
| Parameters | `contamination`, `pca_variance`, `n_components`, `n_estimators` |
| Metrics | `pca_explained_variance`, `n_anomalies_detected`, `anomaly_rate`, `silhouette_score` |
| Model | Full sklearn Pipeline (StandardScaler + PCA + IsolationForest) |

---

### 13. Pandas Bridge for sklearn

**Decision:** Convert aggregated Spark DataFrame to Pandas before applying sklearn.

scikit-learn runs on a single node — it cannot operate on Spark DataFrames. The bridge is safe because the Gold table has only 105 rows.

```
Spark (6.3M rows) → aggregate → Pandas (105 rows) → sklearn → Spark (write results)
```

---

## 🛡️ Data Governance

### Unity Catalog Tags

Tags applied on the Volume (`aneel_files`):
- `data_source: aneel`
- `data_classification: public`
- `pii: false`
- `environment: dev`
- `owner: zara_louise`

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

2. **Create the Volume:**
   ```sql
   CREATE VOLUME IF NOT EXISTS energy_project.raw.aneel_files;
   ```

3. **Upload ANEEL data to the Volume:**
   - Create subdirectories: `samp/` and `inadimplencia/`
   - Upload CSV files to respective folders

4. **Import notebooks to Databricks Workspace**

5. **Run the pipeline in order:**
   ```
   01_bronze_ingestion → 02_silver_transformation → 03_gold_features → 04_anomaly_model
   ```

---

## 📚 References

- [Databricks Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [Delta Lake Documentation](https://delta.io/learn/getting-started/)
- [Unity Catalog Best Practices](https://docs.databricks.com/data-governance/unity-catalog/best-practices.html)
- [Isolation Forest — scikit-learn](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html)
- [ANEEL Resolution Nº 1003/2022](https://www2.aneel.gov.br/cedoc/ren20221003.pdf)
