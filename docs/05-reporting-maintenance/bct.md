---
tags: [bct, block-change-tracking, incremental, performance]
---

# Block Change Tracking (BCT)

Without BCT, RMAN scans every block in every datafile to find which ones changed since the last backup — even if only 1% of the data changed. With BCT enabled, a small binary tracking file records exactly which blocks changed, so RMAN reads only those during incremental backups.

**When to use:** databases where less than ~20% of blocks change between backups. The more stable your data, the bigger the benefit.

- Disabled by default
- BCT file grows by 10 MB per datafile
- Only 8 incremental backup levels can be tracked simultaneously
- Default location: `DB_CREATE_FILE_DEST`

---

## Enable / Disable BCT

```sql
-- Enable with default location (DB_CREATE_FILE_DEST)
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING;

-- Enable with a specific file path
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
  USING FILE '/u01/app/oracle/bct/ORCL_bct.chg';

-- Disable (Oracle deletes the BCT file automatically)
ALTER DATABASE DISABLE BLOCK CHANGE TRACKING;
```

---

## Rename the BCT File

Must be in MOUNT state to rename.

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;

-- Oracle renames the file and updates its internal pointer
ALTER DATABASE RENAME FILE
  '/u01/old_path/ORCL_bct.chg'
  TO '/u02/new_path/ORCL_bct.chg';

ALTER DATABASE OPEN;
```

---

## Check BCT Status

```sql
-- STATUS = ENABLED or DISABLED
SELECT FILENAME, STATUS, BYTES/1024/1024 AS SIZE_MB
FROM V$BLOCK_CHANGE_TRACKING;
```

---

## Measure BCT Effectiveness

```sql
-- PCT_READ_FOR_BACKUP shows what % of blocks RMAN had to read
-- If BCT is working well, this should be much less than 100%
SELECT
    FILE#,
    AVG(DATAFILE_BLOCKS)                     AS TOTAL_BLOCKS,
    AVG(BLOCKS_READ)                         AS BLOCKS_READ,
    AVG(BLOCKS_READ / DATAFILE_BLOCKS) * 100 AS PCT_READ_FOR_BACKUP,
    AVG(BLOCKS)                              AS BLOCKS_CHANGED
FROM V$BACKUP_DATAFILE
WHERE USED_CHANGE_TRACKING = 'YES'   -- only rows where BCT was used
  AND INCREMENTAL_LEVEL > 0          -- only level 1 backups
GROUP BY FILE#;
```

---

## Section Commands Summary

```sql
-- Enable
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING;
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING USING FILE '/path/bct.chg';

-- Disable
ALTER DATABASE DISABLE BLOCK CHANGE TRACKING;

-- Rename (MOUNT state required)
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE RENAME FILE '/old/bct.chg' TO '/new/bct.chg';
ALTER DATABASE OPEN;

-- Check status
SELECT FILENAME, STATUS, BYTES/1024/1024 AS SIZE_MB FROM V$BLOCK_CHANGE_TRACKING;

-- Measure effectiveness
SELECT FILE#, AVG(BLOCKS_READ/DATAFILE_BLOCKS)*100 AS PCT_READ_FOR_BACKUP
FROM V$BACKUP_DATAFILE
WHERE USED_CHANGE_TRACKING='YES' AND INCREMENTAL_LEVEL > 0
GROUP BY FILE#;
```
