---
name: Revenue Revision by Sector — Time Series Chart Skill
description: Charts the time series of daily revenue estimate revisions versus a YTD start date for the S&P Global BMI Index, aggregated to the sector level and pre-filtered to the top 5 sectors by absolute revision at the latest revision date. Execution is a single SELECT against the deployed Snowflake user-defined table function (UDTF) `QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REVENUE_REVISION`, which encapsulates the universe join, sector aggregation, and top-5 filter; the skill renders that result set as a multi-line time series chart of REV_PCT_CHANGE over REVISIONDATE.
---

## Objective

Returns a long-format, tabular result set (revision date × sector × revision metrics) suitable for plotting a multi-line time series of cumulative revenue estimate revisions since `START_REVISION_DATE`. All universe membership (BMI constituents via `BASKETS_1`), the new-vs-old self-join on `revisiondate`, the dollar-weighted sector aggregation, and the top-5-by-absolute-revision filter are encapsulated inside the deployed UDTF `QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REVENUE_REVISION`, so every invocation produces an identical schema and identical aggregation logic regardless of caller.

---

## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY

### Step 1 — Query the table function

Execute exactly:

```sql
SELECT *
FROM TABLE(
    QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REVENUE_REVISION(
        DATE '{START_REVISION_DATE}',
        DATE '{END_REVISION_DATE}',
        '{PERIOD}',
        '{INDEX_NAME}'
    )
);
```

Substitute the four parameters in the order shown. The first two are `DATE`s and **must** be passed either as `DATE 'YYYY-MM-DD'` literals (preferred) or via `TO_DATE('YYYY-MM-DD')`; the last two are `VARCHAR` string literals. Defaults when the user does not specify otherwise: `START_REVISION_DATE='2026-01-01'` (YTD start), `END_REVISION_DATE` = today (e.g. `'2026-05-12'`), `PERIOD='FY-2026'`, `INDEX_NAME='Global'` (S&P Global BMI).

Do not wrap the function call in anything other than `SELECT * FROM TABLE(...)`, do not add a `LIMIT`, do not add a `WHERE` clause to narrow sectors or dates, and do not pass additional arguments — the function signature is fixed at exactly four positional arguments `(START_REVISION_DATE DATE, END_REVISION_DATE DATE, PERIOD_INPUT VARCHAR, INDEX_NAME VARCHAR)`.

### Step 2 — Render the result as a time series chart

Plot the returned result set as a multi-line time series chart:

- **x-axis:** `REVISIONDATE`
- **y-axis:** `REV_PCT_CHANGE` formatted as a percentage with at least 1 decimal place
- **series (one line per value):** `SECTOR`
- **reference line:** horizontal at `y = 0`
- **title:** `S&P Global BMI — YTD Revenue Estimate Revisions by Sector ({PERIOD}, {START_REVISION_DATE} → {END_REVISION_DATE})`

The function already pre-filters to the top 5 sectors by `ABS(rev_pct_change)` at the latest revision date, so do not re-filter or re-rank client-side. The function returns Snowflake-default UPPERCASE identifiers — preserve them. Do not rename, re-case, or post-process columns. Drop only the auxiliary columns (`MIN_REVISION_DATE`, `MAX_REVISION_DATE`, `COUNT_REVISION_DATE`, `DISTINCT_COMPANY_COUNT`) from the chart itself; they may be surfaced in a small accompanying table if the user asks for diagnostics.

### Step 3 — Narrate in ≤ 4 sentences

State (i) the window covered and the forward `PERIOD` being revised, (ii) which of the 5 sectors saw the largest upward and downward cumulative revisions and by roughly how much, and (iii) any notable inflection points or divergence/convergence across sectors. Do not enumerate every sector on every date.

---

## Function Contract

