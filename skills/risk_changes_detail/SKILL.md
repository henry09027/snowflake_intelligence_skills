---
name: risk-changes-detail
description: For a single company and fiscal year, surface which specific risk factors were added in the current 10-K and which were removed compared to the prior year. Use as a follow-up to the changers-statistics skill, once a portfolio manager has identified a high-change company, this skill drills into the actual text to show what changed. Triggers: "what changed for company X", "show me the added/removed risks for {company}", "diff the risk section for {company} in {year}", or any drill-in request after a changer leaderboard.
---

## Objective

The portfolio manager has already identified a company of interest (typically from the `changers-statistics` skill). They now want to know **what specifically changed** in the risk factor section: which risks appear in this year's 10-K that were not in last year's, and which risks were dropped.

The skill has two phases:
1. **Cortex Analyst** retrieves the two raw text blobs (prior-year risk section, current-year risk section) plus identifying metadata for the requested `(companyid, fiscal_year)` pair.
2. **LLM** performs a semantic diff over those two texts and renders the result as two bullet lists.

The Cortex Analyst phase is deterministic and load-bearing. The diff phase is interpretive — the LLM is doing the work, not Snowflake.

---

## 🎯 Execution Procedure — Follow Exactly

### Step 1 — Send the literal prompt to Cortex Analyst

Pass the prompt from the **Cortex Analyst Prompt Library** below to Cortex Analyst **verbatim**, with `{COMPANYID}` and `{YEAR}` substituted. Do not paraphrase. Do not "improve" the prompt.

The query must return **exactly one row**. If it returns 0 rows: stop, report "No 10-K filing found for company `{COMPANYID}` with fiscal year `{YEAR}`," and do not proceed to Step 2. If it returns more than 1 row: stop and report a data-grain anomaly — the underlying view should be unique on `(companyid, next_perioddate)`.

### Step 2 — Parse each text into a list of discrete risk factors

For both `PREV_FILING_TEXT` and `NEXT_FILING_TEXT`:

1. Identify the structure used to demarcate individual risk factors. 10-Ks typically use one of: bolded inline headings, all-caps sub-headers, or numbered/bulleted lists. The structure is usually consistent within a single filing but differs across companies.
2. Extract each distinct risk factor as a `(heading, body)` pair. The heading is the short risk title (often a single sentence stating the risk, e.g. "Our supply chain is concentrated in a small number of suppliers"). The body is the supporting paragraph(s).
3. If the filing has no clear structural delimiters, fall back to paragraph-level segmentation and treat the first sentence as the implicit heading.

### Step 3 — Match risks semantically across years

Compare the two lists. For each pair of risks (one from prior, one from current), decide whether they describe the **same underlying risk** even if rephrased, reordered, or expanded. Examples of what should match:

- "Cybersecurity incidents could materially harm our business" ↔ "A breach of our information systems could result in significant losses"
- "We depend on a small number of key customers" ↔ "Loss of any of our largest customers would adversely affect revenue"

**Be conservative when matching.** When in doubt about whether two risks are the same, treat them as the same. Only flag a risk as added/removed if it is clearly absent from the other year. The cost of a false-positive flag (PM wastes time investigating a non-change) is higher than the cost of a false negative (PM misses a subtle rephrasing).

A risk that was substantially **modified** but covers the same underlying topic is **not** an addition or removal. Skip it. (If the user explicitly asks about modifications, that is out of scope for this skill — direct them to read the underlying text.)

### Step 4 — Render the output

Use the format in the **Output Format** section below. Both lists may be empty; report empty lists explicitly rather than omitting the section.

### Step 5 — Brief narration (≤ 3 sentences)

State (i) the cosine similarity for context, (ii) the count of added vs removed risks, (iii) one sentence on the dominant theme of the changes if a theme is obvious (e.g. "Most additions relate to AI and regulatory risk"). Do not summarize every bullet.

---

## 🗣️ Cortex Analyst Prompt Library

### PROMPT_RISK_DIFF — Single Company, Single Year, Two Texts

