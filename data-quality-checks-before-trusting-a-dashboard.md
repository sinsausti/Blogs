---
title: "Data Quality Checks Before Trusting a Dashboard"
description: "Validate freshness, completeness, uniqueness, units, and collection behavior before making operational decisions from a dashboard."
author: "Sebastian Insausti"
date: "2026-08-18"
tags: ["Infrastructure", "Monitoring"]
canonical_url: "https://insaustis.com/blog/data-quality-checks-before-trusting-a-dashboard.html"
---

# Data Quality Checks Before Trusting a Dashboard

A polished dashboard can be precisely wrong. Before treating a graph as evidence, verify that the underlying observations are recent, complete, correctly labeled, and measured with consistent semantics.

## 1. Start with a Data Contract

Write down what each metric means: source, unit, type, labels, collection interval, expected delay, and owner. A counter such as `queries_total` should only increase except after a process restart; a gauge such as `connections_active` may move in either direction.

- Define whether timestamps represent event time or collection time.
- Record whether latency is measured client-side, server-side, or at a proxy.
- Specify which failures, retries, and timeouts are included.
- Document label values and expected cardinality.

## 2. Check Freshness and Coverage

A healthy-looking line may simply be stale. Display the newest sample timestamp and alert on collection gaps separately from service failures. Compare the expected number of samples with the number received.

```sql
SELECT
  max(recorded_at) AS newest_sample,
  now() - max(recorded_at) AS ingestion_delay,
  count(*) FILTER (
    WHERE recorded_at >= now() - interval '1 hour'
  ) AS samples_last_hour
FROM infrastructure_metrics
WHERE host = 'db-01';
```

If one sample is expected every minute, approximately 60 rows should exist for the last hour. Account for the current partial interval and planned maintenance instead of requiring an exact count.

## 3. Test Completeness and Uniqueness

Nulls, duplicate events, and silently missing hosts change aggregates. A duplicate is defined by the data contract, not necessarily by comparing every column.

```sql
SELECT host, metric_name, recorded_at, count(*)
FROM infrastructure_metrics
GROUP BY host, metric_name, recorded_at
HAVING count(*) > 1;

SELECT
  count(*) FILTER (WHERE metric_value IS NULL) AS null_values,
  count(*) FILTER (WHERE recorded_at > now() + interval '1 minute') AS future_rows
FROM infrastructure_metrics;
```

## 4. Validate Units, Ranges, and Labels

Milliseconds interpreted as seconds create a thousand-fold error. Percentages may arrive as either 0–1 or 0–100. Validate units at ingestion and reject impossible values such as negative durations or unknown environments.

- Compare label sets with inventory: are all production database hosts present?
- Look for label growth caused by request IDs, raw SQL, or user IDs.
- Convert counters with a rate function before graphing per-second activity.
- Keep missing values distinct from real zeroes.

## 5. Reconcile with an Independent Source

Spot-check the dashboard against the database, operating system, load balancer, or raw logs for the same time window. Differences are not automatically defects—the sources may measure different boundaries—but they must be explainable.

Compare application errors with proxy status codes and database connection failures. Compare exporter uptime with scrape success. Use UTC internally and make the displayed timezone visible when matching events.

## 6. Make Quality Visible

Treat telemetry collection as a production pipeline. Add panels or alerts for scrape failures, ingestion delay, rejected records, missing targets, and sudden cardinality changes. Rerun the checks after collector, schema, or instrumentation changes.

> A dashboard is trustworthy only when you can also observe the pipeline that produced it.
