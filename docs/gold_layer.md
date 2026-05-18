# 🥇 Gold Layer — Deep Dive

This document provides a comprehensive technical breakdown of the Gold layer implementation, covering feature engineering decisions, aggregation strategies, and lessons learned.

---

## 🎯 Purpose

The Gold layer is the **feature engineering and ML-ready stage** of the medallion architecture. It reads from Silver Delta tables and produces a single aggregated feature matrix — one row per distributor — ready for the anomaly detection model.

### Gold Philosophy

> **"Gold is where data becomes insight."**

The Gold layer must:
- ✅ Aggregate monthly data into distributor-level features
- ✅ Engineer signals that capture consumption behavior, volatility, and default risk
- ✅ Join SAMP and inadimplência features into a single matrix
- ✅ Produce a clean, typed, null-free feature table ready for ML
- ❌ NOT apply ML models (that is the anomaly model's responsibility)
- ❌ NOT expose raw monthly data — only aggregated features

---

## 📊 Tables Produced

| Delta Table | Source | Rows | Columns | Partitioned |
|---|---|---|---|---|
| `energy_project.gold.features_distributor` | `silver.samp` + `silver.inadimplencia` | 105 | 15 | No |
| `energy_project.gold.risk_scores` | `gold.features_distributor` + ML model | 105 | 6 | No |

> Both tables have ~105 rows — too small to benefit from partitioning.

---

## 📐 Feature Matrix (12 features)

| Feature | Source | Description | Signal |
|---|---|---|---|
| `avg_consumption_kwh` | silver.samp | Average monthly consumption in kWh | Baseline volume |
| `std_consumption_kwh` | silver.samp | Standard deviation of monthly consumption | Volatility |
| `cv_consumption` | silver.samp | Coefficient of variation (std/avg) | Instability signal |
| `consumption_trend` | silver.samp | Slope of consumption over time | Growth or decline |
| `pct_residential` | silver.samp | % of consumption from residential class | Consumer mix |
| `pct_industrial` | silver.samp | % of consumption from industrial class | Consumer mix |
| `pct_commercial` | silver.samp | % of consumption from commercial class | Consumer mix |
| `avg_default_rate` | silver.inadimplencia | Average ITotCrt (total default rate) | Default risk |
| `avg_aging_12` | silver.inadimplencia | Average ITot12 (12-month aging) | Aging risk |
| `avg_aging_24` | silver.inadimplencia | Average ITot24 (24-month aging) | Aging risk |
| `default_trend` | silver.inadimplencia | Slope of ITotCrt over time | Worsening or improving |
| `default_consumption_ratio` | both | avg_default_rate / avg_consumption_kwh | Combined risk signal |

---

## 🔑 Key Engineering Decisions

### 1. Filtering market_detail for Consumption

`market_value` in SAMP is polymorphic — the same column represents different things depending on `market_detail` (consumers, MWh, R$, taxes, etc.). We filter specifically for `"Energia Consumida (kWh)"` to get the actual billed consumption.

This was confirmed by inspecting all 25+ distinct `market_detail` values before writing any aggregation.

---

### 2. Coefficient of Variation as Instability Signal

Simple average consumption doesn't reveal anomalies. The coefficient of variation (cv = std/avg) captures **instability**:

```python
.withColumn("cv_consumption",
    F.col("std_consumption_kwh") / F.col("avg_consumption_kwh"))
```

A high `cv_consumption` means consumption fluctuates wildly month to month — a potential fraud signal (energy being diverted or meters being tampered).

---

### 3. Trend Calculation via numpy.polyfit

Time series slope is calculated using `numpy.polyfit(degree=1)` — the slope of the best-fit line through the monthly series:

- **Positive slope** → consumption growing (normal for expanding distributors)
- **Negative slope** → consumption declining (potential fraud or customer loss)
- **Extreme positive slope** → unusual growth (also worth investigating)

A `safe_slope()` wrapper handles edge cases where polyfit fails:

```python
def safe_slope(g):
    try:
        return np.polyfit(range(len(g)), g["default_rate"], 1)[0]
    except Exception:
        return 0.0
```

---

### 4. Pandas Bridge for sklearn

PySpark DataFrames cannot be used directly with scikit-learn. The bridge pattern:

```
Spark DataFrame (6.3M rows) → aggregate → Pandas DataFrame (105 rows) → sklearn
```

`toPandas()` is safe here because the aggregated result is only 105 rows.

---

### 5. Null Filling Strategy

Distributors present in SAMP but not in inadimplência produce nulls after the join. These are filled with `0.0`:

```python
.fillna(0.0)
```

A distributor with no reported default data has no known default risk — zero is the correct signal, not null.

---

### 6. default_consumption_ratio as Combined Signal

A distributor with high default AND high consumption is less concerning than one with high default AND low consumption. The ratio captures this:

```python
.withColumn("default_consumption_ratio",
    F.col("avg_default_rate") / F.col("avg_consumption_kwh"))
```

High ratio → default is large relative to consumption → stronger risk signal.

---

## 📊 Feature Statistics

| Feature | Min | Mean | Max | Notes |
|---|---|---|---|---|
| `avg_consumption_kwh` | 3,026 | 406,113 | 5,626,622 | 1,860x range — StandardScaler essential |
| `cv_consumption` | 0.77 | 4.21 | 130.18 | Max outlier: extreme instability |
| `avg_default_rate` | 0.0 | 12,464 | 121,737 | Clear outlier at 121K |
| `default_consumption_ratio` | 0.0 | 0.017 | 0.106 | Small but meaningful signal |

The wide ranges confirm the need for `StandardScaler` before PCA.

---

## 🎓 Lessons Learned

### 1. Always inspect polymorphic columns before aggregating
`market_value` in SAMP has 25+ distinct meanings. Aggregating without filtering would mix kWh with R$ and consumer counts — producing meaningless averages.

### 2. Coefficient of variation is more informative than raw standard deviation
Raw stddev is correlated with mean — bigger distributors have bigger stddev. CV normalizes by mean, making it comparable across distributors of different sizes.

### 3. The Pandas bridge is acceptable for small aggregated outputs
Converting 6.3M rows to Pandas would crash the driver. Converting 105 rows is trivial. The key is aggregating first, converting second.

### 4. Null filling must be domain-driven
Filling nulls with 0 here is correct because it means "no reported default." Always justify null strategy with domain knowledge.

---

## 🔗 Related Documentation

- [Architecture Overview](architecture.md)
- [Silver Layer Deep Dive](silver_layer.md)
- [Data Dictionary (PT-BR → EN)](data_dictionary.md)
- [Gold Notebook Source](../notebooks/03_gold_features.py)
- [Anomaly Model Notebook](../notebooks/04_anomaly_model.py)
