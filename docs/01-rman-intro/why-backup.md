---
tags: [rman, backup, failures]
---

# Why Backup & Failure Types

## Why Backups Exist

Data is the most valuable asset in any organization. Hardware fails, humans make mistakes, and disasters happen. A DBA's core responsibility is not just keeping the database running — it's ensuring data can be **recovered** when the inevitable happens.

> **The question is never "will we lose data?" — it's "how much data can we afford to lose, and how fast must we recover?"**

---

## Categories of Failure

Oracle classifies failures into six major categories. Understanding each helps you choose the right protection strategy.

| Failure Type | What Happens | Example | Fix |
|---|---|---|---|
| **User Process** | A client application or session crashes | `CTRL+C` on a running query, app crash | Automatic — Oracle rolls back uncommitted work |
| **Instance** | The Oracle instance itself dies unexpectedly | Server power cut, OS crash | Automatic instance recovery on next `STARTUP` |
| **Media / Storage** | A disk, datafile, or storage device fails | Bad sector corrupts a `.dbf` file | Restore from backup + apply archive logs |
| **Inaccessible Component** | A required file can't be opened | Wrong permissions on a datafile | Fix OS permissions or restore file |
| **Physical Corruption** | Block header is damaged but file is accessible | Disk write error mid-block | RMAN block media recovery, restore datafile |
| **Logical Corruption (Inconsistency)** | SCN mismatch — data is structurally valid but logically wrong | Accidental `DELETE` or `DROP TABLE` | Flashback, point-in-time recovery |
| **Disaster** | Entire site is lost | Fire, flood, data center outage | Disaster Recovery (Oracle Data Guard, OSB) |

---

## The DBA's Responsibility

A backup strategy must answer four questions before any incident occurs:

```
1. What do we back up?       → entire DB, specific tablespaces, archivelogs?
2. How often?                → daily full, hourly incremental?
3. Where do we store it?     → local disk, tape, remote site?
4. How fast must we recover? → seconds (RAC/DataGuard) or hours (RMAN restore)?
```

!!! warning "A backup you can't restore is not a backup"
    Always test your recovery process in a separate environment. Many organizations discover their backups are broken only during an actual disaster.

---

## Oracle Protection Solutions by RTO

Oracle offers different solutions depending on how quickly you need to recover:

=== "Seconds / Minutes (Infrastructure Failure)"
    | Solution | Description |
    |---|---|
    | **Oracle RAC** | Multiple instances on the same database — instance fails, others keep serving |
    | **Oracle Data Guard** | Standby database on separate hardware — switchover in seconds |
    | **Oracle GoldenGate** | Real-time data replication across databases |

=== "Minutes / Hours (Human Error)"
    | Solution | Description |
    |---|---|
    | **Oracle Flashback** | Roll back a table or entire DB to a past point without restore |
    | **Oracle RMAN** | Restore specific datafiles, tablespaces, or whole DB |
    | **Data Guard** | Activate a standby to replace the primary |

=== "Hours / Days (Catastrophic)"
    | Solution | Description |
    |---|---|
    | **Oracle RMAN + OSB** | Full restore from backup — local tape or Oracle Secure Backup cloud |
    | **Offsite backups** | Backups shipped to a geographically separate location |
