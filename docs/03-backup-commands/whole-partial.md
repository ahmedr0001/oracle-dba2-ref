---
tags: [backup, whole, partial, database, tablespace, datafile]
---

# Whole & Partial Backup

---

## Whole Database Backup

```sql
-- Simplest full backup (to FRA or configured channel)
RMAN> BACKUP DATABASE;

-- Include archive logs (PLUS ARCHIVELOG also switches log before and after backup)
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;

-- Compressed — smaller files, uses more CPU
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;

-- Compressed + archive logs (most common production command)
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;

-- Custom output path (overrides configured channel format)
RMAN> BACKUP DATABASE FORMAT '/u01/rman_bkp/db_%d_%T_%U';

-- Label this backup with a tag for easy reference later
RMAN> BACKUP DATABASE TAG 'pre_upgrade_backup';

-- Full job: backup then clean up old backups in one block
RMAN> RUN {
  BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
  DELETE NOPROMPT OBSOLETE;
}
```

!!! tip "What PLUS ARCHIVELOG actually does"
    1. Switch current redo log → archives it
    2. Back up all archive logs
    3. Back up the database
    4. Switch log again → archives logs created during backup
    5. Back up those new logs

    Result: a self-contained backup you can restore and recover completely.

---

## Partial Backup — Tablespace

```sql
-- Back up one tablespace
RMAN> BACKUP TABLESPACE users;

-- Back up multiple tablespaces in one command
RMAN> BACKUP TABLESPACE users, tools, app_data;

-- Tablespace backup + archive logs
RMAN> BACKUP TABLESPACE users PLUS ARCHIVELOG;

-- Incremental tablespace backup
RMAN> BACKUP INCREMENTAL LEVEL 1 TABLESPACE users;

-- Custom path
RMAN> BACKUP TABLESPACE users FORMAT '/u01/rman_bkp/ts_users_%T_%U';
```

---

## Partial Backup — Datafile

```sql
-- Find datafile numbers first
-- SELECT FILE_ID, FILE_NAME, TABLESPACE_NAME FROM DBA_DATA_FILES;
-- Or: SELECT FILE#, NAME FROM V$DATAFILE;

-- Back up specific datafiles by number
RMAN> BACKUP DATAFILE 1, 2, 3, 4;

-- By full path
RMAN> BACKUP DATAFILE '/u01/app/oradata/ORCL/users01.dbf';

-- Incremental datafile backup
RMAN> BACKUP INCREMENTAL LEVEL 1 DATAFILE 4, 5;
```

---

## Backup SPFILE

```sql
-- Manual SPFILE backup
RMAN> BACKUP SPFILE;

-- To specific location
RMAN> BACKUP SPFILE FORMAT '/u01/rman_bkp/spfile_%d_%T.bkp';
```

!!! info "SPFILE is also backed up automatically"
    When `CONFIGURE CONTROLFILE AUTOBACKUP ON`, the SPFILE is always included in the auto backup after every operation.

---

## Monitor a Running Backup

```sql
-- From a separate SQL*Plus session while RMAN is running:
-- Shows % complete for long-running operations
SELECT SID, SERIAL#, CONTEXT, SOFAR, TOTALWORK,
       ROUND(SOFAR/TOTALWORK*100,2) AS PCT_DONE,
       MESSAGE
FROM V$SESSION_LONGOPS
WHERE OPNAME LIKE 'RMAN%'
  AND TOTALWORK > 0
  AND SOFAR < TOTALWORK;

-- View completed backup job history
SELECT SESSION_KEY, INPUT_TYPE, STATUS,
       TO_CHAR(START_TIME,'DD-MON-YYYY HH24:MI') AS START_TIME,
       TO_CHAR(END_TIME,'DD-MON-YYYY HH24:MI')   AS END_TIME,
       ELAPSED_SECONDS
FROM V$RMAN_BACKUP_JOB_DETAILS
ORDER BY START_TIME DESC
FETCH FIRST 10 ROWS ONLY;
```

---

## Identify Files Before Backing Up

```sql
-- List all datafiles with size and tablespace
SELECT FILE_ID, FILE_NAME, TABLESPACE_NAME,
       ROUND(BYTES/1024/1024,0) AS SIZE_MB
FROM DBA_DATA_FILES
ORDER BY FILE_ID;

-- Live view from V$
SELECT FILE#, NAME, STATUS, ROUND(BYTES/1024/1024,0) AS SIZE_MB
FROM V$DATAFILE;

-- List all tablespaces
SELECT TABLESPACE_NAME, STATUS, CONTENTS
FROM DBA_TABLESPACES;
```

---

## Section Commands Summary

```sql
-- Whole database
RMAN> BACKUP DATABASE;
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;
RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
RMAN> BACKUP DATABASE FORMAT '/path/%d_%T_%U';
RMAN> BACKUP DATABASE TAG 'my_tag';

-- Incremental whole
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;

-- Tablespace
RMAN> BACKUP TABLESPACE users;
RMAN> BACKUP TABLESPACE users, tools, app_data;
RMAN> BACKUP TABLESPACE users PLUS ARCHIVELOG;
RMAN> BACKUP INCREMENTAL LEVEL 1 TABLESPACE users;

-- Datafile
RMAN> BACKUP DATAFILE 1, 2, 3;
RMAN> BACKUP DATAFILE '/u01/oradata/ORCL/users01.dbf';
RMAN> BACKUP INCREMENTAL LEVEL 1 DATAFILE 4, 5;

-- SPFILE
RMAN> BACKUP SPFILE;

-- Monitor
SELECT ROUND(SOFAR/TOTALWORK*100,2) PCT_DONE, MESSAGE FROM V$SESSION_LONGOPS
WHERE OPNAME LIKE 'RMAN%' AND SOFAR < TOTALWORK;
```
