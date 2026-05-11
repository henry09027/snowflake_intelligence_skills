---
name: risk-changes-detail
description: >-
  For a single company and fiscal year, surface which specific risk factors
  were added in the current 10-K and which were removed compared to the prior
  year. Calls the SP_RISK_FACTOR_DIFF stored procedure, which performs the
  semantic diff inside Snowflake using AI_COMPLETE and returns the analysis as
  a single markdown string. This skill is a thin pass-through: invoke the
  procedure, render the returned string. Use as a follow-up to the
  changers-statistics skill, once a portfolio manager has identified a
  high-change company. Triggers: "what changed for company X", "show me the
  added or removed risks for {company}", "diff the risk section for {company}
  in {year}", or any drill-in request after a changer leaderboard.
---

## Objective

The portfolio manager has identified a company of interest (typically from the `changers-statistics` skill). They want to know **what specifically changed** in the risk factor section: which risks appear in this year's 10-K that were not in last year's, and which risks were dropped.

This skill is a **pass-through wrapper**. All substantive work — reading both filing texts, parsing them into discrete risks, matching risks across years, and formatting the diff — is performed by the Snowflake stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF`. The procedure invokes `AI_COMPLETE` server-side over the full filing texts and returns a single `VARCHAR` containing the analysis already formatted as markdown. The orchestrator's only job is to call the procedure and render the returned string.

This architecture exists specifically to avoid the retrieval-payload limits that would apply to passing two full 10-K risk sections back through Cortex Analyst.

---

## 🎯 Execution Procedure — Follow Exactly

### Step 1 — Call the stored procedure

Execute exactly this CALL, substituting `{COMPANY_ID}` and `{YEAR}`:

```sql
CALL QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF({COMPANY_ID}, {YEAR});
```

Do not pass a third argument. The procedure defaults to `claude-haiku-4-5`, which is the sanctioned model for this task. Override the model only if the user explicitly asks.

The procedure returns a single `VARCHAR` value (one row, one column) containing markdown.

### Step 2 — Validate the returned value

The expected return is non-empty markdown beginning with `#### Risks Removed`. Handle failure cases as follows:

- **Call errored with a no-rows / not-found exception** — no 10-K exists for the requested `(company_id, fiscal_year)` pair. Report: "No 10-K filing found for company `{COMPANY_ID}` with fiscal year `{YEAR}`." Stop.
- **Call returned NULL or empty string** — the procedure ran but `AI_COMPLETE` produced nothing. Report the failure verbatim and stop. Do not retry with a different model or attempt to construct an analysis yourself.
- **Returned text is obviously truncated** (ends mid-bullet, missing the `Brief Narration` section, etc.) — report the truncation and stop. Do not pad the analysis with fabricated content.

### Step 3 — Render the output

Use the format in the **Output Format** section below. The returned string already contains the `#### Risks Removed`, `#### Risks Added`, and `#### Brief Narration` subsections — **pass it through verbatim**. Prepend only the minimal header described below. Do not rewrite, reorder, condense, or "improve" the analysis itself.

---

## Output Format

Render the result as inline markdown (not a file) with this structure:

```
### Risk Section Changes — Company {COMPANY_ID}, Fiscal Year {YEAR}

{procedure return value — verbatim, includes the Removed / Added / Narration subsections}
```

If a company name is already established in the surrounding chat context (e.g. the PM just selected the row from a changers-statistics leaderboard), substitute it into the header in place of `Company {COMPANY_ID}` — but only when the name is already present in context. Do **not** make a separate query to look it up. The procedure is the only sanctioned access path.

The returned string already contains the `#### Risks Removed`, `#### Risks Added`, and `#### Brief Narration` subsections — do **not** re-create them, and do **not** add a narration of your own. The procedure has already done that work.

---

## ❌ Do Not

1. Do **not** rewrite, paraphrase, edit, condense, reorder, or "polish" the procedure's return value. The diff was computed inside Snowflake against the full filing texts; modifying it here breaks the audit trail and risks layering hallucination on top of a known output.
2. Do **not** query the underlying view directly, attempt to re-fetch the filing texts, or call any other procedure to enrich the output with metadata (cosine similarity, token counts, filing dates, sector, etc.). The procedure is the only sanctioned access path — direct reads of `PREV_FILING_TEXT` / `NEXT_FILING_TEXT` will hit retrieval-payload limits and are the reason this procedure exists.
3. Do **not** retry the procedure with a different model unless the user explicitly requests one. If the first call fails, surface the error verbatim.
4. Do **not** override the `MODEL_NAME` parameter silently or "for performance."
5. Do **not** infer materiality, directionality, or trade ideas from the analysis — surface what changed, not what to do about it.
6. Do **not** fabricate content if the return value is NULL, empty, or truncated. Report the failure and stop.
7. Do **not** add your own commentary alongside the analysis (e.g. "Interestingly, this aligns with…"). The procedure's output is the answer; embellishment dilutes the audit trail.

---

## Source

- **Procedure:** `QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF`
- **Signature:** `(COMPANY_ID NUMBER(38,0), FISCAL_YEAR NUMBER(38,0), MODEL_NAME VARCHAR DEFAULT 'claude-haiku-4-5')`
- **Returns:** `VARCHAR` — a single string containing pre-formatted markdown with the diff. A no-rows match raises a procedure-execution exception (no 10-K exists for that `(company_id, fiscal_year)` pair).
- **Underlying view (do not query directly):** `QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`. Grain: one row per `(companyid, next_perioddate)`.

---

## Inputs

| Placeholder     | Type           | Description                                                                                                  |
|-----------------|----------------|--------------------------------------------------------------------------------------------------------------|
| `{COMPANY_ID}`  | NUMBER(38,0)   | The S&P Capital IQ company ID. Typically obtained from the `changers-statistics` output.                     |
| `{YEAR}`        | NUMBER(38,0)   | Fiscal year of the **current** 10-K (the year of `next_perioddate`). The prior year is implicit in the view. |
