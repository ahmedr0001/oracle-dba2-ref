---
tags: [scenario, dropped-tables, pitr, incomplete-recovery]
---

# Scenario 1 — Recover Dropped Tables

**Situation:** Tables were accidentally dropped. You need to recover them by going back in time to before the DROP happened.
**Requirement:** Database must be in **ARCHIVELOG** mode. A backup must exist from before the tables were created (or at minimum before the drop).

---

## Setup (Before the Incident)

```sql
-- Set NLS date format so SYSDATE output is readable (persistent after restart)
ALTER SYSTEM SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI:SS' SCOPE=SPFILE;
-- Restart needed for SPFILE; or use SCOPE=MEMORY for current session only

-- Note the current time BEFORE making changes you want to protect
SELECT SYSDATE FROM DUAL;
-- e.g. output: 2025-05-26 08:45:00  ← this is your safe point

-- Archive the current log so recent changes are preserved
ALTER SYSTEM SWITCH LOGFILE;
```

---

## Simulate the Accident

```sql
-- These tables are about to be lost — note the time BEFORE this
SELECT SYSDATE FROM DUAL;  -- e.g. 2025-05-26 09:10:00

DROP TABLE PROJECT     CASCADE CONSTRAINTS;
DROP TABLE EMPLOYEES   CASCADE CONSTRAINTS;
DROP TABLE DEPARTMENTS CASCADE CONSTRAINTS;

-- Note time AFTER the drop (recovery must be BEFORE this time)
SELECT SYSDATE FROM DUAL;  -- e.g. 2025-05-26 09:11:00
```

---

## Assumption A — Recover UNTIL TIME

Goal: go back to `2025-05-26 09:00:00` — before the drop.

```sql
-- Step 1: shut down the database
SHUTDOWN IMMEDIATE;
```

```bash
rman target /
```

```sql
-- Step 2: start in MOUNT — don't open (needed for database restore)
RMAN> STARTUP MOUNT;

-- Step 3: restore all datafiles from the backup
RMAN> RESTORE DATABASE;

-- Step 4: apply archive logs UP TO (but not including) the drop time
-- Use a timestamp a few minutes BEFORE the drop occurred
RMAN> RECOVER DATABASE UNTIL TIME
      "TO_DATE('2025-05-26 09:00:00', 'YYYY-MM-DD HH24:MI:SS')";

-- Step 5: open with RESETLOGS — mandatory after incomplete recovery
SQL> ALTER DATABASE OPEN RESETLOGS;

-- Step 6: verify tables are back
SELECT COUNT(*) FROM EMPLOYEES;
SELECT COUNT(*) FROM DEPARTMENTS;
```

!!! info "What you'll get"
    Tables exist with data as of `09:00:00`. Any inserts made between 09:00 and the drop are lost — that's the trade-off of PITR.

---

## Assumption B — Recover UNTIL SEQUENCE

Goal: recover to before a specific log sequence (you checked `ARCHIVE LOG LIST` before the drop).

```sql
-- Step 1: check current sequence BEFORE the accident
ARCHIVE LOG LIST;
-- Note: "Current log sequence" e.g. = 44

-- After the drop, you want to recover UP TO sequence 44
-- (not including it, since the drop may be in it)
```

```sql
SHUTDOWN IMMEDIATE;
```

```bash
rman target /
```

```sql
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;

-- Recover applying logs up to (not including) sequence 44
RMAN> RECOVER DATABASE UNTIL SEQUENCE 44;

SQL> ALTER DATABASE OPEN RESETLOGS;

-- Verify tables
SELECT COUNT(*) FROM EMPLOYEES;
```

---

## Assumption C — All-in-One RUN Block (Cleanest Approach)

```sql
RMAN> RUN {
  -- SET UNTIL applies to all commands inside this block
  SET UNTIL TIME = '2025-05-26:09:00:00';
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}
```

Other UNTIL options in a RUN block:

```sql
-- By SCN number
RMAN> RUN {
  SET UNTIL SCN 987654;
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}

-- By restore point (if you created one before the accident)
RMAN> RUN {
  SET UNTIL RESTORE POINT before_batch;
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}
```

---

## Cancel-Based Recovery (When You Don't Know the Exact Time)

Apply archive logs one at a time from SQL*Plus — stop when the bad transaction hasn't appeared yet:

```sql
-- From SQL*Plus (DB must be in MOUNT state)
SQL> RECOVER DATABASE UNTIL CANCEL;
-- Oracle prompts: Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
-- Press Enter to apply the suggested log, or type CANCEL to stop
```

---

## Section Commands Summary

```sql
-- Check/set date format
ALTER SYSTEM SET NLS_DATE_FORMAT='YYYY-MM-DD HH24:MI:SS' SCOPE=SPFILE;
SELECT SYSDATE FROM DUAL;

-- Archive current log
ALTER SYSTEM SWITCH LOGFILE;

-- Check archive sequence
ARCHIVE LOG LIST;

-- Recover UNTIL TIME
SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2025-05-26 09:00:00','YYYY-MM-DD HH24:MI:SS')";
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Recover UNTIL SEQUENCE
RMAN> RECOVER DATABASE UNTIL SEQUENCE 44;
SQL>  ALTER DATABASE OPEN RESETLOGS;

-- Recover UNTIL SCN
RMAN> RECOVER DATABASE UNTIL SCN 987654;
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
```
