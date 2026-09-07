---
title: "Understanding the InnoDB Doublewrite Buffer"
description: "Learn how the InnoDB Doublewrite Buffer protects MySQL data integrity against torn page writes and corruption during crashes."
author: "Sebastian Insausti"
date: "2025-10-07"
tags: ["Databases", "Linux"]
canonical_url: "https://insaustis.com/blog/innodb-doublewrite-buffer.html"
---

# Understanding the InnoDB Doublewrite Buffer

The InnoDB Doublewrite Buffer is one of those features that quietly keeps your MySQL data safe without anyone noticing — until something goes wrong. It protects against **torn page writes**, a subtle form of corruption that can occur when a system crashes in the middle of writing a 16KB InnoDB page to a storage device that uses a smaller atomic write size (typically 512 bytes or 4KB).

In this post, we'll break down how the Doublewrite Buffer works, why it exists, and the implications of disabling it.

## The Problem: Torn Pages

InnoDB stores data in pages that are 16KB by default. Most operating systems and storage devices, however, guarantee atomicity only at the sector level — 512 bytes or 4096 bytes. This means that when InnoDB flushes a 16KB page to disk, it involves multiple smaller write operations under the hood.

If the operating system, storage subsystem, or `mysqld` exits unexpectedly during those writes, a page can be only partially written. The first half might contain new data and the second half old data. Redo recovery alone may not be enough because it expects a valid base page on disk.

> A torn page is essentially half-old, half-new — and the redo log has no way to know which half is which.

## The Solution: The Doublewrite Buffer

InnoDB solves this with a two-phase write approach:

1. **Phase 1 — Write and flush the Doublewrite Buffer:** Before flushing dirty pages to their final tablespace locations, InnoDB writes them in a large sequential batch to the Doublewrite Buffer and flushes that copy to durable storage.
2. **Phase 2 — Write to actual tablespace positions:** Once the Doublewrite Buffer is safely flushed, InnoDB writes each page to its real location in the data file.

If a crash happens during Phase 2, InnoDB can recover by reading the complete, known-good copy of the page from the Doublewrite Buffer and using it as the base for redo log application.

## Where Is It Stored?

Historically, the Doublewrite Buffer was stored inside the system tablespace (`ibdata1`). Since MySQL 8.0.20, it has been moved to its own dedicated files, separate from the system tablespace, which improves I/O efficiency:

```
# Typical location in MySQL 8.0.20+; names and count vary
find /var/lib/mysql -maxdepth 1 -name '#ib_*.dblwr' -ls
```

You can configure the location with the `innodb_doublewrite_dir` variable, useful if you want to put it on a faster or more durable storage device.

## Checking the Status

You can verify the Doublewrite Buffer status and see its activity via:

```
SHOW GLOBAL STATUS LIKE 'Innodb_dblwr%';

+----------------------------+----------+
| Variable_name              | Value    |
+----------------------------+----------+
| Innodb_dblwr_pages_written | 18430    |
| Innodb_dblwr_writes        | 4107     |
+----------------------------+----------+
```

The ratio of `Innodb_dblwr_pages_written` to `Innodb_dblwr_writes` tells you how many pages are being batched per write. A higher ratio means better efficiency.

## Should You Disable It?

You can disable the Doublewrite Buffer with:

```
# my.cnf — requires a controlled MySQL restart
[mysqld]
innodb_doublewrite = OFF
```

Disabling it can improve some write-heavy benchmarks, but the result depends on the storage and workload and must be measured. It also removes protection against incomplete page writes. Reasonable cases are limited:

- The complete storage stack provides a documented atomic-write guarantee compatible with the InnoDB page size; filesystem block or record size alone is not sufficient evidence.
- You're running a non-critical environment where you can afford to recreate data from backups
- You are running a disposable replica that can be rebuilt and have accepted the longer recovery time.

For most production systems, keep it enabled. From MySQL 8.0.30, `DETECT_ONLY` is also available when incomplete-write detection is required without recovery copies. Changes between an enabled mode and `OFF` require a restart.

## Conclusion

The InnoDB Doublewrite Buffer protects against a real class of incomplete-page corruption. It adds write overhead in exchange for a recoverable page copy; it does not replace backups or protect against every corruption scenario. Unless you have a measured reason backed by documented storage guarantees, leave it on.

---

Have questions or a different experience with the Doublewrite Buffer in production? Feel free to [reach out](https://insaustis.com/#contact).
