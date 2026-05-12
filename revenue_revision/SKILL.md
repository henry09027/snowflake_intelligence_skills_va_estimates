---

name: Revenue Revision by Sector — Time Series Chart Skill
description: Charts the time series of daily revenue estimate revisions versus a YTD start date for the S&P Global BMI Index, aggregated to the sector level and pre-filtered to the top 5 sectors by absolute revision at the latest revision date. Execution is a single CALL to a deployed Snowflake stored procedure that encapsulates the universe join, sector aggregation, and top-5 filter; the skill renders that result set as a multi-line time series chart of REV_PCT_CHANGE over REVISIONDATE.

---

## Objective
Returns a long-format, tabular result set (revision date × sector × revision metrics) suitable for plotting a multi-line time series of cumulative revenue estimate revisions since `START_REVISION_DATE`. All universe membership (BMI constituents via `BASKETS_1`), the new-vs-old self-join on `revisiondate`, the dollar-weighted sector aggregation, and the top-5-by-absolute-revision filter are encapsulated inside the deployed stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.SP_REVENUE_REVISION_BY_SECTOR`, so every invocation produces an identical schema and identical aggregation logic regardless of caller.

---
## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY
### Step 1 — Call the stored procedure
Execute exactly:
```sql
CALL QRSLLM_POC_DB.HENRY_SCHEMA.SP_REVENUE_REVISION_BY_SECTOR(
    '{START_REVISION_DATE}',
    '{END_REVISION_DATE}',
    '{PERIOD}',
    '{INDEX_NAME}'
);
```
Substitute the four parameters as string literals in the order shown. Defaults when the user does not specify otherwise: `START_REVISION_DATE='2026-01-01'` (YTD start), `END_REVISION_DATE` = today (e.g. `'2026-05-05'`), `PERIOD='FY-2026'`, `INDEX_NAME='Global'` (S&P Global BMI). Do not wrap the call in a `SELECT`, do not add a `LIMIT`, do not add a `WHERE` clause, and do not pass additional arguments — the procedure signature accepts exactly four `VARCHAR`s in the order `(START_REVISION_DATE, END_REVISION_DATE, PERIOD, INDEX_NAME)`.
### Step 2 — Render the result as a time series chart
Plot the returned result set as a multi-line time series chart:
- **x-axis:** `REVISIONDATE`
- **y-axis:** `REV_PCT_CHANGE` formatted as a percentage with at least 1 decimal place
- **series (one line per value):** `SECTOR`
- **reference line:** horizontal at `y = 0`
- **title:** `S&P Global BMI — YTD Revenue Estimate Revisions by Sector ({PERIOD}, {START_REVISION_DATE} → {END_REVISION_DATE})`

The procedure already pre-filters to the top 5 sectors by `ABS(rev_pct_change)` at the latest revision date, so do not re-filter or re-rank client-side. The procedure returns Snowflake-default UPPERCASE identifiers — preserve them. Do not rename, re-case, or post-process columns. Drop only the auxiliary columns (`MIN_REVISION_DATE`, `MAX_REVISION_DATE`, `COUNT_REVISION_DATE`, `DISTINCT_COMPANY_COUNT`) from the chart itself; they may be surfaced in a small accompanying table if the user asks for diagnostics.

### Step 3 — Narrate in ≤ 4 sentences

State (i) the window covered and the forward `PERIOD` being revised, (ii) which of the 5 sectors saw the largest upward and downward cumulative revisions and by roughly how much, and (iii) any notable inflection points or divergence/convergence across sectors. Do not enumerate every sector on every date.

---
## Procedure Contract
- **Fully qualified name:** `QRSLLM_POC_DB.HENRY_SCHEMA.SP_REVENUE_REVISION_BY_SECTOR`
- **Inputs (all `VARCHAR`, positional):**
  - `START_REVISION_DATE` — anchor date in `'YYYY-MM-DD'`; every later date is compared against this one (it is the `old` side of the self-join).
  - `END_REVISION_DATE` — inclusive upper bound on the `new` side, `'YYYY-MM-DD'`.
  - `PERIOD` — forward estimate period label, e.g. `'FY-2026'`.
  - `INDEX_NAME` — `indexshort` value in `BASKETS_1`, e.g. `'Global'` for S&P Global BMI.
- **Output:** nine-column table, one row per `(REVISIONDATE × SECTOR)` cell for the top 5 sectors only, ordered by `REVISIONDATE ASC, SECTOR ASC`.
- **Semantics:** `REV_PCT_CHANGE = (Σ new_revenue_estimate − Σ old_revenue_estimate) / Σ old_revenue_estimate`, where `old.revisiondate = START_REVISION_DATE` for every row — so the series is **cumulative since YTD start**, not a daily delta. `NEW_REVENUE` and `OLD_REVENUE` are returned in absolute units (the procedure multiplies the source `revenue_estimate` by 1,000,000).
---
## Output Table Form
| Column                    | Type    | Description                                                                 |
|---------------------------|---------|-----------------------------------------------------------------------------|
| `REVISIONDATE`            | DATE    | Date of the revision (the `new` side of the self-join)                      |
| `SECTOR`                  | STRING  | GICS-style sector from `COMPANYFIRMOGRAPHICS`                               |
| `REV_PCT_CHANGE`          | FLOAT   | Cumulative % change in aggregated revenue estimate vs. `START_REVISION_DATE`|
| `NEW_REVENUE`             | FLOAT   | Σ revenue estimate on `REVISIONDATE`, in absolute units (×1,000,000)        |
| `OLD_REVENUE`             | FLOAT   | Σ revenue estimate on `START_REVISION_DATE`, in absolute units              |
| `MIN_REVISION_DATE`       | DATE    | Diagnostic — min revision date contributing to the row                      |
| `MAX_REVISION_DATE`       | DATE    | Diagnostic — max revision date contributing to the row                      |
| `COUNT_REVISION_DATE`     | INT     | Diagnostic — count of contributing rows                                     |
| `DISTINCT_COMPANY_COUNT`  | INT     | Diagnostic — distinct constituents contributing on that date                |
---
## ❌ DOs and DON'Ts
1. **Do not re-create, alter, or replace the procedure** from within this skill. The procedure is managed and deployed out-of-band; the skill is a pure consumer.
2. **Do not inline the join / aggregation / top-5 SQL** as a substitute for the `CALL`. The point of the procedure is that universe membership, the new-vs-old self-join, the dollar-weighted aggregation, and the top-5 filter are defined once, server-side — ad-hoc SQL defeats reproducibility and risks silently changing the universe.
3. **Do not append a `LIMIT`** to the `CALL`. The procedure already returns at most `5 × N_dates` rows for the requested window.
4. **Do not client-side re-filter or re-rank sectors.** The top-5 selection is fixed by `ABS(rev_pct_change)` at the latest `REVISIONDATE` inside the procedure; reranking client-side will produce a chart that disagrees with the procedure contract.
5. **Do not interpret `REV_PCT_CHANGE` as a daily delta.** It is cumulative vs. `START_REVISION_DATE` by construction.
6. **Do not lowercase or quote-wrap column names** in display. Preserve Snowflake-default UPPERCASE identifiers.
7. **Do not change `INDEX_NAME` silently.** If the user asks for a non-BMI universe (e.g. S&P 500, MSCI Taiwan), confirm the corresponding `indexshort` value in `BASKETS_1` before substituting.
---
