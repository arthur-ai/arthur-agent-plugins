---
name: arthur-dashboards
description: Write SQL for Upsolve dashboard charts, stat cards and alert rules against Arthur's metrics tables. Covers which metrics exist and their dimensions, the rule_count vs inference_count denominator trap that silently produces wrong rates, Upsolve's SQL wrapper and its syntax constraints, and KLL sketch handling for latency and score distributions. Use whenever writing or debugging a query for an Arthur dashboard chart, a stat card, or an alert rule.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Arthur dashboard SQL

Charts on Arthur dashboards are SQL against two TimescaleDB tables, embedded through Upsolve. Most of what goes wrong is not SQL: it is picking a metric whose denominator answers a different question than the one asked, or hitting a constraint of Upsolve's query wrapper that produces a blank chart with no visible error.

Read `references/metrics.md` for the full metric and dimension reference before writing a query against an unfamiliar metric. Read `references/upsolve.md` for the wrapper's constraints and the failure modes they produce.

## The tables

| Table | Holds | `value` is |
|---|---|---|
| `metrics_numeric_latest_version` | counts and sums | a number |
| `metrics_sketch_latest_version` | distributions | a KLL sketch object |

Both carry the same top-level columns: `timestamp`, `model_id`, `model_name`, `project_id`, `project_name`, `workspace_id`, `workspace_name`, `organization_id`, `metric_name`, `metric_version`, `value`, `dimensions` (JSONB).

Metrics are bucketed at 5 minutes by the aggregation pipeline. That is the finest granularity available; there are no per-inference rows.

**`model_name` the column is the Arthur application.** `dimensions->>'model_name'` is the LLM the inference reported (`gpt-4o`, `claude-sonnet-4-5`). These are different things and the naming invites confusion. Filter applications on the column; group LLMs by the dimension.

## Pick the right denominator

This is the single most common source of wrong numbers, and it is silent — both queries run and return plausible percentages.

- **`inference_count`** — one row per request. An inference's `result` is `Fail` if *any* rule failed.
- **`rule_count`** — one row per rule evaluation. An inference running 7 rules produces 7 rows.

On a representative dataset the same failure rendered as **35% of inferences** and **9% of rule evaluations**. Both correct. A dashboard mixing them shows a stat card and a chart that contradict each other, and an alert rule that fires at the wrong threshold.

Rules of thumb:

- "What fraction of requests were blocked" → `inference_count`. This is what users experience and what stakeholders mean.
- "Which rule is doing the work" → `rule_count`, grouped by `rule_type` or `name`.
- Any chart that must agree with a headline number → whatever the headline uses. Check before assuming.

Two further traps inside `rule_count`:

**Per-rule-type rates are not comparable across types.** Each rule type has its own evaluation count. Five configured regex rules means 5× the denominator of a single PII rule, so regex shows a lower rate for the same real-world impact. Group by `dimensions->>'name'` to see per-rule-instance numbers.

**Failures double-count across rules.** An inference tripping PII and two regex rules contributes 1 to `inference_count` failures and 3 to `rule_count` failures. There is no way to count distinct failing inferences from `rule_count` — no inference id survives aggregation.

## Writing a query

Start from this shape:

```sql
SELECT
    time_bucket('1 day', timestamp) AS bucket,
    SUM(CASE WHEN dimensions->>'result' = 'Fail' THEN value ELSE 0 END) AS failures,
    SUM(value) AS total,
    ROUND((100.0 * SUM(CASE WHEN dimensions->>'result' = 'Fail' THEN value ELSE 0 END)
         / NULLIF(SUM(value), 0))::numeric, 2) AS failure_rate
FROM metrics_numeric_latest_version
WHERE metric_name = 'inference_count'
    [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
GROUP BY bucket
ORDER BY bucket
```

Five things in that template are load-bearing:

1. **No trailing semicolon.** Upsolve wraps the query in a CTE; a semicolon terminates the statement mid-parenthesis and the chart fails with a syntax error that preview never shows.
2. **`[[ ]]` around the date filter.** The block is stripped when the variable is unbound. Without it the literal `{{dateStart}}` reaches Postgres as `invalid input syntax for type timestamp`.
3. **`::numeric` before `ROUND(x, 2)`.** Postgres has no `round(double precision, integer)`.
4. **`NULLIF(..., 0)`** on every denominator.
5. **Expose numerator and denominator** when the number will be compared against another chart. Two rates that disagree are almost always two denominators.

Always include the raw counts while developing, even if the final chart shows only the rate. Debugging a disagreement without them means re-deriving the query.

## Stat cards and period comparison

A stat card returning one column cannot display the wrong one. Prefer that over returning three and configuring which to show.

Week-over-week cards fail in a specific way worth recognising: when the previous window has no data, `NULLIF(prev, 0)` yields null, and a `COALESCE(..., 0)` wrapper renders that as `0%` — indistinguishable from "no change". Omit the outer `COALESCE` so a missing baseline renders blank.

Before building any period comparison, check the data actually spans both windows:

```sql
SELECT MIN(timestamp), MAX(timestamp), COUNT(*)
FROM metrics_numeric_latest_version
WHERE metric_name = '<metric>'
```

