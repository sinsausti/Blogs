---
title: "What to Do After a Server Is Compromised"
description: "A practical first-response guide for isolating a compromised server, preserving evidence, scoping the incident, and recovering safely."
author: "Sebastian Insausti"
date: "2026-08-23"
tags: ["Infrastructure", "Security"]
canonical_url: "https://insaustis.com/blog/after-server-compromise.html"
---

# What to Do After a Server Is Compromised

A suspected compromise is an incident, not a normal troubleshooting session. The priorities are to protect people and operations, stop further damage, preserve useful evidence, determine the scope, and recover from a trustworthy state.

> If the server supports critical operations, contains regulated data, or may be part of a wider intrusion, activate the incident response plan and involve qualified security, legal, and forensic support.

## 1. Record What Triggered the Investigation

Open an incident record using a trusted system. Preserve the original alert, timestamps, host and cloud identifiers, source, affected accounts, observed indicators, and the person who detected them. Do not assume the first affected server is the initial access point.

## 2. Contain Without Destroying Evidence

Isolate the host in a coordinated manner using an EDR isolation function, hypervisor or switch control, or restrictive cloud security rules. Prefer a method that blocks attacker communication while preserving power and system state.

Do not immediately reboot, power off, wipe files, run cleanup tools, or start broad malware scans unless the incident lead determines that safety or continuing damage outweighs evidence preservation. Powering down loses volatile memory; leaving a host online may allow more damage. That trade-off must be recorded.

If isolation is not immediately possible and destructive activity is continuing, protecting the environment takes priority. Use an approved emergency action and document exactly what was changed.

## 3. Preserve Evidence Safely

Every command executed on the host changes evidence. Use approved forensic tooling and trained personnel whenever legal, regulatory, insurance, or disciplinary action is possible. Typical collection may include volatile memory, processes, network connections, logged-in users, scheduled tasks, service definitions, logs, disk images or snapshots, and relevant control-plane audit records.

```
# Examples for triage only; run only when authorized and record the time
date -u
uptime
who
ps auxf
ss -plant
systemctl --failed
journalctl --since "24 hours ago" --utc

# Hash exported evidence on trusted storage
sha256sum evidence-file > evidence-file.sha256
```

Store originals read-only where possible. Record who collected each item, when, how, where it is stored, and every transfer or access. A hash helps demonstrate integrity but does not replace chain-of-custody documentation.

## 4. Determine the Scope

Use evidence from outside the compromised host: identity-provider logs, EDR, firewalls, DNS, proxies, cloud audit logs, database audit logs, and centralized logging. Investigate:

- How initial access occurred and when activity began.
- Accounts, keys, tokens, certificates, and secrets available to the host.
- Persistence, privilege escalation, lateral movement, and command-and-control.
- Data accessed, modified, staged, encrypted, or exfiltrated.
- Other systems showing the same indicators or behaviors.

Do not reset one password and declare the incident closed. A stolen session, API key, SSH key, service credential, or identity-provider token may remain valid.

## 5. Eradicate and Recover

For a confirmed operating-system compromise, rebuilding from a known-good image is generally more trustworthy than trying to remove every attacker change. Before reconnecting:

1. Close the initial access path and remove persistence across the full scope.
2. Rotate exposed credentials from a known-clean system, in dependency order.
3. Patch vulnerabilities and correct insecure configuration.
4. Restore only validated data and required configuration.
5. Re-enroll monitoring, logging, backup, and endpoint controls.
6. Validate the application and increase monitoring during a controlled return.

Do not restore a vulnerable image or reconnect the system before the root cause and credential exposure have been addressed.

## 6. Close the Incident Properly

Document the timeline, root cause, affected data and systems, decisions, notifications, recovery evidence, and remaining risk. Assign owners and due dates to corrective actions. Preserve evidence according to legal, contractual, insurance, and organizational retention requirements.

## References

Primary guidance: [NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final), the [CISA #StopRansomware Guide](https://www.cisa.gov/stopransomware/ransomware-guide), and [NIST SP 800-86](https://csrc.nist.gov/pubs/sp/800/86/final) for forensic techniques in incident response.
