---
name: Cosine Similarity Distribution Analysis Skill
description: Analyzes the distribution of LM cosine similarity scores across pre-defined buckets for S&P 500 10-K/10-Q risk factor filings, with optional fiscal-year filtering. Designed for a Cortex-Analyst-only execution environment — the skill steers Cortex Analyst to produce a stable, chart-ready SQL output via prescriptive natural-language prompts.
---

## Objective
Returns a flat, chart-ready tabular result set (similarity buckets × fiscal year × filing counts) for the Lazy-Prices–style YoY risk-factor change analysis. **Execution runs through Cortex Analyst**, so this skill's primary job is to constrain Cortex Analyst's text-to-SQL output so that bucket boundaries, column projections, year filters, and ordering are deterministic across calls.

---

## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY

### Step 1 — Classify user intent

| Case | Trigger phrases | Use Prompt | Use Chart Spec |
|------|-----------------|------------|----------------|
| **A. Single year** | "FY2025", "for 2024", "this year's distribution" | `PROMPT_A` | `CHART_A` |
| **B. Multi-year comparison** | "compare 2023 vs 2024", "YoY", "across years" | `PROMPT_B` | `CHART_B` |
| **C. All years aggregated** | "overall", "across the whole dataset", "total" | `PROMPT_C` | `CHART_C` |
| **D. Year totals only** | "how many filings per year", "annual counts" | `PROMPT_D` | `CHART_D` |

If user intent does not match A–D, ask one clarifying question. Do not invent a fifth path.

### Step 2 — Send the literal prompt string to Cortex Analyst
Pass the matching prompt from the **Cortex Analyst Prompt Library** below to Cortex Analyst **verbatim**, with `{YEAR}` / `{YEAR_LIST}` substituted. Do not paraphrase. Do not "improve" the prompt. The exact phrasing has been tuned to coerce Cortex Analyst into producing SQL that conforms to the *Expected SQL Contracts* section.

### Step 3 — Validate Cortex Analyst's generated SQL before accepting the result
Compare the SQL Cortex Analyst returns against the **Expected SQL Contract** for that case. The SQL must:

- Reference `V_GET_COSINE_SIMILARITY_DISTRIBUTION` (the view), **not** the base table.
- **Not** contain a `CASE WHEN lm_cosine_similarity ...` expression. If it does, Cortex Analyst regenerated the bucketing — re-issue the prompt with the prefix `"Use the view's existing similarity_bucket column. Do not recompute buckets."` prepended.
- Contain the year filter exactly: `WHERE FISCAL_YEAR = {YEAR}` (Case A) or `WHERE FISCAL_YEAR IN ({YEAR_LIST})` (Case B).
- Contain `WHERE SIMILARITY_BUCKET <> 'Other'` (or equivalent) to exclude floating-point artifacts.
- Order by `BUCKET_ORDER ASC` (and `FISCAL_YEAR DESC` for Case B).

If validation fails twice in a row, surface the raw result table to the user with a note that the chart layer is the failure point — do not retry indefinitely.

### Step 4 — Pass the result directly to the chart tool
Use the matching `CHART_*` spec from the *Chart Spec Library*. Field names in the spec are **UPPERCASE** to match Cortex Analyst's default Snowflake column-name folding. Do not rename, re-case, or post-process columns.

### Step 5 — Narrate in ≤ 4 sentences
State (i) total filing count, (ii) the modal bucket, (iii) the share of filings at similarity ≥ 0.98, (iv) any year-over-year shift if Case B. Do not enumerate every bucket.

---

## 🗣️ CORTEX ANALYST PROMPT LIBRARY

Each prompt is a **literal string** to send to Cortex Analyst. Do not edit beyond `{YEAR}` / `{YEAR_LIST}` substitution.

