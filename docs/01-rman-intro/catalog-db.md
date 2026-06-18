---
tags: [rman, catalog, catdb]
---

# Catalog Database Setup

## What Is the Recovery Catalog?

The Recovery Catalog is a separate Oracle database that stores RMAN metadata — backup history, datafile information, archive log details — for one or more target databases.

```
┌──────────────┐    register    ┌─────────────────┐
│  Target DB   │◄──────────────►│  Catalog DB     │
│  (ORCL)      │    metadata    │  (CatDB)        │
└──────────────┘                └─────────────────┘
         │                              │
         │ Target DB2, DB3...           │
         └─────────── also registered ──┘
```

**Why use a catalog instead of just the control file?**

| Feature | Control File Only | Recovery Catalog |
|---|---|---|
| History retention | 7 days (default) | Unlimited |
| Multiple target DBs | ❌ No | ✅ Yes |
| Stored RMAN scripts | ❌ No | ✅ Yes |
| Resync after gap | Manual | Automatic |
| Survives target DB loss | ❌ No | ✅ Yes |

---

## Step 1: Create the Catalog Database

Use DBCA to create a separate database instance (e.g. SID = `CatDB`). It can be a small database — the catalog schema doesn't need much space.

```bash
# Launch Database Configuration Assistant
dbca
# Follow the wizard: create a General Purpose database, SID = CatDB
```

---

## Step 2: Create the Catalog Schema

```sql
-- Connect to CatDB as SYSTEM
sqlplus system/pass@CatDB

-- Create a dedicated tablespace for catalog data
CREATE TABLESPACE cat_ts
  DATAFILE '/u01/oradata/CatDB/cat_ts01.dbf'
  SIZE 100M
  AUTOEXTEND ON NEXT 50M MAXSIZE 2G;

-- Create the catalog owner user
CREATE USER rmanadmin
  IDENTIFIED BY "RmanP@ss1"
  DEFAULT TABLESPACE cat_ts
  QUOTA UNLIMITED ON cat_ts
  TEMPORARY TABLESPACE temp;

-- Grant required privileges
-- RECOVERY_CATALOG_OWNER bundles all needed catalog privileges
GRANT CONNECT, RESOURCE, RECOVERY_CATALOG_OWNER TO rmanadmin;
```

---

## Step 3: Create the Catalog

```bash
# Connect RMAN to CatDB as the catalog user (no target yet)
rman catalog rmanadmin/RmanP@ss1@CatDB

# Create the catalog schema objects (tables, views, packages)
RMAN> CREATE CATALOG;
-- Output: recovery catalog created

RMAN> EXIT;
```

---

## Step 4: Register the Target Database

```bash
# Now connect to BOTH target AND catalog
rman target / catalog rmanadmin/RmanP@ss1@CatDB

# Register this target DB with the catalog
# (copies control file metadata into the catalog)
RMAN> REGISTER DATABASE;
-- Output: database registered in recovery catalog
--         starting full resync of recovery catalog
--         full resync complete

RMAN> EXIT;
```

!!! warning "Register once per target database"
    Run `REGISTER DATABASE` only once. Running it again will cause an error. If the catalog loses sync, use `RESYNC CATALOG` instead.

---

## Step 5: Verify Registration

```sql
-- From inside RMAN (connected to target + catalog):
RMAN> LIST DB_UNIQUE_NAME ALL;

-- Or query the catalog directly:
sqlplus rmanadmin/RmanP@ss1@CatDB

SELECT DB_KEY, DB_ID, NAME, RESETLOGS_TIME
FROM RC_DATABASE;

SELECT DB_KEY, NAME, STATUS
FROM RC_TABLESPACE;
```

---

## Resync the Catalog

After a period of backups done without the catalog (e.g. catalog DB was down), resync copies all new metadata from the target's control file into the catalog:

```bash
rman target / catalog rmanadmin/RmanP@ss1@CatDB

# Full resync — copies all control file records to catalog
RMAN> RESYNC CATALOG;

# Partial resync (happens automatically during most RMAN operations)
```

---

## Unregister a Target Database

```bash
# If you decommission a target DB:
rman catalog rmanadmin/RmanP@ss1@CatDB

RMAN> UNREGISTER DATABASE 'ORCL';
```

---

## Catalog Maintenance Queries

```sql
-- Connect to catalog DB directly:
sqlplus rmanadmin/RmanP@ss1@CatDB

-- All registered databases
SELECT DB_ID, NAME FROM RC_DATABASE;

-- All backups recorded in catalog
SELECT BS_KEY, COMPLETION_TIME, STATUS, BACKUP_TYPE
FROM RC_BACKUP_SET
ORDER BY COMPLETION_TIME DESC;

-- All archived logs in catalog
SELECT DB_NAME, SEQUENCE#, FIRST_TIME, NEXT_TIME, STATUS
FROM RC_ARCHIVED_LOG
WHERE DB_NAME = 'ORCL'
ORDER BY SEQUENCE# DESC;

-- Datafile backup history
SELECT FILE#, COMPLETION_TIME, STATUS
FROM RC_BACKUP_DATAFILE
WHERE DB_ID = (SELECT DB_ID FROM RC_DATABASE WHERE NAME='ORCL')
ORDER BY COMPLETION_TIME DESC;
```

---

## CONTROL_FILE_RECORD_KEEP_TIME

When using control file only (no catalog), this parameter controls how long RMAN records are kept in the control file:

```sql
-- Check current setting (default: 7 days)
SHOW PARAMETER CONTROL_FILE_RECORD_KEEP_TIME;

-- Change it (requires restart for SPFILE, or use SCOPE=BOTH)
ALTER SYSTEM SET CONTROL_FILE_RECORD_KEEP_TIME = 10 SCOPE=BOTH;
```

!!! danger "Don't set above 10 days without a catalog"
    Setting this too high bloats the control file. Oracle's best practice: if you need retention beyond 10 days, use a **Recovery Catalog** instead of increasing this parameter.
