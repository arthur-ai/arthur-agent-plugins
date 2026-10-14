# Metric reference

Source of truth is `arthur-common/src/arthur_common/aggregations/functions/` — `shield_aggregations.py` and `agentic_aggregations.py`. Each class declares `METRIC_NAME` and a `group_by_dims` list, and that list is exactly the keys available in `dimensions`. When a metric here disagrees with what a query returns, re-read those files; this reference is a snapshot.

A metric exists for an app only if the app's problem type matches the aggregation's declared `model_problem_type`. Shield metrics do not appear for agentic apps and vice versa, so a workspace-level chart spanning mixed app types silently covers a subset.

## Shield (guardrails)

All shield aggregations declare `model_problem_type=ModelProblemType.ARTHUR_SHIELD`.

| Metric | Table | Dimensions |
|---|---|---|
| `inference_count` | numeric | `result`, `prompt_result`, `response_result`, `model_name` |
| `rule_count` | numeric | `location`, `rule_type`, `result`, `name`, `id` |
| `rule_latency` | sketch | `result`, `rule_type`, `location`, `model_name` |
| `token_count` | numeric | `location`, `model_name` |
| `token_cost` | numeric | `location`, `model_name` |
| `pii_score` | sketch | `result`, `location`, `entity`, `model_name` |
| `toxicity_score` | sketch | `result`, `location`, `model_name` |
| `hallucination_count` | numeric | `model_name` |
| `claim_count` | sketch | `result`, `model_name` |
| `claim_valid_count` | sketch | `result`, `model_name` |
| `claim_invalid_count` | sketch | `result`, `model_name` |

All of them extend `group_by_dims` with `conversation_id` and `user_id` when `INFERENCE_USER_CONVERSATION_SEGMENTATION` is enabled.

### inference_count

One row per request. `result` is the overall outcome; `prompt_result` and `response_result` split it by stage. `value` is a count.

The headline failure rate comes from here. `result = 'Fail'` means at least one rule failed.

### rule_count

One row per rule evaluation, from unnesting `inference_prompt.prompt_rule_results` and `inference_response.response_rule_results`.

`location` is `'prompt'` or `'response'`. `rule_type` is the class (`PIIDataRule`, `PromptInjectionRule`, `RegexRule`, `ToxicityRule`). `name` and `id` identify the configured rule instance — use these to tell several regex rules apart, since `rule_type` collapses them.

`result` can include `'Partially Unavailable'` alongside Pass and Fail. Exclude it from rate calculations.

### rule_latency

Sketch of `latency_ms`, already in milliseconds. Do not divide.

p50 across all rule types is typically 0 — regex and PII complete sub-millisecond and dominate the volume. Filter to `rule_type = 'PromptInjectionRule'` or use p95 for a meaningful number.

### token_count and token_cost

`token_count` sums `inference_prompt.tokens` and `inference_response.tokens` per bucket, split by `location`.

`token_cost` prices those tokens via litellm at the rate for the model the inference reported, prompt tokens as input and response tokens as completion. Rows whose model is missing or unknown to litellm get no cost row at all, so cost silently undercounts for unrecognised models.

**Historical note.** Before `b1d0562` ("update token cost agg to not use hard-coded list"), this emitted one series per model in a hardcoded `SUPPORTED_MODELS` list, named `token_cost.<model>` — pricing every app's tokens against all ~22 models simultaneously. Summing `token_cost.%` across those is meaningless: the same tokens counted once per model at different rates. Deployments with metric versions computed before that commit still hold those series. If `SELECT DISTINCT metric_name ... LIKE 'token_cost%'` returns dotted names, you are looking at the old shape.

### pii_score

Sketch of detector confidence, one value per detected entity. `entity` is the PII type from the detector's vocabulary — check the actual values, they are not guessable.

**Only failures appear here.** A PII evaluation that detects nothing produces no row. There is no Pass population, so an entity-filtered *rate* is not computable from this metric: the numerator (detections) and denominator (evaluations, in `rule_count`) live in different tables with no shared key.

Related: you cannot determine which inferences were flagged *only* for a given entity. Each `pii_score` row is independent, with no inference id, so "what would the failure rate be without DATE_TIME" is answerable only as bounds, or by changing the rule config and measuring.

Confidence scores may be constant or unpopulated depending on the detector — the aggregation uses `TRY_CAST`, so nulls pass through silently. Verify spread before building a distribution chart:

```sql
SELECT entity, kll_float_sketch_get_quantiles(kll_float_sketch_merge(value), ARRAY[0.0, 0.5, 1.0])
FROM metrics_sketch_latest_version
WHERE metric_name = 'pii_score'
GROUP BY entity
```

Min = median = max means the scores carry no information.

## Agentic

Declared `model_problem_type=ModelProblemType.AGENTIC_TRACE`. These produce nothing for shield apps — a `span_latency` chart on a guardrails app is correctly empty, and that is the usual explanation for a blank agentic chart.

Trace level: `trace_count`, `trace_latency`, `annotation_count`, `annotation_cost_sum`, `annotation_cost_distribution`, and token sums and distributions (`total_`, `prompt_`, `completion_token_cost_sum` / `_distribution`, and the `token_count` equivalents).

Span level: `span_count`, `span_latency`, `tool_span_count`, `tool_span_latency`, `agent_span_count`, `llm_span_latency`, plus `span_`- and `llm_span_`-prefixed token cost and count series.

`total_token_cost_sum` groups only by `user_id` — there is no model dimension on it. Group by the `model_name` *column* for per-application cost.

## Generic aggregations

`categorical_count` and `numeric_sketch` are matched to columns by tag, via `aggregation_matcher.py` intersecting a parameter's `tag_hints` with a column's. They are not app-type specific in principle.

In practice they do not apply to shield apps. `SHIELD_SCHEMA` (in `arthur_common/models/schema_definitions.py`) has eleven fixed columns, none carrying tag hints, and the schema is fixed by the validate_prompt/validate_response contract rather than user-defined. There is nothing to tag.

This matters because the PSI drift charts on the stock dashboards are built on `categorical_count` and `numeric_sketch`. They are empty for guardrails apps and will stay empty. The drift question for guardrails is answered by failure rate over time instead.

## Metric versions

`metrics_numeric_latest_version` resolves to the newest computed version per timestamp. Recalculation writes a new version rather than updating in place, so a running ml-engine can change query results between executions.

Multiple versions coexisting over different time spans is normal — each metrics-calculation job covers the range it was asked for. `max_version_for_timestamp` is available if you need to pin to the newest per timestamp explicitly.

Metrics are computed in batches, never live. A dimension added by a config change (such as enabling user segmentation) appears only on versions computed after the change; existing rows keep their old shape and no recompute of a *different* time range will backfill them.
