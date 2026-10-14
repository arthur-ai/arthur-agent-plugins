# Query patterns

Working queries to adapt. All use `[[ ]]` date filters and carry no trailing semicolon.

## Failure rate over time, per application

```sql
SELECT
    time_bucket('1 day', timestamp) AS bucket,
    model_name AS application,
    SUM(CASE WHEN dimensions->>'result' = 'Fail' THEN value ELSE 0 END) AS failures,
    SUM(value) AS inferences,
    ROUND((100.0 * SUM(CASE WHEN dimensions->>'result' = 'Fail' THEN value ELSE 0 END)
         / NULLIF(SUM(value), 0))::numeric, 2) AS failure_rate
FROM metrics_numeric_latest_version
WHERE metric_name = 'inference_count'
    [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
GROUP BY bucket, model_name
ORDER BY bucket
```

Line chart, `application` as the series. Drop `model_name` for a single combined line.

## Per-rule-instance table, including rules that never fired

The `LEFT JOIN` from evaluations is what keeps silent rules visible — a rule with zero failures has no row in `failures`, and an inner join would hide exactly the rule worth noticing.

```sql
WITH total_evaluations AS (
    SELECT dimensions->>'name' AS rule_name,
           dimensions->>'rule_type' AS rule_type,
           SUM(value) AS total_evaluations
    FROM metrics_numeric_latest_version
    WHERE metric_name = 'rule_count'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
    GROUP BY rule_name, rule_type
), failures AS (
    SELECT dimensions->>'name' AS rule_name,
           dimensions->>'rule_type' AS rule_type,
           SUM(value) AS total_failures,
           SUM(CASE WHEN dimensions->>'location' = 'prompt' THEN value ELSE 0 END) AS on_prompts,
           SUM(CASE WHEN dimensions->>'location' = 'response' THEN value ELSE 0 END) AS on_responses
    FROM metrics_numeric_latest_version
    WHERE metric_name = 'rule_count'
        AND dimensions->>'result' = 'Fail'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
    GROUP BY rule_name, rule_type
)
SELECT
    regexp_replace(total_evaluations.rule_type, 'Rule$', '') || ': ' ||
        regexp_replace(total_evaluations.rule_name, '^<app-prefix>-', '') AS rule,
    COALESCE(total_failures, 0) AS failures,
    COALESCE(on_prompts, 0) AS on_prompts,
    COALESCE(on_responses, 0) AS on_responses,
    total_evaluations.total_evaluations AS evaluations,
    ROUND((COALESCE(total_failures, 0) / total_evaluations.total_evaluations * 100)::numeric, 2) AS failure_percent
FROM total_evaluations LEFT JOIN failures
    ON failures.rule_name = total_evaluations.rule_name
    AND failures.rule_type = total_evaluations.rule_type
ORDER BY failures DESC
```

Rule names often carry an application prefix; the `regexp_replace` strips it for readability. Group by `id` as well if names are reused or edited.

## Sketch percentiles per bucket and application

```sql
SELECT
    bucket,
    model_name AS application,
    ROUND((kll_float_sketch_get_quantiles(sketch, ARRAY[0.95]))[1]::numeric, 1) AS p95_ms
FROM (
    SELECT time_bucket('1 day', timestamp) AS bucket,
           model_name,
           kll_float_sketch_merge(value) AS sketch
    FROM metrics_sketch_latest_version
    WHERE metric_name = 'rule_latency'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
    GROUP BY bucket, model_name
) x
ORDER BY bucket
```

The subquery is required: the quantile function cannot be applied to the merge result in the same SELECT as its GROUP BY.

## Sketch spread check

Run before building any distribution chart. Confirms the metric carries real variation rather than a constant.

```sql
WITH merged AS (
    SELECT kll_float_sketch_merge(value) AS sketch
    FROM metrics_sketch_latest_version
    WHERE metric_name = '<metric>'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
)
SELECT
    kll_float_sketch_get_n(sketch) AS n,
    (kll_float_sketch_get_quantiles(sketch, ARRAY[0.0, 0.5, 0.95, 1.0]))[1] AS min,
    (kll_float_sketch_get_quantiles(sketch, ARRAY[0.0, 0.5, 0.95, 1.0]))[2] AS p50,
    (kll_float_sketch_get_quantiles(sketch, ARRAY[0.0, 0.5, 0.95, 1.0]))[3] AS p95,
    (kll_float_sketch_get_quantiles(sketch, ARRAY[0.0, 0.5, 0.95, 1.0]))[4] AS max
FROM merged
```

`n = 0` means no data in range. min = p50 = max means the values are constant and the chart will say nothing.

## Period-over-period comparison

Follows the date picker's span rather than a fixed week, so the comparison length matches whatever is selected.

```sql
WITH current_period AS (
    SELECT COALESCE(SUM(value), 0) AS v
    FROM metrics_numeric_latest_version
    WHERE metric_name = 'token_cost'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
),
previous_period AS (
    SELECT COALESCE(SUM(value), 0) AS v
    FROM metrics_numeric_latest_version
    WHERE metric_name = 'token_cost'
        [[AND timestamp BETWEEN '{{dateStart}}'::timestamp - ('{{dateEnd}}'::timestamp - '{{dateStart}}'::timestamp)
                             AND '{{dateStart}}']]
)
SELECT ROUND(((current_period.v - previous_period.v)
     / NULLIF(previous_period.v, 0) * 100)::numeric, 1) AS percent_change
FROM current_period, previous_period
```

