---
name: Revenue Revision by Region for a Sector — Time Series Chart Skill
description: Charts the time series of daily revenue estimate revisions versus a YTD start date for the S&P Global BMI Index, restricted to a single GICS-style sector and broken down by Capital IQ `aggregateregion`. Execution is a single SELECT against the deployed Snowflake user-defined table function (UDTF) `QRSLLM_POC_DB.HENRY_SCHEMA.REGION_REVENUE_REVISION`, which encapsulates the universe join, sector filter, and regional aggregation; the skill renders that result set as a multi-line time series chart of REV_PCT_CHANGE over REVISIONDATE.
---

## Objective

Returns a long-format, tabular result set (revision date × aggregate region × revision metrics) suitable for plotting a multi-line time series of cumulative revenue estimate revisions since `START_REVISION_DATE` within a single sector. All universe membership (BMI constituents via `BASKETS_1`), the new-vs-old self-join on `revisiondate`, the single-sector filter on `COMPANYFIRMOGRAPHICS.sector`, and the dollar-weighted regional aggregation by `aggregateregion` are encapsulated inside the deployed UDTF `QRSLLM_POC_DB.HENRY_SCHEMA.REGION_REVENUE_REVISION`, so every invocation produces an identical schema and identical aggregation logic regardless of caller (Snowflake Intelligence agent, Cortex Analyst, notebook, or ad-hoc SQL).

This is the regional companion to `SECTOR_REVENUE_REVISION`: that function answers *"which sectors are being revised most across the BMI?"*, while this one drills into a chosen sector and answers *"across which regions is the revision concentrated within that sector?"*. A typical workflow is to call the sector skill first, identify a sector of interest, and then call this skill with that sector as `SECTOR_INPUT` to localise the revision geographically.

---

## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY

### Step 1 — Query the table function

Execute exactly:

```sql
SELECT *
FROM TABLE(
    QRSLLM_POC_DB.HENRY_SCHEMA.REGION_REVENUE_REVISION(
        '{START_REVISION_DATE}',
        '{END_REVISION_DATE}',
        '{PERIOD}',
        '{INDEX_NAME}',
        '{SECTOR}'
    )
);
```

All five arguments are `VARCHAR` and must be passed as string literals — `'YYYY-MM-DD'` for the two dates, and the raw label values for `PERIOD`, `INDEX_NAME`, and `SECTOR`. The function casts the date strings to `DATE` internally via `TO_DATE`, so no client-side `CAST` or `DATE 'YYYY-MM-DD'` literal is needed (and none should be added — keeping inputs as plain strings is what makes the function callable identically from a Snowflake Intelligence agent, where JSON tool arguments are always serialised as strings).

Defaults when the user does not specify otherwise: `START_REVISION_DATE='2026-01-01'` (YTD start), `END_REVISION_DATE` = today (e.g. `'2026-05-12'`), `PERIOD='FY-2026'`, `INDEX_NAME='Global'` (S&P Global BMI). `SECTOR` has **no default** — it is a required input because the function returns a regional decomposition *within* a single sector. If the user has not specified one (and has not been routed in from the sector skill with one already in context), ask before invoking; do not silently fall back to a placeholder sector.

`SECTOR` must match a value of `COMPANYFIRMOGRAPHICS.sector` exactly, case-sensitive, including spacing (e.g. `'Information Technology'`, `'Health Care'`, `'Consumer Staples'`). If the user provides a near-match (`'Tech'`, `'IT'`, `'Healthcare'`), normalise to the canonical GICS-style label before invocation.

Do not wrap the function call in anything other than `SELECT * FROM TABLE(...)`, do not add a `LIMIT`, do not add a `WHERE` clause to narrow regions or dates, and do not pass additional arguments — the function signature is fixed at exactly five positional `VARCHAR` arguments `(START_REVISION_DATE, END_REVISION_DATE, PERIOD_INPUT, INDEX_NAME, SECTOR_INPUT)`.

### Step 2 — Render the result as a time series chart

Plot the returned result set as a multi-line time series chart:

- **x-axis:** `REVISIONDATE`
- **y-axis:** `REV_PCT_CHANGE` formatted as a percentage with at least 1 decimal place
- **series (one line per value):** `AGGREGATEREGION`
- **reference line:** horizontal at `y = 0`
- **title:** `S&P Global BMI — YTD Revenue Estimate Revisions for {SECTOR} by Region ({PERIOD}, {START_REVISION_DATE} → {END_REVISION_DATE})`

Unlike `SECTOR_REVENUE_REVISION`, this function **does not pre-filter to a top-N by absolute revision**. `aggregateregion` is naturally low-cardinality (typically 5–8 values globally — e.g. United States and Canada, Europe, Asia/Pacific, Latin America and Caribbean, Africa/Middle East), so the chart should plot every region returned. Do not drop, re-rank, or re-bucket regions client-side. If the user explicitly asks for a top-N narrowing or a regional grouping (e.g. "developed vs. emerging"), surface that separately as a small table and keep the chart complete.