- **Object type:** SQL user-defined table function (UDTF).
- **Fully qualified name:** `QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REVENUE_REVISION`
- **Invocation syntax:** `SELECT * FROM TABLE(QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REVENUE_REVISION(...))` — never `CALL`, since this is a function, not a stored procedure.
- **Inputs (positional):**
  - `START_REVISION_DATE` `DATE` — anchor date; every later date is compared against this one (it is the `old` side of the self-join).
  - `END_REVISION_DATE` `DATE` — inclusive upper bound on the `new` side.
  - `PERIOD_INPUT` `VARCHAR` — forward estimate period label, e.g. `'FY-2026'`.
  - `INDEX_NAME` `VARCHAR` — `indexshort` value in `BASKETS_1`, e.g. `'Global'` for S&P Global BMI.
- **Output:** nine-column table, one row per `(REVISIONDATE × SECTOR)` cell for the top 5 sectors only, ordered by `REVISIONDATE ASC, SECTOR ASC`.
- **Semantics:** `REV_PCT_CHANGE = (Σ new_revenue_estimate − Σ old_revenue_estimate) / Σ old_revenue_estimate`, where `old.revisiondate = START_REVISION_DATE` for every row — so the series is **cumulative since YTD start**, not a daily delta. `NEW_REVENUE` and `OLD_REVENUE` are returned in absolute units (the function multiplies the source `revenue_estimate` by 1,000,000).

---

## Output Table Form

| Column                    | Type    | Description                                                                 |
|---------------------------|---------|-----------------------------------------------------------------------------|
| `REVISIONDATE`            | DATE    | Date of the revision (the `new` side of the self-join)                      |
| `SECTOR`                  | VARCHAR | GICS-style sector from `COMPANYFIRMOGRAPHICS`                               |
| `REV_PCT_CHANGE`          | FLOAT   | Cumulative % change in aggregated revenue estimate vs. `START_REVISION_DATE`|
| `NEW_REVENUE`             | FLOAT   | Σ revenue estimate on `REVISIONDATE`, in absolute units (×1,000,000)        |
| `OLD_REVENUE`             | FLOAT   | Σ revenue estimate on `START_REVISION_DATE`, in absolute units              |
| `MIN_REVISION_DATE`       | DATE    | Diagnostic — min revision date contributing to the row                      |
| `MAX_REVISION_DATE`       | DATE    | Diagnostic — max revision date contributing to the row                      |
| `COUNT_REVISION_DATE`     | NUMBER  | Diagnostic — count of contributing rows                                     |
| `DISTINCT_COMPANY_COUNT`  | NUMBER  | Diagnostic — distinct constituents contributing on that date                |

---

## ❌ DOs and DON'Ts

1. **Do not use `CALL`** to invoke this object — it is a UDTF, not a stored procedure. `CALL SECTOR_REVENUE_REVISION(...)` will fail. Always wrap it in `SELECT * FROM TABLE(...)`.
2. **Do not re-create, alter, or replace the function** from within this skill. The UDTF is managed and deployed out-of-band; the skill is a pure consumer.
3. **Do not inline the join / aggregation / top-5 SQL** as a substitute for the table-function call. The point of the UDTF is that universe membership, the new-vs-old self-join, the dollar-weighted aggregation, and the top-5 filter are defined once, server-side — ad-hoc SQL defeats reproducibility and risks silently changing the universe.
4. **Do not pass the date arguments as bare strings** (`'2026-01-01'`) when calling the function. The signature is typed `DATE`; use `DATE '2026-01-01'` or `TO_DATE('2026-01-01')` to make the cast explicit and avoid silent coercion surprises.
5. **Do not append a `LIMIT`** to the `SELECT`. The function already returns at most `5 × N_dates` rows for the requested window.
6. **Do not client-side re-filter or re-rank sectors.** The top-5 selection is fixed by `ABS(rev_pct_change)` at the latest `REVISIONDATE` inside the function; reranking client-side will produce a chart that disagrees with the function contract.
7. **Do not interpret `REV_PCT_CHANGE` as a daily delta.** It is cumulative vs. `START_REVISION_DATE` by construction.
8. **Do not lowercase or quote-wrap column names** in display. Preserve Snowflake-default UPPERCASE identifiers.
9. **Do not change `INDEX_NAME` silently.** If the user asks for a non-BMI universe (e.g. S&P 500, MSCI Taiwan), confirm the corresponding `indexshort` value in `BASKETS_1` before substituting.

---
