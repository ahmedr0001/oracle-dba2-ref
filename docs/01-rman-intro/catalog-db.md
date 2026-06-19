---
tags: [rman, catalog, catdb]
---

# Catalog Database Setup

The Recovery Catalog is a separate Oracle database that stores RMAN backup metadata — backup history, datafile info, archive log records — for one or more target databases.

Use it when you need history beyond 7 days, want to manage multiple databases with one RMAN, or need stored RMAN scripts.

| | Control File Only | Recovery Catalog |
|---|---|---|
| History retention | 7 days default | Unlimited |
| Multiple targets | No | Yes |
| Stored RMAN scripts | No | Yes |
| Survives target DB loss | No | Yes |

---

## Step 1 — Create the Catalog Database

Use DBCA to create a small separate database (SID = `CatDB`):

```bash
# Launch the DB creation wizard
dbca
```

---

## Step 2 — Create the Catalog Schema

```sql
-- Connect to CatDB as SYSTEM
sqlplus system/pass@CatDB

-- Dedicated tablespace for RMAN catalog tables
CREATE TABLESPACE cat_ts
  DATAFILE '/u01/oradata/CatDB/cat_ts01.dbf'
  SIZE 100M
  AUTOEXTEND ON NEXT 50M MAXSIZE 2G;

-- Catalog owner user
CREATE USER rmanadmin
  IDENTIFIED BY "RmanP@ss1"
  DEFAULT TABLESPACE cat_ts
  QUOTA UNLIMITED ON cat_ts
  TEMPORARY TABLESPACE temp;

-- RECOVERY_CATALOG_OWNER grants all privileges needed to manage the catalog
GRANT CONNECT, RESOURCE, RECOVERY_CATALOG_OWNER TO rmanadmin;
```

---

## Step 3 — Create the Catalog (Schema Objects)

```bash
# Connect RMAN to CatDB as the catalog user — no target yet
rman catalog rmanadmin/RmanP@ss1@CatDB

# Create the catalog tables, views, and packages inside CatDB
RMAN> CREATE CATALOG;
# Output: recovery catalog created

RMAN> EXIT;
```

---

## Step 4 — Register the Target Database

```bash
# Now connect to BOTH the target DB and the catalog
rman target / catalog rmanadmin/RmanP@ss1@CatDB

# Copy target DB's control file metadata into the catalog
# Run once per target database — never run again on the same DB
RMAN> REGISTER DATABASE;
# Output: database registered in recovery catalog
#         starting full resync of recovery catalog
#         full resync complete

RMAN> EXIT;
```

!!! warning "Register only once"
    Running `REGISTER DATABASE` on an already-registered DB causes an error. Use `RESYNC CATALOG` to update it after that.

---

## Step 5 — Verify Registration

```bash
rman target / catalog rmanadmin/RmanP@ss1@CatDB
RMAN> LIST DB_UNIQUE_NAME ALL;  -- shows all DBs registered in this catalog
```

```sql
-- Or query the catalog schema directly
sqlplus rmanadmin/RmanP@ss1@CatDB

-- All registered databases
SELECT DB_KEY, DB_ID, NAME FROM RC_DATABASE;

-- All tablespaces RMAN knows about for each DB
SELECT DB_KEY, NAME, STATUS FROM RC_TABLESPACE;
```

---

## Resync the Catalog

If the catalog DB was down while backups were running, resync copies the missed metadata from the target's control file into the catalog:

```sql
rman target / catalog rmanadmin/RmanP@ss1@CatDB

-- Pull all new backup records from control file into catalog
RMAN> RESYNC CATALOG;
```

---

## Unregister a Database

```bash
rman catalog rmanadmin/RmanP@ss1@CatDB

# Remove a target DB from the catalog (use when decommissioning)
RMAN> UNREGISTER DATABASE 'ORCL';
```

---

## Useful Catalog Queries

```sql
sqlplus rmanadmin/RmanP@ss1@CatDB

-- All registered databases
SELECT DB_ID, NAME FROM RC_DATABASE;

-- Recent backup sets (latest first)
SELECT BS_KEY, COMPLETION_TIME, STATUS, BACKUP_TYPE
FROM RC_BACKUP_SET
ORDER BY COMPLETION_TIME DESC;

-- Archive log history in catalog
SELECT DB_NAME, SEQUENCE#, FIRST_TIME, NEXT_TIME, STATUS
FROM RC_ARCHIVED_LOG
WHERE DB_NAME = 'ORCL'
ORDER BY SEQUENCE# DESC;
```

---

## CONTROL_FILE_RECORD_KEEP_TIME

When using control file only (no catalog), this controls how long RMAN records stay in the control file:

```sql
-- Check current setting (default = 7 days)
SHOW PARAMETER CONTROL_FILE_RECORD_KEEP_TIME;

-- Increase to 10 days (do not go higher — use a catalog instead)
ALTER SYSTEM SET CONTROL_FILE_RECORD_KEEP_TIME = 10 SCOPE=BOTH;
```

---

## Section Commands Summary

```bash
# Connect catalog only (no target)
rman catalog rmanadmin/pass@CatDB

# Connect target + catalog
rman target / catalog rmanadmin/pass@CatDB
```

```sql
RMAN> CREATE CATALOG;              -- build catalog schema (run once ever)
RMAN> REGISTER DATABASE;           -- register target DB (run once per DB)
RMAN> RESYNC CATALOG;              -- sync catalog with control file
RMAN> LIST DB_UNIQUE_NAME ALL;     -- list all registered databases
RMAN> UNREGISTER DATABASE 'ORCL'; -- remove a DB from catalog

-- Catalog views (query from sqlplus as rmanadmin)
SELECT DB_ID, NAME FROM RC_DATABASE;
SELECT BS_KEY, COMPLETION_TIME, STATUS FROM RC_BACKUP_SET ORDER BY 2 DESC;
SELECT SEQUENCE#, FIRST_TIME, STATUS FROM RC_ARCHIVED_LOG WHERE DB_NAME='ORCL';

-- Control file retention (no catalog)
SHOW PARAMETER CONTROL_FILE_RECORD_KEEP_TIME;
ALTER SYSTEM SET CONTROL_FILE_RECORD_KEEP_TIME = 10 SCOPE=BOTH;
```
