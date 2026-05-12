---

name: Revenue Revision by Sector — Time Series Chart Skill
description: Renders a multi-line time series chart of daily revenue estimate revisions versus a YTD start date for the S&P Global BMI Index, aggregated to the sector level and pre-filtered to the top 5 sectors by absolute revision at the latest revision date. Execution is a single CALL to a deployed Snowflake stored procedure that encapsulates the universe join, sector aggregation, top-5 filter, and SVG construction; the skill passes the returned SVG string directly to the Snowflake Intelligence chart renderer.

---
## Objective

Returns a single scalar `VARCHAR` containing a fully-formed SVG document that visualises the cumulative revenue estimate revision (in %) since `START_REVISION_DATE` for the top 5 sectors of the requested index, one line per sector, with a zero reference line and a sector legend. All universe membership (BMI constituents via `BASKETS_1`), the new-vs-old self-join on `revisiondate`, the dollar-weighted sector aggregation, the top-5-by-absolute-revision filter, and the SVG layout (axes, gridlines, colours, legend, title) are encapsulated inside the deployed stored procedure `QRSLLM_POC_DB.HENRY_SCHEMA.SP_REVENUE_REVISION_BY_SECTOR` and its companion UDF `QRSLLM_POC_DB.HENRY_SCHEMA.GENERATE_SVG_TIMESERIES`, so every invocation produces an identical chart specification regardless of caller.

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
### Step 2 — Pass the returned SVG directly to the Snowflake Intelligence chart renderer
The `CALL` returns a single scalar `VARCHAR` whose value is a complete, self-contained SVG document beginning with `<svg ...>` and ending with `</svg>`. Hand this string verbatim to the Snowflake Intelligence chart generation function for inline rendering. Do not parse, edit, re-encode, escape, prettify, wrap in markdown code fences, truncate, or attempt to convert the SVG into any other chart format (Vega-Lite, Plotly, table, etc.) — the procedure has already finalised the visual specification, and any client-side transformation will either break rendering or silently diverge from the server-side contract.
### Step 3 — Narrate in ≤ 4 sentences

State (i) the window covered and the forward `PERIOD` being revised, (ii) which of the 5 sectors saw the largest upward and downward cumulative revisions and by roughly how much (read these off the rendered chart), and (iii) any notable inflection points or divergence/convergence across sectors. Do not enumerate every sector on every date, and do not re-describe the chart's axes, colours, or legend — they are visible to the user.

---
## Procedure Contract
- **Fully qualified name:** `QRSLLM_POC_DB.HENRY_SCHEMA.SP_REVENUE_REVISION_BY_SECTOR`
- **Inputs (all `VARCHAR`, positional):**
  - `START_REVISION_DATE` — anchor date in `'YYYY-MM-DD'`; every later date is compared against this one (it is the `old` side of the self-join).
  - `END_REVISION_DATE` — inclusive upper bound on the `new` side, `'YYYY-MM-DD'`.
  - `PERIOD` — forward estimate period label, e.g. `'FY-2026'`.
  - `INDEX_NAME` — `indexshort` value in `BASKETS_1`, e.g. `'Global'` for S&P Global BMI.
- **Output:** single scalar `VARCHAR` containing one complete SVG document. Not a result set, not a table, not a row — a single string.
- **Semantics encoded in the SVG:** each polyline corresponds to one of the top 5 sectors (selected server-side by `ABS(rev_pct_change)` at the latest `REVISIONDATE`); the y-axis is `(Σ new_revenue_estimate − Σ old_revenue_estimate) / Σ old_revenue_estimate` in %, where `old.revisiondate = START_REVISION_DATE` for every row — so the series is **cumulative since YTD start**, not a daily delta. The dashed horizontal line at y = 0 is the no-revision reference. The auxiliary diagnostic columns (`MIN_REVISION_DATE`, `MAX_REVISION_DATE`, `COUNT_REVISION_DATE`, `DISTINCT_COMPANY_COUNT`) are aggregated server-side but are not surfaced in the SVG output; they are not retrievable from this procedure.
---
## Output Form
| Return slot | Type      | Description                                                                                              |
|-------------|-----------|----------------------------------------------------------------------------------------------------------|
| (scalar)    | `VARCHAR` | A single SVG document, 900 × 500 px, white background, title `"Revenue Estimate Revision % Change by Sector"`, with x-axis date labels, y-axis percentage labels, a dashed zero reference line, one coloured polyline per sector, and a right-side sector legend (sector labels truncated to 18 characters). |
---
## ❌ DOs and DON'Ts
1. **Do not re-create, alter, or replace the procedure or the SVG UDF** from within this skill. Both `SP_REVENUE_REVISION_BY_SECTOR` and `GENERATE_SVG_TIMESERIES` are managed and deployed out-of-band; the skill is a pure consumer.
2. **Do not inline the join / aggregation / top-5 SQL** as a substitute for the `CALL`. The point of the procedure is that universe membership, the new-vs-old self-join, the dollar-weighted aggregation, the top-5 filter, and the chart construction are defined once, server-side — ad-hoc SQL defeats reproducibility and risks silently changing the universe or the chart spec.
3. **Do not append a `LIMIT`** to the `CALL`. The procedure returns a single scalar; `LIMIT` is meaningless and may error.
4. **Do not attempt to re-chart the data client-side.** The underlying tabular result set is not returned — only the SVG is. Asking the model to "make a nicer chart" or "use Plotly instead" is incompatible with this skill's contract; if a different chart is required, the procedure or UDF must be redeployed.
5. **Do not parse, modify, escape, or re-encode the SVG string.** Pass it verbatim to the Snowflake Intelligence chart renderer. In particular, do not wrap it in markdown code fences, HTML-escape the angle brackets, base64-encode it, or strip the XML namespace.
6. **Do not interpret `REV_PCT_CHANGE` (the y-axis) as a daily delta.** It is cumulative vs. `START_REVISION_DATE` by construction.
7. **Do not change `INDEX_NAME` silently.** If the user asks for a non-BMI universe (e.g. S&P 500, MSCI Taiwan), confirm the corresponding `indexshort` value in `BASKETS_1` before substituting.
---
