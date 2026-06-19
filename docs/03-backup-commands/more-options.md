---
tags: [backup, options, tags, device-type, not-backed-up]
---

# More Backup Options

---

## NOT BACKED UP — Skip Already-Backed-Up Files

Avoids re-backing up files already covered, useful in frequent/incremental schedules.

```sql
-- Back up archive logs not yet backed up at all
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 1 TIMES;

-- Back up logs backed up fewer than 2 times (ensure dual copies)
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 2 TIMES;

-- Back up the database if it hasn't been backed up in the last 24 hours
RMAN> BACKUP NOT BACKED UP SINCE TIME 'SYSDATE-1'
      DATABASE PLUS ARCHIVELOG;

-- Tablespace backup if not done in the last week
RMAN> BACKUP NOT BACKED UP SINCE TIME 'SYSDATE-7'
      TABLESPACE users;
```

---

## DEVICE TYPE — Override Where Backup Is Written

```sql
-- Use disk for this specific backup (regardless of configured default)
RMAN> BACKUP DEVICE TYPE DISK DATABASE;

-- Use tape (SBT = System Backup to Tape)
RMAN> BACKUP DEVICE TYPE SBT DATABASE;

-- Change default for all future backups
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO DISK;
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO SBT;

-- Override per channel in a RUN block
RMAN> RUN {
  ALLOCATE CHANNEL tape1 DEVICE TYPE SBT;  -- use tape for this job only
  BACKUP DATABASE;
  RELEASE CHANNEL tape1;                    -- release when done
}
```

---

## TAG — Label Your Backups

A tag is a text label attached to a backup. Lets you reference it by name later instead of by key number.

```sql
-- Tag a database backup
RMAN> BACKUP DATABASE TAG 'pre_patching_backup';

-- Tag a tablespace backup
RMAN> BACKUP TABLESPACE users TAG 'before_purge_20260618';

-- Tag archive logs
RMAN> BACKUP ARCHIVELOG ALL TAG 'weekly_archivelogs';

-- Restore using a tag (instead of needing to know backup key)
RMAN> RESTORE TABLESPACE users FROM TAG 'before_purge_20260618';
RMAN> RESTORE DATABASE FROM TAG 'pre_patching_backup';
```

Tags are case-insensitive, up to 30 characters. Multiple backup sets can share a tag.

---

## FORMAT — Custom Backup File Names

```sql
-- Custom format string (overrides configured channel format for this job)
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/db_%d_%T_%s_%p';
-- Result: db_ORCL_20260618_42_1

-- Unique ID (auto-generated, guaranteed no collisions)
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/%U';

-- Date-organized directories
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/%Y/%M/%D/%U';
-- Result: /u01/rman_bkp/2026/06/18/3df5k2a1_1_1
```

---

## DELETE INPUT — Delete Source After Backup

```sql
-- After backing up archive logs, delete them from disk
-- (deletes only the specific logs backed up in this command)
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;

-- Delete ALL copies of archive logs from all destinations after backup
RMAN> BACKUP ARCHIVELOG ALL DELETE ALL INPUT;

-- With a filter
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100 UNTIL SEQUENCE 200 DELETE INPUT;
```

!!! warning "Never use DELETE INPUT with datafile backups"
    `DELETE INPUT` on a datafile backup would delete your live datafile after copying it — crashing the database.

---

## COMPRESSED BACKUPSET

```sql
-- Compress the backup set (smaller files, more CPU)
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;
RMAN> BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL 1 DATABASE;

-- Set compression level (BASIC = no license required)
-- MEDIUM and HIGH require Oracle Advanced Compression license
RMAN> CONFIGURE COMPRESSION ALGORITHM 'BASIC';
RMAN> CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';
RMAN> CONFIGURE COMPRESSION ALGORITHM 'HIGH';
```

---

## VALIDATE — Check Files Without Backing Up

```sql
-- Check database for block corruption (no backup created)
RMAN> VALIDATE DATABASE;

-- Check specific tablespace
RMAN> VALIDATE TABLESPACE users;

-- Check specific datafile
RMAN> VALIDATE DATAFILE 4;

-- Check an existing backup piece
RMAN> VALIDATE BACKUPSET 42;

-- Also check for logical corruption (slower but more thorough)
RMAN> VALIDATE DATABASE CHECK LOGICAL;
```

---

## Section Commands Summary

```sql
-- NOT BACKED UP
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 1 TIMES;
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 2 TIMES;
RMAN> BACKUP NOT BACKED UP SINCE TIME 'SYSDATE-1' DATABASE PLUS ARCHIVELOG;

-- DEVICE TYPE
RMAN> BACKUP DEVICE TYPE DISK DATABASE;
RMAN> BACKUP DEVICE TYPE SBT DATABASE;

-- TAG
RMAN> BACKUP DATABASE TAG 'pre_patching_backup';
RMAN> BACKUP TABLESPACE users TAG 'before_purge';
RMAN> RESTORE DATABASE FROM TAG 'pre_patching_backup';

-- FORMAT
RMAN> BACKUP DATABASE FORMAT '/path/%d_%T_%s_%p_%U';

-- DELETE INPUT (archive logs only)
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;
RMAN> BACKUP ARCHIVELOG ALL DELETE ALL INPUT;

-- COMPRESSED
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;
RMAN> CONFIGURE COMPRESSION ALGORITHM 'BASIC';

-- VALIDATE
RMAN> VALIDATE DATABASE;
RMAN> VALIDATE DATABASE CHECK LOGICAL;
RMAN> VALIDATE TABLESPACE users;
RMAN> VALIDATE DATAFILE 4;
RMAN> VALIDATE BACKUPSET 42;
```
