---
name: Sector × Region Revenue, Earnings, and Margin Revision — Time Series Chart Skill
description: Charts the time series of daily revenue, earnings, and margin estimate revisions versus a YTD start date for the S&P Global BMI Index, restricted to a single user-specified sector AND a single user-specified region. Execution is a single SELECT against the deployed Snowflake user-defined table function (UDTF) `QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REGION_LEVEL_ALL_REVISION`, which encapsulates the universe join, sector + region filter, dollar-weighted aggregation, and new-vs-old self-join on `revisiondate`; the skill renders that result set as a dual-axis time series chart with `REVENUE_PCT_CHANGE` and `EARNINGS_PCT_CHANGE` on the primary (left) y-axis and `MARGIN_PCT_CHANGE` on the secondary (right) y-axis, with `REVISIONDATE` on the x-axis.
---

## Objective

Returns a long-format, tabular result set (revision date × revision metrics) suitable for plotting a dual-axis time series of cumulative revenue, earnings, and margin estimate revisions since `START_REVISION_DATE` for a single (sector, region) pair. All universe membership (BMI constituents via `BASKETS_1`), the sector restriction via `SECTOR_INPUT`, the region restriction via `REGION_INPUT`, the new-vs-old self-join on `revisiondate`, and the dollar-weighted aggregation are encapsulated inside the deployed UDTF `QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REGION_LEVEL_ALL_REVISION`, so every invocation produces an identical schema and identical aggregation logic regardless of caller (Snowflake Intelligence agent, Cortex Analyst, notebook, or ad-hoc SQL).

This skill is the regional drill-down of `SECTOR_LEVEL_ALL_REVISION`. Where the latter aggregates a sector across all regions globally, this one further restricts the universe to a single `aggregateregion` value (e.g. Energy companies in Latin America and Caribbean only). Use this skill when the user wants to compare how the same sector's estimate revisions differ across geographies, by calling the function once per region and assembling the views side by side.

---

## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY

### Step 1 — Query the table function

Execute exactly:

```sql
SELECT *
FROM TABLE(
    QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REGION_LEVEL_ALL_REVISION(
        '{START_REVISION_DATE}',
        '{END_REVISION_DATE}',
        '{PERIOD}',
        '{INDEX_NAME}',
        '{SECTOR_INPUT}',
        '{REGION_INPUT}'
    )
);
```

All six arguments are `VARCHAR` and must be passed as string literals in `'YYYY-MM-DD'` form for the two dates. The function casts the date strings to `DATE` internally via `TO_DATE`, so no client-side `CAST` or `DATE 'YYYY-MM-DD'` literal is needed (and none should be added — keeping inputs as plain strings is what makes the function callable identically from a Snowflake Intelligence agent, where JSON tool arguments are always serialised as strings).

Defaults when the user does not specify otherwise: `START_REVISION_DATE='2026-01-01'` (YTD start), `END_REVISION_DATE` = today (e.g. `'2026-05-14'`), `PERIOD='FY-2026'`, `INDEX_NAME='Global'` (S&P Global BMI). `SECTOR_INPUT` and `REGION_INPUT` have **no defaults** and must both be specified by the user — if the user has not named both a sector and a region, ask for whichever is missing before invoking the function rather than guessing. "Energy" alone is not enough; "Energy in Latin America and Caribbean" is.

`SECTOR_INPUT` must match the exact `sector` string in `QRSLLM_POC_DB.DREW_SCHEMA.COMPANYFIRMOGRAPHICS` (e.g. `'Information Technology'`, `'Energy'`, `'Financials'`, `'Health Care'`, `'Consumer Discretionary'`, `'Consumer Staples'`, `'Industrials'`, `'Materials'`, `'Communication Services'`, `'Utilities'`, `'Real Estate'`). The match is case-sensitive.

`REGION_INPUT` must match the exact `aggregateregion` string in `COMPANYFIRMOGRAPHICS`. The one verified canonical label so far is `'Latin America and Caribbean'`. Other regions follow analogous "long-form, title-case, with conjunctions written out" conventions — so do **not** assume `'LatAm'`, `'EM'`, `'APAC'`, `'US'`, or `'Americas'` will resolve. If the user uses any short form or non-verified label, either confirm with the user or run this lookup once and pick the matching label:

