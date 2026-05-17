# 🏛️ Architecture & Design Decisions

This document captures the architectural decisions and rationale behind the project design.

---

## 📐 Medallion Architecture

The project follows Databricks' **medallion architecture** pattern, organizing data in progressive layers of refinement.

### Layer Responsibilities

| Layer | Purpose | Data Quality | Schema | Audience |
|---|---|---|---|---|
| **🟫 Raw** | Source files preserved as-is | Source-dependent | N/A (files) | Data Engineers |
| **🥉 Bronze** | Lossless ingestion + metadata | Type-preserved (strings) | Source-faithful | Data Engineers |
| **🥈 Silver** | Cleansed, typed, validated | Quality-enforced | Standardized (EN) | Data Scientists |
| **🥇 Gold** | Aggregated, ML-ready | Business-validated | Use-case oriented | Analysts, BI users |

### Catalog Structure (Unity Catalog)

```
energy_project/                    # Catalog
├── raw/                            # Schema
│   └── aneel_files/                # Volume (CSV files)
│       ├── samp/                   # SAMP yearly files
│       └── inadimplencia/          # Default data
├── bronze/                         # Schema
│   ├── samp                        # Delta table
│   ├── inadimplencia               # Delta table
│   └── dominio_indicadores         # Delta table (dimension)
├── silver/                         # Schema (in progress)
└── gold/                           # Schema (planned)
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
| Compression | ✅ | ✅ |
| ACID transactions | ❌ | ✅ |
| Time travel | ❌ | ✅ |
| Schema evolution | ❌ | ✅ |
| MERGE/UPDATE/DELETE | ❌ | ✅ |
| Concurrent writes | ❌ | ✅ |

**Trade-off:** Delta has slight overhead from the transaction log, but the operational benefits (rollback, reproducibility, concurrent access) far outweigh the cost.

---

### 3. Explicit Schema over inferSchema

**Decision:** Define `StructType` schemas explicitly for all readers.

**Rationale:**
During development, we observed that `inferSchema=true` produces **non-deterministic results** — the same dataset returned different row counts (14M vs 6.8M) across executions due to inconsistent type inference across cluster restarts.

**Root cause:** Spark samples data to infer types. When sampling differs between runs, type inference changes, and rows that don't match the inferred types may be silently dropped.

**Resolution:** Use explicit `StringType` schema in Bronze. Type casting is deferred to Silver, where errors can be handled explicitly (NULL on failure, log to dead-letter queue, etc.).

This is documented in [Apache Spark's official guidance](https://spark.apache.org/docs/latest/sql-data-sources-csv.html) as best practice for production pipelines.

See [bronze_layer.md](bronze_layer.md#explicit-schema) for the full investigation.

---

### 4. PT-BR Column Names Preserved in Bronze

**Decision:** Keep original ANEEL column names (Portuguese) in the Bronze layer. Translate to English (snake_case) in Silver.

**Rationale:**
- **Traceability:** A column named `VlrMercado` can be looked up directly in ANEEL's data dictionary. A column named `market_value` cannot.
- **Lineage:** When debugging, the ability to map back to the source is critical.
- **Separation of concerns:** Bronze preserves source; Silver standardizes.

**Implementation:** A `COLUMN_DICTIONARY` is documented in the Bronze notebook for reference, but no translation is applied at this layer.

See [data_dictionary.md](data_dictionary.md) for the full mapping.

---

### 5. Partitioning by Year

**Decision:** Partition Bronze and downstream tables by `_partition_year` extracted from `DatCompetencia`.

**Rationale:**
- Query patterns will heavily filter by year (year-over-year comparisons, trend analysis)
- Partition pruning provides ~7x speedup for year-filtered queries
- 7 yearly partitions are well-balanced (not too few, not too many)

**Trade-off:** Partition columns add a small storage overhead but reduce I/O significantly for typical queries.

See [bronze_layer.md](bronze_layer.md#partitioning) for benchmarks.

---

### 6. Audit Metadata Columns

**Decision:** Every Bronze table includes technical metadata columns prefixed with `_`:

| Column | Type | Purpose |
|---|---|---|
| `_ingestion_timestamp` | timestamp | When the row was loaded |
| `_source_file` | string | Origin file path |
| `_ingestion_date` | date | Load date (for partitioning/filtering) |

**Rationale:**
Enables critical operational capabilities:
- **Debugging:** Trace any row back to its source file and load time
- **Incremental processing:** Filter by `_ingestion_date` for delta loads
- **Auditability:** Compliance and reproducibility requirements
- **Drift detection:** Compare data distributions across ingestion batches

This implements the **Data Lineage** pattern that's foundational to MLOps.

See [bronze_layer.md](bronze_layer.md#metadata) for details.

---

### 7. Modern Unity Catalog Patterns

**Decision:** Use `_metadata.file_path` instead of the deprecated `input_file_name()`.

**Rationale:**
Databricks deprecated `input_file_name()` in Unity Catalog environments. The modern `_metadata` syntax provides richer information (file size, modification time, etc.) and is the recommended pattern.

---

## 🛡️ Data Governance

### Unity Catalog Tags

Tags are applied at multiple levels for data classification and governance:

**Catalog (`energy_project`):**
- `project: energy_fraud_detection`
- `team: data_engineering`
- `environment: dev`

**Schemas:**
- `bronze` → `data_layer: bronze`, `data_quality: ingested`
- `silver` → `data_layer: silver`, `data_quality: cleansed`
- `gold` → `data_layer: gold`, `data_quality: production_ready`

**Volume (`aneel_files`):**
- `data_source: aneel`
- `data_classification: public`
- `pii: false`

These tags enable:
- **Filtering** in Catalog Explorer
- **Access policies** by tag
- **Cost attribution** by project/team
- **Compliance reporting**

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
   - `01_bronze_ingestion`
   - `02_silver_transformation` (when available)
   - `03_gold_features` (when available)
   - `04_anomaly_model` (when available)

---

## 📚 References

- [Databricks Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [Delta Lake Documentation](https://delta.io/learn/getting-started/)
- [Unity Catalog Best Practices](https://docs.databricks.com/data-governance/unity-catalog/best-practices.html)
- [ANEEL Resolution Nº 1003/2022](https://www2.aneel.gov.br/cedoc/ren20221003.pdf)
