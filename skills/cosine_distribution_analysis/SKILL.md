# Cosine Similarity Distribution Analysis Skill

## Metadata
- **Skill Name**: Cosine Distribution Analysis
- **Version**: 3.0.0
- **Created**: 2026-05-08
- **Last Updated**: 2026-05-08
- **Language**: SQL (Snowflake View — direct SELECT only)
- **Type**: Analytical View for Time Series & Distribution Analysis

## Objective
Analyzes the distribution of LM cosine similarity scores across pre-defined buckets for S&P 500 10-K/10-Q risk factor filings, with optional fiscal-year filtering. Returns a flat, chart-ready tabular result set whose column names and value ordering are stable across calls so that no transformation is required between query execution and chart rendering.

---

## 🛑 EXECUTION PROCEDURE — FOLLOW EXACTLY

This skill exists to **eliminate response variance**. The agent MUST follow the procedure below verbatim. Do not improvise SQL. Do not fall back to Cortex Analyst. Do not call any stored procedure.

### Step 1 — Classify user intent into one of four cases

| Case | Trigger phrases | Use Query Template |
|------|-----------------|--------------------|
| **A. Single year** | "FY2025", "for 2024", "this year's distribution" | `Q_SINGLE_YEAR` |
| **B. Multi-year comparison** | "compare 2023 vs 2024", "YoY", "across years" | `Q_MULTI_YEAR` |
| **C. All years aggregated** | "overall", "across the whole dataset", "total" | `Q_ALL_YEARS` |
| **D. Year totals only** | "how many filings per year", "annual counts" | `Q_YEAR_TOTALS` |

If the user's intent does not match A–D, ask one clarifying question. Do not invent a fifth path.

### Step 2 — Execute the SQL via the standard query tool
Use the **direct SQL execution tool** (e.g. `run_snowflake_query` or the platform-equivalent text-to-result tool). Submit the template from Step 1 verbatim, with `{YEAR}` / `{YEAR_LIST}` substituted. The view does the bucketing, ordering, and aggregation — there is nothing for the agent to compute downstream.

### Step 3 — Pass results directly to the chart function
Use the matching chart spec from the **Chart Spec Library** below. Field names in the spec are lowercase and match the view's quoted aliases exactly. Do not rename, re-case, or post-process the result columns.

### Step 4 — Narrate the result in ≤4 sentences
State (i) total filing count, (ii) the modal bucket, (iii) the share of filings at similarity ≥ 0.98, (iv) any year-over-year shift if Case B. Do not restate every bucket.

---

## ❌ DO NOT

The following actions caused prior failures and are explicitly forbidden:

1. **Do not call `GET_COSINE_SIMILARITY_DISTRIBUTION` or any stored procedure.** Procedures return JSON-wrapped VARIANT, which the chart tool cannot bind to a tabular schema. SELECT against the view instead.
2. **Do not invoke Cortex Analyst / text-to-SQL** as a fallback. If the chart fails, re-read this skill — the failure is upstream, not a SQL synthesis problem.
3. **Do not rewrite the bucket CASE expression** in the calling query. The bucketing lives in the view; calling queries only filter and project.
4. **Do not alias columns to UPPERCASE or PascalCase** in the calling SELECT. The chart spec field names are lowercase. Preserve the view's casing.
5. **Do not add an `ORDER BY` that breaks ordinal axis ordering.** The view already orders by bucket. If you must re-order, use the `bucket_order` column (see schema).
6. **Do not include the synthetic `'Other'` bucket in charts** unless explicitly asked — it captures floating-point artifacts (similarity > 1.0) and distracts from the true distribution.

---

## Input Parameters
- **fiscal_year** (INT, optional): single year for Case A. Extract from user message (e.g. "FY2025" → `2025`). Reject non-integer or pre-2000 values with a clarification.
- **fiscal_year_list** (LIST[INT], optional): comma-separated list for Case B (e.g. `(2023, 2024, 2025)`).

---

## Output Schema (stable contract)

| Column | Type | Description |
|--------|------|-------------|
| `similarity_bucket` | STRING | Ordered bucket label, e.g. `'[0.90, 0.95)'` |
| `bucket_order` | INT | Integer 1–8 for deterministic ordinal sorting |
| `fiscal_year` | INT | Year extracted from `next_perioddate` |
| `filing_count` | INT | Filings in this (bucket × year) cell — primary Y-axis metric |
| `min_similarity` | FLOAT | Min cosine similarity in cell |
| `max_similarity` | FLOAT | Max cosine similarity in cell |
| `avg_similarity` | FLOAT | Mean cosine similarity in cell |
| `min_next_perioddate` | DATE | Earliest filing date in cell |
| `max_next_perioddate` | DATE | Latest filing date in cell |

All column names are **lowercase** and **quoted** in the view definition to defeat Snowflake's default uppercase folding when results serialize to JSON.

---

## View Definition