The function returns Snowflake-default UPPERCASE identifiers — preserve them. Do not rename, re-case, or post-process columns. Drop only the auxiliary columns (`MIN_REVISION_DATE`, `MAX_REVISION_DATE`, `DISTINCT_COMPANY_COUNT`) from the chart itself; they may be surfaced in a small accompanying table if the user asks for diagnostics, and `DISTINCT_COMPANY_COUNT` is particularly worth showing when a region looks thinly populated (e.g. fewer than ~5 constituents on the latest date), since small-N regional aggregates are noisy.

### Step 3 — Narrate as a short bulleted summary

Render the narration as a Markdown bulleted list of **3–5 bullets**, not as a single prose paragraph. Lead the response with a one-line header (e.g. `**Summary:**`) and then the bullets. Each bullet is a single short sentence; collectively the bullets cover:

- The window covered, the forward `PERIOD` being revised, and the sector being broken down.
- The region with the largest cumulative upward revision and roughly its magnitude.
- The region with the largest cumulative downward revision (or the most muted, if all regions are positive) and roughly its magnitude.
- Any notable inflection point, sign flip, or divergence/convergence across regions over the window — e.g. United States and Canada and Europe moving together while Asia/Pacific diverges.
- (Optional, only if it genuinely adds signal) one cross-regional observation, e.g. that the sector-level number from `SECTOR_REVENUE_REVISION` is being driven primarily by a single region rather than broadly across geographies, or a regional aggregate driven by a thin distinct-company count.

Do not enumerate every region on every date, do not restate the chart's axes, and do not add a closing paragraph after the bullets.

**Formatting rule for approximate values.** Use the words `roughly` or `approximately` to qualify imprecise figures — do **not** prefix numbers with `~` or `~~`. Double-tilde is GitHub-flavored Markdown for strikethrough, and an unclosed `~~+15.0%` will render as the literal characters `~~+15.0%` in the Snowflake Intelligence chat surface. Single `~` is also unsafe because some Markdown flavors (e.g. Pandoc with the `subscript` extension) treat `~x~` as subscript. If a symbol is required, use `≈` (U+2248); otherwise drop the qualifier entirely — a number like `+15.0%` is already obviously approximate in this context. Examples: `North America led with roughly +15.0%` ✅ — `North America led with ~~+15.0%` ❌ — `North America led with ~+15.0%` ❌.

---

## Function Contract

- **Object type:** SQL user-defined table function (UDTF).
- **Fully qualified name:** `QRSLLM_POC_DB.HENRY_SCHEMA.REGION_REVENUE_REVISION`
- **Invocation syntax:** `SELECT * FROM TABLE(QRSLLM_POC_DB.HENRY_SCHEMA.REGION_REVENUE_REVISION(...))` — never `CALL`, since this is a function, not a stored procedure.
- **Inputs (all `VARCHAR`, positional):**
  - `START_REVISION_DATE` — `'YYYY-MM-DD'`. Anchor date; every later date is compared against this one (it is the `old` side of the self-join). Cast to `DATE` internally.
  - `END_REVISION_DATE` — `'YYYY-MM-DD'`. Inclusive upper bound on the `new` side. Cast to `DATE` internally.
  - `PERIOD_INPUT` — forward estimate period label, e.g. `'FY-2026'`.
  - `INDEX_NAME` — `indexshort` value in `BASKETS_1`, e.g. `'Global'` for S&P Global BMI.
  - `SECTOR_INPUT` — `sector` value in `COMPANYFIRMOGRAPHICS`, e.g. `'Information Technology'`. Case-sensitive, must match exactly.
