---
tags: [backup, controlfile, autobackup]
---

# Backup Control Files

The control file is Oracle's master index — every datafile, redo log, archive log, and RMAN backup record lives here. Losing all copies without a backup means the database cannot mount.

---

## Auto Backup (Recommended)

```sql
-- Automatically back up control file + SPFILE after every RMAN operation
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;

-- Disable auto backup
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP OFF;

-- Set a custom path for auto backup files
-- %F = unique name: c-DBID-YYYYMMDD-nn
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT
      FOR DEVICE TYPE DISK
      TO '/u01/rman_bkp/cf_%F';

-- Reset path back to default (FRA or configured channel)
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT
      FOR DEVICE TYPE DISK CLEAR;
```

!!! warning "Datafile 1 always triggers auto backup"
    Backing up datafile 1 (SYSTEM tablespace) always creates an automatic control file backup — even if AUTOBACKUP is OFF. This is hardcoded Oracle behavior.

---

## Manual Backup

```sql
-- Back up control file as a backupset (RMAN format)
RMAN> BACKUP CURRENT CONTROLFILE;

-- Back up control file along with a tablespace backup
RMAN> BACKUP TABLESPACE users INCLUDE CURRENT CONTROLFILE;

-- Back up control file with a custom path
RMAN> BACKUP CURRENT CONTROLFILE
      FORMAT '/u01/rman_bkp/cf_%T_%s.bkp';

-- Backing up datafile 1 also triggers a control file backup
RMAN> BACKUP DATAFILE 1;
```

---

## Image Copy of Control File

```sql
-- Exact binary copy — can be used directly without restore
RMAN> BACKUP AS COPY CURRENT CONTROLFILE
      FORMAT '/u01/rman_bkp/control01.bkp';

-- List control file image copies
RMAN> LIST COPY OF CONTROLFILE;
```

---

## Backup Control File to Trace (SQL Script)

The trace backup creates a SQL script that can recreate the control file from scratch — the ultimate fallback if all RMAN backups are lost.

```sql
-- From SQL*Plus as SYSDBA
-- Creates a .trc file in the trace directory automatically
ALTER DATABASE BACKUP CONTROLFILE TO TRACE;

-- Or specify exact output path
ALTER DATABASE BACKUP CONTROLFILE TO TRACE
  AS '/u01/rman_bkp/cf_trace.sql';

-- Without RESETLOGS — use this when all redo logs are still intact
ALTER DATABASE BACKUP CONTROLFILE TO TRACE NORESETLOGS;
```

```bash
# Find auto-generated trace file (if no AS clause used)
# SQL> SELECT VALUE FROM V$DIAG_INFO WHERE NAME = 'Diag Trace';
ls -lt $ORACLE_BASE/diag/rdbms/orcl/ORCL/trace/*.trc | head -5
```

The trace file contains two `CREATE CONTROLFILE` statements — one with RESETLOGS and one without. Use NORESETLOGS when all redo logs are present.

---

## List and Verify

```sql
-- List all control file backups (backupsets)
RMAN> LIST BACKUP OF CONTROLFILE;

-- List control file image copies
RMAN> LIST COPY OF CONTROLFILE;

-- Validate a backup is readable (no restore — just check)
RMAN> VALIDATE BACKUPSET <bs_key>;
```

---

## Restore Control File (Recovery Scenario)

```bash
# If all control files are lost, start RMAN with DB in NOMOUNT
rman target /
```

```sql
RMAN> STARTUP NOMOUNT;

-- RMAN finds the auto backup automatically in FRA
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;

-- Or specify exact backup piece
RMAN> RESTORE CONTROLFILE FROM '/u01/rman_bkp/cf_20260618_1.bkp';

-- After restoring: mount, recover, open
RMAN> ALTER DATABASE MOUNT;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

---

## Section Commands Summary

```sql
-- Auto backup
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP OFF;
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/path/cf_%F';
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK CLEAR;

-- Manual backup
RMAN> BACKUP CURRENT CONTROLFILE;
RMAN> BACKUP TABLESPACE users INCLUDE CURRENT CONTROLFILE;
RMAN> BACKUP CURRENT CONTROLFILE FORMAT '/path/cf_%T_%s.bkp';
RMAN> BACKUP AS COPY CURRENT CONTROLFILE FORMAT '/path/control01.bkp';

-- Trace backup (SQL*Plus)
ALTER DATABASE BACKUP CONTROLFILE TO TRACE;
ALTER DATABASE BACKUP CONTROLFILE TO TRACE AS '/path/cf_trace.sql';
ALTER DATABASE BACKUP CONTROLFILE TO TRACE NORESETLOGS;

-- List / validate
RMAN> LIST BACKUP OF CONTROLFILE;
RMAN> LIST COPY OF CONTROLFILE;
RMAN> VALIDATE BACKUPSET <bs_key>;

-- Restore (when all CFs are lost)
RMAN> STARTUP NOMOUNT;
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
RMAN> RESTORE CONTROLFILE FROM '/path/backup.bkp';
RMAN> ALTER DATABASE MOUNT;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```
