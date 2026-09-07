---
title: "Exploratory Data Analysis for Infrastructure Metrics"
description: "Use a repeatable exploratory workflow to understand infrastructure metrics before creating dashboards, thresholds, or alerts."
author: "Sebastian Insausti"
date: "2026-08-28"
tags: ["Infrastructure", "Monitoring"]
canonical_url: "https://insaustis.com/blog/exploratory-data-analysis-infrastructure-metrics.html"
---

# Exploratory Data Analysis for Infrastructure Metrics

Before defining an alert threshold, learn how the signal behaves. Exploratory data analysis (EDA) helps distinguish normal cycles, collection defects, workload changes, and relationships worth investigating.

## 1. Define Scope and Grain

Choose one question, a range that includes normal and busy periods, and a consistent observation grain. For example: “What drives PostgreSQL connection saturation during weekday peaks?” Export connections, pool utilization, transaction rate, query latency, CPU, disk latency, and deployment annotations at one-minute resolution.

Do not mix per-host values with fleet totals without labeling the aggregation. Record timezone, scrape interval, source, and known maintenance.

## 2. Profile the Dataset

Start with row count, time coverage, missingness, distinct hosts, and descriptive statistics. This catches collection issues before they become conclusions.

```sql
SELECT
  host,
  count(*) AS samples,
  min(recorded_at) AS first_sample,
  max(recorded_at) AS last_sample,
  count(*) FILTER (WHERE metric_value IS NULL) AS missing,
  min(metric_value) AS minimum,
  percentile_cont(0.5) WITHIN GROUP (ORDER BY metric_value) AS median,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY metric_value) AS p95,
  max(metric_value) AS maximum
FROM metric_samples
WHERE metric_name = 'db_connections_active'
GROUP BY host;
```

## 3. Plot Time Before Distribution

First graph the metric over time. Look for trends, step changes, periodic patterns, gaps, flat lines, restarts, and spikes. Add deployment and maintenance annotations. Then inspect a histogram or percentiles.

- A flat zero may be a stopped workload or a failed collector.
- A step change may follow configuration, capacity, or instrumentation changes.
- A daily pattern may need time-aware thresholds.
- A bimodal distribution may contain two workloads that should be separated.

## 4. Segment by Failure Domain

Aggregate views orient the investigation but can hide one unhealthy host, region, database, or query class. Compare the same metric by a small set of meaningful dimensions and avoid high-cardinality monitoring labels.

Useful database segments include primary versus replica, cluster, instance, query class, application, and outcome. Compare rates with rates and gauges with gauges; do not sum values whose semantics do not support it.

## 5. Explore Relationships Carefully

Align signals to the same timestamps. Rising connections alongside longer query latency and lock waits is a useful lead, not proof of causation. Traffic mix, retries, deployments, or a shared dependency may affect all three.

Test the hypothesis with query plans, locks, system telemetry, logs, and controlled changes. Consider lag: saturation may precede application latency, while retries may increase traffic after errors begin.

## 6. Produce Testable Actions

Finish with observations, open questions, and proposed actions. A useful result might be a segmented panel, a collector-quality alert, or a candidate threshold evaluated against historical incidents. Backtest alerts before paging anyone.

> EDA should reduce uncertainty and generate hypotheses; it should not manufacture certainty from correlated graphs.
