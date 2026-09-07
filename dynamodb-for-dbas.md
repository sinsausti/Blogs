---
title: "DynamoDB for DBAs: When to Use It and How to Operate It"
description: "A concise DBA guide to DynamoDB workload fit, data modeling, capacity, topology, monitoring, backups, and recovery."
author: "Sebastian Insausti"
date: "2026-09-01"
tags: ["Databases", "Infrastructure"]
canonical_url: "https://insaustis.com/blog/dynamodb-for-dbas.html"
---

# DynamoDB for DBAs: When to Use It and How to Operate It

Amazon DynamoDB is a fully managed key-value and document database. There are no servers to patch or replicas to promote, but that does not remove database work: a DBA still needs to understand access patterns, partition behavior, capacity, security, cost, and recovery.

## 1. When DynamoDB Is a Good Fit

DynamoDB works best when an application has known, repeatable access patterns and needs predictable low latency at almost any scale. Common examples include:

- Sessions, shopping carts, user profiles, and application state.
- Event metadata, device state, gaming data, and high-volume APIs.
- Serverless applications where capacity changes quickly.
- Multi-Region services that require local reads and writes.

It is usually a poor fit for workloads built around joins, ad hoc reporting, frequent full-table scans, or query requirements that change constantly. In those cases, a relational database or an analytical platform may be simpler.

## 2. Model Access Patterns First

In DynamoDB, table design starts with the queries—not with normalized entities. Every table has a partition key and may also have a sort key. Choose a high-cardinality partition key that distributes activity evenly; a popular key can create a hot partition and throttling.

```
PK          SK
USER#1042   PROFILE
USER#1042   ORDER#2026-0001
USER#1042   ORDER#2026-0002
```

This composite-key pattern keeps related items together and supports targeted `Query` operations. Add a global secondary index only for a defined alternate access pattern: each index adds storage and write cost. Application paths should normally use `GetItem` or `Query`, not `Scan`.

## 3. Capacity and Recommended Topology

Start with **on-demand capacity** when traffic is new, unpredictable, or operational simplicity matters most. Provisioned capacity with auto scaling can be more cost-effective after the workload becomes predictable. Monitor both the table and every secondary index.

A standard single-Region table is already distributed across multiple Availability Zones. Use **Global Tables** only when the application stack and recovery design are also multi-Region. Multi-Region eventual consistency is the usual choice; multi-Region strong consistency has stricter Region support and latency trade-offs. The consistency mode cannot be changed after the global table is created.

With eventual consistency, concurrent updates use last-writer-wins conflict resolution. Prefer a clear write owner per item or Region when business rules cannot tolerate that behavior.

## 4. Create a Basic Table

```
aws dynamodb create-table \
  --table-name AppData \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --deletion-protection-enabled
```

Keep encryption enabled, grant applications least-privilege IAM access, tag resources, and manage tables and indexes through infrastructure as code.

## 5. Basic Administration

Alert on throttled requests, consumed capacity, latency, and system errors. For global tables, also watch replication latency and failures. Contributor Insights can help identify frequently accessed or throttled keys.

```
aws dynamodb describe-table --table-name AppData
aws dynamodb describe-continuous-backups --table-name AppData
aws dynamodb describe-time-to-live --table-name AppData
```

Use TTL for data such as expired sessions, but treat deletion as asynchronous rather than an exact expiration mechanism:

```
aws dynamodb update-time-to-live \
  --table-name AppData \
  --time-to-live-specification \
  "Enabled=true,AttributeName=expiresAt"
```

## 6. Backups and Recovery

Enable point-in-time recovery (PITR) for production tables. It provides continuous recovery points for up to 35 days:

```
aws dynamodb update-continuous-backups \
  --table-name AppData \
  --point-in-time-recovery-specification \
  PointInTimeRecoveryEnabled=true
```

Create an on-demand backup before major application or data changes:

```
aws dynamodb create-backup \
  --table-name AppData \
  --backup-name AppData-before-release

aws dynamodb list-backups --table-name AppData
```

A restore creates a new table; it does not overwrite the original. Test the full procedure: restore, validate item counts and representative queries, recreate dependent integrations if required, and switch the application deliberately.

## 7. DBA Checklist

- Document every access pattern before creating keys or indexes.
- Test for hot keys with production-like traffic.
- Set alarms, budgets, deletion protection, and least-privilege IAM.
- Enable PITR. Native on-demand backups remain until deleted; use AWS Backup when you need scheduled retention policies.
- Run restore tests and record recovery time.
- Review unused indexes, scans, and capacity cost regularly.

---

The main DynamoDB mindset change is simple: administration moves away from servers and toward access-pattern design, guardrails, observability, and tested recovery.
