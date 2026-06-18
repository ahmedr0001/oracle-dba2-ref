---
tags: [backup, options, tags, device-type, not-backed-up]
---

# More Backup Options

## NOT BACKED UP

Avoid redundant backups by skipping files that have already been backed up a certain number of times. Essential for incremental or frequent backup schedules.

```sql
-- Back up archive logs not yet backed up at all (0 times)
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 1 TIMES;

-- Back up archive logs backed up fewer than 2 times
-- (ensures you have dual copies)
RMAN> BACKUP ARCHIVELOG NOT BACKED UP 2 TIMES;

-- Back up the database if not backed up in the last day
RMAN> BACKUP NOT BACKED UP SINCE TIME 'SYSDATE-1'
      DATABASE PLUS ARCHIVELOG;

-- Tablespace backup only if not backed up recently
RMAN> BACKUP NOT BACKED UP SINCE TIME 'SYSDATE-7'
      TABLESPACE users;
```

**Use case:** Hourly archive log backup job — each run only backs up logs created since the last run, without duplicating previously-backed-up logs.

---

## DEVICE TYPE

Override the default device type for a specific backup command:

```sql
-- Check and set the default
RMAN> SHOW DEFAULT DEVICE TYPE;
RMAN> CONFIGURE DEFAULT DEVICE TYPE TO DISK;   -- or SBT

-- Override for a single backup command
RMAN> BACKUP DEVICE TYPE DISK DATABASE;
RMAN> BACKUP DEVICE TYPE SBT DATABASE;

-- Override per channel in a RUN block
RMAN> RUN {
  ALLOCATE CHANNEL tape1 DEVICE TYPE SBT;
  BACKUP DATABASE;
  RELEASE CHANNEL tape1;
}
```

| Device | When to Use |
|---|---|
| `DISK` | Filesystem or ASM — fast, easy, most common |
| `SBT` | Tape library or cloud via Oracle Secure Backup / media agent |

---

## TAG — Naming Your Backups

A tag is a **label** you attach to a backup set or image copy. Makes it easy to reference specific backups later without knowing the backup key number.

```sql
-- Backup with a custom tag
RMAN> BACKUP DATAFILE 1, 2, 3, 4 TAG 'monthly_backup';

RMAN> BACKUP DATABASE TAG 'pre_patching_backup';

RMAN> BACKUP TABLESPACE users TAG 'before_purge_20260618';

-- Archive logs with a tag
RMAN> BACKUP ARCHIVELOG ALL TAG 'weekly_archivelogs';
```

**Rules for tags:**
- Multiple backup sets can share the same tag
- Tags are case-insensitive
- Up to 30 characters

**Restore using a tag:**
```sql
-- Reference a backup by its tag during restore
RMAN> RESTORE TABLESPACE users FROM TAG 'before_purge_20260618';
RMAN> RESTORE DATABASE FROM TAG 'pre_patching_backup';
```

---

## FORMAT — Custom Backup File Naming

```sql
-- Custom format for a specific backup
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/db_%d_%T_%s_%p';
-- produces: db_ORCL_20260618_42_1

-- Format with unique ID
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/%U';
-- %U = unique 8-character identifier

-- Date-organized directories
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/%Y/%M/%D/%U';
-- produces: /u01/rman_bkp/2026/06/18/3df5k2a1_1_1
```

---

## DELETE INPUT — Delete After Backup

```sql
-- Delete archive logs from disk after backing them up
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;

-- Delete ALL copies of archive logs (all destinations) after backup
RMAN> BACKUP ARCHIVELOG ALL DELETE ALL INPUT;

-- Same with filter
RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 100
      UNTIL SEQUENCE 200
      DELETE INPUT;
```

!!! warning "DELETE INPUT only for archive logs"
    You should never use `DELETE INPUT` with datafile backups — it would delete your live datafile after backing it up, crashing the database.

---

## COMPRESSED BACKUPSET

Compression reduces backup size significantly (often 50–70%) at the cost of CPU:

```sql
-- Compressed database backup
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;

-- Compressed incremental
RMAN> BACKUP AS COMPRESSED BACKUPSET INCREMENTAL LEVEL 1 DATABASE;

-- Set compression algorithm (configure once)
RMAN> CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';
-- Options: BASIC (default, no license), LOW, MEDIUM, HIGH
-- MEDIUM/HIGH require Oracle Advanced Compression license
```

---

## VALIDATE — Check Without Backing Up

Validate checks files for corruption without creating a backup. Useful for pre-backup health checks.

```sql
-- Validate entire database (check for block corruption)
RMAN> VALIDATE DATABASE;

-- Validate specific tablespace
RMAN> VALIDATE TABLESPACE users;

-- Validate a specific datafile
RMAN> VALIDATE DATAFILE 4;

-- Validate a backup piece (check backup readability)
RMAN> VALIDATE BACKUPSET 42;

-- Check for logical corruption too (more thorough, slower)
RMAN> VALIDATE DATABASE CHECK LOGICAL;
```

---

## Full BACKUP Command Reference

```
RMAN> BACKUP
        [AS {BACKUPSET | COMPRESSED BACKUPSET | COPY}]
        [INCREMENTAL LEVEL {0 | 1} [CUMULATIVE]]
        [DEVICE TYPE {DISK | SBT}]
        [NOT BACKED UP [n TIMES]]
        [NOT BACKED UP SINCE TIME 'expr']
        [FORMAT 'format_string']
        [TAG 'tag_name']
        {
          DATABASE |
          TABLESPACE ts_name [, ts_name ...] |
          DATAFILE file_num [, file_num ...] |
          DATAFILE 'filepath' |
          CURRENT CONTROLFILE |
          SPFILE |
          ARCHIVELOG {ALL | FROM SCN n | UNTIL SCN n | 
                      FROM SEQUENCE n | UNTIL SEQUENCE n |
                      FROM TIME 'expr' | UNTIL TIME 'expr' |
                      SCN BETWEEN n AND n |
                      SEQUENCE BETWEEN n AND n}
        }
        [INCLUDE CURRENT CONTROLFILE]
        [PLUS ARCHIVELOG]
        [DELETE [ALL] INPUT];
```
