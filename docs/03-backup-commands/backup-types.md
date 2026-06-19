---
tags: [backup, full, incremental, differential, cumulative]
---

# Backup Types Explained

Every RMAN backup has two independent choices: **what files** to include (scope) and **which blocks** to copy (strategy).

---

## Full vs Incremental

**Full backup** — copies every used block in the target files, regardless of when they last changed.

**Incremental backup** — copies only blocks that changed since a previous backup. Uses levels:

- **Level 0** — the baseline. Same data as a full backup, but RMAN can use it as the base for Level 1.
- **Level 1** — changes only. Copies blocks modified since the last Level 0 or Level 1.

!!! info "Full ≠ Whole"
    `FULL` = which blocks (all of them). `WHOLE` = which files (the entire database). These are independent choices.

---

## Differential vs Cumulative (Level 1 sub-types)

**Differential** (default) — backs up blocks changed since the **last Level 0 or Level 1**, whichever is more recent.

**Cumulative** — backs up blocks changed since the **last Level 0 only** (ignores other Level 1s).

| | Differential | Cumulative |
|---|---|---|
| Daily backup size | Small (only today's changes) | Grows during the week |
| Recovery steps | Apply L0 + all L1s in sequence | Apply L0 + latest L1 only |
| Recovery speed | Slower | Faster |

---

## Commands

```sql
-- Full backup (all used blocks, cannot be used as incremental base)
RMAN> BACKUP DATABASE;

-- Level 0 — baseline for incremental strategy
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;

-- Level 1 Differential (default — changes since last L0 or L1)
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;

-- Level 1 Cumulative (changes since last L0 only)
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;

-- Tablespace incremental
RMAN> BACKUP INCREMENTAL LEVEL 1 TABLESPACE users;

-- Include archive logs with any incremental
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
```

---

## Typical Weekly Strategies

=== "Differential (small daily backups)"
    ```sql
    -- Sunday: Level 0 baseline
    RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;

    -- Mon-Sat: Level 1 Differential (today's changes only)
    RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
    -- Recovery: restore L0 → apply each L1 in order
    ```

=== "Cumulative (faster recovery)"
    ```sql
    -- Sunday: Level 0 baseline
    RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;

    -- Mon-Sat: Level 1 Cumulative (all changes since Sunday)
    RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE PLUS ARCHIVELOG;
    -- Recovery: restore L0 → apply only the latest L1
    ```

=== "Simple Full (small databases)"
    ```sql
    -- Every night: full compressed backup
    RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
    RMAN> DELETE NOPROMPT OBSOLETE;
    ```

---

## Check Backup Status

```sql
-- High-level summary of all backups
RMAN> LIST BACKUP SUMMARY;

-- Full detail of all backups
RMAN> LIST BACKUP;

-- Backups of specific objects
RMAN> LIST BACKUP OF DATABASE;
RMAN> LIST BACKUP OF TABLESPACE users;
RMAN> LIST BACKUP OF DATAFILE 1;
RMAN> LIST BACKUP OF ARCHIVELOG ALL;

-- What still needs backing up?
RMAN> REPORT NEED BACKUP;
RMAN> REPORT NEED BACKUP DAYS 3;         -- not backed up in 3 days
RMAN> REPORT NEED BACKUP REDUNDANCY 2;   -- fewer than 2 backup copies
```

---

## Section Commands Summary

```sql
-- Full backup
RMAN> BACKUP DATABASE;
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;

-- Incremental
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;
RMAN> BACKUP INCREMENTAL LEVEL 1 TABLESPACE users;
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;

-- List and report
RMAN> LIST BACKUP SUMMARY;
RMAN> LIST BACKUP OF DATABASE;
RMAN> LIST BACKUP OF TABLESPACE users;
RMAN> LIST BACKUP OF DATAFILE 1;
RMAN> REPORT NEED BACKUP;
RMAN> REPORT NEED BACKUP DAYS 3;
RMAN> REPORT NEED BACKUP REDUNDANCY 2;
```