```sql
SELECT DISTINCT aggregateregion
FROM QRSLLM_POC_DB.DREW_SCHEMA.COMPANYFIRMOGRAPHICS
ORDER BY 1;
```

If a call returns zero rows, the most likely cause is a casing or naming mismatch on either `SECTOR_INPUT` or `REGION_INPUT`, not an empty universe. The diagnostic `DISTINCT_COMPANY_COUNT` in the result set is also useful here: if it's zero on every row, the join produced no constituents and the labels are almost certainly wrong.

Do not wrap the function call in anything other than `SELECT * FROM TABLE(...)`, do not add a `LIMIT`, do not add a `WHERE` clause to narrow dates or further restrict the universe, and do not pass additional arguments — the function signature is fixed at exactly six positional `VARCHAR` arguments `(START_REVISION_DATE, END_REVISION_DATE, PERIOD_INPUT, INDEX_NAME, SECTOR_INPUT, REGION_INPUT)`.

### Step 2 — Render the result as a dual-axis time series chart

The chart is a **dual-axis Vega-Lite line chart with exactly two layers**. Do not add a third layer for a zero reference rule — Vega-Lite's continuous y-scales include zero by default (`scale.zero = true`), so the axis gridline at zero acts as the reference and a third layer triggers overlapping y-axis label rendering in the Snowflake Intelligence chart surface.

Required structure:

- **Layer 1 (primary, left axis):** fold `REVENUE_PCT_CHANGE` and `EARNINGS_PCT_CHANGE` into a long form, rename the keys to `'Revenue (L)'` and `'Earnings (L)'` via a `calculate` transform that writes into a field called `Metric`, plot `value` as a line. Axis title `"Revenue / Earnings Revision (%)"`, axis `format: ".1%"`, axis `orient: "left"`.
- **Layer 2 (secondary, right axis):** pre-multiply `MARGIN_PCT_CHANGE` by 100 via a `calculate` transform into a new field called `MARGIN_PP` (so the value is in percentage points, not raw fraction), and set a constant `Metric` field equal to `'Margin (R)'` via another `calculate`. Plot `MARGIN_PP` as a line. Axis title `"Margin Revision (pp)"`, axis `format: ".2f"` (no `%` — the unit is carried by the title), axis `orient: "right"`.
- **Top-level color encoding:** a single `color` encoding on `Metric` (nominal), placed at the top level of the spec (not inside individual layers). Use an explicit `scale.range` of three distinct colors so all three series render distinguishably, e.g. `["#1f77b4", "#ff7f0e", "#2ca02c"]` for Revenue / Earnings / Margin respectively.
- **Mark differentiation:** keep all three lines solid by default. If the renderer makes them hard to distinguish on grayscale or print, set `mark.strokeDash: [6, 3]` on the margin layer's mark — but only on the mark, not in an encoding, since `strokeDash` encoded by `Metric` would force a fold across layers that breaks the dual-axis structure.
- **Y-axis resolution:** `resolve: {scale: {y: "independent"}}` at the top level. This is what creates the two physically distinct y-axes; without it both layers' y-values collapse onto a single scale.
- **X-axis:** `REVISIONDATE` as `temporal`, axis title `"Revision Date"`, axis `format: "%b %d"`.
- **Title:** `"S&P Global BMI — {SECTOR_INPUT} in {REGION_INPUT}: YTD Revenue, Earnings, and Margin Revisions ({PERIOD}, {START_REVISION_DATE} → {END_REVISION_DATE})"`. Both the sector and the region must appear in the title; either derive them from the `SECTOR_INPUT` and `REGION_INPUT` parameters (preferred, since you have them in hand) or read them off the result set's `SECTOR` and `REGION` columns (both constant across rows).

Concrete Vega-Lite skeleton — emit this shape verbatim, substituting only the placeholder values:

```json
{
  "title": "S&P Global BMI — {SECTOR_INPUT} in {REGION_INPUT}: YTD Revenue, Earnings, and Margin Revisions ({PERIOD}, {START_REVISION_DATE} → {END_REVISION_DATE})",
  "encoding": {
    "color": {
      "field": "Metric",
      "type": "nominal",
      "title": "Metric",
      "scale": {
        "domain": ["Revenue (L)", "Earnings (L)", "Margin (R)"],
        "range":  ["#1f77b4",     "#ff7f0e",      "#2ca02c"]
      }
    }
  },
  "layer": [
    {
      "transform": [
        {"fold": ["REVENUE_PCT_CHANGE", "EARNINGS_PCT_CHANGE"], "as": ["MetricRaw", "value"]},
        {"calculate": "datum.MetricRaw === 'REVENUE_PCT_CHANGE' ? 'Revenue (L)' : 'Earnings (L)'", "as": "Metric"}
      ],
      "mark": {"type": "line", "strokeWidth": 2},
      "encoding": {
        "x": {"field": "REVISIONDATE", "type": "temporal", "axis": {"title": "Revision Date", "format": "%b %d"}},
        "y": {
          "field": "value",
          "type": "quantitative",
          "axis": {
            "title":  "Revenue / Earnings Revision (%)",
            "format": ".1%",
            "orient": "left"
          }
        }
      }
    },
    {
      "transform": [
        {"calculate": "datum.MARGIN_PCT_CHANGE * 100", "as": "MARGIN_PP"},
        {"calculate": "'Margin (R)'",                  "as": "Metric"}
      ],
      "mark": {"type": "line", "strokeWidth": 2},
      "encoding": {
        "x": {"field": "REVISIONDATE", "type": "temporal"},
        "y": {
          "field": "MARGIN_PP",
          "type": "quantitative",
          "axis": {
            "title":  "Margin Revision (pp)",
            "format": ".2f",
            "orient": "right"
          }
        }
      }
    }
  ],
  "resolve": {"scale": {"y": "independent"}}
}
```

The function returns Snowflake-default UPPERCASE identifiers — preserve them in field references exactly as shown above (`REVENUE_PCT_CHANGE`, `EARNINGS_PCT_CHANGE`, `MARGIN_PCT_CHANGE`, `REVISIONDATE`). Do not rename, re-case, or post-process columns at the dataframe level before passing to the chart spec. The `SECTOR` and `REGION` columns are not referenced by the chart spec (they are constant across rows and already encoded in the chart title via the input parameters). The auxiliary columns (`NEW_REVENUE`, `NEW_EARNINGS`, `NEW_MARGIN`, `OLD_REVENUE`, `OLD_EARNINGS`, `OLD_MARGIN`, `MIN_REVISION_DATE`, `MAX_REVISION_DATE`, `COUNT_REVISION_DATE`, `DISTINCT_COMPANY_COUNT`) are not referenced by the chart spec either; they may be surfaced in a small accompanying table if the user asks for diagnostics or for the absolute revenue / earnings levels behind the percentages.

### Step 3 — Narrate as a short bulleted summary

Render the narration as a Markdown bulleted list of **3–5 bullets**, not as a single prose paragraph. Lead the response with a one-line header (e.g. `**Summary:**`) and then the bullets. Each bullet is a single short sentence; collectively the bullets cover:

- The window covered, the forward `PERIOD` being revised, and **both** the sector and region pinned by `SECTOR_INPUT` and `REGION_INPUT` (e.g. "Energy in Latin America and Caribbean" — never just "Energy" or just "Latin America and Caribbean").
- The cumulative revenue revision at the latest `REVISIONDATE` (direction and roughly the magnitude).
- The cumulative earnings revision at the latest `REVISIONDATE` (direction and roughly the magnitude).
- The cumulative margin revision at the latest `REVISIONDATE` in **percentage points**, plus whether the margin story is consistent with the revenue/earnings story (e.g. revenue up + earnings up more → margin expansion; revenue up + earnings flat → margin compression; both down with earnings falling faster → margin compression).
- (Optional, only if it genuinely adds signal) one observation about inflection points, sign flips, or the relative timing of revenue vs. earnings revisions over the window. If the user has separately surfaced the same sector for a different region or for the global aggregate in the same session, you may use one bullet to note whether this region's revision trajectory looks consistent with or divergent from that comparison — but do not invent a comparison; only narrate one if the data was actually retrieved in this session.