### PROMPT_A — Single-Year Distribution
```
Query the view QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION.
Return columns: SIMILARITY_BUCKET, BUCKET_ORDER, FISCAL_YEAR, FILING_COUNT,
MIN_SIMILARITY, MAX_SIMILARITY, AVG_SIMILARITY.
Filter to fiscal_year = {YEAR} and exclude rows where similarity_bucket = 'Other'.
Order by BUCKET_ORDER ascending.
Do not recompute the similarity bucket — use the view's existing similarity_bucket column verbatim.
Do not query the base table master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned directly.
```

### PROMPT_B — Multi-Year Comparison
```
Query the view QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION.
Return columns: SIMILARITY_BUCKET, BUCKET_ORDER, FISCAL_YEAR, FILING_COUNT, AVG_SIMILARITY.
Filter to fiscal_year IN ({YEAR_LIST}) and exclude rows where similarity_bucket = 'Other'.
Order by FISCAL_YEAR descending, then BUCKET_ORDER ascending.
Do not recompute the similarity bucket — use the view's existing similarity_bucket column verbatim.
Do not query the base table directly.
```

### PROMPT_C — All-Years Aggregate
```
Query the view QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION.
Aggregate across all fiscal years. Return columns: SIMILARITY_BUCKET, BUCKET_ORDER,
FILING_COUNT (as SUM of filing_count), MIN_SIMILARITY (as MIN), MAX_SIMILARITY (as MAX),
AVG_SIMILARITY (as filing-count-weighted average:
  SUM(filing_count * avg_similarity) / SUM(filing_count) ).
Exclude rows where similarity_bucket = 'Other'.
Group by SIMILARITY_BUCKET and BUCKET_ORDER. Order by BUCKET_ORDER ascending.
Do not recompute the similarity bucket. Do not query the base table directly.
```

### PROMPT_D — Annual Filing Totals
```
Query the view QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION.
Return columns: FISCAL_YEAR, TOTAL_FILINGS (as SUM of filing_count),
WEIGHTED_AVG_SIMILARITY (as SUM(filing_count * avg_similarity) / SUM(filing_count)).
Exclude rows where similarity_bucket = 'Other'.
Group by FISCAL_YEAR. Order by FISCAL_YEAR descending.
Do not query the base table directly.
```

### Prompt construction rules (why these prompts are written this way)

| Element | Purpose |
|---------|---------|
| Lead with the fully-qualified view name | Forces Cortex Analyst to bind to the view, not retrieve via the base table. |
| Explicit column list | Pins projection — prevents `SELECT *` or column drift. |
| `Do not recompute the similarity bucket` | Stops Cortex Analyst from re-emitting its own `CASE WHEN lm_cosine_similarity < 0.9 THEN 'below 0.9' ...` boilerplate (the v2 failure mode). |
| `Do not query the base table directly` | Closes the only other path Cortex Analyst would take to invent buckets. |
| Explicit filter and ordering | Removes degrees of freedom; reduces variance to near zero. |

---

## ✅ EXPECTED SQL CONTRACTS

These are the SQL strings Cortex Analyst should produce. Use them to validate Step 3.

### Contract A — produced by `PROMPT_A`
```sql
SELECT SIMILARITY_BUCKET, BUCKET_ORDER, FISCAL_YEAR,
       FILING_COUNT, MIN_SIMILARITY, MAX_SIMILARITY, AVG_SIMILARITY
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE FISCAL_YEAR = {YEAR}
  AND SIMILARITY_BUCKET <> 'Other'
ORDER BY BUCKET_ORDER ASC;
```

### Contract B — produced by `PROMPT_B`
```sql
SELECT SIMILARITY_BUCKET, BUCKET_ORDER, FISCAL_YEAR,
       FILING_COUNT, AVG_SIMILARITY
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE FISCAL_YEAR IN ({YEAR_LIST})
  AND SIMILARITY_BUCKET <> 'Other'
ORDER BY FISCAL_YEAR DESC, BUCKET_ORDER ASC;
```

