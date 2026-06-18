---
tags: [archivelog, archive, redo]
---

# Archive Log Mode

## NOARCHIVELOG vs ARCHIVELOG

When Oracle's online redo log fills up, the Log Writer (LGWR) must switch to the next group. What happens to the old group depends on the archiving mode:

| Mode | What Happens to Old Log | Hot Backup? | Point-in-Time Recovery? |
|---|---|---|---|
| `NOARCHIVELOG` | Overwritten immediately | ❌ No | ❌ No |
| `ARCHIVELOG` | Copied to archive destination first | ✅ Yes | ✅ Yes |

!!! danger "NOARCHIVELOG is the default — change it for any production DB"
    A database in NOARCHIVELOG mode can only be backed up cold (database closed). If a datafile is lost, you can only restore to the last cold backup — everything since is gone.

---

## Check Current Mode

```sql
-- Method 1: ARCHIVE LOG LIST (SQL*Plus command)
ARCHIVE LOG LIST;
-- Output:
-- Database log mode              Archive Mode
-- Automatic archival             Enabled
-- Archive destination            USE_DB_RECOVERY_FILE_DEST
-- Oldest online log sequence     100
-- Next log sequence to archive   102
-- Current log sequence           102

-- Method 2: V$DATABASE
SELECT NAME, LOG_MODE FROM V$DATABASE;

-- Method 3: V$INSTANCE
SELECT ARCHIVER FROM V$INSTANCE;
-- STARTED = archiving enabled, STOPPED = not archiving
```

---

## Enable ARCHIVELOG Mode

!!! warning "Must be in MOUNT state — database must be closed first"

```sql
-- Step 1: Clean shutdown
SHUTDOWN IMMEDIATE;

-- Step 2: Start to MOUNT only (no OPEN)
STARTUP MOUNT;

-- Step 3 (optional): Set archive destination if not using FRA
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1
  = 'LOCATION=/u01/arch/ORCL'
  SCOPE = BOTH;

-- Step 4: Enable ARCHIVELOG mode
ALTER DATABASE ARCHIVELOG;

-- Step 5: Open the database
ALTER DATABASE OPEN;

-- Step 6: Verify
ARCHIVE LOG LIST;
SELECT LOG_MODE FROM V$DATABASE;
```

---

## Disable ARCHIVELOG Mode (Not Recommended)

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE NOARCHIVELOG;
ALTER DATABASE OPEN;
```

---

## Set Archive Destinations

Oracle supports up to 31 archive destinations (`LOG_ARCHIVE_DEST_1` through `LOG_ARCHIVE_DEST_31`):

```sql
-- Primary destination: local filesystem
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1
  = 'LOCATION=/u01/arch/ORCL'
  SCOPE = BOTH;

-- Secondary destination: another path (manual mirror)
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2
  = 'LOCATION=/u02/arch/ORCL'
  SCOPE = BOTH;

-- Use the FRA as the archive destination (recommended)
-- Set DB_RECOVERY_FILE_DEST and Oracle handles it automatically
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1
  = 'LOCATION=USE_DB_RECOVERY_FILE_DEST'
  SCOPE = BOTH;

-- Check archive destination settings
SHOW PARAMETER LOG_ARCHIVE_DEST;

-- Archive log naming format
SHOW PARAMETER LOG_ARCHIVE_FORMAT;
-- Default: %t_%s_%r.dbf  (thread_sequence_resetlogs.dbf)
```

---

## Archive Log Format

```sql
-- Customize archive log file names
ALTER SYSTEM SET LOG_ARCHIVE_FORMAT
  = 'arch_%d_%t_%s_%r.arc'
  SCOPE = SPFILE;
-- Requires restart

-- Variables:
-- %d = DB name
-- %t = thread number
-- %s = log sequence number
-- %r = resetlogs ID
```

---

## Manual Archiving Operations

```sql
-- Force a log switch (archives the current log)
ALTER SYSTEM SWITCH LOGFILE;

-- Archive all unarchived logs
ALTER SYSTEM ARCHIVE LOG ALL;

-- Archive a specific sequence number
ALTER SYSTEM ARCHIVE LOG SEQUENCE 105;

-- Archive the current log
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

---

## Monitor Archive Logs

```sql
-- View archived log history
SELECT SEQUENCE#,
       NAME,
       FIRST_TIME,
       NEXT_TIME,
       BLOCKS * BLOCK_SIZE / 1024 / 1024 AS SIZE_MB,
       ARCHIVED,
       STATUS
FROM V$ARCHIVED_LOG
WHERE DEST_ID = 1
ORDER BY SEQUENCE# DESC
FETCH FIRST 20 ROWS ONLY;

-- Check for gaps in archive log sequence
SELECT SEQUENCE#
FROM V$ARCHIVED_LOG
WHERE DEST_ID = 1
  AND STANDBY_DEST = 'NO'
MINUS
SELECT SEQUENCE#
FROM V$ARCHIVED_LOG
WHERE DEST_ID = 1
  AND STANDBY_DEST = 'NO'
  AND SEQUENCE# - 1 IN (
    SELECT SEQUENCE# FROM V$ARCHIVED_LOG WHERE DEST_ID=1
  );

-- Archive destination status
SELECT DEST_ID, STATUS, TARGET, ARCHIVER, DESTINATION, ERROR
FROM V$ARCHIVE_DEST
WHERE STATUS != 'INACTIVE'
ORDER BY DEST_ID;
```

---

## Delete Old Archive Logs via RMAN

```sql
-- Connect to RMAN
-- rman target /

-- Delete all archive logs already backed up
RMAN> DELETE ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK;

-- Delete archive logs older than 3 days
RMAN> DELETE NOPROMPT ARCHIVELOG ALL
      COMPLETED BEFORE 'SYSDATE-3';

-- Delete archive logs already backed up to tape
RMAN> DELETE NOPROMPT ARCHIVELOG ALL
      BACKED UP 1 TIMES TO SBT;
```

!!! warning "Never delete archive logs manually with rm"
    Always delete archive logs through RMAN so it can update its repository. Deleting with `rm` leaves stale records in the catalog/control file and can cause confusion during recovery.
