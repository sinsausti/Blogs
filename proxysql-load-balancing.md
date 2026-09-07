---
title: "ProxySQL: Load Balancing MySQL Traffic"
description: "A practical guide to ProxySQL connection management, topology monitoring, and safe workload-specific query routing for MySQL."
author: "Sebastian Insausti"
date: "2026-02-20"
tags: ["Databases", "Linux"]
canonical_url: "https://insaustis.com/blog/proxysql-load-balancing.html"
---

# ProxySQL: Load Balancing MySQL Traffic

ProxySQL is a high-performance MySQL proxy that sits between applications and database servers. Unlike a simple TCP load balancer, it understands the MySQL protocol and can route queries, multiplex compatible sessions, and apply query rules. Applications still need tested consistency and transaction behavior.

## Why ProxySQL

When you have a primary-replica setup, applications typically need to send writes to the primary and selected reads to replicas. ProxySQL centralizes that routing, but the policy must reflect each application's consistency and transaction requirements:

- **Connection pooling** — thousands of application threads share a much smaller pool of backend connections, reducing connection overhead on MySQL significantly.
- **Read/write splitting** — selected read-only queries can be routed to replicas after consistency requirements are understood.
- **Health awareness** — monitors can shun failed backends; writer promotion still requires replication-hostgroup logic or an external failover manager.
- **Query rules** — route, rewrite, mirror, or block specific queries without touching application code.

## Installation

Configure the signed ProxySQL repository for your distribution and the current stable 3.0.x tier. Do not install the old 2.6.3 package shown in earlier versions of this guide.

```
# Debian / Ubuntu, after adding the official repository
apt update
apt install proxysql

# RHEL-compatible distributions, after adding the official repository
dnf install proxysql

systemctl enable --now proxysql
proxysql --version
```

ProxySQL listens on two ports: `6033` for MySQL client connections and `6032` for its admin interface.

## The Admin Interface

All ProxySQL configuration is done through its admin interface, which speaks MySQL protocol. Connect with:

```
mysql -u admin -padmin -h 127.0.0.1 -P 6032 --prompt "ProxySQL> "
```

> Change the default admin credentials immediately in production: UPDATE global_variables SET variable_value='admin:new_password' WHERE variable_name='admin-admin_credentials'; LOAD ADMIN VARIABLES TO RUNTIME; SAVE ADMIN VARIABLES TO DISK;

## Adding Backend Servers

ProxySQL organizes backends into **hostgroups**. By convention, hostgroup `10` is the writer (primary) and hostgroup `20` is the reader pool (replicas).

```
-- Add the primary (writer)
INSERT INTO mysql_servers (hostgroup_id, hostname, port)
VALUES (10, '10.0.1.100', 3306);

-- Add replicas (readers)
INSERT INTO mysql_servers (hostgroup_id, hostname, port)
VALUES (20, '10.0.1.101', 3306);
INSERT INTO mysql_servers (hostgroup_id, hostname, port)
VALUES (20, '10.0.1.102', 3306);

-- Apply and persist
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

For asynchronous replication, map the writer and reader hostgroups so the monitor can move servers according to MySQL's `read_only` state:

```
INSERT INTO mysql_replication_hostgroups
  (writer_hostgroup, reader_hostgroup, check_type, comment)
VALUES
  (10, 20, 'read_only', 'production cluster');

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

This does not promote a MySQL replica. Use Orchestrator or another tested failover mechanism to fence the failed primary, promote a candidate, and change its read-only state.

Create a restricted monitoring account on MySQL, then configure ProxySQL to use it:

```
-- Run on MySQL and replicate or create on every backend.
CREATE USER 'proxysql_monitor'@'10.0.1.%'
  IDENTIFIED BY 'monitor_password';
GRANT USAGE, REPLICATION CLIENT ON *.*
  TO 'proxysql_monitor'@'10.0.1.%';

-- Run on the ProxySQL admin interface.
UPDATE global_variables
SET variable_value = 'proxysql_monitor'
WHERE variable_name = 'mysql-monitor_username';

UPDATE global_variables
SET variable_value = 'monitor_password'
WHERE variable_name = 'mysql-monitor_password';

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

## Read/Write Splitting with Query Rules

Do not route every `SELECT` to replicas with a generic regular expression in production. That can break transactions, session state, and read-after-write consistency. Start with all traffic on the writer, inspect query digests, and route only reads proven safe for replica lag:

```
-- First identify a specific safe query.
SELECT digest, digest_text, count_star, sum_time
FROM stats.stats_mysql_query_digest
WHERE digest_text LIKE 'SELECT%'
ORDER BY sum_time DESC
LIMIT 10;

-- Route one reviewed digest to replicas.
INSERT INTO mysql_query_rules
  (rule_id, active, digest, destination_hostgroup, apply)
VALUES
  (100, 1, '0xREPLACE_WITH_REVIEWED_DIGEST', 20, 1);

LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

Create a MySQL user in ProxySQL that maps to a real MySQL account:

```
INSERT INTO mysql_users (username, password, default_hostgroup)
VALUES ('appuser', 'app_password', 10);

LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

The `default_hostgroup = 10` means non-matched queries (writes, DDL) go to the primary.

## Monitoring and Stats

ProxySQL's `stats` schema gives you real-time visibility into what's happening:

```
-- Query digest: most frequent queries, latency, error rate
SELECT hostgroup, digest_text, count_star, sum_time/count_star AS avg_us
FROM stats.stats_mysql_query_digest
ORDER BY sum_time DESC
LIMIT 10;

-- Connection pool health
SELECT hostgroup, srv_host, status, ConnUsed, ConnFree, Latency_us
FROM stats.stats_mysql_connection_pool;

-- Which servers are currently UP
SELECT hostgroup_id, hostname, port, status
FROM runtime_mysql_servers;
```

## Conclusion

ProxySQL provides a powerful control point for MySQL routing and connection management, but safe behavior depends on workload-specific rules. Start with the writer as the default, enable monitoring, analyze digests, and add targeted replica routing gradually. Test transactions, failover, replica lag, and read-after-write behavior before relying on it in production.

---

Questions about ProxySQL configuration or a tricky query routing scenario? [Reach out.](https://insaustis.com/#contact)
