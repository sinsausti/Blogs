---
title: "PostgreSQL Migration Strategies"
description: "A practical guide to PostgreSQL migration strategies: pg_upgrade, dump and restore, minimal-downtime logical replication, and pgloader."
author: "Sebastian Insausti"
date: "2026-04-04"
tags: ["Databases", "Linux"]
canonical_url: "https://insaustis.com/blog/postgresql-migration-strategies.html"
---

# PostgreSQL Migration Strategies

PostgreSQL migrations fall into three categories: major version upgrades (e.g., PG 16 → PG 18), migrations from another database engine (MySQL, Oracle, SQLite), and table or schema changes within the same running instance. The right tool depends almost entirely on two questions — how much downtime you can accept, and how large your dataset is.

This guide covers the four strategies that handle the vast majority of real-world PostgreSQL migrations.

## Choosing the Right Strategy

- **pg_upgrade** — major version upgrade, same server or same filesystem. Minimal downtime (minutes, not hours). The fastest path for large datasets.
- **Dump and restore** — any version, any host, any size. Requires downtime proportional to dataset size. Best for small-to-medium databases or when moving across servers with different architectures.
- **Logical replication** — minimal-downtime PostgreSQL-to-PostgreSQL migration, including version upgrades. It keeps the target synchronized before a short, controlled write pause and cutover.
- **pgloader** — migrating from a different database engine (MySQL, SQLite, MS SQL Server). Handles type mapping and encoding conversion automatically.

## In-Place Upgrade with pg_upgrade

`pg_upgrade` upgrades a PostgreSQL cluster in place by reusing the existing data files, avoiding the full rewrite that dump/restore requires. The old and new server binaries must both be installed.

Run it as the PostgreSQL operating-system user, install compatible builds of every required extension first, and take a verified backup. Distribution tools may use different service and data-directory names.

```
# 1. Initialize the new cluster
/usr/lib/postgresql/18/bin/initdb -D /var/lib/postgresql/18/main

# 2. Run a compatibility check first — no data is modified
/usr/lib/postgresql/18/bin/pg_upgrade \
  -b /usr/lib/postgresql/16/bin \
  -B /usr/lib/postgresql/18/bin \
  -d /var/lib/postgresql/16/main \
  -D /var/lib/postgresql/18/main \
  --check --link

# 3. Stop both clusters, then run the upgrade
systemctl stop postgresql@16-main
systemctl stop postgresql@18-main

/usr/lib/postgresql/18/bin/pg_upgrade \
  -b /usr/lib/postgresql/16/bin \
  -B /usr/lib/postgresql/18/bin \
  -d /var/lib/postgresql/16/main \
  -D /var/lib/postgresql/18/main \
  --link
```

The four core flags: `-b` (old binary directory), `-B` (new binary directory), `-d` (old data directory), `-D` (new data directory).

Three transfer modes are available:

- **Default (copy)** — copies data files to the new cluster. Safe everywhere, but slow for large datasets.
- **--link** — creates hard links instead of copying. Near-instant for any dataset size, but both data directories must be on the same filesystem. Once you start the new cluster, the old one's data is shared — don't start the old cluster again.
- **--clone** — uses reflinks (copy-on-write at the block level). Available since PostgreSQL 12; requires a reflink-capable filesystem (XFS, Btrfs). Faster than a full copy and safer than hard links since old and new data are independent.

> As of PostgreSQL 18, pg_upgrade preserves optimizer statistics from the old cluster by default, so an immediate ANALYZE is no longer required. On earlier versions, pg_upgrade generates an analyze_new_cluster.sh script — run it before putting the new cluster under load. In both cases, update any extensions after the upgrade: ALTER EXTENSION name UPDATE;

## Dump and Restore with pg_dump

`pg_dump` produces a consistent snapshot of a single database. For large datasets, use directory format (`-Fd`) which supports parallel dump and restore:

```
# Dump using 4 parallel workers (requires directory format)
pg_dump -h old-host -U postgres -Fd -j 4 -f /tmp/pgdump/ mydb

# Restore using 4 parallel workers
pg_restore -h new-host -U postgres -d mydb -j 4 /tmp/pgdump/
```

Custom format (`-Fc`) is a single compressed file that supports parallel restore but not parallel dump. Plain SQL format (`-Fp`) supports neither. For most migrations, directory format is the right choice.

Roles and tablespaces are not included in `pg_dump` output — use `pg_dumpall` for those:

```
# Dump only global objects (roles, tablespaces) — no table data
pg_dumpall -h old-host -U postgres --globals-only > globals.sql

# Apply on the new server before restoring the database
psql -h new-host -U postgres -f globals.sql
```

