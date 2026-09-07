---
title: "Detecting Outliers Without Hiding Real Incidents"
description: "Detect unusual production behavior with robust statistics while preserving the anomalies that may represent real incidents."
author: "Sebastian Insausti"
date: "2026-08-19"
tags: ["Infrastructure", "Monitoring"]
canonical_url: "https://insaustis.com/blog/detecting-outliers-without-hiding-real-incidents.html"
---

# Detecting Outliers Without Hiding Real Incidents

Outliers are not dirty data by default. A five-second query among thousands of 20 ms queries may be a measurement error, but it may also be the only visible evidence of lock contention or a failing storage path.

## 1. Define the Operational Question

Decide whether you are cleaning data for capacity analysis, detecting incidents, or measuring user experience. Removing extremes may help estimate normal demand, but it is dangerous when the objective is reliability.

Keep raw observations immutable. Add a classification or filtered view instead of deleting records so another analyst can reproduce the original and cleaned results.

## 2. Avoid a Fixed Mean-Based Rule

Rules such as “more than three standard deviations from the mean” work best with stable, roughly symmetric distributions. Latency, query duration, and queue depth are often skewed and long-tailed. Extreme values move both the mean and standard deviation.

For skewed data, compare percentiles, the interquartile range (IQR), or median absolute deviation (MAD). Segment first: a backup job and an interactive API should not share one baseline.

## 3. Use IQR as a Robust Baseline

Calculate Q1 and Q3 for a comparable window. The IQR is `Q3 - Q1`; a common exploratory rule flags values below `Q1 - 1.5 × IQR` or above `Q3 + 1.5 × IQR`. These are investigation candidates, not proven defects.

```sql
WITH limits AS (
  SELECT
    percentile_cont(0.25) WITHIN GROUP (ORDER BY duration_ms) AS q1,
    percentile_cont(0.75) WITHIN GROUP (ORDER BY duration_ms) AS q3
  FROM query_samples
  WHERE recorded_at >= now() - interval '24 hours'
    AND query_class = 'checkout'
)
SELECT s.*
FROM query_samples AS s
CROSS JOIN limits AS l
WHERE s.query_class = 'checkout'
  AND s.recorded_at >= now() - interval '24 hours'
  AND (s.duration_ms < l.q1 - 1.5 * (l.q3 - l.q1)
       OR s.duration_ms > l.q3 + 1.5 * (l.q3 - l.q1));
```

## 4. Respect Time and Seasonality

Compare like with like. Monday morning traffic, nightly maintenance, and month-end reporting may need different baselines. A global threshold can flag normal batch work while missing a serious daytime regression.

- Use rolling baselines that exclude the point being evaluated.
- Compare the same hour and weekday when seasonality is strong.
- Require a minimum sample count before evaluating percentiles.
- Annotate deployments, failovers, backups, and maintenance.

## 5. Investigate Before Suppressing

For each flagged interval, check errors, traffic, saturation, lock waits, disk latency, replication lag, query plans, and changes. Repeated extremes across related signals are stronger evidence than one extreme sample.

Suppress a value only for a documented reason such as sensor failure, an impossible timestamp, or test traffic. Version the rule and measure how many records it affects.

## 6. Alert on Impact

An anomaly detector asks “is this unusual?” An operational alert must also ask “does this require action?” Combine anomaly signals with an SLO breach, error increase, sustained duration, or affected traffic volume.

> Flag unusual data, preserve it, add context, and let operational impact determine the response.
