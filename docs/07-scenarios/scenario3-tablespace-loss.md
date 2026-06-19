---
tags: [scenario, tablespace-loss, partial-restore, archivelog]
---

# Scenario 3 — User Tablespace Loss

**Situation:** One or more user tablespace datafiles are lost, but the SYSTEM and UNDO tablespaces are fine — the database can stay **open** for other users while you recover the affected tablespace.

**Assumption:** Database is running in ARCHIVELOG mode.

---

## Step 1 — Identify Which Tablespace Lost a File

```sql
-- Find the tablespace name from the datafile number
-- (use the file# reported in the ORA-01157 error)
SELECT NAME FROM V$TABLESPACE
WHERE TS# = (
    SELECT D.TS#
    FROM V$DATAFILE D
    WHERE FILE# = 4      -- replace 4 with your actual file#
);

-- Verify the file is actually missing
SELECT FILE#, NAME, STATUS FROM V$DATAFILE WHERE FILE# = 4;

-- See what's waiting for recovery
SELECT FILE#, ERROR FROM V$RECOVER_FILE;
```

---

## Step 2 — Take Tablespace Offline

```sql
-- IMMEDIATE forces the tablespace offline without a clean checkpoint
-- (needed because the datafile is missing — we can't write to it)
ALTER TABLESPACE users OFFLINE IMMEDIATE;
```

---

## Step 3 — Restore and Recover (DB Stays Open)

```bash
rman target /
```

```sql
-- Restore only the lost datafile(s) for this tablespace
RMAN> RESTORE TABLESPACE users;

-- Apply archive logs to bring the tablespace forward to current SCN
RMAN> RECOVER TABLESPACE users;
```

---

## Step 4 — Bring Tablespace Back Online

```sql
-- Bring the tablespace online — now accessible to users again
ALTER TABLESPACE users ONLINE;

-- Verify
SELECT TABLESPACE_NAME, STATUS FROM DBA_TABLESPACES WHERE TABLESPACE_NAME = 'USERS';
```

---

## Full Command Sequence (Copy-Paste Ready)

```sql
-- Step 1: check which tablespace owns the bad datafile
SELECT NAME FROM V$TABLESPACE
WHERE TS# = (SELECT D.TS# FROM V$DATAFILE D WHERE FILE# = 4);

-- Step 2: take tablespace offline
ALTER TABLESPACE users OFFLINE IMMEDIATE;
```

```bash
rman target /
```

```sql
-- Step 3: restore + recover (database stays open throughout)
RMAN> RESTORE TABLESPACE users;
RMAN> RECOVER TABLESPACE users;
```

```sql
-- Step 4: bring back online
ALTER TABLESPACE users ONLINE;
```

---

## Restore by Datafile Path (When Tablespace Name Is Unknown)

```sql
-- If you only know the file path, not the tablespace name:
RMAN> RESTORE DATAFILE '/u01/oradata/ORCL/users01.dbf';
RMAN> RECOVER DATAFILE '/u01/oradata/ORCL/users01.dbf';
```

---

## Section Commands Summary

```sql
-- Identify the tablespace
SELECT NAME FROM V$TABLESPACE
WHERE TS# = (SELECT TS# FROM V$DATAFILE WHERE FILE# = 4);

SELECT FILE#, ERROR FROM V$RECOVER_FILE;
SELECT TABLESPACE_NAME, STATUS FROM DBA_TABLESPACES;

-- Take offline
ALTER TABLESPACE users OFFLINE IMMEDIATE;
```

```bash
rman target /
```

```sql
-- Restore + recover
RMAN> RESTORE TABLESPACE users;
RMAN> RECOVER TABLESPACE users;

-- Or by file
RMAN> RESTORE DATAFILE '/u01/oradata/ORCL/users01.dbf';
RMAN> RECOVER DATAFILE '/u01/oradata/ORCL/users01.dbf';

-- Bring back online
ALTER TABLESPACE users ONLINE;
```