### Contract C — produced by `PROMPT_C`
```sql
SELECT SIMILARITY_BUCKET, BUCKET_ORDER,
       SUM(FILING_COUNT)                                      AS FILING_COUNT,
       MIN(MIN_SIMILARITY)                                    AS MIN_SIMILARITY,
       MAX(MAX_SIMILARITY)                                    AS MAX_SIMILARITY,
       SUM(FILING_COUNT * AVG_SIMILARITY) / SUM(FILING_COUNT) AS AVG_SIMILARITY
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE SIMILARITY_BUCKET <> 'Other'
GROUP BY SIMILARITY_BUCKET, BUCKET_ORDER
ORDER BY BUCKET_ORDER ASC;
```

### Contract D — produced by `PROMPT_D`
```sql
SELECT FISCAL_YEAR,
       SUM(FILING_COUNT)                                      AS TOTAL_FILINGS,
       SUM(FILING_COUNT * AVG_SIMILARITY) / SUM(FILING_COUNT) AS WEIGHTED_AVG_SIMILARITY
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE SIMILARITY_BUCKET <> 'Other'
GROUP BY FISCAL_YEAR
ORDER BY FISCAL_YEAR DESC;
```

> **Validation rule:** Cortex Analyst output is acceptable if it is *semantically equivalent* to the contract — exact whitespace and CTE wrapping (e.g. `WITH __view AS (...)`) is fine. The prohibited deviations are: querying the base table, regenerating bucket `CASE` logic, omitting the `'Other'` filter, or returning columns outside the contract.

---

## ❌ DO NOT

1. **Do not call any stored procedure**, including `GET_COSINE_SIMILARITY_DISTRIBUTION`. Procedures emit JSON-wrapped VARIANT, which the chart tool cannot bind to a tabular schema.
2. **Do not rephrase the prompts in the *Cortex Analyst Prompt Library*.** Their wording is load-bearing.
3. **Do not let Cortex Analyst query the base table** `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned` directly. The view is the only sanctioned entry point because it owns the bucket definition.
4. **Do not accept SQL that regenerates the bucket `CASE` expression.** Re-issue the prompt with `"Use the view's existing similarity_bucket column. Do not recompute buckets."` prepended.
5. **Do not include the synthetic `'Other'` bucket** in charts. It captures floating-point artifacts (similarity > 1.0) that distract from the true distribution.
6. **Do not lowercase or quote-wrap column names** in the chart spec. Cortex Analyst returns Snowflake-default UPPERCASE; chart specs must match.

---

## Output Schema (stable contract)

| Column | Type | Description |
|--------|------|-------------|
| `SIMILARITY_BUCKET` | STRING | Ordered bucket label, e.g. `'[0.90, 0.95)'` |
| `BUCKET_ORDER` | INT | Integer 1–8 for deterministic ordinal sorting |
| `FISCAL_YEAR` | INT | Year extracted from `next_perioddate` |
| `FILING_COUNT` | INT | Filings in this (bucket × year) cell |
| `MIN_SIMILARITY` | FLOAT | Min cosine similarity in cell |
| `MAX_SIMILARITY` | FLOAT | Max cosine similarity in cell |
| `AVG_SIMILARITY` | FLOAT | Mean cosine similarity in cell |
| `MIN_NEXT_PERIODDATE` | DATE | Earliest filing date in cell |
| `MAX_NEXT_PERIODDATE` | DATE | Latest filing date in cell |

Casing is **UPPERCASE** because Cortex Analyst returns columns under Snowflake's default unquoted-identifier folding rules.

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
  END                                                                    AS similarity_bucket,
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
  END                                                                    AS bucket_order,
  YEAR(next_perioddate)                                                  AS fiscal_year,
  COUNT(*)                                                               AS filing_count,
  MIN(lm_cosine_similarity)                                              AS min_similarity,
  MAX(lm_cosine_similarity)                                              AS max_similarity,
  AVG(lm_cosine_similarity)                                              AS avg_similarity,
  MIN(next_perioddate)                                                   AS min_next_perioddate,
  MAX(next_perioddate)                                                   AS max_next_perioddate
