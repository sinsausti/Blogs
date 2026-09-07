---
title: "A Backup Is Not a Recovery Plan"
description: "Learn why successful backup jobs are not enough and how to build a tested, secure, and operational database recovery plan."
author: "Sebastian Insausti"
date: "2026-08-29"
tags: ["Databases", "Infrastructure"]
canonical_url: "https://insaustis.com/blog/a-backup-is-not-a-recovery-plan.html"
---

# A Backup Is Not a Recovery Plan

A green backup job only proves that a tool produced an output. It does not prove that the data is complete, that the credentials are available, or that the team can restore the service within the required time.

> A backup is a copy of data. A recovery plan is the tested process that turns that copy back into a working service.

## 1. What a Recovery Plan Must Cover

A database rarely works alone. Recovery may also require configuration files, users and roles, encryption keys, certificates, extensions, scheduled jobs, network rules, application secrets, and a known application version.

A useful plan answers five questions:

- What systems and data must be recovered first?
- How much data loss and downtime are acceptable?
- Where are the backups, keys, credentials, and instructions?
- Who can declare a disaster and perform the restore?
- How will the result be validated before applications reconnect?

## 2. Replication Is Not a Backup

Replication improves availability, but it usually copies accidental deletes, bad updates, and some forms of corruption to every replica. Snapshots help, but a snapshot in the same account and failure domain may disappear with the production environment.

Use several independent layers: replication for availability, point-in-time recovery for recent mistakes, and separate retained backups for larger failures. Keep at least one copy offline or immutable and protect backup administration with separate credentials.

## 3. Capture the Right Database Data

For PostgreSQL, a physical base backup plus archived WAL supports point-in-time recovery. Logical dumps are portable and useful for object-level restores, but may be too slow for a large database.

```
pg_basebackup \
  --host=db-primary \
  --username=backup_user \
  --pgdata=/backup/postgresql/base \
  --format=plain \
  --wal-method=stream \
  --progress

pg_verifybackup /backup/postgresql/base
pg_dumpall --host=db-primary --username=backup_user --globals-only > globals.sql
```

For a moderate MySQL InnoDB database, a logical backup can be created without locking transactional tables for the entire dump:

```
mysqldump \
  --host=db-primary \
  --user=backup_user \
  --password \
  --single-transaction \
  --routines --events --triggers \
  --all-databases \
  > all-databases.sql
```

The `--single-transaction` guarantee does not extend to nontransactional tables, which need separate handling. For large databases, use an appropriate physical backup tool and retain the binary logs required for point-in-time recovery. Never place database passwords directly in scripts or command-line arguments.

## 4. Validate More Than the Backup File

Checksums can detect damaged files, but they cannot prove that the correct data was captured. Validation should happen at several levels:

- Confirm every expected database and recovery file exists.
- Verify checksums, encryption, retention, and replication to the secondary location.
- Restore into an isolated environment.
- Run database consistency checks and representative application queries.
- Record the actual recovery point, restore duration, failures, and manual steps.

## 5. Write a Short Runbook

The runbook should be usable during a stressful incident. Include prerequisites, exact backup locations, restore commands, expected output, validation queries, DNS or proxy changes, rollback steps, owners, and escalation contacts.

Test it regularly and after major changes to storage, database versions, encryption, authentication, or topology. A restore exercise is successful only when the application works and the measured recovery point and time meet the business objectives.

## 6. Practical Checklist

- Define recovery objectives for each critical service.
- Keep multiple copies across independent failure domains.
- Use encryption in transit and at rest, with recoverable key management.
- Protect one copy from modification or deletion.
- Monitor backup age, size, duration, and failures.
- Perform scheduled restore tests and document the evidence.
- Review the plan whenever the architecture changes.

---

The real deliverable is not the backup file. It is a repeatable recovery with known data loss, known duration, and evidence that the restored service is usable.