```sql
CREATE OR REPLACE VIEW QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION AS
SELECT
  CASE
    WHEN lm_cosine_similarity < 0.90                                     THEN '[0.0, 0.90)'
    WHEN lm_cosine_similarity >= 0.90  AND lm_cosine_similarity < 0.95   THEN '[0.90, 0.95)'
    WHEN lm_cosine_similarity >= 0.95  AND lm_cosine_similarity < 0.97   THEN '[0.95, 0.97)'
    WHEN lm_cosine_similarity >= 0.97  AND lm_cosine_similarity < 0.98   THEN '[0.97, 0.98)'
    WHEN lm_cosine_similarity >= 0.98  AND lm_cosine_similarity < 0.99   THEN '[0.98, 0.99)'
    WHEN lm_cosine_similarity >= 0.99  AND lm_cosine_similarity < 0.995  THEN '[0.99, 0.995)'
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999  THEN '[0.995, 0.999)'
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0   THEN '[0.999, 1.0]'
    ELSE 'Other'
  END                                                                    AS "similarity_bucket",
  CASE
    WHEN lm_cosine_similarity < 0.90                                     THEN 1
    WHEN lm_cosine_similarity >= 0.90  AND lm_cosine_similarity < 0.95   THEN 2
    WHEN lm_cosine_similarity >= 0.95  AND lm_cosine_similarity < 0.97   THEN 3
    WHEN lm_cosine_similarity >= 0.97  AND lm_cosine_similarity < 0.98   THEN 4
    WHEN lm_cosine_similarity >= 0.98  AND lm_cosine_similarity < 0.99   THEN 5
    WHEN lm_cosine_similarity >= 0.99  AND lm_cosine_similarity < 0.995  THEN 6
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999  THEN 7
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0   THEN 8
    ELSE 9
  END                                                                    AS "bucket_order",
  YEAR(next_perioddate)                                                  AS "fiscal_year",
  COUNT(*)                                                               AS "filing_count",
  MIN(lm_cosine_similarity)                                              AS "min_similarity",
  MAX(lm_cosine_similarity)                                              AS "max_similarity",
  AVG(lm_cosine_similarity)                                              AS "avg_similarity",
  MIN(next_perioddate)                                                   AS "min_next_perioddate",
  MAX(next_perioddate)                                                   AS "max_next_perioddate"
FROM QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned
WHERE lm_cosine_similarity IS NOT NULL
  AND next_perioddate <= CURRENT_DATE
GROUP BY 1, 2, 3
ORDER BY "fiscal_year" DESC, "bucket_order" ASC;
```

> **Casing note:** the double-quoted aliases are deliberate. They force Snowflake to emit lowercase column names in the result set, so downstream JSON keys match the chart spec field names without translation.

---

## Query Template Library

Substitute `{YEAR}` and `{YEAR_LIST}` at call time. Do not modify anything else.

### Q_SINGLE_YEAR (Case A)
```sql
SELECT "similarity_bucket", "bucket_order", "fiscal_year",
       "filing_count", "min_similarity", "max_similarity", "avg_similarity"
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE "fiscal_year" = {YEAR}
  AND "similarity_bucket" <> 'Other'
ORDER BY "bucket_order" ASC;
```

### Q_MULTI_YEAR (Case B)
```sql
SELECT "similarity_bucket", "bucket_order", "fiscal_year",
       "filing_count", "avg_similarity"
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE "fiscal_year" IN ({YEAR_LIST})
  AND "similarity_bucket" <> 'Other'
ORDER BY "fiscal_year" DESC, "bucket_order" ASC;
```

### Q_ALL_YEARS (Case C)
```sql
SELECT "similarity_bucket", "bucket_order",
       SUM("filing_count")                                            AS "filing_count",
       MIN("min_similarity")                                          AS "min_similarity",
       MAX("max_similarity")                                          AS "max_similarity",
       SUM("filing_count" * "avg_similarity") / SUM("filing_count")   AS "avg_similarity"
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE "similarity_bucket" <> 'Other'
GROUP BY "similarity_bucket", "bucket_order"
ORDER BY "bucket_order" ASC;
```

### Q_YEAR_TOTALS (Case D)
```sql
SELECT "fiscal_year",
       SUM("filing_count")                                            AS "total_filings",
       SUM("filing_count" * "avg_similarity") / SUM("filing_count")   AS "weighted_avg_similarity"
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE "similarity_bucket" <> 'Other'
GROUP BY "fiscal_year"
ORDER BY "fiscal_year" DESC;
```

---

## Chart Spec Library

Each spec uses an explicit `sort` array on the ordinal axis so bucket order is preserved even if the result rows are reshuffled in transit.

