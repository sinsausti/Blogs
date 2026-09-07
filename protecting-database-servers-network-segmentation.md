---
title: "Protecting Database Servers with Network Segmentation"
description: "A practical guide to isolating database servers with security zones, explicit traffic flows, firewalls, and verification."
author: "Sebastian Insausti"
date: "2026-08-25"
tags: ["Databases", "Networking"]
canonical_url: "https://insaustis.com/blog/protecting-database-servers-network-segmentation.html"
---

# Protecting Database Servers with Network Segmentation

A database should not be reachable from every workstation, container, or public subnet. Network segmentation limits which systems can initiate connections, reduces lateral movement, and creates useful enforcement and logging points.

## 1. Start with Required Flows

Document traffic before writing firewall rules. A small production environment may need only these paths:

- Application subnet to the database listener: PostgreSQL `5432/TCP` or MySQL `3306/TCP`.
- Administration subnet or bastion to SSH and the database administration endpoint.
- Backup systems to the database or backup agent endpoint.
- Monitoring systems to approved exporters or agents.
- Database nodes to one another on the exact replication and cluster ports required.

Everything else should be denied by default. Port numbers are conventions, not proof of application identity, so combine network rules with database authentication, authorization, and TLS.

## 2. Separate Security Zones

```
Internet
   |
Reverse proxy / load balancer zone
   |
Application zone
   |
Database zone

Administration, backup, and monitoring use separate controlled paths.
```

A VLAN or subnet creates a boundary, but not necessarily enforcement. Apply stateful firewall rules, cloud security groups, network policies, or distributed firewalls at the boundary. Keep production, development, backups, and management in separate trust zones.

## 3. Keep Databases Off Public Networks

Bind the database to private interfaces and avoid public IP addresses. PostgreSQL `listen_addresses` and MySQL `bind_address` determine where the service listens; host-based access rules and database grants determine who may authenticate.

```
sudo ss -lntp | grep -E ':(5432|3306)\b'

# Run from an authorized application host
nc -vz db.internal.example 5432

# Run from a zone that must not have access; this should fail
nc -vz -w 3 db.internal.example 5432
```

Test both permitted and denied paths. A successful application test alone cannot prove that unwanted routes are blocked.

## 4. Restrict Administration and Egress

Route administrative sessions through a hardened bastion, VPN, or identity-aware access service. Require individual accounts, strong authentication, short-lived access where possible, and session logging. Do not expose SSH or database listeners to the Internet.

Outbound rules matter too. Database servers usually need only specific DNS, time, package, backup, monitoring, or key-management destinations. Controlled egress can limit command-and-control traffic and data exfiltration, but exceptions must account for updates and recovery operations.

## 5. Add Defense in Depth

- Use host firewalls in addition to network controls.
- Encrypt application and replication traffic with TLS.
- Use separate service identities and least-privilege database roles.
- Log allowed and denied connections without overwhelming the logging platform.
- Review rules after migrations, topology changes, and service retirement.
- Manage rules as code and require peer review.

## 6. Roll Out Safely

Build an inventory, capture current flows, create an explicit rule matrix, test in a non-production environment, and stage production changes with console access and a rollback plan. Observe denied traffic before removing temporary exceptions.

> Segmentation is effective only when the boundary is enforced, monitored, and tested from both sides.

## References

See the [NIST Guidelines on Firewalls and Firewall Policy](https://csrc.nist.gov/pubs/sp/800/41/r1/final) and the [PostgreSQL client authentication documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).