Do not enumerate every date, do not restate the chart's axes, and do not add a closing paragraph after the bullets.

**Formatting rule for approximate values.** Use the words `roughly` or `approximately` to qualify imprecise figures — do **not** prefix numbers with `~` or `~~`. Double-tilde is GitHub-flavored Markdown for strikethrough, and an unclosed `~~+15.0%` will render as the literal characters `~~+15.0%` in the Snowflake Intelligence chat surface. Single `~` is also unsafe because some Markdown flavors (e.g. Pandoc with the `subscript` extension) treat `~x~` as subscript. If a symbol is required, use `≈` (U+2248); otherwise drop the qualifier entirely — a number like `+15.0%` is already obviously approximate in this context.

**Unit rule for the margin series.** Always qualify the margin number with `pp` or the words `percentage points` (e.g. `+1.2 pp`, `roughly +120 bps`, `approximately 1.2 percentage points`). Do **not** narrate it with a `%` suffix on its own (`+1.2%`), because that conflates a level change in margin with a percent change in margin and is the single most common misreading of this function's output. Note that the raw value of `MARGIN_PCT_CHANGE` is a fraction (e.g. `0.012`), so when reading the narration number off the raw column you must multiply by 100 before stating `pp`. (The chart spec already multiplies in the `calculate` transform — but only for axis display, not for narration values.)

---

## Function Contract

- **Object type:** SQL user-defined table function (UDTF).
- **Fully qualified name:** `QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REGION_LEVEL_ALL_REVISION`
- **Invocation syntax:** `SELECT * FROM TABLE(QRSLLM_POC_DB.HENRY_SCHEMA.SECTOR_REGION_LEVEL_ALL_REVISION(...))` — never `CALL`, since this is a function, not a stored procedure.
- **Inputs (all `VARCHAR`, positional):**
  - `START_REVISION_DATE` — `'YYYY-MM-DD'`. Anchor date; every later date is compared against this one (it is the `old` side of the self-join). Cast to `DATE` internally.
  - `END_REVISION_DATE` — `'YYYY-MM-DD'`. Inclusive upper bound on the `new` side. Cast to `DATE` internally.
  - `PERIOD_INPUT` — forward estimate period label, e.g. `'FY-2026'`.
  - `INDEX_NAME` — `indexshort` value in `BASKETS_1`, e.g. `'Global'` for S&P Global BMI.
  - `SECTOR_INPUT` — exact `sector` string in `COMPANYFIRMOGRAPHICS`, e.g. `'Information Technology'`.
  - `REGION_INPUT` — exact `aggregateregion` string in `COMPANYFIRMOGRAPHICS`, e.g. `'Latin America and Caribbean'`.