No outer `COALESCE` — a missing baseline renders blank rather than a misleading `0%`. For a fixed week-over-week card instead, anchor both windows to `{{dateEnd}}` with `INTERVAL '7 days'` and `'14 days'`; note the card then ignores the picker's span.

## Two-proportion z-test

Use instead of the plain z-score when daily volumes vary widely. Derives expected variation from sample sizes, so a low-volume day with a swingy rate correctly scores low.

```sql
WITH daily AS (
    SELECT time_bucket('1 day', timestamp) AS bucket,
           model_name,
           SUM(CASE WHEN dimensions->>'result' = 'Fail' THEN value ELSE 0 END) AS failures,
           SUM(value) AS total
    FROM metrics_numeric_latest_version
    WHERE metric_name = 'inference_count'
        [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
    GROUP BY bucket, model_name
),
with_baseline AS (
    SELECT bucket, model_name, failures, total,
           SUM(failures) OVER w AS base_failures,
           SUM(total)    OVER w AS base_total
    FROM daily
    WINDOW w AS (PARTITION BY model_name ORDER BY bucket ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING)
),
pooled AS (
    SELECT *,
           failures::numeric / NULLIF(total, 0) AS p1,
           base_failures::numeric / NULLIF(base_total, 0) AS p2,
           (failures + base_failures)::numeric / NULLIF(total + base_total, 0) AS p_pool
    FROM with_baseline
)
SELECT bucket,
       model_name,
       ROUND((100 * p1)::numeric, 2) AS rate,
       ROUND((100 * p2)::numeric, 2) AS baseline_rate,
       ROUND(((p1 - p2) / NULLIF(SQRT(p_pool * (1 - p_pool)
            * (1.0/NULLIF(total,0) + 1.0/NULLIF(base_total,0))), 0))::numeric, 2) AS z_score
FROM pooled
ORDER BY bucket
```

|z| > 2 notable, |z| > 3 unusual, sign gives direction. First row per application is null — no baseline.

Caveat in the other direction: at very high volume the standard error is tiny, so a half-point difference clears 3 sigma while meaning nothing operationally. Add a minimum effect size if that produces noise.

A chi-squared test on the same 2×2 table is equivalent (χ² = z²) but loses direction, and its threshold scales with sample size rather than staying interpretable. The z form is preferable for a dashboard.

## Day's share of range total

```sql
ROUND((100.0 * SUM(value) / NULLIF(SUM(SUM(value)) OVER (), 0))::numeric, 2) AS pct_of_total
```

`SUM(SUM(value)) OVER ()` totals across all result rows. Add `PARTITION BY model_name` to make each day's share relative to its own application.

This column is the quickest way to explain why a pooled rate disagrees with a chart's daily rates: a stat card is volume-weighted across the whole range, while each day on a chart is weighted independently. One high-volume day dominates the card without appearing as an outlier on the chart.

## Reference line

```sql
SELECT
    time_bucket('1 day', timestamp) AS bucket,
    ROUND((100.0 * SUM(CASE WHEN dimensions->>'result' = 'Pass' THEN value ELSE 0 END)
         / NULLIF(SUM(value), 0))::numeric, 2) AS success_rate,
    90 AS target
FROM metrics_numeric_latest_version
WHERE metric_name = 'inference_count'
    [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
GROUP BY bucket
ORDER BY bucket
```

A constant column renders as a flat second series. Use this when the chart type has no built-in threshold line.

## Stacked composition over time

```sql
SELECT
    time_bucket('1 hour', timestamp) AS bucket,
    dimensions->>'rule_type' AS rule_type,
    SUM(value) AS failures
FROM metrics_numeric_latest_version
WHERE metric_name = 'rule_count'
    AND dimensions->>'result' = 'Fail'
    [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
GROUP BY bucket, rule_type
ORDER BY bucket
```

Stacked bar, `rule_type` as the series. Bucket size against `LIMIT 2000`: 5 minutes for hours of data, 1 hour for days, 1 day for weeks — and the series dimension multiplies the row count.

Grouping by `rule_type` merges rules of the same class, so an inference tripping two regex rules contributes 2. Group by `name` to keep instances separate.

## Series with an explicit total

```sql
SELECT time_bucket('1 day', timestamp) AS bucket,
       dimensions->>'entity' AS entity,
       kll_float_sketch_get_n(kll_float_sketch_merge(value)) AS detections
FROM metrics_sketch_latest_version
WHERE metric_name = 'pii_score'
    [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
GROUP BY bucket, entity

UNION ALL

SELECT time_bucket('1 day', timestamp) AS bucket,
       'TOTAL' AS entity,
       kll_float_sketch_get_n(kll_float_sketch_merge(value)) AS detections
FROM metrics_sketch_latest_version
WHERE metric_name = 'pii_score'
    [[AND timestamp BETWEEN '{{dateStart}}' AND '{{dateEnd}}']]
GROUP BY bucket

ORDER BY bucket
```

Keep both halves' filters identical or the total will not match the lines. The total merges sketches rather than summing per-entity counts, which is the correct way to combine them.

A stacked area chart gives the total as stack height without the extra series, and avoids the TOTAL line compressing the others against the axis.