- **Why all `VARCHAR`?** Snowflake Intelligence and Cortex Agents serialise tool arguments as JSON strings and invoke the underlying function with **named arguments** (`arg => value`). Since the [2023_03 BCR](https://docs.snowflake.com/en/release-notes/bcr-bundles/2023_03/bcr-1017), named arguments resolve by name *and* type, so a `DATE`-typed parameter cannot be bound by an agent that passes `'2026-01-01'` as a `VARCHAR` — the call fails with *"named arguments do not match any signature"* and the agent falls back to Cortex Analyst. Declaring everything `VARCHAR` and casting in the body keeps a single signature that resolves cleanly for every caller. The deployed signature for this function **must** therefore use `VARCHAR` for `START_REVISION_DATE` and `END_REVISION_DATE` (matching `SECTOR_REVENUE_REVISION`); if the function is currently deployed with `DATE`-typed date parameters, agent calls will fail at bind time and the function must be redeployed before this skill is usable end-to-end.
- **Output:** eight-column table, one row per `(REVISIONDATE × AGGREGATEREGION)` cell — **no top-N filter is applied**, every region with at least one contributing constituent on a given date is returned. Ordered by `REVISIONDATE ASC, AGGREGATEREGION ASC`.
- **Semantics:** `REV_PCT_CHANGE = (Σ new_revenue_estimate − Σ old_revenue_estimate) / Σ old_revenue_estimate`, where `old.revisiondate = START_REVISION_DATE` for every row — so the series is **cumulative since YTD start**, not a daily delta. `NEW_REVENUE` and `OLD_REVENUE` are returned in absolute units (the function multiplies the source `revenue_estimate` by 1,000,000). The denominator `OLD_REVENUE` is over the **same set of companies that report on `REVISIONDATE`** (the self-join keys on `companyid`), so a constituent that drops out of estimates after the anchor date does not contribute to either side of the ratio on subsequent dates.

---

## Output Table Form

| Column                    | Type    | Description                                                                          |
|---------------------------|---------|--------------------------------------------------------------------------------------|
| `REVISIONDATE`            | DATE    | Date of the revision (the `new` side of the self-join)                               |
| `AGGREGATEREGION`         | VARCHAR | Capital IQ `aggregateregion` from `COMPANYFIRMOGRAPHICS`                             |
| `REV_PCT_CHANGE`          | FLOAT   | Cumulative % change in aggregated revenue estimate vs. `START_REVISION_DATE`         |
| `NEW_REVENUE`             | FLOAT   | Σ revenue estimate on `REVISIONDATE`, in absolute units (×1,000,000)                 |
| `OLD_REVENUE`             | FLOAT   | Σ revenue estimate on `START_REVISION_DATE`, in absolute units                       |
| `DISTINCT_COMPANY_COUNT`  | NUMBER  | Diagnostic — distinct constituents contributing on that `(REVISIONDATE, REGION)` cell |
| `MIN_REVISION_DATE`       | DATE    | Diagnostic — min revision date contributing to the row                               |
| `MAX_REVISION_DATE`       | DATE    | Diagnostic — max revision date contributing to the row                               |

Note the column ordering and count differ from `SECTOR_REVENUE_REVISION`: there is no `COUNT_REVISION_DATE` column, and `DISTINCT_COMPANY_COUNT` is positioned ahead of the min/max diagnostic dates. Downstream consumers that select columns by position (rather than by name) must be updated accordingly.

---

## ❌ DOs and DON'Ts

1. **Do not use `CALL`** to invoke this object — it is a UDTF, not a stored procedure. `CALL REGION_REVENUE_REVISION(...)` will fail. Always wrap it in `SELECT * FROM TABLE(...)`.
2. **Do not change the parameter types to `DATE`** to "be correct". Agent callers pass JSON strings, and a `DATE` signature breaks named-argument resolution (see Function Contract above). The string→date conversion belongs *inside* the function body, not at the boundary. This applies to both `START_REVISION_DATE` and `END_REVISION_DATE`.
3. **Do not omit `SECTOR_INPUT` or default it silently.** There is no sensible global default for sector; if the user has not specified one, ask. Picking an arbitrary sector (or the alphabetically-first one) will produce a chart that looks reasonable but answers a question nobody asked.
4. **Do not pass a near-match sector label** — e.g. `'Tech'`, `'IT'`, `'tech'`, `'Information Tech'`. The function filter is `cf.sector = SECTOR_INPUT` with no fuzzy match; a mistyped label returns zero rows and a blank chart. Normalise to the canonical `COMPANYFIRMOGRAPHICS.sector` value before invocation.
5. **Do not re-create, alter, or replace the function** from within this skill. The UDTF is managed and deployed out-of-band; the skill is a pure consumer.
6. **Do not inline the join / aggregation / sector-filter SQL** as a substitute for the table-function call. The point of the UDTF is that universe membership, the new-vs-old self-join, the sector filter, and the regional aggregation are defined once, server-side — ad-hoc SQL defeats reproducibility and risks silently changing the universe.
7. **Do not append a `LIMIT`** to the `SELECT`. The function already returns at most `~8 × N_dates` rows for the requested window (bounded by the cardinality of `aggregateregion`).
8. **Do not client-side filter to a top-N of regions.** Unlike the sector function, no top-N is applied server-side, and the expected behaviour for this skill is to plot every region. If the user explicitly asks for a top-N or a developed/emerging grouping, do that as a separate post-step and keep the underlying chart complete.
9. **Do not interpret `REV_PCT_CHANGE` as a daily delta.** It is cumulative vs. `START_REVISION_DATE` by construction.
10. **Do not lowercase or quote-wrap column names** in display. Preserve Snowflake-default UPPERCASE identifiers, including `AGGREGATEREGION`.
11. **Do not change `INDEX_NAME` silently.** If the user asks for a non-BMI universe (e.g. S&P 500, MSCI Taiwan), confirm the corresponding `indexshort` value in `BASKETS_1` before substituting.
12. **Do not over-interpret thinly-populated regions.** If a region's `DISTINCT_COMPANY_COUNT` is very small (e.g. one or two constituents for the chosen sector), its `REV_PCT_CHANGE` is effectively a single-name move and should be flagged as such in the narration rather than treated as a regional signal.

---
