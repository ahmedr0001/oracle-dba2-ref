---
tags: [rman, architecture, catalog, channels]
---

# RMAN Architecture

RMAN is an Oracle client application — not a separate database. You run it from the Linux shell, it connects to your database as SYSDBA, and handles all backup/recovery logic automatically.

---

## Key Components

**Target Database** — the database you're backing up. RMAN connects to it as SYSDBA.

**RMAN Repository** — where RMAN stores metadata about every backup (what files, when, where). Stored in one of two places:

| Option | Where | Pros | Cons |
|---|---|---|---|
| **Control File** (default) | Inside the target DB | No setup needed | Only 7 days history by default |
| **Recovery Catalog** | Separate database | Unlimited history, multi-DB support | Requires a second DB to maintain |

**Channel** — an I/O worker process that reads datafiles and writes to backup destination. More channels = faster backup (one per disk is optimal).

**Backup Destination** — disk directory, ASM disk group, or tape library (SBT).

---

## Backup Output: Backup Set vs Image Copy

| | Backup Set (default) | Image Copy |
|---|---|---|
| Format | RMAN proprietary (compressed) | Exact copy of the original file |
| Space | Less (can compress) | Same as original |
| Restore needed? | Yes — must decompress first | No — can use directly |
| Recovery speed | Slower (restore then recover) | Faster (just recover) |
| Command | `BACKUP DATABASE` | `BACKUP AS COPY DATABASE` |

---

## What RMAN Can and Cannot Back Up

| ✅ Can Back Up | ❌ Cannot Back Up |
|---|---|
| Datafiles | Temp files |
| Control files | Online redo log files |
| SPFILE | Password file |
| Archived redo logs | `listener.ora`, `tnsnames.ora` |

!!! warning "Online redo logs are never backed up directly"
    RMAN archives them first (log switch), then backs up the resulting archive log.

---

## Section Commands Summary

```bash
# Connect to RMAN (see full syntax in Connections page)
rman target /

# Check what RMAN can see
RMAN> REPORT SCHEMA;       -- lists all datafiles RMAN knows about
RMAN> LIST BACKUP SUMMARY; -- lists all recorded backups
RMAN> SHOW ALL;            -- shows current RMAN configuration
```
