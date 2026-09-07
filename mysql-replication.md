---
title: "MySQL Primary-Replica Replication: A Practical Setup Guide"
description: "Step-by-step guide to configuring MySQL primary-replica replication — binary log setup, GTID-based replication, replica provisioning, and common troubleshooting."
author: "Sebastian Insausti"
date: "2026-01-15"
tags: ["Databases", "Linux"]
canonical_url: "https://insaustis.com/blog/mysql-replication.html"
---

# MySQL Primary-Replica Replication: A Practical Setup Guide

MySQL replication is the foundation of most production database architectures. It lets you maintain one or more read replicas in near-real-time sync with a primary, enabling read scale-out, failover, and backup offloading without touching the primary under load.

This guide walks through setting up asynchronous primary-replica replication with GTIDs on MySQL 8.0, a sensible default for new replication topologies.

## How Replication Works

MySQL replication is event-driven: the source records changes in the **binary log** (binlog). A replica receives those events and applies them locally:

- **Receiver thread** — connects to the source and streams binlog events into a local **relay log**.
- **Applier** — reads the relay log and applies transactions. With parallel replication this includes a coordinator and multiple worker threads.

With **GTIDs** (Global Transaction Identifiers), every transaction gets a unique ID assigned by the primary. This makes failover and replica re-pointing dramatically simpler — no more tracking binlog filenames and byte offsets.

## Configuring the Primary

Edit `/etc/mysql/mysql.conf.d/mysqld.cnf` (or your distro's equivalent) and add the following under `[mysqld]`:

```
[mysqld]
server-id       = 1
log_bin         = /var/log/mysql/mysql-bin.log
binlog_format   = ROW
gtid_mode       = ON
enforce_gtid_consistency = ON
bind-address    = 0.0.0.0
```

Then restart MySQL and create a dedicated replication user:

```
CREATE USER 'repl'@'10.0.1.101'
  IDENTIFIED BY 'strong_password_here'
  REQUIRE SSL;
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'10.0.1.101';
```

> Use the replica's exact address or network, require encrypted transport, and configure certificate verification appropriate to your environment.

## Provisioning the Replica

The replica needs a consistent copy of the source data. Use `mysqldump` for smaller InnoDB datasets or **Percona XtraBackup** for larger ones. XtraBackup is designed for hot backups, but metadata operations, non-InnoDB tables, and version compatibility still require planning.

```
# With mysqldump (consistent snapshot via --single-transaction, no table locks for InnoDB)
mysqldump -h primary-host -u root -p \
  --all-databases \
  --single-transaction \
  --source-data=2 \
  --set-gtid-purged=ON \
  > /tmp/full-dump.sql

# Restore on the replica
mysql -u root -p < /tmp/full-dump.sql
```

Configure the replica's `my.cnf`:

```
[mysqld]
server-id                = 2
gtid_mode                = ON
enforce_gtid_consistency = ON
read_only                = ON
super_read_only          = ON
log_bin                  = /var/log/mysql/mysql-bin.log
relay_log                = /var/log/mysql/mysql-relay-bin.log
```

## Starting Replication

With GTIDs, pointing the replica at the primary is straightforward:

```
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST     = '10.0.1.100',
  SOURCE_USER     = 'repl',
  SOURCE_PASSWORD = 'strong_password_here',
  SOURCE_SSL      = 1,
  SOURCE_SSL_CA   = '/etc/mysql/ca.pem',
  SOURCE_SSL_VERIFY_SERVER_CERT = 1,
  SOURCE_AUTO_POSITION = 1;

START REPLICA;
```

`SOURCE_AUTO_POSITION = 1` tells MySQL to use GTIDs to automatically determine where to start — no filename or position needed.

## Verifying the Replica

Check the replica's status immediately after starting:

```
SHOW REPLICA STATUS\G
```

The two lines you care about most:

```
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

Both must show `Yes`. Also inspect `Last_IO_Error`, `Last_SQL_Error`, relay-log position, GTID sets, and application-level data checks. `Seconds_Behind_Source` is an estimate; zero alone does not prove that the replica is healthy or fully current.

## Common Issues

### Error 1062: Duplicate entry

The replica already has a row the source is trying to insert. This usually means the data diverged or the initial snapshot was inconsistent. Identify the affected GTID and compare the source and replica data before changing replication state. Creating an empty transaction for the GTID skips the source transaction and can hide further divergence, so use it only as a documented last resort:

```
SET GTID_NEXT = 'uuid:transaction_id';
BEGIN; COMMIT;
SET GTID_NEXT = AUTOMATIC;
START REPLICA;
```

### Error 1032: Row not found

The replica is missing a row the primary wants to update or delete. Same root cause as above — data divergence. Investigate with `pt-table-checksum` from Percona Toolkit, then fix with `pt-table-sync`.

### High replication lag

Check whether the bottleneck is I/O (replica disk can't keep up) or CPU (SQL thread is slow). Enable **parallel replication** to replay transactions in parallel based on the primary's commit order:

```
# In replica my.cnf
replica_parallel_workers = 4
replica_preserve_commit_order = ON
```

`replica_parallel_type` is deprecated in current MySQL 8.0 releases. Benchmark the worker count instead of assuming that more workers will reduce lag.

## Conclusion

GTID-based replication is the preferred default for new MySQL 8.0 topologies. It simplifies positioning and failover workflows, but does not automate promotion, prevent data loss, or replace tested backups. Once validated, replication can support read scaling, online maintenance, and disaster-recovery procedures.

---

Running into a replication issue not covered here? [Get in touch](https://insaustis.com/#contact) — replication debugging is one of my favorite things to untangle.
