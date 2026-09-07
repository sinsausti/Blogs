---
title: "Practical Database Monitoring with Prometheus and Grafana"
description: "Build a practical PostgreSQL and MySQL monitoring stack with Prometheus, Grafana, database exporters, useful metrics, alerts, and daily DBA workflows."
author: "Sebastian Insausti"
date: "2026-09-05"
tags: ["Databases", "Monitoring"]
canonical_url: "https://insaustis.com/blog/database-monitoring-prometheus-grafana.html"
---

# Practical Database Monitoring with Prometheus and Grafana

A database monitor should answer three questions quickly: Is the service available? Is it meeting its performance objective? If not, where should the DBA investigate first? Prometheus and Grafana remain a strong open-source combination because they separate metric collection, storage, visualization, and alert delivery without tying the design to one database engine.

> Monitor the database, the operating system, and the application path. A healthy database process does not guarantee a healthy service.

## 1. Choose the Stack

- **[Prometheus](https://prometheus.io/docs/introduction/overview/):** scrapes and stores time-series metrics and evaluates alert rules.
- **[Grafana](https://grafana.com/docs/grafana/latest/introduction/):** provides dashboards, exploration, and visualization.
- **[postgres_exporter](https://github.com/prometheus-community/postgres_exporter):** exposes PostgreSQL statistics in Prometheus format.
- **[mysqld_exporter](https://github.com/prometheus/mysqld_exporter):** exposes MySQL and MariaDB status, configuration, and replication metrics.
- **[node_exporter](https://github.com/prometheus/node_exporter):** adds CPU, memory, filesystem, disk, and network metrics from Linux.
- **[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/):** groups, routes, inhibits, and silences Prometheus alerts.

Nagios is still useful for direct service checks and existing operational workflows, but Prometheus is better suited to multidimensional time-series analysis. The stack below favors flexibility, open components, and transparent configuration.

## 2. Create Read-Only Monitoring Users

In PostgreSQL, use the predefined `pg_monitor` role instead of a superuser. Connect to a database that exists on every monitored instance:

```
CREATE USER prometheus WITH PASSWORD 'replace_this_password';
GRANT pg_monitor TO prometheus;
GRANT CONNECT ON DATABASE postgres TO prometheus;
```

For query-level PostgreSQL statistics, enable `pg_stat_statements`. Adding it to `shared_preload_libraries` requires a server restart; create the extension afterward in the databases where it is needed:

```
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'

-- Run after the restart
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

For MySQL, create a dedicated account restricted to the monitoring host. The grants below follow the official exporter requirements and limit concurrent monitoring connections:

```
CREATE USER 'exporter'@'10.0.20.15'
  IDENTIFIED BY 'replace_this_password'
  WITH MAX_USER_CONNECTIONS 3
  REQUIRE SSL;

GRANT PROCESS, REPLICATION CLIENT, SELECT
  ON *.* TO 'exporter'@'10.0.20.15';
```

Store credentials in files readable only by the exporter process or in a secrets manager. Do not place passwords in Git, image definitions, command histories, or Prometheus target labels.

## 3. Run the Monitoring Server

The following Compose file runs Prometheus, Grafana, and both database exporters on one monitoring host. Replace the database addresses and mount the two secret files before starting it:

```
services:
  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    ports: ["127.0.0.1:9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    ports: ["127.0.0.1:3000:3000"]
    volumes:
      - grafana-data:/var/lib/grafana

  alertmanager:
    image: prom/alertmanager:latest
    restart: unless-stopped
    ports: ["127.0.0.1:9093:9093"]
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro

  postgres-exporter:
    image: quay.io/prometheuscommunity/postgres-exporter:latest
    restart: unless-stopped
    environment:
      DATA_SOURCE_URI: "postgres.example.net:5432/postgres?sslmode=require"
      DATA_SOURCE_USER: "prometheus"
      DATA_SOURCE_PASS_FILE: "/run/secrets/postgres_password"
    volumes:
      - ./secrets/postgres_password:/run/secrets/postgres_password:ro
    ports: ["127.0.0.1:9187:9187"]

  mysql-exporter:
    image: prom/mysqld-exporter:latest
    restart: unless-stopped
    command: ["--config.my-cnf=/cfg/.my.cnf"]
    volumes:
      - ./secrets/mysql-exporter.cnf:/cfg/.my.cnf:ro
      - ./secrets/mysql-ca.pem:/cfg/mysql-ca.pem:ro
    ports: ["127.0.0.1:9104:9104"]

volumes:
  prometheus-data:
  grafana-data:
```

Create the MySQL exporter configuration and restrict its permissions:

```
[client]
user=exporter
password=replace_this_password
host=mysql.example.net
port=3306
ssl-ca=/cfg/mysql-ca.pem
```

Make each secret readable by its container process but not by unrelated host users; container UIDs differ by image, so verify ownership after pinning the images. The ports above bind only to loopback. Use a VPN or an authenticated TLS reverse proxy if administrators need remote access.

Using `latest` keeps the example readable. In production, pin tested image versions or digests, review release notes, and upgrade deliberately.

## 4. Configure Prometheus and Grafana

Create `prometheus.yml`:

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: postgres
    static_configs:
      - targets: ["postgres-exporter:9187"]
        labels:
          environment: production
          cluster: orders-postgres

  - job_name: mysql
    static_configs:
      - targets: ["mysql-exporter:9104"]
        labels:
          environment: production
          cluster: customer-mysql
```

Create a minimal `alertmanager.yml`. It accepts alerts and exposes them in Alertmanager while you configure the notification integration appropriate to your environment:

```
route:
  receiver: dba-team
  group_by: [alertname, cluster]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: dba-team
```

Start and verify the stack:

```
docker compose config
docker compose up -d
docker compose ps

curl -fsS http://localhost:9090/-/ready
curl -fsS http://localhost:3000/api/health
curl -fsS http://localhost:9093/-/ready
curl -fsS http://localhost:9187/metrics | grep '^pg_up'
curl -fsS http://localhost:9104/metrics | grep '^mysql_up'
```

In Grafana, open `http://MONITORING_HOST:3000`, change the initial administrator password, and add Prometheus as a data source using `http://prometheus:9090`. Provision the data source and dashboards from files once the design is stable so changes can be reviewed and version-controlled.

## 5. Monitor Metrics That Explain Risk

Start with a small dashboard that connects symptoms to causes:

- **Availability:** exporter reachability, database uptime, failed connections, and application connection errors.
- **Workload:** transactions or queries per second, rows processed, reads versus writes, and query latency.
- **Connections:** active, idle, waiting, rejected, and percentage of the configured limit.
- **Contention:** locks, deadlocks, long transactions, and waiting sessions.
- **Replication:** health, lag measured in time and bytes where available, receiver/applier state, and replication slot or binlog retention.
- **Storage:** database growth, filesystem usage, IOPS, latency, throughput, and temporary files.
- **Cache:** PostgreSQL buffer activity and MySQL InnoDB buffer-pool efficiency, interpreted with workload context.
- **Maintenance:** PostgreSQL vacuum and transaction-ID age; MySQL history-list growth, purge behavior, and backup freshness.

Never interpret a ratio alone. A high cache-hit ratio does not rule out slow queries, and zero replication lag does not prove that replication is running. Combine state, rate, latency, errors, and saturation.

## 6. Add Useful PromQL Panels

Metric names depend on exporter version and enabled collectors, so confirm them in Prometheus before importing a dashboard. These common examples provide a starting point:

```
# PostgreSQL transactions per second
sum by (instance) (
  rate(pg_stat_database_xact_commit[5m])
  + rate(pg_stat_database_xact_rollback[5m])
)

# PostgreSQL connection usage
100 * sum by (instance) (pg_stat_database_numbackends)
  / max by (instance) (pg_settings_max_connections)

# MySQL queries per second
rate(mysql_global_status_queries[5m])

# MySQL connection usage
100 * mysql_global_status_threads_connected
  / mysql_global_variables_max_connections
```

Label dashboards by environment, cluster, instance, role, region, and engine version. Avoid high-cardinality labels such as full SQL text, user IDs, or request IDs. Use query digests, logs, or traces for detailed query analysis.

## 7. Alert on Actionable Conditions

Create `rules/database.yml` with basic reachability alerts:

```
groups:
  - name: database-availability
    rules:
      - alert: PostgreSQLExporterCannotReachDatabase
        expr: pg_up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is unreachable from the exporter"
          runbook_url: "https://runbooks.example.net/postgresql-unreachable"

      - alert: MySQLExporterCannotReachDatabase
        expr: mysql_up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MySQL is unreachable from the exporter"
          runbook_url: "https://runbooks.example.net/mysql-unreachable"
```

Add warning alerts for sustained connection pressure, replication delay, disk exhaustion forecasts, long transactions, deadlocks, and backup age—but set thresholds from baselines and service objectives, not copied dashboards. Test each rule by creating a controlled failure.

Prometheus evaluates alert rules; Alertmanager handles grouping, routing, silences, and inhibition before notifying email, chat, or an on-call platform. During maintenance, create a time-bounded silence rather than disabling rules.

## 8. Operate It Day to Day

- Review active alerts, replication, disk forecasts, long transactions, and unusual workload changes daily.
- Compare current behavior with the same hour and weekday, not only the previous few minutes.
- Annotate deployments, failovers, schema changes, and maintenance in dashboards.
- Keep Prometheus retention sized for available disk and back up Grafana configuration and dashboards.
- Monitor the monitoring system: scrape failures, rule evaluation errors, cardinality, storage growth, and notification delivery.
- Expose exporters only to authorized monitoring networks and use TLS or an authenticated proxy across untrusted links.
- Update dashboards and runbooks whenever topology, engine versions, or service objectives change.

---

A useful DBA dashboard is not a wall of graphs. Begin with availability, workload, latency, errors, connections, contention, replication, and storage. Add detail only when it helps someone make a faster and safer operational decision.
