---
tags: [scenario, image-copy, switch-database, instant-recovery]
---

# Scenario 4 — Recover by Image Copy (Instant Recovery)

**Situation:** One or more datafiles are lost. You have image copies of the database in the FRA (from a previous `BACKUP AS COPY DATABASE`).

**Advantage:** No restore step needed — SWITCH tells Oracle to use the image copy directly, then recover applies archive logs to make it current. Much faster than restoring from a backup set.

**Assumptions:**
- Database is in ARCHIVELOG mode
- Image copies of the database are available in the FRA

---

## Verify Image Copies Exist

```bash
rman target /
```

```sql
-- List all available image copies
RMAN> LIST COPY OF DATABASE;

-- If RMAN shows them, you're ready
```

---

## Recovery Steps

```sql
-- Step 1: start in MOUNT (can't recover an open database)
RMAN> STARTUP MOUNT;

-- Step 2: SWITCH ALL datafiles to use the image copies in the FRA
-- This is instant — no data is moved, Oracle just updates its internal pointers
RMAN> SWITCH DATABASE TO COPY;

-- Step 3: open the database
-- Oracle applies archive logs automatically to make the copies current
RMAN> ALTER DATABASE OPEN;

-- Step 4: confirm the new datafile paths (they now point to the FRA copies)
RMAN> REPORT SCHEMA;
```

!!! tip "Why is this faster than normal restore?"
    `RESTORE DATABASE` decompresses and copies gigabytes of data from a backup set to disk — that takes time proportional to DB size.
    `SWITCH DATABASE TO COPY` just updates a pointer in the control file. It's nearly instantaneous regardless of DB size.

---

## After Recovery — Move Files Back (Optional)

After `SWITCH DATABASE TO COPY`, the database is now using files in the FRA as its live datafiles. The FRA is for backup/recovery, not for production. You should move files back to the original location when convenient (during a maintenance window).

```sql
-- Move files back to original production location using RMAN
RMAN> RUN {
  -- Tell RMAN the new target name for each datafile
  SET NEWNAME FOR DATAFILE '/u01/fra/ORCL/datafile/users01.dbf'
    TO '/u01/oradata/ORCL/users01.dbf';

  -- Switch the pointer and copy the file
  SWITCH DATAFILE ALL;
  RECOVER DATABASE;
}

SQL> ALTER DATABASE OPEN;
```

---

## Section Commands Summary

```bash
rman target /
```

```sql
-- Verify image copies
RMAN> LIST COPY OF DATABASE;
RMAN> LIST COPY OF DATAFILE 4;

-- Instant recovery using image copy
RMAN> STARTUP MOUNT;
RMAN> SWITCH DATABASE TO COPY;    -- update all pointers to FRA copies
RMAN> ALTER DATABASE OPEN;        -- open and apply archive logs automatically

-- Check new paths after switch
RMAN> REPORT SCHEMA;

-- Optional: move files back to production location
RMAN> RUN {
  SET NEWNAME FOR DATAFILE '/fra_path/file.dbf' TO '/prod_path/file.dbf';
  SWITCH DATAFILE ALL;
  RECOVER DATABASE;
}
SQL> ALTER DATABASE OPEN;
```
