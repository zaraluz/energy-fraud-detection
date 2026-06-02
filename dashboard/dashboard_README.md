# 📈 Power BI Dashboard — Energy Fraud & Default Risk

> Interactive 3-page Power BI report delivering insights from the anomaly detection pipeline. Built on Gold layer tables (`features_distributor` and `risk_scores`) connected via Databricks DirectQuery.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Pages](#pages)
- [Data Model](#data-model)
- [DAX Measures](#dax-measures)
- [Calculated Columns](#calculated-columns)
- [Custom Tables](#custom-tables)
- [Visuals & Configuration](#visuals--configuration)
- [Theme](#theme)
- [Setup](#setup)

---

## Overview

| Property | Value |
|---|---|
| **File** | `energy-fraud.pbix` |
| **Canvas size** | 1600 × 900 px |
| **Connection** | Databricks DirectQuery |
| **Gold tables** | `features_distributor`, `risk_scores` |
| **Custom tables** | `dim_uf` (state mapping with coordinates) |
| **Pages** | 3 (Executive Overview · Anomaly Analysis · Distributor Profile) |
| **Theme** | Custom dark — `energy-fraud-theme.json` |

---

## Pages

### Page 1 — Executive Overview

High-level summary of the risk landscape across all 105 Brazilian electricity distributors.

**Visuals:**

| Visual | Type | Fields | Notes |
|---|---|---|---|
| Header | Text box | — | Title + subtitle with project metadata |
| Total Distributors | Card | `[Total Distributors]` | Count of all distributors |
| High Risk Distributors | Card | `[High Risk Distributors]` | Count of anomaly-labeled distributors |
| Silhouette Score | Card | `[Silhouette Score]` | Fixed value — model quality metric |
| Avg Risk Score (High) | Card | `[Avg Risk Score High]` | Average risk score among HIGH RISK group |
| % High Risk | Card | `[Pct High Risk]` | Share of HIGH RISK distributors |
| Risk Map | Azure Maps | Lat/Lng from `dim_uf`, Size = `avg_default_rate` (Average), Legend = `anomaly_label` | Bubble size proportional to average default rate per state; Night map style |
| HIGH RISK Ranking | Table | `distributor_name`, `risk_score` (Average), `anomaly_label` | Filtered to HIGH RISK; sorted by risk score DESC; conditional formatting (gradient bar) on risk_score |
| Risk Distribution | Donut | Legend = `anomaly_label`, Values = `[Total Distributors]` | HIGH RISK = `#FF6B35`, NORMAL = `#2ECC71` |

---

### Page 2 — Anomaly Analysis

Deep-dive into consumption patterns and model outputs across all 105 distributors.

**Visuals:**

| Visual | Type | Fields | Notes |
|---|---|---|---|
| Header | Text box | — | Page title |
| Slicer | Tile slicer | `anomaly_label` | Filters all visuals on page; HIGH RISK button color `#FF6B35` |
| Consumption vs Default Rate | Bubble chart (Scatter) | X = `avg_consumption_kwh`, Y = `avg_default_rate`, Size = `risk_score`, Legend = `anomaly_label` | HIGH RISK = `#FF6B35`, NORMAL = `#2ECC71` |
| Risk Score Distribution | Clustered column chart | X = `Risk Band`, Y = Count of `distributor_code`, Legend = `anomaly_label` | Ordered via `Risk Band Order` column; HIGH RISK = `#FF6B35`, NORMAL = `#2ECC71` |
| Consumer Mix by Distributor | Stacked bar chart | Y = `distributor_name`, X = Average of `Residential`, `Industrial`, `Commercial` | Residential = `#3498DB`, Industrial = `#F39C12`, Commercial = `#1ABC9C` |
| Consumption Trend by Risk Rank | Line chart | X = `risk_rank`, Y = Average of `consumption_trend`, Legend = `anomaly_label` | Shows whether high-ranked (high-risk) distributors exhibit distinct trend patterns |

---

### Page 3 — Distributor Profile

Drill-through page showing individual distributor metrics. Accessed by right-clicking any distributor name in Pages 1 or 2 → Drill through → Distributor Profile.

**Drill-through field:** `Risk Scores[distributor_name]`

**Visuals:**

| Visual | Type | Fields | Notes |
|---|---|---|---|
| Risk Score | Card | `risk_score` (Average) | Individual distributor risk score |
| Anomaly Label | Card | `anomaly_label` (First) | HIGH RISK or NORMAL |
| Avg Default Rate | Card | `avg_default_rate` (Average) | Distributor's average default rate |
| Avg Consumption (kWh) | Card | `avg_consumption_kwh` (Average) | Distributor's average monthly consumption |
| Consumer Mix | Stacked bar chart | Y = `distributor_name`, X = Average of `Residential`, `Industrial`, `Commercial` | Single bar showing the distributor's consumer profile |
| Aging 12m | Card | `avg_aging_12` (Average) | Average debt aging at 12 months |
| Aging 24m | Card | `avg_aging_24` (Average) | Average debt aging at 24 months |
| Consumption Variability | Card | `cv_consumption` (Average) | Coefficient of variation — instability signal used by the ML model |

---

## Data Model

```
features_distributor ──── Risk Scores
  distributor_code    1:1   distributor_code
  
dim_uf ──────────────────── Risk Scores
  distributor_code    *:1   distributor_code
```

`dim_uf` is a manually created lookup table that maps each distributor code to its Brazilian state (UF) and geographic coordinates (latitude/longitude), enabling the Azure Maps visualization.

All measures are stored in a dedicated `measures_` table to keep the model organized.

---

## DAX Measures

All measures live in the `measures_` table.

```dax
Total Distributors =
DISTINCTCOUNT('Risk Scores'[distributor_code])
```

```dax
High Risk Distributors =
CALCULATE(
    DISTINCTCOUNT('Risk Scores'[distributor_code]),
    'Risk Scores'[anomaly_label] = "HIGH RISK"
)
```

```dax
Silhouette Score = 0.5206
-- Fixed value from MLflow model run
```

```dax
Avg Risk Score High =
CALCULATE(
    AVERAGE('Risk Scores'[risk_score]),
    'Risk Scores'[anomaly_label] = "HIGH RISK"
)
```

```dax
Pct High Risk =
DIVIDE([High Risk Distributors], [Total Distributors], 0)
-- Format as percentage
```

---

## Calculated Columns

Calculated columns added to the `Risk Scores` table.

### Risk Band

Groups distributors into risk score buckets for histogram visualization. Prefixed with a zero-padded number to enforce sort order.

```dax
Risk Band =
SWITCH(
    TRUE(),
    'Risk Scores'[risk_score] <= 20, "01 | 0-20",
    'Risk Scores'[risk_score] <= 40, "02 | 21-40",
    'Risk Scores'[risk_score] <= 60, "03 | 41-60",
    'Risk Scores'[risk_score] <= 80, "04 | 61-80",
    "05 | 81-100"
)
```

### Risk Band Order

Numeric sort key for `Risk Band`. Set as the **Sort by column** for `Risk Band` via Column Tools → Sort by Column.

```dax
Risk Band Order =
SWITCH(
    TRUE(),
    'Risk Scores'[risk_score] <= 20, 1,
    'Risk Scores'[risk_score] <= 40, 2,
    'Risk Scores'[risk_score] <= 60, 3,
    'Risk Scores'[risk_score] <= 80, 4,
    5
)
```

---

## Custom Tables

### dim_uf

Manually created in Power Query (blank query → Advanced Editor) to map distributor codes to geographic coordinates for the Azure Maps visual.

**Columns:** `distributor_code` (text), `state` (text), `latitude` (decimal), `longitude` (decimal)

**Sample rows:**

| distributor_code | state | latitude | longitude |
|---|---|---|---|
| ENEL CE | CE | -3.73 | -38.52 |
| ELETROPAULO | SP | -23.55 | -46.63 |
| CEMIG-D | MG | -19.92 | -43.94 |
| EQUATORIAL MA | MA | -2.53 | -44.30 |
| COELBA | BA | -12.97 | -38.51 |

> Note: Distributors sharing the same state use slightly offset coordinates to avoid full overlap on the map.

---

## Visuals & Configuration

### Azure Maps
- **Tenant requirement:** Three Azure Maps switches must be enabled in the Power BI Admin Portal → Tenant Settings → Integration Settings
- **Map style:** Night
- **Bubble size series:** HIGH RISK max = 20px, NORMAL max = 20px
- **Color:** HIGH RISK = `#FF6B35`, NORMAL = `#2ECC71`

### HTML Content (Daniel Marsh-Patrick)
- Used during development for SVG map fallback; not present in final version

### Conditional Formatting — Risk Score column (Page 1 table)
- **Format style:** Gradient
- **Minimum color:** `#16213E`
- **Maximum color:** `#FF6B35`
- **Apply to:** Values only

---

## Theme

Custom JSON theme file: `energy-fraud-theme.json`

**Color palette:**

| Role | Hex |
|---|---|
| Primary accent (HIGH RISK) | `#FF6B35` |
| Secondary accent (NORMAL) | `#2ECC71` |
| Warning | `#F39C12` |
| Background (page) | `#1A1A2E` |
| Background (visuals) | `#16213E` |
| Background (sidebar / cards) | `#0F3460` |
| Text primary | `#E8E8F0` |
| Text secondary | `#A0A0C0` |

To apply: **View → Themes → Browse for themes** → select `energy-fraud-theme.json`

---

## Setup

### Prerequisites
- Power BI Desktop (Windows)
- Databricks workspace with Gold layer tables available
- Azure Maps enabled in your Power BI tenant (see [Azure Maps](#azure-maps) above)

### Steps

1. **Clone the repository**
   ```bash
   git clone https://github.com/zaraluz/energy-fraud-detection.git
   ```

2. **Open the report**
   ```
   Open dashboard/energy-fraud.pbix in Power BI Desktop
   ```

3. **Update the Databricks connection**
   - Home → Transform Data → Data Source Settings
   - Update the Databricks server hostname and HTTP path to your workspace
   - Authenticate with your Databricks personal access token

4. **Apply the theme**
   - View → Themes → Browse for themes → select `energy-fraud-theme.json`

5. **Refresh data**
   - Home → Refresh

### Troubleshooting

| Issue | Solution |
|---|---|
| Azure Maps shows "not enabled for your organization" | Enable all three Azure Maps switches in Power BI Admin Portal → Tenant Settings → Integration Settings |
| Native map visual blocked | Same admin portal fix — enable "Map and filled map visuals" |
| `dim_uf` relationship broken | Verify distributor codes match exactly between `dim_uf` and `Risk Scores[distributor_code]` — check for trailing spaces or encoding differences |
| DAX errors referencing `risk_scores` | Table name in Power BI is `Risk Scores` (with space and capital S) — always wrap in single quotes: `'Risk Scores'[column]` |

---

*Built as part of the [energy-fraud-detection](https://github.com/zaraluz/energy-fraud-detection) portfolio project.*
