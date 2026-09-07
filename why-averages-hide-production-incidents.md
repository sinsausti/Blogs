---
title: "Why Averages Hide Production Incidents"
description: "Learn why average latency can hide production incidents and how percentiles, histograms, segmentation, and error rates reveal the real user experience."
author: "Sebastian Insausti"
date: "2026-09-03"
tags: ["Infrastructure", "Monitoring"]
canonical_url: "https://insaustis.com/blog/why-averages-hide-production-incidents.html"
---

# Why Averages Hide Production Incidents

The dashboard says average latency is 120 ms, comfortably below the 200 ms target. Support tickets still report timeouts. Both observations can be true: most requests are fast enough to pull the average down while a smaller, important group waits several seconds.

> An average answers “what was the total divided by the count?” It does not show how the individual observations were distributed.

## 1. The Incident Hidden Inside the Mean

Imagine 100 database requests. Ninety-five complete in 100 ms and five take 4 seconds. The average is 295 ms. That sounds degraded, but it still hides the operational reality: 95% of requests are healthy while 5% are catastrophically slow.

A second service could average the same 295 ms because every request takes roughly that long. The two services have the same mean and completely different failure modes. The first suggests tail latency, contention, retries, a slow replica, or one expensive query class. The second suggests a broad capacity or dependency problem.

## 2. Add Percentiles

A percentile reports the value below which a percentage of observations falls. If p95 latency is 800 ms, 95% of measured requests completed in 800 ms or less; the remaining 5% were slower. It does not mean that every request in the slowest 5% took 800 ms.

- **p50:** the median or typical request.
- **p95:** exposes a slow minority without focusing only on extremes.
- **p99:** shows deeper tail behavior, but needs enough traffic to be stable.
- **Maximum:** useful for investigation, but often too noisy for alerting by itself.

Percentiles should match a user-facing objective. A p95 target explicitly accepts that up to 5% of observations may exceed it, so a critical workflow may need a stricter percentile, an error-rate objective, or both.

## 3. Calculate Them in PostgreSQL

Given a table containing one row per request, PostgreSQL can calculate exact continuous percentiles:

```
SELECT
  date_trunc('minute', recorded_at) AS minute,
  count(*) AS requests,
  avg(duration_ms)::numeric(10,2) AS avg_ms,
  percentile_cont(0.50) WITHIN GROUP (ORDER BY duration_ms) AS p50_ms,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_ms,
  percentile_cont(0.99) WITHIN GROUP (ORDER BY duration_ms) AS p99_ms,
  max(duration_ms) AS max_ms
FROM request_latency
WHERE recorded_at >= now() - interval '1 hour'
GROUP BY 1
ORDER BY 1;
```

Exact percentiles require raw observations and can be expensive over large windows. In production, pre-aggregate appropriately or use a telemetry system designed to retain distributions. Do not average percentiles from several servers or time windows: the result is not the percentile of the combined requests.

## 4. Keep the Distribution

Percentiles summarize the distribution but do not show its shape. A histogram can reveal two clusters—for example, cache hits around 20 ms and cache misses around 600 ms—or a long tail caused by lock waits.

With Prometheus histogram buckets, calculate a fleet-wide p95 by aggregating cumulative bucket counters first:

```
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

Bucket boundaries determine the resolution of the estimate. Choose them around meaningful latency targets, and aggregate only comparable requests. Mixing batch jobs with interactive traffic may produce a number that represents neither workload well.

## 5. Segment Before You Conclude

A global metric can hide a regional, tenant-specific, endpoint-specific, or database-node-specific incident. Break latency down by dimensions that can identify an owner or failure domain:

- endpoint or query class;
- region, availability zone, or host;
- primary versus replica;
- success versus error status;
- application version or deployment cohort.

Keep label cardinality under control. Raw user IDs, query text, and request IDs do not belong in metric labels; use logs or traces for high-cardinality investigation.

## 6. Pair Latency with Errors and Load

Latency alone is incomplete. A fast failed request still produces a good latency value. During overload, timeouts may disappear from a server-side duration metric if they occur upstream. Review latency together with request rate, error rate, timeout count, retries, saturation, and queue depth.

For databases, correlate the interval with active sessions, lock waits, connection-pool utilization, disk latency, replication lag, slow query classes, and deployment events. Correlation narrows the investigation; it does not prove that one metric caused another.

## 7. Build an Alert That Reflects Impact

A useful alert combines a meaningful threshold, a sustained evaluation window, and enough traffic to make the signal credible. For example: page when p95 latency exceeds the service objective for ten minutes and the request rate is above a minimum volume. Add a separate alert for error-budget consumption or elevated timeouts.

Do not remove the average. It remains useful for capacity calculations and detecting broad shifts. Put it beside percentiles, distributions, errors, and segmented views so that one compact statistic cannot declare a system healthy on its own.

---

The best monitoring question is not “What is the average?” but “Which users or requests are suffering, how badly, and where does that behavior begin?” Preserve enough of the distribution to answer it.
