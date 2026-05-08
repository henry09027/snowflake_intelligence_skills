---
name: Cosine Similarity Distribution Analysis Skill
description: Analyzes the distribution of LM cosine similarity scores across pre-defined buckets for S&P 500 10-K/10-Q risk factor filings, with optional fiscal-year filtering. Designed for a Cortex-Analyst-only execution environment — the skill steers Cortex Analyst to produce a stable, chart-ready SQL output via prescriptive natural-language prompts.
---

## Objective
Returns a flat, chart-ready tabular result set (similarity buckets × fiscal year × filing counts) for the Lazy-Prices–style YoY risk-factor change analysis. **Execution runs through Cortex Analyst**, so this skill's primary job is to constrain Cortex Analyst's text-to-SQL output so that bucket boundaries, column projections, year filters, and ordering are deterministic across calls.

---

## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY

### Step 1 — Send the literal prompt string to Cortex Analyst
Pass the matching prompt from the **Cortex Analyst Prompt Library** below to Cortex Analyst **verbatim**, with `{YEAR}` / `{YEAR_LIST}` substituted. Do not paraphrase. Do not "improve" the prompt. The exact phrasing has been tuned to coerce Cortex Analyst into producing SQL that conforms to the *Expected SQL Contracts* section.

### Step 2 — Pass the result directly to the chart tool
Use the spec from the *Chart Spec Library*. Field names in the spec are **UPPERCASE** to match Cortex Analyst's default Snowflake column-name folding. Do not rename, re-case, or post-process columns.

### Step 3 — Narrate in ≤ 4 sentences
State (i) total filing count, (ii) the modal bucket, (iii) the share of filings at similarity ≥ 0.98, (iv) any year-over-year shift if Case B. Do not enumerate every bucket.

---+

## 🗣️ CORTEX ANALYST PROMPT LIBRARY

```
Query the table:
  QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned

Return exactly these columns, in this order:
  SIMILARITY_BUCKET, BUCKET_ORDER, FISCAL_YEAR,
  FILING_COUNT, MIN_SIMILARITY, MAX_SIMILARITY, AVG_SIMILARITY

Definitions (use these exactly — do not substitute other aggregations or source columns):
  - SIMILARITY_BUCKET  = the view's existing similarity_bucket column, used verbatim. Do NOT recompute or rebucket.
CASE
    WHEN lm_cosine_similarity < 0.90
    THEN 'below 0.90'
    WHEN lm_cosine_similarity >= 0.90 AND lm_cosine_similarity < 0.95
    THEN '0.90-0.95'
    WHEN lm_cosine_similarity >= 0.95 AND lm_cosine_similarity < 0.97
    THEN '0.95-0.97'
    WHEN lm_cosine_similarity >= 0.97 AND lm_cosine_similarity < 0.98
    THEN '0.97-0.98'
    WHEN lm_cosine_similarity >= 0.98 AND lm_cosine_similarity < 0.99
    THEN '0.98-0.99'
    WHEN lm_cosine_similarity >= 0.99 AND lm_cosine_similarity < 0.995
    THEN '0.99-0.995'
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999
    THEN '0.995-0.999'
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0
    THEN '0.999-1.0'
    ELSE 'other'
  END AS similarity_bucket,
  - BUCKET_ORDER       = the view's existing bucket_order column, used verbatim.
CASE
    WHEN lm_cosine_similarity < 0.90
    THEN 1
    WHEN lm_cosine_similarity >= 0.90 AND lm_cosine_similarity < 0.95
    THEN 2
    WHEN lm_cosine_similarity >= 0.95 AND lm_cosine_similarity < 0.97
    THEN 3
    WHEN lm_cosine_similarity >= 0.97 AND lm_cosine_similarity < 0.98
    THEN 4
    WHEN lm_cosine_similarity >= 0.98 AND lm_cosine_similarity < 0.99
    THEN 5
    WHEN lm_cosine_similarity >= 0.99 AND lm_cosine_similarity < 0.995
    THEN 6
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999
    THEN 7
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0
    THEN 8
  ELSE 9 END AS BUCKET_ORDER,
  - FISCAL_YEAR        = DATE_PART('year', next_perioddate)
  - FILING_COUNT       = COUNT(*)
  - MIN_SIMILARITY     = MIN(lm_cosine_similarity)
  - MAX_SIMILARITY     = MAX(lm_cosine_similarity)
  - AVG_SIMILARITY     = AVG(lm_cosine_similarity)

Filters (apply ALL of the following in the WHERE clause):
  - DATE_PART('year', next_perioddate) = {YEAR}
  - lm_cosine_similarity IS NOT NULL
  - next_perioddate <= CURRENT_DATE
  - similarity_bucket <> 'Other'

Grouping:
  - GROUP BY SIMILARITY_BUCKET, BUCKET_ORDER, FISCAL_YEAR
  - One row per similarity_bucket. Do NOT return ungrouped/filing-level rows.

Ordering:
  - ORDER BY BUCKET_ORDER ASC
  - Do NOT order by SIMILARITY_BUCKET (the bucket labels are strings and sort incorrectly alphabetically — e.g. '0.999-1.0' would appear before '0.99-0.995').

Other constraints:
  - Return all matching buckets. Do NOT append a LIMIT clause.
  - Do NOT add any columns beyond the seven listed above.
  - Do NOT add HAVING filters unless explicitly requested.
  - Treat all column identifiers as unquoted (case-insensitive) Snowflake identifiers.
```

---

## ❌ DO NOT

1. **Do not call any stored procedure. Procedures emit JSON-wrapped VARIANT, which the chart tool cannot bind to a tabular schema.
2. **Do not rephrase the prompts in the *Cortex Analyst Prompt Library*.** Their wording is load-bearing.
3. **Do not include the synthetic `'Other'` bucket** in charts. It captures floating-point artifacts (similarity > 1.0) that distract from the true distribution.
4. **Do not lowercase or quote-wrap column names** in the chart spec. Cortex Analyst returns Snowflake-default UPPERCASE; chart specs must match.

---

## Chart Spec Library

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
---

## Source Table

- **Database:** `QRSLLM_POC_DB`
- **Schema:** `HENRY_SCHEMA`
- **Table:** `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`
- **Required columns:** `next_perioddate` (DATE), `lm_cosine_similarity` (FLOAT)

---