When migrating across environments where ownership differs, add `--no-owner --no-acl` to the `pg_dump` command to strip ownership and privilege statements from the dump. On PostgreSQL 18, `pg_dump --statistics` includes most optimizer statistics in the dump, reducing the time before the planner has good estimates after restore — though running `ANALYZE` after restore is still recommended to ensure completeness.

## Minimal-Downtime Migration with Logical Replication

PostgreSQL's built-in logical replication (available since version 10) replicates row-level changes in real time. You can run source and target in parallel, let them sync, then cut over with a brief write pause — no extended maintenance window required.

> Faster path for same-version replicas (PostgreSQL 17+): pg_createsubscriber can convert an existing physical standby into a logical subscriber without copying the table data again. The publisher, standby, and utility must use the same PostgreSQL major version, so this is not a direct major-version upgrade method.

**Step 1: Configure the source for logical replication.**

```
# postgresql.conf on source
wal_level = logical
max_replication_slots = 10  # must be >= subscriptions + temporary table sync slots
max_wal_senders = 10
```

Restart the source after changing `wal_level`. Then create a replication user:

```
-- On source
CREATE USER repl WITH REPLICATION LOGIN PASSWORD 'strong_password';
-- pg_read_all_data (PostgreSQL 14+) grants SELECT on all tables in all schemas
GRANT pg_read_all_data TO repl;
```

**Step 2: Copy the schema to the target.** Logical replication handles data, not DDL — you must create the schema first.

```
pg_dump -h old-host -U postgres --schema-only mydb | \
  psql -h new-host -U postgres mydb
```

**Step 3: Create the publication on the source and the subscription on the target.**

```
-- On source
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- On target
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=old-host dbname=mydb user=repl password=strong_password'
  PUBLICATION my_pub;
```

The subscription immediately begins an initial data copy of all tables, then switches to streaming live changes. Monitor progress:

```
-- On target: check per-table sync state
-- srsubstate: 'i' = initializing, 'd' = copying data,
--             's' = synchronized, 'r' = ready (live replication)
SELECT rel::regclass, srsubstate
FROM pg_subscription_rel;

-- Check replication lag
SELECT subname, received_lsn, latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
```

**Step 4: Sync sequences, then cut over.**

Logical replication does not replicate sequence state. Before the cutover, generate and run `setval` commands on the target:

```
-- Run on source, apply output on target
SELECT format(
         'SELECT pg_catalog.setval(%L, %s, %L);',
         format('%I.%I', schemaname, sequencename),
         last_value,
         true
       )
FROM pg_sequences
WHERE last_value IS NOT NULL;
```

To cut over: stop application writes to the source, record `pg_current_wal_lsn()` there, and wait until the subscriber has received and applied that position. Re-run the sequence sync, disable the subscription, validate critical data, and then redirect application traffic. Keep a tested rollback plan and do not allow independent writes on both databases.

> Tables without a primary key require explicit replica identity for UPDATE and DELETE to replicate: ALTER TABLE t REPLICA IDENTITY FULL; — or add a primary key. Without a replica identity, any UPDATE or DELETE on the table causes an error on the publisher and breaks replication.

## Cross-Platform Migration with pgloader

`pgloader` migrates data from MySQL, SQLite, MS SQL Server, and other sources into PostgreSQL, handling type mapping, character set conversion, and index recreation automatically.

```
# Install
apt install pgloader

# Migrate an entire MySQL database to PostgreSQL
pgloader mysql://user:pass@mysql-host/source_db \
         postgresql://user:pass@pg-host/target_db
```

pgloader streams rows from the source into the target in batches, converting types on the fly (e.g., MySQL's `TINYINT(1)` → `BOOLEAN`, `DATETIME` → `TIMESTAMP`, `AUTO_INCREMENT` → `BIGSERIAL`). For more control, use a load file:

```
LOAD DATABASE
  FROM      mysql://user:pass@mysql-host/source_db
  INTO      postgresql://user:pass@pg-host/target_db

WITH include drop, create tables, create indexes,
     reset sequences, foreign keys

SET work_mem to '128MB',
    maintenance_work_mem to '512MB'

EXCLUDING TABLE NAMES MATCHING ~/^tmp_/, ~/^log_/;
```

What pgloader does *not* migrate: stored procedures, triggers, events, and any MySQL-specific SQL syntax in your application. Those require manual rewriting. Always do a row-count comparison after the migration and spot-check data in critical tables before cutting over application traffic.

## Conclusion

For many major-version upgrades, `pg_upgrade --link` is a fast and well-tested option when the source and target storage meet its requirements. When only a short write interruption is acceptable, logical replication can reduce the cutover window, but sequences, schema changes, large objects, and rollback still need planning. For cross-platform work, pgloader handles much of the type mapping, while application-specific SQL still requires validation.

---

Working through a PostgreSQL migration and hit a snag? [Get in touch.](https://insaustis.com/#contact)
