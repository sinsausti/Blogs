---
title: "The Anatomy of a Cyberattack: Where Defenders Can Break the Chain"
description: "Follow common adversary objectives from reconnaissance to impact and identify practical opportunities to prevent, detect, and contain them."
author: "Sebastian Insausti"
date: "2026-08-20"
tags: ["Infrastructure", "Security"]
canonical_url: "https://insaustis.com/blog/anatomy-of-a-cyberattack.html"
---

# The Anatomy of a Cyberattack: Where Defenders Can Break the Chain

Cyberattacks are often described as a chain, but real intrusions are not reliably linear. Adversaries may skip, repeat, or perform objectives in parallel. MITRE ATT&CK is better understood as a knowledge base of observed tactics and techniques—not a guaranteed timeline.

Still, grouping common objectives helps defenders identify where prevention, detection, and containment can interrupt an operation.

## 1. Reconnaissance and Resource Development

The adversary gathers information about people, domains, exposed services, suppliers, technologies, and credentials, and may prepare infrastructure or accounts for the operation.

- **Reduce exposure:** maintain an external asset inventory, remove obsolete services, protect registration accounts, and limit unnecessary public technical detail.
- **Detect:** monitor domain changes, leaked credentials, unusual scanning, and repeated enumeration of public services.

Reconnaissance is difficult to eliminate because much of it uses public information. The goal is to reduce useful exposure and find preparation early.

## 2. Initial Access and Execution

Common entry paths include phishing, valid accounts, exposed remote services, public-facing application vulnerabilities, trusted relationships, and supply-chain compromise. After entry, the adversary attempts to execute code or abuse legitimate functionality.

- **Prevent:** use phishing-resistant MFA where feasible, patch Internet-facing systems, restrict remote administration, harden applications, and control third-party access.
- **Detect:** correlate identity, endpoint, application, and network telemetry; alert on unusual logins, new devices, exploit behavior, and unexpected interpreters or child processes.

## 3. Persistence, Privilege, and Defense Impairment

An intruder may create or alter accounts, SSH keys, scheduled tasks, services, startup mechanisms, IAM policies, images, or application components. They may seek more privileges, hide activity, or impair security controls and logging.

- **Prevent:** enforce least privilege, separate administrative identities, protect configuration pipelines, and tightly control security-tool administration.
- **Detect:** monitor privileged-group and IAM changes, new startup entries, modified `authorized_keys`, disabled agents, cleared logs, and unexpected policy changes.

## 4. Credential Access, Discovery, and Lateral Movement

The adversary looks for credentials and maps hosts, services, trust relationships, cloud resources, databases, and backups. Stolen identities or remote-management tools can then provide access to additional systems.

- **Prevent:** segment networks, isolate management paths, use unique service credentials, protect secrets, limit credential exposure, and restrict east-west traffic.
- **Detect:** baseline authentication and administrative protocols; alert on password spraying, unusual source-to-destination pairs, remote execution, new session patterns, and access outside an identity's normal scope.

This is often a strong containment point: isolating a small number of affected identities and hosts may prevent an initial foothold from becoming an environment-wide incident.

## 5. Collection, Command and Control, and Exfiltration

Attackers may collect files, database records, email, credentials, screenshots, or cloud data, then stage and transfer it through an existing control channel or another service.

- **Prevent:** minimize retained sensitive data, restrict bulk export, control egress, protect backups, and apply least privilege to storage and database access.
- **Detect:** watch DNS, proxy, firewall, storage, and database audit logs for unusual destinations, volumes, compression, archive creation, bulk reads, or access at unusual times.

Encryption can limit content inspection, so behavioral and metadata-based detection remain important.

## 6. Impact

The final objective may be disruption, encryption, destruction, manipulation, fraud, or loss of availability. Exfiltration and impact can occur together, as in double-extortion ransomware.

- **Limit impact:** maintain tested offline or immutable backups, separate backup credentials, use resilient architectures, protect recovery consoles, and define manual continuity procedures.
- **Respond:** isolate in a coordinated manner, preserve evidence, rotate exposed credentials, rebuild from trusted sources, and validate data before restoring service.

## 7. Build Detection Around Behaviors

Do not rely on a single product or indicator. Map the techniques relevant to your environment, verify that required telemetry exists, create detections for high-risk behaviors, and test them with authorized exercises. Prioritize:

- Identity and privilege changes.
- Execution on critical servers.
- Management-plane and security-control changes.
- Unexpected lateral movement.
- Large or unusual access to databases and backups.
- Recovery systems being modified or disabled.

> The objective is not to stop an abstract attack at one perfect point. It is to create multiple opportunities to prevent, observe, contain, and recover.

## References

Primary references: the live [MITRE ATT&CK Enterprise tactics](https://attack.mitre.org/tactics/enterprise/), the [NIST Cybersecurity Framework 2.0](https://www.nist.gov/cyberframework), and [NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final).