- **Why all `VARCHAR`?** Snowflake Intelligence and Cortex Agents serialise tool arguments as JSON strings and invoke the underlying function with **named arguments** (`arg => value`). Since the [2023_03 BCR](https://docs.snowflake.com/en/release-notes/bcr-bundles/2023_03/bcr-1017), named arguments resolve by name *and* type, so a `DATE`-typed parameter cannot be bound by an agent that passes `'2026-01-01'` as a `VARCHAR` — the call fails with *"named arguments do not match any signature"* and the agent falls back to Cortex Analyst. Declaring everything `VARCHAR` and casting in the body keeps a single signature that resolves cleanly for every caller.
- **Output:** sixteen-column table, one row per `REVISIONDATE` for the single pinned `(sector, region)` pair, ordered by `REVISIONDATE ASC`. Because `cf.sector = SECTOR_INPUT` and `cf.aggregateregion = REGION_INPUT` are both enforced inside the function, the `SECTOR` and `REGION` columns are constant across all rows of a given result set and equal the corresponding input parameters.
- **Semantics:**
  - `REVENUE_PCT_CHANGE = (Σ new_revenue_estimate − Σ old_revenue_estimate) / Σ old_revenue_estimate` — cumulative % change in aggregated revenue since `START_REVISION_DATE`, restricted to the (sector, region) pair.
  - `EARNINGS_PCT_CHANGE = (Σ new_earnings_estimate − Σ old_earnings_estimate) / Σ old_earnings_estimate` — cumulative % change in aggregated earnings since `START_REVISION_DATE`, restricted to the (sector, region) pair.
  - `MARGIN_PCT_CHANGE = (Σ new_earnings_estimate / Σ new_revenue_estimate) − (Σ old_earnings_estimate / Σ old_revenue_estimate)` — **difference of two aggregate margin ratios**, expressed in raw fraction units (e.g. `0.012` means `+1.2 percentage points`). Not a percent change in margin.
  - `NEW_REVENUE` and `NEW_EARNINGS` are returned in absolute units (the function multiplies the source estimates by 1,000,000). `NEW_MARGIN` and `OLD_MARGIN` are aggregate margin ratios (e.g. `0.12` = 12% margin) and are *not* scaled.
  - The series is **cumulative since YTD start**, not a daily delta — every row's `old` side is anchored to `START_REVISION_DATE`.

---

## Output Table Form

| Column                    | Type    | Description                                                                              |
|---------------------------|---------|------------------------------------------------------------------------------------------|
| `REVISIONDATE`            | DATE    | Date of the revision (the `new` side of the self-join)                                   |
| `SECTOR`                  | VARCHAR | GICS-style sector — constant across rows, equal to `SECTOR_INPUT`                        |
| `REGION`                  | VARCHAR | Aggregate region from `COMPANYFIRMOGRAPHICS.aggregateregion` — constant across rows, equal to `REGION_INPUT` |
| `REVENUE_PCT_CHANGE`      | FLOAT   | Cumulative % change in aggregated revenue estimate vs. `START_REVISION_DATE`             |
| `EARNINGS_PCT_CHANGE`     | FLOAT   | Cumulative % change in aggregated earnings estimate vs. `START_REVISION_DATE`            |
| `MARGIN_PCT_CHANGE`       | FLOAT   | Difference of aggregate margin ratios (new − old), in fraction units (× 100 for pp)       |
| `NEW_REVENUE`             | FLOAT   | Σ revenue estimate on `REVISIONDATE`, in absolute units (× 1,000,000)                     |
| `NEW_EARNINGS`            | FLOAT   | Σ earnings estimate on `REVISIONDATE`, in absolute units (× 1,000,000)                    |
| `NEW_MARGIN`              | FLOAT   | Aggregate margin ratio on `REVISIONDATE` (Σ earnings / Σ revenue, unscaled)               |
| `OLD_REVENUE`             | FLOAT   | Σ revenue estimate on `START_REVISION_DATE`, in absolute units                            |
| `OLD_EARNINGS`            | FLOAT   | Σ earnings estimate on `START_REVISION_DATE`, in absolute units                           |
| `OLD_MARGIN`              | FLOAT   | Aggregate margin ratio on `START_REVISION_DATE` (Σ earnings / Σ revenue, unscaled)        |
| `MIN_REVISION_DATE`       | DATE    | Diagnostic — min revision date contributing to the row                                   |
| `MAX_REVISION_DATE`       | DATE    | Diagnostic — max revision date contributing to the row                                   |
| `COUNT_REVISION_DATE`     | NUMBER  | Diagnostic — count of contributing rows                                                  |
| `DISTINCT_COMPANY_COUNT`  | NUMBER  | Diagnostic — distinct constituents contributing on that date. Useful sanity check: if zero on every row, `SECTOR_INPUT` or `REGION_INPUT` is almost certainly mislabeled. |

---

## ❌ DOs and DON'Ts

### Calling the function

1. **Do not use `CALL`** to invoke this object — it is a UDTF, not a stored procedure. `CALL SECTOR_REGION_LEVEL_ALL_REVISION(...)` will fail. Always wrap it in `SELECT * FROM TABLE(...)`.
2. **Do not change the parameter types to `DATE`** to "be correct". Agent callers pass JSON strings, and a `DATE` signature breaks named-argument resolution (see Function Contract above). The string→date conversion belongs *inside* the function body, not at the boundary.
3. **Do not re-create, alter, or replace the function** from within this skill. The UDTF is managed and deployed out-of-band; the skill is a pure consumer.
4. **Do not inline the join / aggregation SQL** as a substitute for the table-function call. The point of the UDTF is that universe membership, the sector + region filter, the new-vs-old self-join, and the dollar-weighted aggregation are defined once, server-side — ad-hoc SQL defeats reproducibility and risks silently changing the universe.
5. **Do not append a `LIMIT`** to the `SELECT`. The function already returns at most one row per `REVISIONDATE` in the requested window.
6. **Do not aggregate across sectors or regions client-side, and do not slice by sub-industry or country.** The function pins the universe to a single `(SECTOR_INPUT, REGION_INPUT)` pair; any cross-sector or cross-region comparison should call the function once per cell. Use `SECTOR_LEVEL_ALL_REVISION` for "this sector across all regions globally" and `SECTOR_REVENUE_REVISION` for cross-sector revenue revision.
7. **Do not silently substitute either input** when the user-supplied labels don't match `COMPANYFIRMOGRAPHICS`. For sectors, the GICS standard is reliable (`'Information Technology'`, `'Communication Services'`, etc.). For regions, only `'Latin America and Caribbean'` is verified — other regions follow analogous long-form, title-case conventions but you should not assume `'LatAm'`, `'EM'`, `'APAC'`, `'US'`, `'Americas'`, or other short forms will resolve. If unsure, either ask the user or `SELECT DISTINCT aggregateregion FROM COMPANYFIRMOGRAPHICS` once to enumerate the valid labels.
8. **Do not change `INDEX_NAME` silently.** If the user asks for a non-BMI universe (e.g. S&P 500, MSCI Taiwan), confirm the corresponding `indexshort` value in `BASKETS_1` before substituting.
9. **Do include both the sector and the region in every chart title and narration header.** "S&P Global BMI — Energy in Latin America and Caribbean" is correct; "S&P Global BMI — Energy" loses the regional restriction and is the most common misread of this skill's output.

### Building the chart spec

10. **Do use exactly two layers**, one per y-axis. Do not add a third `rule` layer for a zero reference — Vega-Lite's continuous y-scales include zero by default and the axis gridline at zero acts as the reference. A third layer causes the Snowflake Intelligence chart renderer to overlap y-axis tick labels.
11. **Do place the `color` encoding at the top level** of the chart spec, not inside individual layers. Both layers must produce a `Metric` field via a `calculate` transform so the top-level color scale unifies all three series into a single legend.
12. **Do set `axis.orient` explicitly** — `"left"` for the primary (Revenue / Earnings) axis and `"right"` for the secondary (Margin) axis. Default placement under `resolve.scale.y = "independent"` is renderer-dependent and may place both axes on the same side.
13. **Do pre-multiply `MARGIN_PCT_CHANGE` by 100** in a `calculate` transform (producing a `MARGIN_PP` field) and format the secondary y-axis numerically (`.2f` or `.1f`), **not** as `.1%`. The axis title `"Margin Revision (pp)"` carries the unit. Formatting as `%` would conflate percentage points with percent change and is the single most common rendering bug for this output.
14. **Do not interpret `MARGIN_PCT_CHANGE` as a percent change in margin.** It is `new_margin − old_margin` as a difference of ratios. A raw value of `0.012` is `+1.2 percentage points`, not `+1.2%` and not `+12%`.
15. **Do not interpret `REVENUE_PCT_CHANGE` or `EARNINGS_PCT_CHANGE` as a daily delta.** Both are cumulative vs. `START_REVISION_DATE` by construction.
16. **Do not lowercase or quote-wrap column names** in the spec or display. Preserve Snowflake-default UPPERCASE identifiers in every field reference (`REVENUE_PCT_CHANGE`, not `Revenue_Pct_Change`, not `"revenue_pct_change"`).
17. **Do not use `datum` for positional encodings** (e.g. `"y": {"datum": 0}` for a zero rule). It renders at position 0 of the screen, not the data scale, in the Snowflake Intelligence chart surface. Use `calculate` transforms to inject constant string fields like `"Metric": "Margin (R)"` instead.
18. **Do not encode `strokeDash` by `Metric`** to differentiate the margin line. That would force a single fold across layers and break the dual-axis structure. Set `strokeDash` as a static mark property on layer 2 only, if visual differentiation beyond color is needed.

---
