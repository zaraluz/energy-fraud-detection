# ⚡ Energy Fraud & Default Risk Detection

> End-to-end Big Data pipeline analyzing Brazil's electricity market to detect anomalous consumption patterns and predict default risk across 60+ power distributors.

![Status](https://img.shields.io/badge/status-in_development-yellow)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache_Spark-E25A1C?logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-00ADD4?logo=delta&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-0194E2?logo=mlflow&logoColor=white)
![Power BI](https://img.shields.io/badge/Power_BI-F2C811?logo=powerbi&logoColor=black)
![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Business Problem](#-business-problem)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Data Sources](#-data-sources)
- [Project Structure](#-project-structure)
- [Pipeline Layers](#-pipeline-layers)
- [Machine Learning Approach](#-machine-learning-approach)
- [Dashboard](#-dashboard)
- [Getting Started](#-getting-started)
- [Results](#-results)
- [Roadmap](#-roadmap)
- [Author](#-author)

---

## 🎯 Overview

This project builds a complete Big Data pipeline on Databricks using public data from **ANEEL** (Brazil's National Electric Energy Agency) to detect consumption anomalies and predict default risk across the Brazilian electricity distribution sector.

The pipeline ingests ~1.5GB of raw monthly market data spanning 5+ years, processes it through a **medallion architecture** (Bronze → Silver → Gold), trains an unsupervised anomaly detection model with MLflow tracking, and delivers actionable insights through a Power BI dashboard.

**Why it matters:** Brazilian power distributors lose an estimated **R$10+ billion per year** to non-technical losses (energy theft, meter fraud, and default). Detecting anomalies early enables targeted field operations and reduces revenue leakage.

---

## 💡 Business Problem

Brazilian electricity distributors face two interconnected challenges:

1. **Non-technical energy losses** — fraud, theft, and metering errors that aren't billed
2. **Customer default** — billed energy that is never paid

Both problems share a common signal: **anomalous consumption patterns**. A consumer who suddenly drops consumption may be tampering with their meter; a distributor with rising default rates may be facing economic stress in its region.

This project answers three core questions:

- 📉 Which distributors show the highest default risk in their aging list?
- 🚨 Which consumption patterns deviate significantly from historical baselines?
- 🗺️ How are risk indicators distributed geographically across Brazil?

---

## 🏗️ Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   DATA SOURCE   │     │   INGESTION     │     │  BRONZE LAYER   │
│                 │────▶│                 │────▶│                 │
│  ANEEL Open     │     │  Databricks     │     │  Raw Delta      │
│  Data Portal    │     │  Autoloader     │     │  Lake (1.5GB+)  │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                          │
                                                          ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   POWER BI      │     │   GOLD LAYER    │     │  SILVER LAYER   │
│                 │◀────│                 │◀────│                 │
│   Dashboard     │     │  Features +     │     │  Cleaned +      │
│   (DirectQuery) │     │  Risk Scores    │     │  Joined Data    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                  │
                                  ▼
                        ┌─────────────────┐
                        │  ML PIPELINE    │
                        │                 │
                        │  Spark ML +     │
                        │  MLflow         │
                        └─────────────────┘
```

---

## 🛠 Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Orchestration** | Databricks Workflows | Pipeline scheduling and dependency management |
| **Storage** | Delta Lake | ACID transactions, schema evolution, time travel |
| **Processing** | Apache Spark (PySpark) | Distributed data transformation |
| **ML / Tracking** | MLflow + Spark ML | Model training, versioning, registry |
| **Governance** | Unity Catalog | Data lineage and access control |
| **Visualization** | Power BI | Executive dashboards with DirectQuery |
| **Environment** | Databricks Free Edition | Free serverless compute for development |

---

## 📊 Data Sources

All data is **publicly available** through ANEEL's open data portal.

### 1. SAMP — Market Information System
> *Sistema de Acompanhamento de Informações de Mercado para Regulação Econômica*

Monthly market data declared by distributors and permission holders, regulated by **ANEEL Resolution Nº 1003/2022**.

- **Granularity:** Monthly × Distributor × Consumption class
- **Coverage:** 2020–2025 (focused window for the project)
- **Volume:** ~1.5 GB across 5 yearly CSV files
- **Source:** [dadosabertos.aneel.gov.br](https://dadosabertos.aneel.gov.br/)

### 2. Indqual — Default & Aging List
> *Indicadores de Qualidade — Inadimplência*

Distributor-level default indicators, including aging list buckets (30, 60, 90+ days) and suspension counts per consumption class.

- **Granularity:** Monthly × Distributor × Class
- **Key file:** `inadimplencia.csv` (~58 MB)
- **Dimension table:** `dominio-indicadores.csv` (indicator code lookup)

---

## 📁 Project Structure

```
energy-fraud-detection/
├── README.md
├── .gitignore
├── requirements.txt
│
├── docs/
│   ├── architecture.png
│   ├── data_dictionary.md
│   └── methodology.md
│
├── notebooks/
│   ├── 01_bronze_ingestion.py        # Raw CSV → Delta (Bronze)
│   ├── 02_silver_transformation.py   # Cleaning + joins (Silver)
│   ├── 03_gold_features.py           # Feature engineering (Gold)
│   ├── 04_anomaly_model.py           # Spark ML + MLflow
│   └── 05_risk_scoring.py            # Score generation
│
├── src/
│   ├── utils/
│   │   ├── schemas.py                # Explicit Spark schemas
│   │   └── transformations.py        # Reusable PySpark functions
│   └── config/
│       └── pipeline_config.yaml
│
├── sql/
│   ├── create_tables.sql
│   └── analytical_queries.sql
│
├── dashboard/
│   ├── energy_risk_dashboard.pbix
│   └── screenshots/
│
└── tests/
    └── test_transformations.py
```

---

## 🔄 Pipeline Layers

### 🥉 Bronze Layer — Raw Ingestion
- Reads CSV files from ANEEL with explicit schemas (avoiding inference overhead)
- Persists raw data as **Delta tables** with metadata columns (`ingestion_timestamp`, `source_file`)
- Handles encoding (Latin-1 → UTF-8) and Brazilian decimal format (`,` → `.`)

### 🥈 Silver Layer — Cleansed & Joined
- Deduplicates records by composite keys
- Joins SAMP with the indicator domain table
- Standardizes distributor names and codes
- Validates data quality (null checks, range validation)
- Partitioned by `year` and `month` for query performance

### 🥇 Gold Layer — Analytics-Ready
- Aggregates consumption per UC (consumption unit) and class
- Engineers temporal features:
  - 3-month and 12-month rolling averages
  - Standard deviation from baseline
  - Year-over-year consumption delta
  - Seasonality indicators
- Joins default indicators per distributor
- Final risk-scored tables ready for ML and BI

---

## 🤖 Machine Learning Approach

**Problem framing:** Unsupervised anomaly detection (no labeled fraud data in public sources).

**Algorithm:** **Isolation Forest** trained at scale via Spark ML, with **XGBoost** ranker as a secondary model for distributor-level default prediction.

**Features engineered:**
- Consumption deviation (Z-score vs. peer group)
- Seasonal anomaly indicators
- Default rate trends (3M, 6M, 12M windows)
- Class-level concentration metrics

**MLflow tracking:**
- All experiments logged with parameters, metrics, and artifacts
- Best model registered in **Databricks Model Registry**
- Model versioning enables A/B testing in production

**Evaluation metrics:**
- Top-decile precision (high-risk UCs flagged)
- AUC for distributor default prediction
- Stability across temporal folds

---

## 📈 Dashboard

The Power BI dashboard connects to the Gold layer via **Databricks SQL Warehouse** (DirectQuery) and provides:

### Executive View
- 🇧🇷 National average non-technical loss rate
- 💰 Estimated unrealized revenue (R$ billions/year)
- 🚨 Count of high-risk consumption units

### Geographic Heatmap
- Risk concentration by state and distributor
- Drill-down: state → distributor → consumption class

### Distributor Ranking
- Top 10 distributors by default rate
- Aging list breakdown (30/60/90+ days)
- Quality indicators (DEC/FEC) correlation

### Trend Analysis
- Historical default evolution
- Class-level consumption patterns
- Pre-/post-pandemic comparison

---

## 🚀 Getting Started

### Prerequisites
- [Databricks Free Edition](https://www.databricks.com/learn/free-edition) account (free)
- Power BI Desktop (free, Windows only)
- Python 3.10+ (optional, for local exploration)

### Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-user/energy-fraud-detection.git
   cd energy-fraud-detection
   ```

2. **Download raw data**
   - Visit [ANEEL Open Data](https://dadosabertos.aneel.gov.br/)
   - Download SAMP files (2020–2025) and Indqual Inadimplência
   - Upload to Databricks Volumes or DBFS

3. **Import notebooks to Databricks**
   - Workspace → Import → Select `notebooks/` folder
   - Attach to a serverless cluster

4. **Run the pipeline**
   ```
   01_bronze_ingestion → 02_silver_transformation
                      → 03_gold_features
                      → 04_anomaly_model
                      → 05_risk_scoring
   ```

5. **Connect Power BI**
   - Get → More → Azure → Azure Databricks
   - Use the SQL Warehouse connection string from your workspace
   - Select Gold layer tables

---

## 📊 Results

> 🚧 *To be updated as the project progresses*

Expected deliverables:
- ✅ Reproducible pipeline running end-to-end on Databricks Free Edition
- ✅ Anomaly detection model with documented AUC and precision metrics
- ✅ Interactive Power BI dashboard with 4+ views
- ✅ Risk scores for all distributors across the analyzed period

---

## 🗺️ Roadmap

- [x] Project conception and architecture design
- [x] Data source identification and download
- [ ] **v1.0** — Bronze layer with SAMP + Indqual
- [ ] **v1.1** — Silver layer with full data quality checks
- [ ] **v1.2** — Gold layer with engineered features
- [ ] **v2.0** — ML pipeline with MLflow tracking
- [ ] **v2.1** — Power BI dashboard published
- [ ] **v3.0** — Extended with DEC/FEC quality indicators
- [ ] **v3.1** — Geographic enrichment with IBGE municipality data
- [ ] **v4.0** — Real-time scoring with Structured Streaming

---

## 📚 References

- ANEEL Open Data Portal: https://dadosabertos.aneel.gov.br/
- ANEEL Resolution Nº 1003/2022 (SAMP regulation)
- Databricks Medallion Architecture: https://www.databricks.com/glossary/medallion-architecture
- Brazilian Electricity Loss Reports (ABRADEE)

---

## 👤 Author

**[Your Name]**

Data Engineer | Big Data Enthusiast

- 🔗 LinkedIn: [linkedin.com/in/yourprofile](https://linkedin.com/in/yourprofile)
- 📧 Email: your.email@example.com
- 🌐 Portfolio: [yourportfolio.com](https://yourportfolio.com)

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

ANEEL data is provided under the [Open Data Commons Open Database License (ODbL)](https://opendatacommons.org/licenses/odbl/).

---

<p align="center">
  <strong>⭐ If this project helped you, consider giving it a star! ⭐</strong>
</p>
