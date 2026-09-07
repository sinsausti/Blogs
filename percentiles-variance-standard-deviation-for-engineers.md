---
title: "Percentiles, Variance, and Standard Deviation for Engineers"
description: "Interpret percentiles, variance, and standard deviation correctly when analyzing latency, workload, and database performance."
author: "Sebastian Insausti"
date: "2026-09-06"
tags: ["Infrastructure", "Monitoring"]
canonical_url: "https://insaustis.com/blog/percentiles-variance-standard-deviation-for-engineers.html"
---

# Percentiles, Variance, and Standard Deviation for Engineers

The average gives a center, but operations also needs spread and tail behavior. Percentiles, variance, and standard deviation answer different questions and should not be treated as interchangeable health scores.

## 1. Use Each Statistic for Its Question

- **Mean:** useful for totals, capacity calculations, and broad shifts.
- **Median (p50):** the middle observation; resistant to a few extremes.
- **Percentiles:** tail thresholds such as p95 or p99.
- **Variance:** average squared distance from the mean, in squared units.
- **Standard deviation:** square root of variance, in the original unit.

No single statistic describes a distribution. Put center, spread, sample count, errors, and time context together.

## 2. Interpret Percentiles Precisely

If p95 latency is 800 ms, 95% of observations are at or below 800 ms and 5% are above it. It does not mean that the slowest 5% all took 800 ms.

High percentiles need sufficient traffic. With only 20 requests, p99 depends on the extreme end of a tiny sample. Always display or filter by sample count.

## 3. Calculate a PostgreSQL Summary

```sql
SELECT
  count(*) AS samples,
  avg(duration_ms)::numeric(12,2) AS mean_ms,
  percentile_cont(0.50) WITHIN GROUP (ORDER BY duration_ms) AS p50_ms,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_ms,
  percentile_cont(0.99) WITHIN GROUP (ORDER BY duration_ms) AS p99_ms,
  var_samp(duration_ms)::numeric(14,2) AS sample_variance,
  stddev_samp(duration_ms)::numeric(12,2) AS sample_stddev
FROM query_samples
WHERE recorded_at >= now() - interval '1 hour'
  AND query_class = 'checkout';
```

PostgreSQL also provides population variants, `var_pop` and `stddev_pop`. Use sample statistics when rows represent a sample of a broader process; use population statistics when they are the complete population being described.

## 4. Know When Standard Deviation Misleads

Standard deviation is easiest to interpret for stable, roughly symmetric data. Production latency is commonly right-skewed, and rare delays can dominate it. Show percentiles and a histogram; consider IQR or MAD for robust spread.

Variance is useful in formulas but harder to communicate because its unit is squared. Latency variance is in milliseconds squared; standard deviation returns to milliseconds.

## 5. Aggregate Histograms Correctly

In Prometheus, classic histogram buckets are cumulative counters. Aggregate bucket rates across compatible instances first, then calculate the quantile. Never average per-instance p95 values as a fleet p95.

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(db_query_duration_seconds_bucket[5m])
  )
)
```

Estimates depend on bucket boundaries. Choose boundaries around meaningful objectives and do not merge unrelated workloads merely because they share a metric name.

## 6. Build a Compact Operational View

For latency, show request count, error rate, mean, p50, p95, p99, and a histogram or heatmap. Segment by query class or endpoint and annotate changes. Use means for resource planning, percentiles for tail experience, and robust measures for skewed spread.

> Statistics summarize observations; operational context determines what those summaries mean.
