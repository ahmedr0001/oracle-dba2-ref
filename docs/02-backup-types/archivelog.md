---
tags: [archivelog, archive, redo]
---

# Archive Log Mode

When an online redo log fills up, Oracle must switch to the next group. What happens to the old log depends on the mode:

| Mode | Old log is... | Hot backup? | Point-in-time recovery? |
|---|---|---|---|
| `NOARCHIVELOG` | Overwritten immediately | No | No |
| `ARCHIVELOG` | Copied to archive destination first | Yes | Yes |

!!! danger "NOARCHIVELOG is the default — change it for any production DB"
    In NOARCHIVELOG mode, RMAN can only back up a closed database. Any data written since the last cold backup is permanently lost if a disk fails.

---

## Check Current Mode

```sql
-- SQL*Plus command (not SQL) — shows full archiving status
ARCHIVE LOG LIST;

-- SQL query
SELECT NAME, LOG_MODE FROM V$DATABASE;

-- Check archiver process status
SELECT ARCHIVER FROM V$INSTANCE;
-- STARTED = archiving running | STOPPED = not archiving
```

---

## Enable ARCHIVELOG Mode

```sql
-- Step 1: clean shutdown
SHUTDOWN IMMEDIATE;

-- Step 2: start to MOUNT only — do not open
STARTUP MOUNT;

-- Step 3: enable archivelog mode (optional: set destination first — see below)
ALTER DATABASE ARCHIVELOG;

-- Step 4: open the database
ALTER DATABASE OPEN;

-- Step 5: verify
ARCHIVE LOG LIST;
SELECT LOG_MODE FROM V$DATABASE;
```

---

## Set Archive Destinations

```sql
-- Option A: use FRA as the archive destination (recommended)
-- Just configure DB_RECOVERY_FILE_DEST — Oracle handles the rest
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1
  = 'LOCATION=USE_DB_RECOVERY_FILE_DEST'
  SCOPE = BOTH;

-- Option B: manual filesystem path
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1
  = 'LOCATION=/u01/arch/ORCL'
  SCOPE = BOTH;  -- takes effect now AND after restart

-- Option C: two destinations (local + mirror)
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1 = 'LOCATION=/u01/arch/ORCL' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2 = 'LOCATION=/u02/arch/ORCL' SCOPE=BOTH;

-- Check all archive destination settings
SHOW PARAMETER LOG_ARCHIVE_DEST;
```

---

## Archive Log File Naming

```sql
-- View current archive log format
SHOW PARAMETER LOG_ARCHIVE_FORMAT;
-- Default: %t_%s_%r.dbf   (thread_sequence_resetlogsID.dbf)

-- Customize (requires restart — SPFILE only)
ALTER SYSTEM SET LOG_ARCHIVE_FORMAT
  = 'arch_%d_%t_%s_%r.arc'
  SCOPE = SPFILE;
-- %d=DB name, %t=thread, %s=sequence#, %r=resetlogs ID
```

---

## Manual Archiving Operations

```sql
-- Force a log switch (archives the current online log)
ALTER SYSTEM SWITCH LOGFILE;

-- Archive all online logs that are not yet archived
ALTER SYSTEM ARCHIVE LOG ALL;

-- Archive a specific sequence number
ALTER SYSTEM ARCHIVE LOG SEQUENCE 105;

-- Archive the current online log
ALTER SYSTEM ARCHIVE LOG CURRENT;
```

---

## Monitor Archive Logs

```sql
-- Recent archive log files: sequence, file name, times, status
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

-- Check archive destination status and errors
SELECT DEST_ID, STATUS, TARGET, ARCHIVER, DESTINATION, ERROR
FROM V$ARCHIVE_DEST
WHERE STATUS != 'INACTIVE'
ORDER BY DEST_ID;
```

---

## Disable ARCHIVELOG Mode

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE NOARCHIVELOG;
ALTER DATABASE OPEN;
```

---

## Delete Archive Logs (Always via RMAN)

```bash
rman target /
```

```sql
-- Delete archive logs already backed up (safe — they're in a backup)
RMAN> DELETE NOPROMPT ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK;

-- Delete archive logs older than 3 days
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-3';

-- Delete all archive logs (use carefully)
RMAN> DELETE NOPROMPT ARCHIVELOG ALL;
```

!!! warning "Never use 'rm' to delete archive logs"
    Always delete through RMAN so it updates its repository. Manual deletion with `rm` leaves orphan records in the catalog/control file.

---

## Section Commands Summary

```sql
-- Check mode
ARCHIVE LOG LIST;
SELECT NAME, LOG_MODE FROM V$DATABASE;
SELECT ARCHIVER FROM V$INSTANCE;

-- Enable
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

-- Destinations
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1 = 'LOCATION=/u01/arch/ORCL' SCOPE=BOTH;
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1 = 'LOCATION=USE_DB_RECOVERY_FILE_DEST' SCOPE=BOTH;
SHOW PARAMETER LOG_ARCHIVE_DEST;

-- Manual operations
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM ARCHIVE LOG ALL;
ALTER SYSTEM ARCHIVE LOG SEQUENCE 105;
ALTER SYSTEM ARCHIVE LOG CURRENT;

-- Monitor
SELECT SEQUENCE#, NAME, FIRST_TIME, STATUS FROM V$ARCHIVED_LOG WHERE DEST_ID=1 ORDER BY SEQUENCE# DESC;
SELECT DEST_ID, STATUS, DESTINATION, ERROR FROM V$ARCHIVE_DEST WHERE STATUS != 'INACTIVE';
```

```bash
# Delete via RMAN
rman target /
RMAN> DELETE NOPROMPT ARCHIVELOG ALL BACKED UP 1 TIMES TO DISK;
RMAN> DELETE NOPROMPT ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-3';
```
