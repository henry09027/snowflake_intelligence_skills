---
name: risk-changes-detail
description: >-
  For a single company and fiscal year, surface which specific risk factors
  were added in the current 10-K and which were removed compared to the prior
  year. Calls the SP_RISK_FACTOR_DIFF stored procedure, which performs the
  semantic diff inside Snowflake using AI_COMPLETE — so this skill is a thin
  orchestrator that invokes the procedure and renders the result. Use as a
  follow-up to the changers-statistics skill, once a portfolio manager has
  identified a high-change company. Triggers: "what changed for company X",
  "show me the added or removed risks for {company}", "diff the risk section
  for {company} in {year}", or any drill-in request after a changer
  leaderboard.
---

## Objective

The portfolio manager has identified a company of interest (typically from the `changers-statistics` skill). They want to know **what specifically changed** in the risk factor section: which risks appear in this year's 10-K that were not in last year's, and which risks were dropped.

This skill is a **thin orchestrator**. The substantive work — parsing the two risk sections, matching risks across years, and rendering the added/removed lists — is performed by the Snowflake stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF`. The procedure reads both risk-section texts directly from the source view and invokes `AI_COMPLETE` server-side, so the full filing texts never leave Snowflake. The procedure returns a single row containing filing metadata plus a pre-formatted markdown analysis in the `LLM_ANALYSIS` column.

This architecture exists specifically to avoid the retrieval-payload limits that would apply to passing two full 10-K risk sections back through Cortex Analyst.

---

## 🎯 Execution Procedure — Follow Exactly

### Step 1 — Call the stored procedure

Execute exactly this CALL, substituting `{COMPANY_ID}` and `{YEAR}`:

```sql
CALL QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF({COMPANY_ID}, {YEAR});
```

Do not pass a third argument. The procedure defaults to `claude-haiku-4-5`, which is the sanctioned model for this task. Override the model only if the user explicitly asks.

The procedure returns **at most one row**. If zero rows: stop and report "No 10-K filing found for company `{COMPANY_ID}` with fiscal year `{YEAR}`." Do not proceed to Step 2 and do not attempt a fallback query.

### Step 2 — Validate the returned row

The returned row contains these columns:

| Column | Purpose |
|---|---|
| `COMPANYNAME`, `COMPANYID`, `SECTOR` | Identifying metadata |
| `NEXT_PERIODDATE`, `NEXT_FILINGDATE` | Filing dates |
| `LM_COSINE_SIMILARITY` | Pre-computed similarity vs prior year |
| `RISK_REMOVED_ADDED_FLAG` | Pre-computed flag (numeric or coded) |
| `PREV_FILING_TEXT_TOKEN_COUNT`, `NEXT_FILING_TEXT_TOKEN_COUNT` | Text size indicators |
| `LLM_PROMPT` | The instruction prompt sent to AI_COMPLETE — audit only, **do not display** unless the user asks |
| `LLM_ANALYSIS` | Pre-formatted markdown diff — the substantive output |

If `LLM_ANALYSIS` is NULL, empty, or obviously truncated (e.g. ends mid-bullet), report a procedure-execution failure and stop. Do not invent content to fill the gap.

### Step 3 — Render the output

Use the format in the **Output Format** section below. The `LLM_ANALYSIS` column already contains formatted markdown with `#### Risks Removed`, `#### Risks Added`, and `#### Brief Narration` subsections. **Pass it through verbatim.** Wrap it in the metadata header described below — do not rewrite, reorder, condense, or "improve" the analysis itself.

---

## Output Format

Render the result as inline markdown (not a file) with this structure:

```
### Risk Section Changes — {COMPANYNAME} ({SECTOR})

**Filing:** 10-K for fiscal period ending {NEXT_PERIODDATE}, filed {NEXT_FILINGDATE}
**Cosine similarity vs prior year:** {LM_COSINE_SIMILARITY, 4 decimals}
**Pre-computed flag:** {RISK_REMOVED_ADDED_FLAG, mapped to human-readable string}
**Token counts:** prior year {PREV_FILING_TEXT_TOKEN_COUNT}, current year {NEXT_FILING_TEXT_TOKEN_COUNT}

{LLM_ANALYSIS — verbatim, includes the Removed / Added / Narration subsections}
```

The `LLM_ANALYSIS` block already contains the `#### Risks Removed`, `#### Risks Added`, and `#### Brief Narration` subsections — do **not** re-create them, and do **not** add a separate narration of your own. The procedure has already done that work.

---

## ❌ Do Not

1. Do **not** rewrite, paraphrase, edit, condense, or "polish" the `LLM_ANALYSIS` text. The diff was computed inside Snowflake against the full filing texts; modifying it here breaks the audit trail and risks layering hallucination on top of a known output.
2. Do **not** call any other stored procedure, query the underlying view directly, or attempt to re-fetch the filing texts. The procedure is the only sanctioned access path — direct reads of `PREV_FILING_TEXT` / `NEXT_FILING_TEXT` via Cortex Analyst will hit retrieval-payload limits and is the reason this procedure exists.
3. Do **not** retry the procedure with a different model unless the user explicitly requests one. If the first call fails, surface the error to the user verbatim.
4. Do **not** override the `MODEL_NAME` parameter silently or "for performance."
5. Do **not** display the `LLM_PROMPT` column unless the user explicitly asks ("show me the prompt", "what instructions did the model get"). It is an audit field, not user-facing content.
6. Do **not** infer materiality, directionality, or trade ideas from the analysis — surface what changed, not what to do about it.
7. Do **not** fabricate risks if `LLM_ANALYSIS` is empty or NULL. Report the failure and stop.
8. Do **not** add your own commentary alongside the analysis (e.g. "Interestingly, this aligns with…"). The procedure's output is the answer; embellishment dilutes the audit trail.

---

## Source

- **Procedure:** `QRSLLM_POC_DB.HENRY_SCHEMA.SP_RISK_FACTOR_DIFF`
- **Signature:** `(COMPANY_ID INTEGER, FISCAL_YEAR INTEGER, MODEL_NAME VARCHAR DEFAULT 'claude-haiku-4-5')`
- **Returns:** Single-row table containing filing metadata, the audit prompt, and the pre-computed markdown diff. Empty result set means no 10-K filing exists for the requested `(company_id, fiscal_year)` pair.
- **Underlying view (do not query directly):** `QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`. Grain: one row per `(companyid, next_perioddate)`.

---

## Inputs

| Placeholder     | Type    | Description                                                                                                  |
|-----------------|---------|--------------------------------------------------------------------------------------------------------------|
| `{COMPANY_ID}`  | INTEGER | The S&P Capital IQ company ID. Typically obtained from the `changers-statistics` output.                     |
| `{YEAR}`        | INTEGER | Fiscal year of the **current** 10-K (the year of `next_perioddate`). The prior year is implicit in the view. |
