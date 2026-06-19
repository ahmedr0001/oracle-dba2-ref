---
tags: [scenario, datafile-loss, noarchivelog, archivelog, complete-recovery, incomplete-recovery]
---

# Scenario 2 — Datafile Loss

**Situation:** One or more datafiles (including possibly SYSTEM) are physically lost or corrupted. The database cannot start normally.

**Symptoms:**
```
ORA-01157: cannot identify/lock data file # - see DBWR trace file
ORA-01110: data file #: '/u01/oradata/ORCL/users01.dbf'
```

---

## Verify the Problem

```bash
# Check if the file actually exists on disk
ls -lh /u01/oradata/ORCL/users01.dbf

# Check the alert log for the error details
tail -100 $ORACLE_BASE/diag/rdbms/orcl/ORCL/trace/alert_ORCL.log
```

```sql
-- From SQL*Plus (DB in MOUNT after failed open):
SELECT FILE#, NAME, STATUS FROM V$DATAFILE;
SELECT FILE#, ERROR FROM V$RECOVER_FILE;  -- shows which files need recovery
```

---

## Assumption A — NOARCHIVELOG, No Log Switch Since Backup

**Conditions:** database in NOARCHIVELOG mode, and no redo log switch occurred between the backup and the crash. The backup is still consistent enough to recover from completely.

**Result: Complete Recovery — No Data Loss**

```sql
-- The database failed to open; it's sitting in MOUNT or NOMOUNT
```

```bash
rman target /
```

```sql
-- Start in MOUNT if not already
RMAN> STARTUP MOUNT;

-- Restore the lost datafile(s) from backup
RMAN> RESTORE DATAFILE 4;      -- restore specific datafile
-- OR restore all:
-- RMAN> RESTORE DATABASE;

-- Recover (in NOARCHIVELOG with no log switch, this applies minimal redo)
RMAN> RECOVER DATAFILE 4;

-- Open normally — no RESETLOGS needed for complete recovery
SQL> ALTER DATABASE OPEN;
```

---

## Assumption B — NOARCHIVELOG, Log Switch DID Occur

**Conditions:** database in NOARCHIVELOG mode, but a redo log switch happened after the backup was taken. Oracle overwrote the old redo — the gap cannot be filled.

**Result: Incomplete Recovery — Data Loss from backup to crash**

```bash
rman target /
```

```sql
RMAN> STARTUP MOUNT;

-- Restore all datafiles (must restore database, not just one file)
RMAN> RESTORE DATABASE;

-- Apply what redo is available, then cancel when no more is found
RMAN> RECOVER DATABASE UNTIL CANCEL;
-- Oracle prompts after each log; type CANCEL when no more logs available

-- Must use RESETLOGS because recovery is incomplete
SQL> ALTER DATABASE OPEN RESETLOGS;
```

!!! danger "Data loss is unavoidable here"
    All transactions committed after the last backup are gone. This is why NOARCHIVELOG mode is dangerous for production databases.

---

## Assumption C — ARCHIVELOG Mode, Most/All Datafiles Lost

**Conditions:** database in ARCHIVELOG mode (correct!). All or most datafiles are lost — even SYSTEM. Archive logs are intact.

**Result: Complete Recovery — No Data Loss**

```sql
-- Generate some transactions and switch logs multiple times to prove archive logs work
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;
ALTER SYSTEM SWITCH LOGFILE;
```

Simulate loss: delete/rename the datafiles at OS level.

```bash
rman target /
```

```sql
-- Start in MOUNT
RMAN> STARTUP MOUNT;

-- Restore all datafiles from the last backup
RMAN> RESTORE DATABASE;

-- Apply all available archive logs to reach current SCN
RMAN> RECOVER DATABASE;

-- No RESETLOGS needed for complete recovery
SQL> ALTER DATABASE OPEN;
```

---

## Restore Specific Datafile (DB Stays Open for Non-SYSTEM Files)

If only a **non-SYSTEM, non-UNDO** datafile is lost and the DB is in ARCHIVELOG mode, you can recover without shutting down the whole database:

```sql
-- Take just the affected tablespace offline
SQL> ALTER TABLESPACE users OFFLINE IMMEDIATE;

-- Restore and recover while the DB is still open
RMAN> RESTORE DATAFILE 4;
RMAN> RECOVER DATAFILE 4;

-- Bring it back online
SQL> ALTER TABLESPACE users ONLINE;
```

---

## Section Commands Summary

```sql
-- Diagnose
ls -lh /path/to/lost/datafile.dbf
SELECT FILE#, ERROR FROM V$RECOVER_FILE;
SELECT FILE#, NAME, STATUS FROM V$DATAFILE;

-- Assumption A (NOARCHIVELOG, no log switch — complete)
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATAFILE 4;       -- or RESTORE DATABASE
RMAN> RECOVER DATAFILE 4;       -- or RECOVER DATABASE
SQL> ALTER DATABASE OPEN;

-- Assumption B (NOARCHIVELOG, log switch occurred — incomplete)
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL CANCEL;
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Assumption C (ARCHIVELOG — complete)
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
SQL> ALTER DATABASE OPEN;

-- Non-SYSTEM datafile, DB stays open (ARCHIVELOG)
SQL> ALTER TABLESPACE users OFFLINE IMMEDIATE;
RMAN> RESTORE DATAFILE 4;
RMAN> RECOVER DATAFILE 4;
SQL> ALTER TABLESPACE users ONLINE;
```
