---
title: "Galera Cluster: Multi-Master MySQL Replication"
description: "A practical guide to Galera Cluster: virtually synchronous replication, quorum, conflicts, state transfers, and health-aware routing."
author: "Sebastian Insausti"
date: "2026-04-01"
tags: ["Databases", "Linux"]
canonical_url: "https://insaustis.com/blog/galera-cluster.html"
---

# Galera Cluster: Multi-Master MySQL Replication

Standard MySQL replication is asynchronous — the primary commits a transaction and immediately moves on; replicas catch up in their own time. That gap is usually milliseconds, but it's real, and if the primary crashes before a replica applies the latest transactions, those writes are gone. For most workloads this is an acceptable trade-off. For some, it isn't.

Galera Cluster takes a fundamentally different approach: writes are replicated and certified across the cluster before being acknowledged. This is *virtually synchronous* replication—remote nodes may still need a short time to apply a certified write-set locally.

## How Galera Works

Galera uses a protocol called **Write-Set Replication (wsrep)**. When a transaction commits on any node, Galera:

1. Packages the transaction's write-set (the actual row changes, not the SQL statements).
2. Broadcasts it to all other nodes simultaneously.
3. Runs **certification** on each node — checking whether the write-set conflicts with any concurrent transactions.
4. If certification succeeds, the originating node commits and the other nodes apply the write-set. A conflicting transaction can be rolled back and must be retried by the application.

Each healthy node maintains a full copy of the data. If a node fails, the component that retains quorum can continue accepting traffic, but clients still need health-aware routing and connection retry. When the failed node returns, it receives missed transactions through an **IST** (Incremental State Transfer) or a full **SST** (State Snapshot Transfer).

## Galera vs. Async Replication

The choice comes down to your tolerance for replication lag and your write pattern:

- **Use Galera** when you need strongly coordinated replication within a low-latency network, can preserve quorum, and your workload has few conflicting writes. Any node can accept writes, although a single-writer routing policy is often easier to operate.
- **Use async replication** when write throughput is the priority, when your workload has high write concurrency with potential row-level conflicts, or when you need geographic distribution across high-latency links (Galera's synchronous commit is sensitive to network latency).

> Certification adds network latency to the commit path. Measure it with the actual workload and network; stretching one cluster across high-latency regions is usually a poor fit.

## Setting Up a 3-Node Cluster

This example uses Percona XtraDB Cluster (PXC). Configure Percona's repository for your exact operating system and PXC release before installing; package and service names can differ between releases:

```
# On all three nodes (Debian/Ubuntu)
apt install -y percona-xtradb-cluster

# On all three nodes (RHEL/AlmaLinux)
dnf install -y percona-xtradb-cluster
```

Configure `/etc/mysql/mysql.conf.d/mysqld.cnf` on each node (adjust IPs and node name):

```
[mysqld]
# Galera settings
wsrep_on                   = ON
wsrep_provider             = /usr/lib/galera4/libgalera_smm.so
wsrep_cluster_name         = "prod_cluster"
wsrep_cluster_address      = "gcomm://10.0.1.10,10.0.1.11,10.0.1.12"
wsrep_node_address         = "10.0.1.10"   # this node's IP
wsrep_node_name            = "node1"
wsrep_sst_method           = xtrabackup-v2

# Required settings
binlog_format              = ROW
default_storage_engine     = InnoDB
innodb_autoinc_lock_mode   = 2
```

Bootstrap the cluster from the first node only:

> Starting a new primary component incorrectly can create divergent clusters. Confirm that node1 has the most advanced state and verify the bootstrap service name in the documentation for your installed PXC package.

```
# On node1 only — starts a new cluster
systemctl start mysql@bootstrap.service

# On node2 and node3 — join the existing cluster
systemctl start mysql
```

Once all three nodes are running, verify cluster size and status before moving node1 back to its normal service unit:

```
# On node1
systemctl stop mysql@bootstrap.service
systemctl start mysql
```

## Monitoring Cluster Health

Connect to any node and check the wsrep status variables:

```
SHOW GLOBAL STATUS LIKE 'wsrep_%';

-- The ones to watch:
-- wsrep_cluster_size      — should equal the number of nodes you expect
-- wsrep_cluster_status    — must be "Primary"
-- wsrep_connected         — must be "ON"
-- wsrep_ready             — must be "ON"
-- wsrep_local_recv_queue  — if consistently > 0, this node is falling behind
```

A node that shows `wsrep_cluster_status = Non-Primary` has lost quorum — it's isolated from the rest of the cluster and will refuse writes. This is the safety mechanism that prevents split-brain.

## Limitations and Gotchas

**No MyISAM support.** Galera only works with InnoDB. Any MyISAM tables will not be replicated.

**Every table should have a primary key.** Galera uses keys during certification and row identification. Tables without a primary key can replicate inefficiently and some operations are restricted; treat a primary key as a production requirement.

**Write conflicts and rollbacks.** If two nodes concurrently modify the same row, one transaction will be rolled back. Your application must handle `deadlock` errors (error 1213) by retrying. This is rare in practice but needs to be accounted for.

**SST and the donor.** A full State Snapshot Transfer consumes disk, network, and donor resources, and its locking behavior depends on the SST method and version. Test it under load and use `wsrep_sst_donor` when you need predictable donor selection.

**Flow control.** If one node falls behind, Galera pauses writes on all nodes to let it catch up. Monitor `wsrep_flow_control_paused` — sustained values above 0 indicate a node that's struggling.

## Conclusion

Galera Cluster is useful when you need virtually synchronous replication and coordinated multi-node availability inside a low-latency network. The trade-offs—certification latency, conflict retries, quorum, flow control, and SST operations—must be tested. Pair it with a correctly configured proxy for health-aware routing, and validate node and network failures before calling the service highly available.

---

Running Galera in production and hitting a specific issue? [Get in touch.](https://insaustis.com/#contact)
