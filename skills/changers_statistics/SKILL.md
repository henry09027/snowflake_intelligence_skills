---
name: changers-statistics
description: Flag the S&P 500 companies whose 10-K risk factor section changed the most year-over-year, ranked by semantic similarity between the prior and current filing. Use when a portfolio manager asks "which companies changed their risk factors this year", "biggest risk section changes", "10-K risk factor changers", or any variant referencing material/largest/biggest year-over-year shifts in 10-K risk language. Returns a fixed-size leaderboard (lowest cosine similarity = largest change), not a distribution.
---

## Objective

The audience is a portfolio manager. The job is to surface S&P 500 companies whose 10-K risk factor section changed substantially year-over-year, so the PM can read the underlying text and decide whether the change is material to their position.

**Execution runs through Cortex Analyst.** The Cortex Analyst prompt below is load-bearing — its wording has been tuned to produce a deterministic SQL contract. Do not paraphrase it.

---

## 🎯 Execution Procedure — Follow Exactly

### Step 1 — Send the literal prompt to Cortex Analyst

Pass the prompt from the **Cortex Analyst Prompt Library** below to Cortex Analyst **verbatim**, with `{YEAR}` substituted to the requested fiscal year (e.g. `2025`). Do not paraphrase. Do not "improve" the prompt. Do not add or remove columns, filters, or ordering clauses.

### Step 2 — Render the result as a table

Use the column mapping in the **Output Table Form** section. Format `LM_COSINE_SIMILARITY` to 4 decimal places. Format `RISK_ADDED_REMOVED_FLAG` values as the human-readable strings in the mapping table (e.g. `RISK_ADDED` → "Risk Added").

---

## 🗣️ Cortex Analyst Prompt Library

### PROMPT_CHANGERS — Top-N Year-Over-Year Risk Section Changers

```
Query the view:
  QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned

Return exactly these columns, in this order:
  COMPANYNAME, COMPANYID, SECTOR, LM_COSINE_SIMILARITY,
  NEXT_FILINGDATE, NEXT_PERIODDATE,
  PREV_FILING_TEXT_TOKEN_COUNT, NEXT_FILING_TEXT_TOKEN_COUNT,
  RISK_ADDED_REMOVED_FLAG

Grain:
  - One row per company per fiscal year. Do NOT aggregate. Do NOT GROUP BY.
  - Use the columns above verbatim from the view. Do NOT recompute or derive them.

Filters (apply ALL of the following in the WHERE clause):
  - DATE_PART('year', next_perioddate) = {YEAR}
  - lm_cosine_similarity IS NOT NULL
  - next_perioddate <= CURRENT_DATE

Ordering:
  - ORDER BY lm_cosine_similarity ASC
  - Tie-break: ORDER BY companyid ASC (so the result is deterministic across calls).

Limit:
  - LIMIT 20  (the 20 companies with the lowest cosine similarity, i.e. the largest year-over-year change).

Other constraints:
  - Return exactly the 9 columns listed above. Do NOT add columns.
  - Do NOT add HAVING filters.
  - Do NOT wrap the query in a CTE unless required by the engine.
  - Treat all column identifiers as unquoted (case-insensitive) Snowflake identifiers.
```

---

## Output Table Form

| Display column        | Source column                    | Type   | Notes                                                            |
|-----------------------|----------------------------------|--------|------------------------------------------------------------------|
| Company               | `COMPANYNAME`                    | STRING |                                                                  |
| Company ID            | `COMPANYID`                      | INT    |                                                                  |
| Sector                | `SECTOR`                         | STRING | GICS sector                                                      |
| Cosine Similarity     | `LM_COSINE_SIMILARITY`           | FLOAT  | Format to 4 decimals. Lower = larger change.                     |
| Filing Date           | `NEXT_FILINGDATE`                | DATE   | Date the current-year 10-K was filed.                            |
| Period End            | `NEXT_PERIODDATE`                | DATE   | Fiscal period end date of the current-year 10-K.                 |
| Prev Tokens           | `PREV_FILING_TEXT_TOKEN_COUNT`   | INT    | Token count of prior-year risk section.                          |
| Curr Tokens           | `NEXT_FILING_TEXT_TOKEN_COUNT`   | INT    | Token count of current-year risk section.                        |
| Risk Change Status    | `RISK_ADDED_REMOVED_FLAG`        | STRING | Map: `RISK_ADDED` → "Risk Added", `RISK_REMOVED` → "Risk Removed", `UNCHANGED` → "Unchanged". |

---

## ❌ Do Not

1. Do **not** call any stored procedure.
2. Do **not** rephrase the prompt in the *Cortex Analyst Prompt Library*. The wording is load-bearing.
3. Do **not** apply additional filters (sector, market cap, token-count thresholds, etc.) unless the user explicitly asks. The skill returns the unconditional top-20 by similarity for the requested year.
4. Do **not** interpret a low cosine similarity as a directional signal (bullish/bearish). It only flags magnitude of change.
5. Do **not** silently fall back to a different year if `{YEAR}` returns fewer than 20 rows — return whatever rows match and say so in the narration.
6. Do **not** output anything other than the required table. I want a clean and professional output.

---

## Source Table

- **Database:** `QRSLLM_POC_DB`
- **Schema:** `HENRY_SCHEMA`
- **Table:** `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`
- **Grain:** one row per `(companyid, next_perioddate)` — i.e. one row per company per fiscal-year 10-K, with the prior year's risk section joined in alongside.
