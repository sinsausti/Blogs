---
title: "MySQL Automated Failover with Orchestrator"
description: "Use Orchestrator for MySQL topology discovery and controlled recovery with candidate selection, fencing, routing, and testing."
author: "Sebastian Insausti"
date: "2026-03-25"
tags: ["Databases", "Linux"]
canonical_url: "https://insaustis.com/blog/mysql-failover-orchestrator.html"
---

# MySQL Automated Failover with Orchestrator

MySQL replication gives you redundancy — replicas hold copies of the primary's data and can serve reads. But replication alone doesn't give you *high availability*. When the primary fails, something needs to detect the failure, pick the best replica to promote, re-point the other replicas at the new primary, and update whatever is routing application traffic. Done manually, that process takes 10–30 minutes. Done wrong, it causes data loss.

Orchestrator is a widely used open-source tool for topology discovery and controlled recovery. The original `openark/orchestrator` repository is archived; active development moved to [ProxySQL/orchestrator](https://github.com/ProxySQL/orchestrator) in 2026.

## What Orchestrator Does

Orchestrator connects to your MySQL instances via a dedicated account and continuously polls them to build a live map of the replication topology. It knows which server is the primary, which are replicas, how far behind each replica is, and the GTID state of every node.

When a primary disappears, Orchestrator:

1. Waits for a configurable detection period to avoid flapping on brief network hiccups.
2. Evaluates candidate replicas using topology state, replication position, promotion rules, and configured data-center constraints.
3. Promotes a suitable candidate and attempts to relocate recoverable replicas.
4. Runs your hook scripts — where you update ProxySQL, DNS, or Consul to redirect application traffic.
5. Records the recovery and prevents an immediate duplicate recovery attempt.

## Installation

Use a signed package or release artifact from the maintained repository when available. To inspect or build the current source:

```
git clone https://github.com/ProxySQL/orchestrator.git
cd orchestrator
git tag --sort=-version:refname | head

# Check out a reviewed release tag before building.
git checkout <release-tag>
go build -o bin/orchestrator ./go/cmd/orchestrator
./bin/orchestrator --version
```

Do not reuse the archived 3.2.6 packages for a new deployment. Production installation also requires a service unit, protected configuration, authenticated HTTP access, and either a durable MySQL backend or a tested highly available configuration.

Orchestrator stores its topology data in its own backend database — MySQL or SQLite for single-node setups.

## Minimal Configuration

Edit `/etc/orchestrator/orchestrator.conf.json`:

```
{
  "MySQLTopologyCredentialsConfigFile": "/etc/orchestrator/topology.cnf",
  "MySQLOrchestratorHost": "127.0.0.1",
  "MySQLOrchestratorPort": 3306,
  "MySQLOrchestratorDatabase": "orchestrator",
  "MySQLOrchestratorCredentialsConfigFile": "/etc/orchestrator/backend.cnf",

  "RecoveryPeriodBlockSeconds": 3600,
  "RecoverMasterClusterFilters": ["*"],
  "FailureDetectionPeriodBlockMinutes": 60,

  "OnFailureDetectionProcesses": [
    "logger -t orchestrator 'Failure detected: {failureType} on {failedHost}'"
  ]
}
```

Create a dedicated topology user on every MySQL instance. Restrict its source host, require TLS when traffic crosses an untrusted network, and grant only the privileges required by the exact Orchestrator and MySQL versions:

```
CREATE USER 'orchestrator'@'10.0.1.%'
  IDENTIFIED BY 'strong_password'
  REQUIRE SSL;

# /etc/orchestrator/topology.cnf — mode 600
[client]
user=orchestrator
password=strong_password

# /etc/orchestrator/backend.cnf — mode 600
[client]
user=orchestrator_backend
password=another_strong_password
```

Avoid copying legacy grants such as unrestricted `SUPER`. Validate discovery and recovery in a disposable topology before enabling automated promotion.

## Discovering Your Topology

Point Orchestrator at your primary and it will crawl the rest of the topology automatically:

```
# Via CLI
orchestrator-client -c discover -i primary-host:3306

# Check what it found
orchestrator-client -c topology -i primary-host:3306
```

The topology output shows the full replication tree with lag and GTID status for each node. The web UI (port 3000 by default) renders this as an interactive graph.

## Automated vs. Manual Failover

Orchestrator supports both modes. Automated failover is controlled by `RecoverMasterClusterFilters` in the config. Setting it to `["*"]` enables auto-recovery for all clusters. You can limit it to specific cluster names if you want manual control over some topologies.

For a manual failover — planned maintenance, for example — use:

```
# Graceful primary switch (primary is healthy, you're choosing to move)
orchestrator-client -c graceful-master-takeover-auto -i primary-host:3306

# Force promotion of a specific replica
orchestrator-client -c force-master-failover -i primary-host:3306
```

## The Failover Hook

The routing layer must be updated as part of recovery. Current ProxySQL-maintained Orchestrator releases provide built-in ProxySQL integration and should be preferred over a shell script containing admin credentials. If custom hooks are required, validate all arguments, load credentials from a mode-600 option file, use timeouts, and make the operation idempotent.

```
# Verify health before enabling recovery for a cluster.
curl --fail --silent http://127.0.0.1:3000/health/ready
curl --fail --silent http://127.0.0.1:3000/api/clusters | jq

# Test a planned takeover before testing an unplanned failure.
orchestrator-client -c graceful-master-takeover-auto \
  -i primary-host:3306
```

## Monitoring the Topology

Check overall cluster health at any time:

```
# List all known clusters
orchestrator-client -c clusters

# List replicas of a given instance
orchestrator-client -c which-replicas -i primary-host:3306

# Check if a specific instance is replicating OK
orchestrator-client -c instance -i replica-host:3306 | jq '.ReplicationLagSeconds, .Slave_SQL_Running'
```

Current ProxySQL-maintained releases expose REST health endpoints and Prometheus metrics at `/metrics`. Protect the HTTP interface with authentication and network controls; it can trigger topology-changing operations.

## Conclusion

Orchestrator can turn failover into a controlled and auditable process, but it cannot guarantee a fixed recovery time or zero data loss. Test detection, candidate selection, fencing of the old primary, replica relocation, routing updates, and application reconnection. Measure recovery time and data-loss exposure against your own topology before enabling unattended recovery.

---

Questions about Orchestrator configuration or integrating it with your routing layer? [Reach out.](https://insaustis.com/#contact)
