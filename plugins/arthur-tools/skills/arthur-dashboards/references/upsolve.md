# Upsolve wrapper and failure modes

Charts are rendered by the Upsolve SDK embedded through `components/shared/ModelDashboard/UpsolveDashboard.tsx`. Your SQL is not executed as written — it is wrapped, filtered and appended to before reaching Postgres. Most blank charts are a consequence of that wrapper rather than the query.

## What the wrapper does

Your query becomes the first CTE of a larger statement:

```sql
WITH upsolve_user_query AS (
<your query>
),
upsolve_user_filtered_query AS (SELECT * FROM upsolve_user_query),
upsolve_filtered_query AS (SELECT * FROM upsolve_user_filtered_query)
SELECT * FROM upsolve_filtered_query LIMIT 2000
```

Three consequences:

**A trailing semicolon is a syntax error.** It terminates the statement inside the CTE parenthesis. Preview does not wrap the query, so the semicolon is harmless there and the chart fails only on the dashboard — the textbook "works in preview, broken on the dashboard" report.

**`LIMIT 2000` is appended.** A query returning more rows is silently truncated, and the chart undercounts with no warning. 5-minute buckets over a multi-day range exceed it easily: a 14-day range is ~4,000 buckets before any grouping dimension multiplies that. Coarsen the bucket.

**The source table is replaced by a filtered subquery.** On an application dashboard it becomes roughly:

```sql
FROM (SELECT ... FROM "public"."metrics_numeric_latest_version"
      WHERE CAST("model_id" AS TEXT)='<uuid>') AS "metrics_numeric_latest_version"
```

So application scoping is injected, not something you write. The same query on a workspace dashboard has no `model_id` filter and spans every app in the token's scope — which is why an app-page query usually works unchanged at workspace level, just with a wider denominator.

Timezone conversion is also injected there (`AT TIME ZONE`), using the viewer's offset.

## Template variables

`{{dateStart}}` and `{{dateEnd}}` are substituted before execution — when the chart's variables are bound.

`[[ ... ]]` marks a block that is **deleted** when its variable is unbound, rather than left as literal text. That is the whole mechanism, and it explains the two failure shapes:

- A date filter inside `[[ ]]` on an unbound chart: block removed, query runs **unfiltered**. The chart renders and looks fine, but ignores the date picker entirely.
- The same filter as a bare `AND`: literal `{{dateStart}}` reaches Postgres as `invalid input syntax for type timestamp with time zone: "{{dateStart}}"`.

A chart whose numbers do not move when the date range changes is the first symptom. Confirm by reading the request payload: if `sqlQuery` still contains `{{dateStart}}`, or the `[[ ]]` block has collapsed to whitespace, the variables are not binding on that chart.

Binding is per-chart configuration in Upsolve's UI, not something SQL can work around. A chart created fresh does not inherit it from the dashboard. Pasting SQL from a working chart into a new one carries the text but not the binding — a common cause of "identical SQL, one works".

## Gapfill cannot be made optional

`time_bucket_gapfill(interval '1 day', timestamp, start, finish)` requires start and finish, and TimescaleDB requires them to be **simple expressions** — literal constants or bound parameters. A subquery raises:

```
invalid time_bucket_gapfill argument: start must be a simple expression
```

So there is no fallback. `[[ ]]` cannot wrap a function argument (removing it breaks the call), and bounds cannot be computed from the data. On a chart where the date variables bind, gapfill works; on one where they do not, gapfill is impossible without hardcoded dates.

When gapfill is unavailable, plain `time_bucket` omits empty buckets rather than zero-filling them. For a z-score or rolling-average chart that is arguably better: gapfilled zeros enter the baseline as real observations and corrupt it across an ingestion gap.

## Diagnosing a blank chart

The banner reads "Unable to load chart data. Please try again later." regardless of cause. Get the real error from the network tab: find the `run-sql-unified` request and read the response body, which carries the Postgres message and the offending line.

The request payload is also worth reading directly. It shows the fully wrapped SQL, whether variables were substituted, the injected `model_id` list, and the attached filters.

In order of likelihood:

1. Trailing semicolon — syntax error inside the CTE.
2. Unbound date variables — literal `{{...}}` in the timestamp comparison.
3. `LIMIT 2000` truncation — chart renders but undercounts.
4. Metric does not exist for this app type — query is valid, returns nothing.
5. Dimension value mismatch — e.g. filtering `IS NOT NULL` where the value is the string `'null'`.

## SQL dialect notes

Postgres with TimescaleDB and the datasketches extension.

- `ROUND(x, 2)` needs `x::numeric`. There is no `round(double precision, integer)`.
- `ORDER BY` on a quoted alias can be rejected; ordinals (`ORDER BY 4 DESC`) always work.
- `NOT IN` takes a list; against an array use `!= ALL(...)`.
- A variable interpolating a list arrives raw. `in ({{entities}})` with a value of `PERSON, EMAIL` produces unquoted identifiers and `column "person" does not exist`. Quoting inside the variable value tends to double up. More robust:
  ```sql
  AND dimensions->>'entity' = ANY(string_to_array(replace('{{entities}}', ' ', ''), ','))
  ```
  with a plain unquoted `PERSON,EMAIL` as the value.

## Which dashboards persist edits

Relevant when someone reports losing chart changes.

The Dashboard tab on an application page renders one of six **global template dashboards**, selected by problem type from `VITE_UPSOLVE_*_DASHBOARD_ID` env vars (`components/shared/ModelDashboard/ModelDashboard.tsx`). Every application of that type in the deployment shares the same dashboard object; only the token's data scope differs. Edits there are not per-application and `canEdit` is permission-gated.

Workspace and project template dashboards are likewise shared, and both pass `canEdit={false}`.

Custom dashboards under Analytics are real Arthur rows (`DashboardsV1`, `hooks/useDashboard.ts`) with their own id, permissions, and a `project_ids` member set defining scope. `canEdit` comes from `DashboardUpdate` on that row. **This is the only surface where chart edits persist.** Clone a template from the Analytics sidebar to get an editable copy.

Visibility (`PRIVATE` / `WORKSPACE`) and scope (project / workspace) are independent axes set at creation: visibility controls who sees the dashboard, scope controls which applications' data it covers.
