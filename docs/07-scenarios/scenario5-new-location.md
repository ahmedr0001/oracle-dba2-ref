---
tags: [scenario, new-location, set-newname, switch-datafile, relocate]
---

# Scenario 5 — Recover to a New Location

**Situation:** You need to restore datafiles to a **different disk path** — not the original location. Common use cases: original disk is full or failed permanently, creating a test/DR environment, migrating to a new storage layout.

---

## Assumption A — Move Whole Database to New Paths

```sql
-- Inside a RUN block, SET NEWNAME maps each datafile to its new path
-- SET NEWNAME must be done BEFORE RESTORE so RMAN writes to the new path
RMAN> RUN {
  -- Map old datafile numbers to new paths
  SET NEWNAME FOR DATAFILE 2 TO '/disk2/datafiles/df1.dbf';
  SET NEWNAME FOR DATAFILE 3 TO '/disk2/datafiles/df2.dbf';
  SET NEWNAME FOR DATAFILE 4 TO '/disk2/datafiles/df3.dbf';

  -- Restore all datafiles — RMAN writes them to the new paths above
  RESTORE DATABASE;

  -- Update Oracle's control file with the new paths for ALL datafiles
  SWITCH DATAFILE ALL;

  -- Apply archive logs to bring all files to current SCN
  RECOVER DATABASE;
}

-- Open the database (no RESETLOGS needed for complete recovery)
SQL> ALTER DATABASE OPEN;
```

---

## Assumption B — Move Only One Tablespace to a New Location

Database stays open; only the affected tablespace goes offline.

```sql
-- Step 1: take the tablespace offline
RMAN> SQL 'ALTER TABLESPACE users OFFLINE IMMEDIATE';

-- Step 2: set new names and restore only that tablespace
RMAN> RUN {
  -- Map old path to new path
  SET NEWNAME FOR DATAFILE '/u01/oradata/ORCL/users01.dbf'
    TO '/u02/oradata/ORCL/users01.dbf';
  SET NEWNAME FOR DATAFILE '/u01/oradata/ORCL/users02.dbf'
    TO '/u02/oradata/ORCL/users02.dbf';

  -- Restore only this tablespace to the new paths
  RESTORE TABLESPACE users;

  -- Update control file with the new paths
  SWITCH DATAFILE ALL;

  -- Apply archive logs
  RECOVER TABLESPACE users;
}

-- Step 3: bring the tablespace back online
RMAN> SQL 'ALTER TABLESPACE users ONLINE';
```

---

## SET NEWNAME Variants

```sql
-- By datafile number
SET NEWNAME FOR DATAFILE 4 TO '/new/path/users01.dbf';

-- By old path
SET NEWNAME FOR DATAFILE '/old/path/users01.dbf' TO '/new/path/users01.dbf';

-- For tempfiles
SET NEWNAME FOR TEMPFILE 1 TO '/new/path/temp01.dbf';

-- Wildcard: redirect ALL datafiles to a new directory
SET NEWNAME FOR DATABASE TO '/new/disk/%b';
-- %b = original filename (basename only)
```

---

## After Recovery — Verify New Paths

```sql
-- Check all datafile paths in the control file
RMAN> REPORT SCHEMA;

-- Or query from SQL*Plus
SELECT FILE#, NAME, STATUS FROM V$DATAFILE;
SELECT FILE_NAME, TABLESPACE_NAME FROM DBA_DATA_FILES;
```

---

## Section Commands Summary

```sql
-- Move whole database
RMAN> RUN {
  SET NEWNAME FOR DATAFILE 2 TO '/new/path/df1.dbf';
  SET NEWNAME FOR DATAFILE 3 TO '/new/path/df2.dbf';
  SET NEWNAME FOR DATAFILE 4 TO '/new/path/df3.dbf';
  RESTORE DATABASE;
  SWITCH DATAFILE ALL;   -- update control file with new paths
  RECOVER DATABASE;
}
SQL> ALTER DATABASE OPEN;

-- Move one tablespace (DB stays open)
RMAN> SQL 'ALTER TABLESPACE users OFFLINE IMMEDIATE';
RMAN> RUN {
  SET NEWNAME FOR DATAFILE '/old/users01.dbf' TO '/new/users01.dbf';
  RESTORE TABLESPACE users;
  SWITCH DATAFILE ALL;
  RECOVER TABLESPACE users;
}
RMAN> SQL 'ALTER TABLESPACE users ONLINE';

-- SET NEWNAME variants
SET NEWNAME FOR DATAFILE 4 TO '/path/file.dbf';             -- by number
SET NEWNAME FOR DATAFILE '/old/path.dbf' TO '/new/path.dbf'; -- by path
SET NEWNAME FOR DATABASE TO '/new/disk/%b';                   -- all files

-- Verify after recovery
RMAN> REPORT SCHEMA;
SELECT FILE#, NAME FROM V$DATAFILE;
```
