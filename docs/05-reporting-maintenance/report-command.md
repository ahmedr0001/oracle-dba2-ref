---
tags: [report, obsolete, need-backup, schema]
---

# REPORT Command

REPORT answers questions about the **state** of your backups — what still needs backing up, what can be safely deleted, and what the database structure looks like from RMAN's perspective.

---

## REPORT NEED BACKUP

Shows which datafiles need a backup based on the current retention policy or a custom policy you specify inline.

```sql
-- Files that need backup under the currently configured retention policy
RMAN> REPORT NEED BACKUP;

-- Files that need backup for a specific tablespace only
RMAN> REPORT NEED BACKUP TABLESPACE users;

-- Files that would need backup under a 2-day recovery window
-- (useful to check "what if I change my policy?")
RMAN> REPORT NEED BACKUP RECOVERY WINDOW OF 2 DAYS;

-- Files that don't have at least 2 backup copies
RMAN> REPORT NEED BACKUP REDUNDANCY 2;

-- Complex filter: 2-day window for the whole DB, skip the users tablespace
RMAN> REPORT NEED BACKUP RECOVERY WINDOW OF 2 DAYS
      DATABASE SKIP TABLESPACE users;

-- Files with fewer than 2 backups for a specific datafile only
RMAN> REPORT NEED BACKUP REDUNDANCY 2 DATAFILE 1;
```

---

## REPORT OBSOLETE

Shows which backups are no longer needed based on the retention policy. Safe to delete these.

```sql
-- List obsolete backups (does not delete them — just shows)
RMAN> REPORT OBSOLETE;

-- List obsolete under a specific redundancy (override current policy)
RMAN> REPORT OBSOLETE REDUNDANCY 2;

-- List obsolete under a specific recovery window
RMAN> REPORT OBSOLETE RECOVERY WINDOW OF 7 DAYS;
```

---

## REPORT SCHEMA

Shows all datafiles and tablespaces that RMAN knows about. Useful to get file numbers before referencing datafiles in BACKUP or RESTORE commands.

```sql
-- List all datafiles, temp files, and tablespaces
RMAN> REPORT SCHEMA;

-- Output includes: file#, tablespace name, datafile path, size
```

---

## REPORT UNRECOVERABLE

Shows datafiles that contain unrecoverable operations (e.g. NOLOGGING inserts) that cannot be recovered from archive logs. These must be backed up again.

```sql
RMAN> REPORT UNRECOVERABLE;
```

---

## Section Commands Summary

```sql
-- REPORT NEED BACKUP
RMAN> REPORT NEED BACKUP;
RMAN> REPORT NEED BACKUP TABLESPACE users;
RMAN> REPORT NEED BACKUP RECOVERY WINDOW OF 2 DAYS;
RMAN> REPORT NEED BACKUP REDUNDANCY 2;
RMAN> REPORT NEED BACKUP RECOVERY WINDOW OF 2 DAYS DATABASE SKIP TABLESPACE users;
RMAN> REPORT NEED BACKUP REDUNDANCY 2 DATAFILE 1;

-- REPORT OBSOLETE
RMAN> REPORT OBSOLETE;
RMAN> REPORT OBSOLETE REDUNDANCY 2;
RMAN> REPORT OBSOLETE RECOVERY WINDOW OF 7 DAYS;

-- REPORT SCHEMA
RMAN> REPORT SCHEMA;

-- REPORT UNRECOVERABLE
RMAN> REPORT UNRECOVERABLE;
```
