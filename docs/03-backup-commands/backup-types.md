---
tags: [backup, full, incremental, differential, cumulative]
---

# Backup Types Explained

## Two Dimensions of Every Backup

Every RMAN backup has two independent attributes that are often confused:

```
Dimension 1: SCOPE        Dimension 2: STRATEGY
─────────────────         ─────────────────────
Whole Backup              Full Backup
Partial Backup            Incremental Backup
                            ├── Differential
                            └── Cumulative
```

These are independent choices. For example: *"Partial Incremental Differential backup of the USERS tablespace"* is completely valid.

---

## Full vs Incremental

### Full Backup
Copies **every used data block** in the target files — regardless of when they were last changed.

```sql
-- Full backup of the whole database
RMAN> BACKUP DATABASE;

-- Full backup of a tablespace
RMAN> BACKUP TABLESPACE users;
```

!!! info "Full ≠ Whole"
    `FULL` refers to which **blocks** are copied (all of them).
    `WHOLE` refers to which **files** are included (the entire database).
    A "Full Whole" backup copies every block in every datafile.

### Incremental Backup
Copies only **blocks that changed** since a reference backup. Uses backup levels: **Level 0** and **Level 1**.

- **Level 0** — the baseline. Copies all used blocks (same data as a full backup, but RMAN can use it as an incremental base).
- **Level 1** — changes only. Copies blocks modified since the last Level 0 or Level 1.

```
              ← Full backup (Level 0 basis)
Level 0 ────────────────────────────────────────────────────►
Level 1 (Differential) ──┼──────────┼──────────┼──────────►
                         Mon        Tue        Wed
                         (changes   (changes   (changes
                          since L0)  since Mon)  since Tue)
```

---

## Differential vs Cumulative Incremental

This is a sub-choice within Level 1 incremental:

### Differential Incremental (Default)
Backs up blocks changed since the **last Level 1 OR Level 0** — whichever is more recent.

```
Sunday: Level 0 (all blocks)           → large backup
Monday: Level 1 Differential           → changes since Sunday
Tuesday: Level 1 Differential          → changes since Monday only
Wednesday: Level 1 Differential        → changes since Tuesday only
```

**Pros:** Each incremental backup is small (only daily changes)
**Cons:** Recovery applies many incremental backups — slower restore

### Cumulative Incremental
Backs up blocks changed since the **last Level 0** only — ignores other Level 1s.

```
Sunday: Level 0 (all blocks)           → large backup
Monday: Level 1 Cumulative             → all changes since Sunday
Tuesday: Level 1 Cumulative            → all changes since Sunday (includes Mon)
Wednesday: Level 1 Cumulative          → all changes since Sunday (includes Mon+Tue)
```

**Pros:** Recovery is simpler — apply Level 0, then latest Level 1 only
**Cons:** Each cumulative backup grows as the week progresses

---

## RMAN Incremental Backup Commands

```sql
-- Level 0 (baseline — must run before any Level 1)
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;

-- Level 1 Differential (default — changes since last Level 0 or 1)
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;

-- Level 1 Cumulative (changes since last Level 0 only)
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;

-- Incremental backup of specific tablespace
RMAN> BACKUP INCREMENTAL LEVEL 1 TABLESPACE users;

-- Incremental + archive logs
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
```

---

## Typical Weekly Backup Strategies

=== "Strategy A — Differential"
    ```sql
    -- Sunday night: Level 0 (full baseline)
    RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;

    -- Mon–Sat nights: Level 1 Differential (daily changes only)
    RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
    ```
    **Recovery:** Apply Level 0 + all Level 1s in sequence

=== "Strategy B — Cumulative"
    ```sql
    -- Sunday night: Level 0
    RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;

    -- Mon–Sat nights: Level 1 Cumulative
    RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE PLUS ARCHIVELOG;
    ```
    **Recovery:** Apply Level 0 + latest Level 1 only (faster)

=== "Strategy C — Simple Full"
    ```sql
    -- Every night: full backup (small databases where speed isn't a concern)
    RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
    RMAN> DELETE NOPROMPT OBSOLETE;
    ```

---

## Check What's Been Backed Up

```sql
-- Summary of all backups
RMAN> LIST BACKUP SUMMARY;

-- Detailed backup info
RMAN> LIST BACKUP;

-- Backups of a specific file
RMAN> LIST BACKUP OF DATAFILE 1;
RMAN> LIST BACKUP OF TABLESPACE users;
RMAN> LIST BACKUP OF ARCHIVELOG ALL;

-- Image copies only
RMAN> LIST COPY;

-- What files still need backup?
RMAN> REPORT NEED BACKUP;
RMAN> REPORT NEED BACKUP DAYS 3;    -- files not backed up in 3 days
RMAN> REPORT NEED BACKUP REDUNDANCY 2;  -- files with fewer than 2 backups
```
