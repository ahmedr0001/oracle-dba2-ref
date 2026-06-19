---
tags: [fra, fast-recovery-area, storage]
---

# Fast Recovery Area (FRA)

The FRA is a single directory (or ASM disk group) where Oracle automatically stores and manages all recovery-related files. Instead of configuring separate paths for archive logs, backups, and flashback logs, you point Oracle at one location and it handles the rest.

**Permanent files** (instance uses actively — never auto-deleted): control file copies, redo log members.
**Transient files** (Oracle auto-manages space by deleting old ones): archived logs, RMAN backup sets, image copies, flashback logs, control file auto backups.

---

## Configure the FRA

```sql
-- Set the FRA size limit first — Oracle won't write past this
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 20G
  SCOPE = BOTH;  -- takes effect immediately AND persists after restart

-- Set the FRA directory path (filesystem or ASM)
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST
  = '/u01/app/oracle/fast_recovery_area'
  SCOPE = BOTH;

-- Verify both settings
SHOW PARAMETER DB_RECOVERY_FILE_DEST;
```

---

## Disable the FRA

```sql
-- Remove the FRA destination (empty string = disabled)
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '' SCOPE = BOTH;
```

---

## What Happens When FRA Fills Up

Oracle tries to free space in this order:
1. Delete obsolete backup sets
2. Delete redundant copies
3. Delete archive logs already backed up

If it still can't free space, you get:

```
ORA-00257: archiver error. Connect internal only, until freed.
```

This means the archiver cannot write new archive logs — **the database stops accepting new transactions**.

### Fix ORA-00257

```sql
-- Connect as SYSDBA (ORA-00257 still allows SYSDBA connections)
sqlplus / as sysdba

-- Check which file type is consuming FRA space
SELECT FILE_TYPE, PERCENT_SPACE_USED, PERCENT_SPACE_RECLAIMABLE
FROM V$RECOVERY_AREA_USAGE;

-- Option A: increase FRA size
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 30G SCOPE=BOTH;
```

```bash
# Option B: delete old archive logs via RMAN
rman target /
RMAN> DELETE NOPROMPT OBSOLETE;
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-3';
```

---

## Monitor FRA Space

```sql
-- Summary: total, used, reclaimable
-- SPACE_LIMIT = max size you configured
-- SPACE_USED = how much is used now
-- SPACE_RECLAIMABLE = how much can be freed (obsolete/already backed up files)
SELECT NAME, SPACE_LIMIT/1024/1024/1024 AS LIMIT_GB,
       SPACE_USED/1024/1024/1024       AS USED_GB,
       SPACE_RECLAIMABLE/1024/1024/1024 AS RECLAIMABLE_GB,
       NUMBER_OF_FILES
FROM V$RECOVERY_FILE_DEST;

-- Breakdown by file type
SELECT FILE_TYPE,
       PERCENT_SPACE_USED        AS PCT_USED,
       PERCENT_SPACE_RECLAIMABLE AS PCT_RECLAIMABLE,
       NUMBER_OF_FILES
FROM V$RECOVERY_AREA_USAGE;
```

---

## Section Commands Summary

```sql
-- Configure
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 20G SCOPE=BOTH;
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '/u01/fra' SCOPE=BOTH;

-- Disable
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '' SCOPE=BOTH;

-- Check settings
SHOW PARAMETER DB_RECOVERY_FILE_DEST;

-- Monitor space
SELECT NAME, SPACE_LIMIT, SPACE_USED, SPACE_RECLAIMABLE FROM V$RECOVERY_FILE_DEST;
SELECT FILE_TYPE, PERCENT_SPACE_USED, PERCENT_SPACE_RECLAIMABLE FROM V$RECOVERY_AREA_USAGE;
```

```bash
# Fix ORA-00257 (FRA full)
rman target /
RMAN> DELETE NOPROMPT OBSOLETE;
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-3';
```
