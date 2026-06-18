---
tags: [backup, whole, partial, database, tablespace, datafile]
---

# Whole & Partial Backup

## Whole Database Backup

A whole backup includes all datafiles, the control file, and the SPFILE. It is the most common form of backup.

```sql
-- Simplest whole backup (backup set, to FRA or configured channel)
RMAN> BACKUP DATABASE;

-- With archive logs included
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- Compressed (reduces backup size significantly — CPU cost)
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;

-- With explicit format (overrides configured channel format)
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/db_%d_%T_%U';

-- Full backup and then clean up obsolete backups in one job
RMAN> RUN {
  BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
  DELETE NOPROMPT OBSOLETE;
}
```

!!! tip "PLUS ARCHIVELOG — what it actually does"
    Running `BACKUP DATABASE PLUS ARCHIVELOG` executes these steps automatically:

    1. `ALTER SYSTEM SWITCH LOGFILE` — archives the current online log
    2. `BACKUP ARCHIVELOG ALL` — backs up all archive logs
    3. Backs up the database (datafiles)
    4. `ALTER SYSTEM SWITCH LOGFILE` — archives again
    5. Backs up any new archive logs created during the DB backup

    This ensures you have a complete, self-consistent backup including all redo needed for recovery.

---

## Partial Backup — Tablespace

A partial backup covers only selected tablespaces. This is valid for both full and incremental strategies.

```sql
-- Backup one tablespace
RMAN> BACKUP TABLESPACE users;

-- Backup multiple tablespaces
RMAN> BACKUP TABLESPACE users, tools, app_data;

-- Backup a tablespace with its archive logs
RMAN> BACKUP TABLESPACE users PLUS ARCHIVELOG;

-- Incremental tablespace backup
RMAN> BACKUP INCREMENTAL LEVEL 1 TABLESPACE users;

-- With format
RMAN> BACKUP TABLESPACE users
      FORMAT '/u01/rman_bkp/ts_users_%T_%U';
```

!!! info "Is a tablespace backup consistent?"
    A tablespace backup taken while the database is **open** is **inconsistent** (hot backup). Recovery requires applying archive logs from the time of the backup. This is perfectly valid in ARCHIVELOG mode.

---

## Partial Backup — Datafile

Back up individual datafiles by file number or path. Use `V$DATAFILE` or `DBA_DATA_FILES` to find file numbers.

```sql
-- Check datafile numbers first
-- SELECT FILE#, NAME FROM V$DATAFILE;
-- or: SELECT FILE_ID, FILE_NAME FROM DBA_DATA_FILES;

-- Backup datafiles by number
RMAN> BACKUP DATAFILE 1, 2, 3, 4;

-- Backup a datafile by full path
RMAN> BACKUP DATAFILE '/u01/app/oradata/ORCL/users01.dbf';

-- Multiple paths
RMAN> BACKUP DATAFILE
        '/u01/app/oradata/ORCL/users01.dbf',
        '/u01/app/oradata/ORCL/system01.dbf';

-- Incremental datafile backup
RMAN> BACKUP INCREMENTAL LEVEL 1 DATAFILE 4, 5;
```

---

## Backup SPFILE

The SPFILE (Server Parameter File) is automatically backed up when `CONTROLFILE AUTOBACKUP` is ON. You can also back it up manually:

```sql
-- Manual SPFILE backup
RMAN> BACKUP SPFILE;

-- Backup SPFILE to a specific location
RMAN> BACKUP SPFILE FORMAT '/u01/rman_bkp/spfile_%d_%T.bkp';
```

---

## Useful Queries for Identifying Files to Back Up

```sql
-- List all datafiles with their tablespace and size
SELECT FILE_ID, FILE_NAME, TABLESPACE_NAME,
       ROUND(BYTES/1024/1024,0) AS SIZE_MB
FROM DBA_DATA_FILES
ORDER BY FILE_ID;

-- Same from V$ (live status)
SELECT FILE#, NAME, STATUS, ROUND(BYTES/1024/1024,0) AS SIZE_MB
FROM V$DATAFILE;

-- Check which tablespaces exist
SELECT TABLESPACE_NAME, STATUS, CONTENTS
FROM DBA_TABLESPACES;
```

---

## Monitoring Backup Progress

```sql
-- Monitor a running RMAN backup (from another SQL*Plus session)
SELECT SID, SERIAL#, CONTEXT, SOFAR, TOTALWORK,
       ROUND(SOFAR/TOTALWORK*100,2) AS PCT_DONE,
       MESSAGE
FROM V$SESSION_LONGOPS
WHERE OPNAME LIKE 'RMAN%'
  AND TOTALWORK > 0
  AND SOFAR < TOTALWORK;

-- Check backup completion status
SELECT SESSION_KEY, INPUT_TYPE, STATUS,
       TO_CHAR(START_TIME,'DD-MON-YYYY HH24:MI') AS START_TIME,
       TO_CHAR(END_TIME,'DD-MON-YYYY HH24:MI')   AS END_TIME,
       ELAPSED_SECONDS
FROM V$RMAN_BACKUP_JOB_DETAILS
ORDER BY START_TIME DESC
FETCH FIRST 10 ROWS ONLY;
```
