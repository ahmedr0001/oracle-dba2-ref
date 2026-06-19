---
tags: [complete-recovery, incomplete-recovery, pitr, resetlogs, incarnation]
---

# Complete vs Incomplete Recovery

## Complete Recovery

Apply all available redo up to the **current point in time** — no data loss. This is the goal for every media failure where archive logs are intact.

```sql
-- Complete recovery (apply all available archive logs)
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;    -- RMAN applies everything up to the current SCN
SQL>  ALTER DATABASE OPEN;
```

Requirements: all archive logs from backup time to now must be available.

---

## Incomplete Recovery (PITR — Point In Time Recovery)

Stop recovery **before** the current time — intentionally leaving out some transactions. Used when:
- Someone accidentally dropped tables or deleted rows
- You need to recover to before a logical error occurred
- Archive logs are missing (you can only recover as far as you have logs)

**Mandatory rule:** must open with `RESETLOGS` after incomplete recovery.

```sql
-- Incomplete recovery by TIME (most common — you know when the error happened)
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL TIME
      "TO_DATE('2025-05-26 09:00:00','YYYY-MM-DD HH24:MI:SS')";
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Incomplete recovery by SEQUENCE (you know the last good log sequence)
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL SEQUENCE 45;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Incomplete recovery by SCN
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL SCN 987654;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Incomplete recovery by RESTORE POINT (named SCN alias)
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL RESTORE POINT before_purge;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- All-in-one RUN block (cleanest approach)
RMAN> RUN {
  SET UNTIL TIME = '2025-05-26:09:00:00';
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}

-- Cancel-based recovery (apply logs one by one, stop when prompted)
SQL> RECOVER DATABASE UNTIL CANCEL;
-- Oracle asks: AUTO / log_filename / CANCEL after each log
-- Type CANCEL when you've applied enough
```

---

## PITR at Tablespace Level (TSPITR)

Recover one tablespace to a past point while the rest of the DB stays current. Useful when a single tablespace was corrupted by a bad batch job.

```sql
-- The DB stays open; only this tablespace is recovered to the past point
RMAN> RECOVER TABLESPACE users
      UNTIL TIME "TO_DATE('2025-05-26 09:00','DD/MM/YYYY HH24:MI')"
      AUXILIARY DESTINATION '/u01/tspitr_temp';
```

Requirements: DB must be in ARCHIVELOG mode, and the tablespace must be self-contained (no cross-tablespace foreign keys).

---

## RESETLOGS — What It Does

When you open with RESETLOGS after incomplete recovery:

- Resets the online redo log sequence back to 1
- Assigns a new **incarnation** to the database
- Updates datafile and controlfile headers with the new incarnation SCN
- Any backups from the previous incarnation cannot be used against the new incarnation without switching incarnation first

```sql
-- You must use RESETLOGS after any incomplete recovery
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Check current incarnation and history
RMAN> LIST INCARNATION OF DATABASE;

-- Switch RMAN to use a previous incarnation's backups
RMAN> RESET DATABASE TO INCARNATION 2;
```

---

## Find the Right Time/SCN/Sequence to Recover To

```sql
-- Find what time a specific sequence started/ended
SELECT SEQUENCE#, FIRST_TIME, NEXT_TIME
FROM V$ARCHIVED_LOG
WHERE DEST_ID = 1
ORDER BY SEQUENCE# DESC;

-- Find current SCN
SELECT CURRENT_SCN FROM V$DATABASE;

-- Check NLS date format (important for UNTIL TIME syntax)
ALTER SYSTEM SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI:SS' SCOPE=SPFILE;
-- Must restart for SPFILE change; use SCOPE=MEMORY for session only

-- Check current date format
SELECT SYSDATE FROM DUAL;
SELECT VALUE FROM V$PARAMETER WHERE NAME='nls_date_format';
```

---

## Section Commands Summary

```sql
-- Complete recovery
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
SQL>  ALTER DATABASE OPEN;

-- Incomplete recovery — by time
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2025-05-26 09:00:00','YYYY-MM-DD HH24:MI:SS')";
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Incomplete recovery — by sequence
RMAN> RECOVER DATABASE UNTIL SEQUENCE 45;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Incomplete recovery — by SCN
RMAN> RECOVER DATABASE UNTIL SCN 987654;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Incomplete recovery — by restore point
RMAN> RECOVER DATABASE UNTIL RESTORE POINT before_purge;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- All-in-one RUN block
RMAN> RUN {
  SET UNTIL TIME = '2025-05-26:09:00:00';
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}

-- Cancel-based
SQL> RECOVER DATABASE UNTIL CANCEL;

-- Incarnation management
RMAN> LIST INCARNATION OF DATABASE;
RMAN> RESET DATABASE TO INCARNATION 2;

-- NLS date format
ALTER SYSTEM SET NLS_DATE_FORMAT='YYYY-MM-DD HH24:MI:SS' SCOPE=SPFILE;
SELECT SYSDATE FROM DUAL;
```
