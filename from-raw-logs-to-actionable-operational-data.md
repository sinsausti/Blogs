---
title: "From Raw Logs to Actionable Operational Data"
description: "Turn structured application and database logs into reliable operational metrics, investigations, and alerts."
author: "Sebastian Insausti"
date: "2026-08-26"
tags: ["Infrastructure", "Monitoring"]
canonical_url: "https://insaustis.com/blog/from-raw-logs-to-actionable-operational-data.html"
---

# From Raw Logs to Actionable Operational Data

Raw logs are evidence, not yet an operational view. A useful pipeline preserves the original event, extracts stable fields, validates them, and produces measurements that answer a specific question.

## 1. Begin with One Question

Start narrowly: “Which database errors increased after the deployment?” or “Which query class caused slow requests?” The question determines the fields and retention you need. Collecting everything without a purpose raises cost and slows investigations.

## 2. Prefer Structured Events

Emit JSON at the source when possible. Include event timestamp, severity, service, environment, host, event name, duration, outcome, and correlation ID. Never log passwords, tokens, connection strings, or sensitive query parameters.

```json
{"timestamp":"2026-08-26T14:05:12.381Z","level":"error","service":"orders-api","environment":"production","event":"db_query","db_system":"postgresql","query_class":"create_order","duration_ms":842,"outcome":"timeout","trace_id":"4f9c..."}
```

Use a normalized query class or fingerprint instead of full SQL as a metric label. Preserve detailed statements only where redaction, access control, and retention policies permit it.

## 3. Inspect and Filter Locally

For a systemd service emitting JSON, `journalctl` and `jq` provide a fast first investigation without modifying the central logging system.

```bash
sudo journalctl -u orders-api.service \
  --since '2026-08-26 14:00:00 UTC' \
  --until '2026-08-26 15:00:00 UTC' \
  -o cat \
| jq -r 'select(.event == "db_query" and .outcome != "success")
  | [.timestamp, .query_class, .outcome, .duration_ms] | @tsv'
```

Record the time window and timezone. Save the original export read-only before transforming it, and record the command used to create derived data.

## 4. Normalize and Validate

At ingestion, parse timestamps, map severity consistently, coerce numeric fields, and route malformed events to a dead-letter destination. Monitor parse failures: silently dropping a new format can make an incident disappear.

- Reject or quarantine invalid timestamps.
- Distinguish absent duration from a real zero.
- Deduplicate retries with a stable event ID when available.
- Measure ingestion delay and source coverage.
- Retain raw events long enough to reprocess parser changes.

## 5. Aggregate Without Losing Context

Create metrics for repeated questions: error rates, duration distributions, and event counts by normalized class. Metrics are efficient for alerting; logs retain event context; traces connect work across services.

```sql
SELECT
  date_trunc('minute', event_time) AS minute,
  query_class,
  count(*) AS executions,
  count(*) FILTER (WHERE outcome <> 'success') AS failures,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95_ms
FROM normalized_db_events
WHERE event_time >= now() - interval '1 hour'
GROUP BY 1, 2
ORDER BY 1, 2;
```

## 6. Turn Results into Operations

Build a small dashboard around traffic, failures, latency distribution, and affected query classes. Link panels to filtered logs with the same service, host, and time window. Alert on sustained user impact and attach a validation runbook.

> The pipeline succeeds when an operator can move from an alert to the relevant raw events without guessing.