FROM QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned
WHERE lm_cosine_similarity IS NOT NULL
  AND next_perioddate <= CURRENT_DATE
GROUP BY 1, 2, 3
ORDER BY fiscal_year DESC, bucket_order ASC;
```

> Aliases are unquoted so Snowflake folds them to UPPERCASE. This matches Cortex Analyst's default identifier behaviour.

---

## Chart Spec Library

Each spec uses an explicit `sort` array on the ordinal axis so bucket order is preserved even if rows are reshuffled in transit.

### CHART_A — Single-Year Distribution (paired with `Contract A`)
```json
{
  "mark": "bar",
  "title": "LM Cosine Similarity Distribution — FY {YEAR}",
  "encoding": {
    "x": {
      "field": "SIMILARITY_BUCKET",
      "type": "ordinal",
      "title": "Cosine Similarity Bucket",
      "sort": ["[0.0, 0.90)", "[0.90, 0.95)", "[0.95, 0.97)", "[0.97, 0.98)",
               "[0.98, 0.99)", "[0.99, 0.995)", "[0.995, 0.999)", "[0.999, 1.0]"],
      "axis": {"labelAngle": -45}
    },
    "y": {"field": "FILING_COUNT", "type": "quantitative", "title": "Number of Filings"},
    "tooltip": [
      {"field": "SIMILARITY_BUCKET", "type": "ordinal"},
      {"field": "FILING_COUNT",      "type": "quantitative"},
      {"field": "AVG_SIMILARITY",    "type": "quantitative", "format": ".4f"},
      {"field": "MIN_SIMILARITY",    "type": "quantitative", "format": ".4f"},
      {"field": "MAX_SIMILARITY",    "type": "quantitative", "format": ".4f"}
    ]
  }
}
```

### CHART_B — Multi-Year Grouped Bars (paired with `Contract B`)
```json
{
  "mark": "bar",
  "title": "YoY Cosine Similarity Distribution",
  "encoding": {
    "x": {
      "field": "SIMILARITY_BUCKET",
      "type": "ordinal",
      "sort": ["[0.0, 0.90)", "[0.90, 0.95)", "[0.95, 0.97)", "[0.97, 0.98)",
               "[0.98, 0.99)", "[0.99, 0.995)", "[0.995, 0.999)", "[0.999, 1.0]"],
      "axis": {"labelAngle": -45}
    },
    "y":      {"field": "FILING_COUNT", "type": "quantitative"},
    "xOffset":{"field": "FISCAL_YEAR",  "type": "nominal"},
    "color":  {"field": "FISCAL_YEAR",  "type": "nominal", "title": "Fiscal Year"},
    "tooltip": [
      {"field": "FISCAL_YEAR",       "type": "nominal"},
      {"field": "SIMILARITY_BUCKET", "type": "ordinal"},
      {"field": "FILING_COUNT",      "type": "quantitative"},
      {"field": "AVG_SIMILARITY",    "type": "quantitative", "format": ".4f"}
    ]
  }
}
```

### CHART_C — All-Years Aggregate (paired with `Contract C`)
Same shape as `CHART_A`, drop the `{YEAR}` from the title.

### CHART_D — Annual Filing Totals (paired with `Contract D`)
```json
{
  "mark": "bar",
  "title": "Filings per Fiscal Year",
  "encoding": {
    "x": {"field": "FISCAL_YEAR",   "type": "ordinal", "title": "Fiscal Year"},
    "y": {"field": "TOTAL_FILINGS", "type": "quantitative", "title": "Total Filings"},
    "tooltip": [
      {"field": "FISCAL_YEAR",             "type": "ordinal"},
      {"field": "TOTAL_FILINGS",           "type": "quantitative"},
      {"field": "WEIGHTED_AVG_SIMILARITY", "type": "quantitative", "format": ".4f"}
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

Bucket boundaries align with Cohen, Malloy & Nguyen (2020) Lazy Prices tail regions — `[0.0, 0.95)` is the "low cosine" treatment group, `[0.999, 1.0]` is the control.

---

## 🛠️ Semantic Model Setup (Required for Cortex Analyst Reliability)

The skill alone is not sufficient — Cortex Analyst's behaviour also depends on its semantic model YAML. To make the prompts in this skill produce stable output, the semantic model must:

1. **Register `V_GET_COSINE_SIMILARITY_DISTRIBUTION` as a logical table** with all nine columns documented and synonyms attached:
   - `fiscal_year`: synonyms `["year", "FY", "fiscal year", "filing year"]`
   - `similarity_bucket`: synonyms `["bucket", "similarity range", "cosine bucket"]`
   - `filing_count`: synonyms `["count", "number of filings", "filings"]`
   - `avg_similarity`: synonyms `["average similarity", "mean cosine"]`

2. **Hide the base table `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`** from the semantic model (or mark it as `hidden: true`). This is the single most important step — without it, Cortex Analyst will sometimes route around the view and regenerate the bucket `CASE`. The view should be the only sanctioned source for cosine-similarity distribution analysis.

3. **Register each contract in the *Expected SQL Contracts* section as a `verified_query`** in the semantic model. Cortex Analyst preferentially returns verified queries for matching questions, which makes the deterministic path the cheapest path.

   Example YAML stub:
   ```yaml
   verified_queries:
     - name: cosine_distribution_single_year
       question: "What is the cosine similarity distribution for fiscal year {YEAR}?"
       sql: |
         SELECT SIMILARITY_BUCKET, BUCKET_ORDER, FISCAL_YEAR,
                FILING_COUNT, MIN_SIMILARITY, MAX_SIMILARITY, AVG_SIMILARITY
         FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
         WHERE FISCAL_YEAR = {YEAR}
           AND SIMILARITY_BUCKET <> 'Other'
         ORDER BY BUCKET_ORDER ASC;
   ```

4. **Drop any stored procedure** named `GET_COSINE_SIMILARITY_DISTRIBUTION` if it still exists, or at minimum remove it from agent-visible tools. Its presence was the original source of the JSON-wrapped VARIANT failure.

---

## Source Table

- **Database:** `QRSLLM_POC_DB`
- **Schema:** `HENRY_SCHEMA`
- **Table:** `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`
- **Required columns:** `next_perioddate` (DATE), `lm_cosine_similarity` (FLOAT)

Direct SQL access is reserved for the view definition above. The skill's prompts forbid Cortex Analyst from querying the base table.

---

## Failure Recovery

| Symptom | Diagnosis | Action |
|---------|-----------|--------|
| Cortex Analyst SQL contains `CASE WHEN lm_cosine_similarity ...` | Cortex regenerated bucketing instead of using the view | Re-issue the prompt with `"Use the view's existing similarity_bucket column. Do not recompute buckets."` prepended. |
| Cortex Analyst SQL references the base table directly | Semantic model has not hidden the base table | Re-issue the prompt with `"Query the view, not the base table."` prepended. Flag to user that semantic model needs `hidden: true` on the base table. |
| Result columns are lowercase | Cortex Analyst added quoted lowercase aliases | Update chart spec field names to lowercase for this single response. Long-term fix: tighten the prompt to remove any aliasing instructions. |
| Result row count is 0 | Year filter excluded everything | Verify `{YEAR}` is in `[2001, current_year]` and that filings have been ingested for that year. |
| Chart still fails after a valid SQL contract | Failure is in the chart layer, not SQL | Surface the raw result table to the user with a one-line note. Do not retry more than twice. |

---