```
Query the view:
  QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned

Return exactly these columns, in this order:
  COMPANYNAME, COMPANYID, SECTOR,
  NEXT_PERIODDATE, NEXT_FILINGDATE,
  LM_COSINE_SIMILARITY, RISK_ADDED_REMOVED_FLAG,
  PREV_FILING_TEXT_TOKEN_COUNT, NEXT_FILING_TEXT_TOKEN_COUNT,
  PREV_FILING_TEXT, NEXT_FILING_TEXT

Grain:
  - Return exactly one row, corresponding to the single (companyid, fiscal_year) pair requested.
  - Do NOT aggregate. Do NOT GROUP BY. Do NOT use DISTINCT.
  - Use all columns verbatim from the view. Do NOT recompute or derive them.

Filters (apply ALL of the following in the WHERE clause):
  - companyid = {COMPANYID}
  - DATE_PART('year', next_perioddate) = {YEAR}
  - next_perioddate <= CURRENT_DATE

Ordering and Limit:
  - Do NOT add ORDER BY. The (companyid, fiscal_year) pair is unique in the view.
  - Do NOT add a LIMIT clause.

Other constraints:
  - Return exactly the 11 columns listed above, in that order. Do NOT add columns.
  - Do NOT add HAVING filters.
  - Do NOT wrap the query in a CTE unless required by the engine.
  - Treat all column identifiers as unquoted (case-insensitive) Snowflake identifiers.
  - Return the full text of PREV_FILING_TEXT and NEXT_FILING_TEXT — do NOT truncate, summarize, or LEFT() them.
```

---

## Output Format

Render the result as inline markdown (not a file) with this structure:

```
### Risk Section Changes — {COMPANYNAME} ({SECTOR})

**Filing:** 10-K for fiscal period ending {NEXT_PERIODDATE}, filed {NEXT_FILINGDATE}
**Cosine similarity vs prior year:** {LM_COSINE_SIMILARITY, 4 decimals}
**Pre-computed flag:** {RISK_ADDED_REMOVED_FLAG, mapped to human-readable string}
**Token counts:** prior year {PREV_FILING_TEXT_TOKEN_COUNT}, current year {NEXT_FILING_TEXT_TOKEN_COUNT}

#### 🟥 Risks Removed (present last year, absent this year)

- **{Heading 1}** — {1-sentence summary of the risk in faithful paraphrase}
- **{Heading 2}** — {…}

(Or: "No risks were removed compared to the prior-year filing." if the list is empty.)

#### 🟩 Risks Added (present this year, absent last year)

- **{Heading 1}** — {1-sentence summary of the risk in faithful paraphrase}
- **{Heading 2}** — {…}

(Or: "No new risks were added compared to the prior-year filing." if the list is empty.)
```

Bullet rules:
- The **heading** for each bullet should be the original risk-factor heading from the source text, lightly trimmed for length if needed (target ≤ 15 words). Do not invent a heading.
- The **summary** is LLM's own concise paraphrase of the body, not a verbatim quote. Keep it to one sentence.
- Order bullets by their order of appearance in the source filing (preserves the company's own narrative structure).

---

## ❌ Do Not

1. Do **not** call any stored procedure.
2. Do **not** rephrase the prompt in the *Cortex Analyst Prompt Library*. The wording is load-bearing.
3. Do **not** truncate `PREV_FILING_TEXT` or `NEXT_FILING_TEXT` in SQL — the full text is needed for the diff. If the texts are too large to fit in context after retrieval, report the failure rather than silently diffing partial text.
4. Do **not** fabricate risks. Every bullet must trace to a heading or paragraph in the source filing.
5. Do **not** flag rephrased-but-equivalent risks as either added or removed.
6. Do **not** report modified risks in either list — that is out of scope for this skill.
7. Do **not** reproduce long verbatim passages from the filing in the bullets. Faithful one-sentence paraphrases only.
8. Do **not** infer materiality, directionality, or trade ideas from the changes — surface what changed, not what to do about it.

---

## Source Table

- **Database:** `QRSLLM_POC_DB`
- **Schema:** `HENRY_SCHEMA`
- **Table:** `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`
- **Grain:** one row per `(companyid, next_perioddate)` — i.e. one row per company per fiscal-year 10-K.
- **Text columns assumed:** `PREV_FILING_TEXT`, `NEXT_FILING_TEXT`. Confirm column names match the view; rename in the prompt if they differ.

---

## Inputs

| Placeholder    | Type | Description                                                            |
|----------------|------|------------------------------------------------------------------------|
| `{COMPANYID}`  | INT  | The S&P Capital IQ company ID. Typically obtained from the changers-statistics output. |
| `{YEAR}`       | INT  | The fiscal year of the **current** 10-K (i.e. the year of `next_perioddate`). The prior year is implicit in the join on the view. |
