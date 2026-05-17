# 📖 Data Dictionary (PT-BR → EN)

This document maps ANEEL's original Portuguese column names (preserved in the Bronze layer) to their English equivalents (applied in the Silver layer).

---

## 🎯 Translation Strategy

The Bronze layer **preserves** ANEEL's original Portuguese column names for full traceability to the official data dictionary. Translation to English (snake_case) is applied in the **Silver layer**, where data is also cleaned and validated.

This approach provides:
- ✅ **Lineage:** any Bronze column maps directly to ANEEL's official dictionary
- ✅ **Standardization:** Silver+ uses consistent English naming for analytics
- ✅ **Separation of concerns:** Bronze preserves; Silver standardizes

---

## 📊 SAMP Dataset

**Source:** [SAMP - Sistema de Acompanhamento de Informações de Mercado para Regulação Econômica](https://dadosabertos.aneel.gov.br/dataset/samp-sistema-de-acompanhamento-de-informacoes-de-mercado-para-regulacao-economica)

**Regulation:** ANEEL Resolution Nº 1003/2022

### Column Mapping

| PT-BR (Bronze) | EN (Silver+) | Type (Silver) | Description |
|---|---|---|---|
| `DatGeracaoConjuntoDados` | `dataset_generation_date` | date | Date when ANEEL generated this dataset version |
| `NumCNPJAgenteDistribuidora` | `distributor_cnpj` | string | Distributor's CNPJ (Brazilian tax ID) |
| `SigAgenteDistribuidora` | `distributor_code` | string | Distributor's short code (e.g., COPEL-DIS) |
| `NomAgenteDistribuidora` | `distributor_name` | string | Distributor's full legal name |
| `NomTipoMercado` | `market_type` | string | Market type (Regular, Sistema de Compensação GD, etc.) |
| `DscModalidadeTarifaria` | `tariff_modality` | string | Tariff modality (Branca, Verde, Azul, Convencional) |
| `DscSubGrupoTarifario` | `tariff_subgroup` | string | Tariff subgroup code (A1-A4, B1-B4) |
| `DscClasseConsumoMercado` | `consumption_class` | string | Consumer class (Industrial, Residencial, Comercial, etc.) |
| `DscSubClasseConsumidor` | `consumer_subclass` | string | Consumer subclass refinement |
| `DscDetalheConsumidor` | `consumer_detail` | string | Additional consumer detail |
| `IdeAgenteAcessante` | `accessing_agent_id` | string | Accessing agent identifier (free market) |
| `NumCNPJAgenteAcessante` | `accessing_agent_cnpj` | string | Accessing agent CNPJ |
| `NomAgenteAcessante` | `accessing_agent_name` | string | Accessing agent name |
| `DscPostoTarifario` | `tariff_period` | string | Tariff period (Ponta, Fora ponta, Não se aplica) |
| `DscOpcaoEnergia` | `energy_option` | string | Energy option (CATIVO = regulated market) |
| `DscDetalheMercado` | `market_detail` | string | What VlrMercado represents (consumers, MWh, R$, etc.) |
| `DatCompetencia` | `reference_date` | date | Month/year of reference (first day of month) |
| `VlrMercado` | `market_value` | double | The actual metric value (meaning depends on market_detail) |

### Derived Columns (added in Silver)

| Column | Type | Logic |
|---|---|---|
| `reference_year` | integer | `F.year(reference_date)` |
| `reference_month` | integer | `F.month(reference_date)` |

### Audit Metadata (added in Bronze)

| Column | Type | Description |
|---|---|---|
| `_ingestion_timestamp` | timestamp | When the row was ingested |
| `_source_file` | string | Original CSV file path |
| `_ingestion_date` | date | Date of ingestion |
| `_partition_year` | integer | Year extracted from DatCompetencia (Bronze partition column) |

---

## 🚨 Important Notes on SAMP Data

### `VlrMercado` is Polymorphic

The `VlrMercado` column **changes meaning** based on `DscDetalheMercado`:

| `DscDetalheMercado` | What `VlrMercado` represents |
|---|---|
| "Número de consumidores" | Count of consumers |
| "Consumo (MWh)" | Energy consumption in MWh |
| "Receita (R$)" | Revenue in Brazilian Reais |
| "PIS/PASEP (R$)" | Tax amount in Brazilian Reais |
| "ICMS (R$)" | Tax amount in Brazilian Reais |
| "Bandeiras (R$)" | Tariff flag amount |

**Implication for analytics:** The Gold layer should **pivot** this column to separate metrics. For example:
```
distributor_code | reference_date | num_consumers | consumption_mwh | revenue_brl
COPEL-DIS        | 2024-01-01     | 4,500,000     | 1,200,000       | 850,000,000
```

### Brazilian Decimal Convention

The `VlrMercado` values use **comma as decimal separator** (Brazilian convention):
- `76,000000` represents `76.0`
- `266754,610000` represents `266,754.61`

Silver layer transformation handles this:
```python
F.regexp_replace(F.col("VlrMercado"), ",", ".").cast("double")
```

### CNPJ Identifiers

Both `NumCNPJAgenteDistribuidora` and `NumCNPJAgenteAcessante` are **identifiers, not numbers**:
- They have a fixed format (14 digits)
- May contain leading zeros (`00123456000189`)
- Should never be cast to numeric types (loses formatting)
- Used for joins and lookups, not arithmetic

---

## 📊 Indqual Dataset (Inadimplência)

**Source:** [Indqual - Inadimplência](https://dadosabertos.aneel.gov.br/)

**Files:**
- `inadimplencia.csv` — Default indicators by distributor (~58 MB, 1.1M rows)
- `dominio-indicadores.csv` — Dimension table with indicator descriptions (~40 KB)

**Coverage:** 2012–2026 (monthly)

### inadimplencia.csv — Column Mapping

| PT-BR (Bronze) | EN (Silver+) | Type (Silver) | Description |
|---|---|---|---|
| `DatGeracaoConjuntoDados` | `dataset_generation_date` | date | Date when ANEEL generated this dataset (format: `dd-MM-yyyy`) |
| `SigAgente` | `distributor_code` | string | Distributor's short code |
| `NumCNPJ` | `distributor_cnpj` | string | Distributor's CNPJ (Brazilian tax ID) |
| `SigIndicador` | `indicator_code` | string | Indicator code — FK to `dominio_indicadores` |
| `AnoIndice` | `reference_year` | integer | Reference year |
| `NumPeriodoIndice` | `reference_month` | integer | Reference month (1–12) |
| `VlrIndiceEnviado` | `indicator_value` | double | Indicator value (comma as decimal separator in source) |

### Derived Columns (added in Silver)

| Column | Type | Logic |
|---|---|---|
| `reference_date` | date | Constructed as first day of month: `concat(AnoIndice, "-", lpad(NumPeriodoIndice, 2, "0"), "-01")` |

> ⚠️ **Note:** `inadimplencia.csv` has no single date column. `reference_date` is built from `AnoIndice` + `NumPeriodoIndice`. The source `DatGeracaoConjuntoDados` uses `dd-MM-yyyy` format (different from SAMP which uses `yyyy-MM-dd`).

### Enriched Column (from join with dominio_indicadores)

| Column | Type | Source | Description |
|---|---|---|---|
| `indicator_description` | string | `dominio_indicadores.DscIndicador` | Full Portuguese description of the indicator |

### dominio-indicadores.csv — Column Mapping

This is a **dimension/lookup table** used only to enrich `inadimplencia` via broadcast join in Silver. It does not produce its own Silver table.

| PT-BR (Bronze) | Used in Silver as | Description |
|---|---|---|
| `SigIndicador` | `indicator_code` (join key) | Indicator code |
| `DscIndicador` | `indicator_description` | Full description of the indicator |
| `DatGeracaoConjuntoDados` | *(dropped)* | Dataset generation date |

### Example Indicators

| `indicator_code` | `indicator_description` |
|---|---|
| `ITotCrt` | Indicador total de crédito |
| `ITot12` | Perc. da rec. fat. 12° mês ant. não receb. no mês de ref. todos os consumidores |
| `ITot21` | Perc. da rec. fat. 21° mês ant. não receb. no mês de ref. todos os consumidores |
| `ITot24` | Perc. da rec. fat. 24° mês ant. não receb. no mês de ref. todos os consumidores |
| `CMM` | Consumo médio mensal, em MWh |
| `AREA` | Área do conj., expressa em km2 |

---

## 🔗 Related Documentation

- [Architecture & Design Decisions](architecture.md)
- [Bronze Layer Deep Dive](bronze_layer.md)
- [Silver Layer Deep Dive](silver_layer.md)
- [ANEEL Official Data Dictionary (PDF)](https://dadosabertos.aneel.gov.br/dataset/samp-sistema-de-acompanhamento-de-informacoes-de-mercado-para-regulacao-economica)

---

## 📚 References

- ANEEL Resolution Nº 1003/2022
- Brazilian tariff classification standards (Subgrupos A1-A4 e B1-B4)
- Regulated Market (Mercado Cativo) vs Free Market (Mercado Livre) frameworks