Arthur deployments frequently have short or gapped ingestion history. A week-over-week card on four days of data is structurally incapable of producing a number.

## Sketches

`metrics_sketch_latest_version.value` is a KLL sketch, not a number. `SUM(value)` is a type error.

```sql
WITH merged AS (
    SELECT kll_float_sketch_merge(value) AS sketch
    FROM metrics_sketch_latest_version
    WHERE metric_name = 'rule_latency'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
)
SELECT
    kll_float_sketch_get_n(sketch) AS n,
    ROUND((kll_float_sketch_get_quantiles(sketch, ARRAY[0.5]))[1]::numeric, 1) AS p50,
    ROUND((kll_float_sketch_get_quantiles(sketch, ARRAY[0.95]))[1]::numeric, 1) AS p95
FROM merged
```

Use `kll_float_sketch_get_quantiles` (plural, array argument, 1-indexed result). The singular form is not reliably available.

Three properties of sketches that shape what you can build:

**There is no mean.** Sketches store no sum, so an exact average is not computable. Averaging several quantiles approximates it badly — the low deciles of a latency distribution are near zero and drag it down. Report percentiles instead, or go to the raw inference data in genai-engine.

**High quantiles jitter between runs.** Merging is order-dependent and KLL carries ~1% rank error, which on a long-tailed distribution means p95 can move 15% run to run. Observed: 82 then 95 on identical inputs. Do not build a percent-change card on p95. Lower quantiles are stabler; rounding to the nearest 10 hides some of it; a time-series chart tolerates the noise better than a stat card.

**Guardrail medians are often 0.** Regex and PII evaluations complete in under a millisecond and dominate the volume, so p50 across all rule types is 0 and any rate built on it divides by zero. Filter to the slow rule type, or use p95.

## Alert rules

An alert query must return `metric_timestamp` (timestamptz), `metric_value` (numeric), contain both `{{dateStart}}` and `{{dateEnd}}`, and use `{{interval}}` as the first argument to `time_bucket`.

```sql
SELECT
    time_bucket('{{interval}}', timestamp) AS metric_timestamp,
    SUM(CASE WHEN dimensions->>'result' = 'Fail' THEN value ELSE 0 END)
          / NULLIF(SUM(value), 0) AS metric_value
FROM metrics_numeric_latest_version
WHERE metric_name = 'inference_count'
    AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}'
GROUP BY metric_timestamp
ORDER BY metric_timestamp
```

Return a fraction, not a percentage, and set the threshold in the same units (`0.4`, not `40`). Do not round — rounding can flip a borderline comparison.

Alerts are scoped to their application by an injected `model_id` filter, the same mechanism the app-page charts use. Do not add one yourself.

Match the alert's metric to whatever the dashboard chart uses. An alert on `inference_count` beside a chart on `rule_count` will appear to fire at the wrong time, and the investigation wastes more time than picking correctly did.

## Anomaly detection

For "is today unusual", a z-score over daily rates is the cheap version:

```sql
WITH daily AS (
    SELECT time_bucket('1 day', timestamp) AS bucket,
           model_name,
           SUM(CASE WHEN dimensions->>'result' = 'Fail' THEN value ELSE 0 END)
             / NULLIF(SUM(value), 0) * 100 AS rate
    FROM metrics_numeric_latest_version
    WHERE metric_name = 'inference_count'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
    GROUP BY bucket, model_name
)
SELECT bucket, model_name, ROUND(rate::numeric, 2) AS rate,
       ROUND(AVG(rate) OVER w::numeric, 2) AS rolling_avg,
       ROUND(((rate - AVG(rate) OVER w) / NULLIF(STDDEV(rate) OVER w, 0))::numeric, 2) AS z_score
FROM daily
WINDOW w AS (PARTITION BY model_name ORDER BY bucket ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING)
ORDER BY bucket
```

It weights every day equally, so a 200-inference day and a 90,000-inference day carry the same influence — and the quiet day's noisy rate gets flagged as an anomaly. If daily volumes vary by more than roughly an order of magnitude, use the two-proportion z-test in `references/patterns.md` instead: it derives expected variation from sample sizes and correctly declines to flag low-volume swings.

Note `ROWS BETWEEN 7 PRECEDING` counts *rows*, not days. With gapped ingestion the baseline silently reaches further back than a week.

## Before building, check the data

Two queries that prevent most wasted work:

```sql
-- what metrics does this app actually emit
SELECT DISTINCT metric_name FROM metrics_numeric_latest_version

-- what values does a dimension take
SELECT DISTINCT dimensions->>'<key>' FROM metrics_numeric_latest_version
WHERE metric_name = '<metric>'
```

Dimension values are rarely what you would guess. `model_name` inside dimensions can be the literal string `'null'`, which needs `!= 'null'` rather than `IS NOT NULL`. Entity names come from the PII detector's own vocabulary. A metric that exists in `arthur-common` may emit nothing for a given app type.

`user_id` and `conversation_id` exist on shield metrics only when `INFERENCE_USER_CONVERSATION_SEGMENTATION=true` is set on the ml-engine process that computes them. Without it the pipeline drops the values even when the integration sends them, and every per-user chart is correctly empty.
