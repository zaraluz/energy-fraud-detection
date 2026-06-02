# ⚡ Energy Fraud & Default Risk Detection

> End-to-end Big Data pipeline analyzing Brazil's electricity market to detect anomalous consumption patterns and predict default risk across 105 power distributors — from raw CSV ingestion to an interactive Power BI dashboard.

![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache_Spark-E25A1C?logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-00ADD4?logo=delta&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-0194E2?logo=mlflow&logoColor=white)
![Power BI](https://img.shields.io/badge/Power_BI-F2C811?logo=powerbi&logoColor=black)
![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

---

## 🎯 Project Overview

This project implements a complete Big Data pipeline on **Databricks** using public data from **ANEEL** (Brazil's National Electric Energy Agency) to detect consumption anomalies and predict default risk across the Brazilian electricity distribution sector.

The pipeline ingests **~1.5 GB** of raw monthly market data spanning 7 years (2020–2026), processes it through a **medallion architecture** (Bronze → Silver → Gold), trains an unsupervised anomaly detection model with MLflow tracking, and delivers insights through a 3-page interactive Power BI dashboard.

### 💡 Business Problem

Brazilian electricity distributors face two interconnected challenges:
1. **Non-technical energy losses** — fraud, theft, and metering errors (~R$10B+/year)
2. **Customer default** — billed energy that is never paid

Both problems share a signal: **anomalous consumption patterns**. This project answers three questions:
- Which distributors show the highest default risk?
- Which consumption patterns deviate from baselines?
- How are risk indicators distributed geographically?

---

## 🏗️ Architecture (Medallion)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   RAW       │────▶│   BRONZE    │────▶│   SILVER    │────▶│   GOLD      │
│             │     │             │     │             │     │             │
│  CSV files  │     │ Delta tables│     │ Cleansed +  │     │ Aggregated  │
│  (Volume)   │     │ All strings │     │ Typed data  │     │ ML features │
│             │     │ + metadata  │     │ + joined    │     │ BI-ready    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                          │                                        │
                          ▼                                        ▼
                    Unity Catalog                              Power BI
                    Governance                             3-Page Dashboard
```

See [docs/architecture.md](docs/architecture.md) for detailed architecture decisions.

---

## 📊 Current Status

| Layer | Status | Records | Tables |
|---|---|---|---|
| 🟫 Raw | ✅ Complete | 9 CSV files (~1.6 GB) | — |
| 🥉 Bronze | ✅ Complete | 6,808,591 rows | `samp`, `inadimplencia`, `dominio_indicadores` |
| 🥈 Silver | ✅ Complete | 7,423,768 rows | `samp`, `inadimplencia`, `samp_quarantine` |
| 🥇 Gold | ✅ Complete | 105 distributors | `features_distributor`, `risk_scores` |
| 🤖 ML Model | ✅ Complete | 11 anomalies detected | PCA + Isolation Forest (MLflow) |
| 📈 Dashboard | ✅ Complete | 3 pages | Executive Overview · Anomaly Analysis · Distributor Profile |

---

## 🤖 Model Results

The anomaly detection pipeline (StandardScaler → PCA → Isolation Forest) was trained on 12 engineered features per distributor.

| Metric | Value |
|---|---|
| Input features | 12 |
| PCA components kept | 7 |
| PCA variance explained | 95.0% |
| Bartlett's test p-value | < 0.000001 ✅ |
| Distributors scored | 105 |
| Anomalies detected | 11 (10.5%) |
| Silhouette Score | 0.5206 (strong separation) |

### 🚨 Top 5 Highest Risk Distributors

| Rank | Distributor | Risk Score | Label |
|---|---|---|---|
| 1 | ENEL CE | 100.0 | HIGH RISK |
| 2 | ELETROPAULO | 76.24 | HIGH RISK |
| 3 | CEA | 75.63 | HIGH RISK |
| 4 | CEMIG-D | 69.70 | HIGH RISK |
| 5 | EQUATORIAL MA | 69.68 | HIGH RISK |

---

## 📈 Dashboard

The Power BI dashboard (`dashboard/energy-fraud.pbix`) delivers interactive insights across 3 pages:

| Page | Description |
|---|---|
| **1 — Executive Overview** | KPI cards · Azure Maps bubble chart · HIGH RISK ranking table · Risk distribution donut |
| **2 — Anomaly Analysis** | Consumption vs Default Rate scatter · Risk Score histogram · Consumer mix by distributor · Consumption trend by risk rank |
| **3 — Distributor Profile** | Drill-through page with individual distributor KPIs · Consumer mix breakdown · Aging and variability indicators |

See [dashboard/README.md](dashboard/README.md) for full documentation including DAX measures, data model, and setup instructions.

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| **Compute** | Databricks Free Edition (Serverless) |
| **Storage** | Delta Lake on Unity Catalog Volumes |
| **Processing** | Apache Spark (PySpark) |
| **ML Pipeline** | Scikit-learn (StandardScaler + PCA + Isolation Forest) |
| **Experiment Tracking** | MLflow + Databricks Model Registry |
| **Governance** | Unity Catalog with tags |
| **Visualization** | Power BI (DirectQuery via Databricks connector) |
| **Version Control** | Git + GitHub |

---

## 📁 Project Structure

```
energy-fraud-detection/
├── README.md                        # This file
├── LICENSE                          # MIT License
├── .gitignore
│
├── docs/                            # Detailed documentation
│   ├── architecture.md              # Architecture decisions
│   ├── bronze_layer.md              # Bronze layer deep dive
│   ├── silver_layer.md              # Silver layer deep dive
│   ├── gold_layer.md                # Gold layer deep dive
│   └── data_dictionary.md           # PT-BR → EN column mapping
│
├── notebooks/                       # Databricks notebooks
│   ├── 01_bronze_ingestion.py       # ✅ Raw CSV → Bronze Delta
│   ├── 02_silver_transformation.py  # ✅ Cleansing + joins
│   ├── 03_gold_features.py          # ✅ Feature engineering
│   └── 04_anomaly_model.py          # ✅ PCA + Isolation Forest + MLflow
│
├── sql/                             # SQL queries and DDL
├── dashboard/                       # Power BI files
│   ├── README.md                    # ✅ Dashboard documentation
│   └── energy-fraud.pbix            # Power BI report file
├── src/                             # Reusable Python modules
└── tests/                           # Unit tests
```

---

## 📊 Data Sources

All data is **publicly available** through [ANEEL's open data portal](https://dadosabertos.aneel.gov.br/).

### 1. SAMP — Market Information System
- **Granularity:** Monthly × Distributor × Consumption class
- **Coverage:** 2020–2026 (7 yearly CSV files)
- **Volume:** ~1.5 GB, 6.8M rows
- **Regulation:** ANEEL Resolution Nº 1003/2022

### 2. Indqual — Default & Aging List
- **Granularity:** Monthly × Distributor × Indicator
- **Files:** `inadimplencia.csv` + `dominio-indicadores.csv`
- **Volume:** ~58 MB, 1.1M rows
- **Coverage:** 2012–2026

---

## 🚀 Getting Started

### Prerequisites
- [Databricks Free Edition](https://www.databricks.com/learn/free-edition) account
- Power BI Desktop (Windows, free)
- ANEEL data downloaded to a Databricks Volume

### Setup
1. **Clone the repository**
   ```bash
   git clone https://github.com/zaraluz/energy-fraud-detection.git
   ```

2. **Import notebooks to Databricks**
   - Workspace → Import → Select `notebooks/` folder

3. **Run the pipeline**
   ```
   01_bronze_ingestion → 02_silver_transformation → 03_gold_features → 04_anomaly_model
   ```

4. **Open the dashboard**
   - Open `dashboard/energy-fraud.pbix` in Power BI Desktop
   - Update the Databricks connection string to your workspace

See detailed setup instructions in [docs/architecture.md](docs/architecture.md#setup) and [dashboard/README.md](dashboard/README.md#setup).

---

## 🎓 Key Engineering Decisions

This project applies industry best practices documented across [docs/](docs/):

- 🥉 **Explicit StringType schema in Bronze** for deterministic, lossless ingestion ([why](docs/bronze_layer.md#explicit-schema))
- 🗂️ **Partitioning by year** for query performance ([details](docs/bronze_layer.md#partitioning))
- 🔍 **Audit metadata columns** for data lineage and drift detection ([approach](docs/bronze_layer.md#metadata))
- 🌐 **PT-BR column names preserved in Bronze** for ANEEL traceability ([rationale](docs/data_dictionary.md))
- 📊 **Delta Lake over Parquet** for ACID + time travel + schema evolution
- 🔤 **EN column names in Silver** — PT-BR → English rename at transformation layer
- ⚡ **Broadcast join** for `dominio_indicadores` (~40 KB) — avoids shuffle
- 🏗️ **reference_date constructed** from `AnoIndice` + `NumPeriodoIndice` in inadimplência
- 🔢 **Decimal separator fix** — ANEEL CSVs use comma; replaced before `DoubleType` cast
- 🚫 **Single `.count()` per table** — action operations are expensive in Spark
- 📐 **Coefficient of variation** as consumption instability signal
- 🧮 **PCA pre-processing** — reduces 12 features to 7 components (95% variance)
- 🌲 **Isolation Forest** — unsupervised anomaly detection, no labels required
- ✅ **Bartlett's test** — prerequisite check confirming PCA appropriateness (p < 0.000001)
- 📏 **Silhouette Score** — model quality metric for unsupervised learning (0.52)

---

## 🗺️ Roadmap

- [x] Project conception and architecture design
- [x] Data source identification and download
- [x] **v1.0** — Bronze layer with SAMP + Inadimplência
- [x] **v1.1** — Silver layer with type casting, validation and joins
- [x] **v1.2** — Gold layer with engineered features
- [x] **v2.0** — ML pipeline with MLflow tracking (PCA + Isolation Forest)
- [x] **v2.1** — Power BI dashboard (3 pages · Azure Maps · drill-through)
- [ ] **v3.0** — Geographic enrichment (IBGE data)
- [ ] **v4.0** — Real-time scoring with Structured Streaming

---

## 📚 Documentation

- [Architecture & Design Decisions](docs/architecture.md)
- [Bronze Layer Deep Dive](docs/bronze_layer.md)
- [Silver Layer Deep Dive](docs/silver_layer.md)
- [Gold Layer Deep Dive](docs/gold_layer.md)
- [Data Dictionary (PT-BR → EN)](docs/data_dictionary.md)
- [Dashboard Documentation](dashboard/README.md)

---

## 👤 Author

**Zara Louise**

Data Analyst | Big Data | MBA in Data Science & AI — USP

- 🔗 LinkedIn: [linkedin.com/in/zaralouise](https://linkedin.com/in/zaralouise)
- 📧 Email: louise_zara@yahoo.com.br

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

ANEEL data is provided under the [Open Data Commons Open Database License (ODbL)](https://opendatacommons.org/licenses/odbl/).

---

<p align="center">
  <strong>⭐ If this project helped you, consider giving it a star!</strong>
</p>