### CHART_A — Single-Year Distribution (paired with `Q_SINGLE_YEAR`)
```json
{
  "mark": "bar",
  "title": "LM Cosine Similarity Distribution — FY {YEAR}",
  "encoding": {
    "x": {
      "field": "similarity_bucket",
      "type": "ordinal",
      "title": "Cosine Similarity Bucket",
      "sort": ["[0.0, 0.90)", "[0.90, 0.95)", "[0.95, 0.97)", "[0.97, 0.98)",
               "[0.98, 0.99)", "[0.99, 0.995)", "[0.995, 0.999)", "[0.999, 1.0]"],
      "axis": {"labelAngle": -45}
    },
    "y": {"field": "filing_count", "type": "quantitative", "title": "Number of Filings"},
    "tooltip": [
      {"field": "similarity_bucket", "type": "ordinal"},
      {"field": "filing_count",      "type": "quantitative"},
      {"field": "avg_similarity",    "type": "quantitative", "format": ".4f"},
      {"field": "min_similarity",    "type": "quantitative", "format": ".4f"},
      {"field": "max_similarity",    "type": "quantitative", "format": ".4f"}
    ]
  }
}
```

### CHART_B — Multi-Year Grouped Bars (paired with `Q_MULTI_YEAR`)
```json
{
  "mark": "bar",
  "title": "YoY Cosine Similarity Distribution",
  "encoding": {
    "x": {
      "field": "similarity_bucket",
      "type": "ordinal",
      "sort": ["[0.0, 0.90)", "[0.90, 0.95)", "[0.95, 0.97)", "[0.97, 0.98)",
               "[0.98, 0.99)", "[0.99, 0.995)", "[0.995, 0.999)", "[0.999, 1.0]"],
      "axis": {"labelAngle": -45}
    },
    "y":     {"field": "filing_count", "type": "quantitative"},
    "xOffset":{"field": "fiscal_year", "type": "nominal"},
    "color": {"field": "fiscal_year",  "type": "nominal", "title": "Fiscal Year"},
    "tooltip": [
      {"field": "fiscal_year",       "type": "nominal"},
      {"field": "similarity_bucket", "type": "ordinal"},
      {"field": "filing_count",      "type": "quantitative"},
      {"field": "avg_similarity",    "type": "quantitative", "format": ".4f"}
    ]
  }
}
```

### CHART_C — All-Years Aggregate (paired with `Q_ALL_YEARS`)
Same shape as `CHART_A`, drop the `{YEAR}` from the title.

### CHART_D — Annual Filing Totals (paired with `Q_YEAR_TOTALS`)
```json
{
  "mark": {"type": "bar"},
  "title": "Filings per Fiscal Year",
  "encoding": {
    "x": {"field": "fiscal_year",    "type": "ordinal", "title": "Fiscal Year"},
    "y": {"field": "total_filings",  "type": "quantitative", "title": "Total Filings"},
    "tooltip": [
      {"field": "fiscal_year",            "type": "ordinal"},
      {"field": "total_filings",          "type": "quantitative"},
      {"field": "weighted_avg_similarity","type": "quantitative", "format": ".4f"}
    ]
  }
}
```

---

## Bucket Reference

| Bucket | Range | Interpretation (re: YoY 10-K risk factor language) |
|--------|-------|-----------------------------------------------------|
| `[0.0, 0.90)`   | < 0.90       | Material rewrite of risk disclosure |
| `[0.90, 0.95)`  | [0.90, 0.95) | Substantial language change |
| `[0.95, 0.97)`  | [0.95, 0.97) | Moderate revision |
| `[0.97, 0.98)`  | [0.97, 0.98) | Light revision |
| `[0.98, 0.99)`  | [0.98, 0.99) | Minor edits |
| `[0.99, 0.995)` | [0.99, 0.995)| Minimal edits |
| `[0.995, 0.999)`| [0.995, 0.999)| Near-identical language |
| `[0.999, 1.0]`  | [0.999, 1.0] | Effectively boilerplate copy-paste |

The bucket boundaries align with the original Lazy Prices (Cohen, Malloy, Nguyen 2020) tail-of-distribution regions — `[0.0, 0.95)` is the "low cosine" treatment group, `[0.999, 1.0]` is the control.

---

## Source Table

- **Database:** `QRSLLM_POC_DB`
- **Schema:** `HENRY_SCHEMA`
- **Table:** `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`
- **Required columns:** `next_perioddate` (DATE), `lm_cosine_similarity` (FLOAT)

---

## Failure Recovery

If the chart tool returns an error after a SELECT against the view:

1. **Check the result schema first** — confirm the result has the lowercase columns listed in *Output Schema*. If they came back UPPERCASE, the view was rebuilt without quoted aliases; re-run the `CREATE OR REPLACE VIEW` DDL above.
2. **Confirm row count > 0** — an empty result means the year filter excluded everything; verify `{YEAR}` is in `[2001, current_year]`.
3. **Do NOT escape to Cortex Analyst.** Re-running the same skill with the same query is the correct retry. If the second attempt also fails, surface the raw result table to the user and explain that the chart layer is the failure point, not the SQL.

---

## Changelog

- **3.0.0** (2026-05-08): Added rigid execution procedure, forbidden-action list, deterministic `bucket_order` column, quoted lowercase aliases, intent → query → chart spec lookup. Removed ambiguous reference to `GET_COSINE_SIMILARITY_DISTRIBUTION` procedure. Removed pie-chart spec (low information density at 3-year scale).
- **2.1.0** (prior): Initial dynamic-year filtering version.
