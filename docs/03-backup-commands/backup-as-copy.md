---
tags: [backup, image-copy, backup-as-copy]
---

# Backup As Copy (Image Copies)

## What Is an Image Copy?

An image copy is a block-for-block duplicate of a datafile, archived log, or control file — identical to the original file. Unlike a backup set (RMAN's proprietary compressed format), an image copy:

- Is a **regular OS file** — the exact same format as the original
- Can be used **directly as a datafile** without a restore step
- Takes exactly the **same space** as the original (no compression)
- Enables **instant recovery** — just switch Oracle to use the copy

```
Backup Set approach:       restore (decompress/reassemble) → then recover
Image Copy approach:       SWITCH DATAFILE TO COPY → just recover (no restore)
                                                      ↑ Much faster!
```

---

## Create Image Copies

### Whole Database as Image Copy

```sql
-- Copy entire database to image copies
RMAN> BACKUP AS COPY DATABASE;

-- With a target directory
RMAN> BACKUP AS COPY DATABASE
      FORMAT '/u01/image_copies/%b';
      -- %b = original file basename
```

### Individual Datafile

```sql
-- Image copy of a datafile (by path)
RMAN> BACKUP AS COPY
      DATAFILE '/u01/app/oradata/ORCL/users01.dbf'
      FORMAT '/u01/image_copies/users01.dbf';

-- Without checksum validation during copy
RMAN> BACKUP AS COPY NOCHECKSUM
      DATAFILE '/u01/app/oradata/ORCL/users01.dbf'
      FORMAT '/u01/image_copies/users01.dbf';

-- Image copy of datafile by number
RMAN> BACKUP AS COPY DATAFILE 4
      FORMAT '/u01/image_copies/df4.dbf';
```

### Tablespace as Image Copy

```sql
-- Image copy of all datafiles in a tablespace
RMAN> BACKUP AS COPY TABLESPACE users;

-- With file name conversion (change path in the copy)
RMAN> BACKUP AS COPY
      DB_FILE_NAME_CONVERT ('/u01/oradata/ORCL', '/u01/image_copies')
      TABLESPACE users;
      -- converts: /u01/oradata/ORCL/users01.dbf
      -- to:       /u01/image_copies/users01.dbf
```

### Control File as Image Copy

```sql
-- Image copy of control file
RMAN> BACKUP AS COPY CURRENT CONTROLFILE
      FORMAT '/u01/image_copies/control01.bkp';
```

### Archive Log as Image Copy

```sql
-- Image copy of archive logs
RMAN> BACKUP AS COPY ARCHIVELOG ALL
      FORMAT '/u01/image_copies/arch_%e_%s.arc';
```

---

## DB_FILE_NAME_CONVERT

This option lets you redirect image copies to a different directory while preserving the original filename:

```sql
RMAN> BACKUP AS COPY
      DB_FILE_NAME_CONVERT (
        '/u01/oradata/ORCL/users', '/rman_backup/users_copy'
      )
      TABLESPACE users;
-- Input:  /u01/oradata/ORCL/users01.dbf
-- Output: /rman_backup/users_copy01.dbf
```

Multiple mappings:
```sql
RMAN> BACKUP AS COPY
      DB_FILE_NAME_CONVERT (
        '/u01/oradata/ORCL', '/u02/image_copies',
        '/u01/oradata/ORCL2', '/u02/image_copies2'
      )
      DATABASE;
```

---

## List Image Copies

```sql
-- List all image copies
RMAN> LIST COPY;

-- List datafile copies only
RMAN> LIST COPY OF DATABASE;

-- List copies of a specific tablespace
RMAN> LIST COPY OF TABLESPACE users;

-- List copies of a specific datafile
RMAN> LIST COPY OF DATAFILE 4;

-- List control file copies
RMAN> LIST COPY OF CONTROLFILE;
```

---

## Instant Recovery Using SWITCH

The key advantage of image copies is **instant recovery** — no restore needed:

```sql
-- Scenario: users01.dbf is lost/corrupted
-- You have an image copy at /u01/image_copies/users01.dbf

-- Step 1: Take the tablespace offline
SQL> ALTER TABLESPACE users OFFLINE IMMEDIATE;

-- Step 2: Switch Oracle to use the image copy as the current datafile
RMAN> SWITCH DATAFILE '/u01/app/oradata/ORCL/users01.dbf'
      TO COPY;
-- or by file number:
RMAN> SWITCH DATAFILE 4 TO COPY;

-- Step 3: Recover (apply archive logs from backup time to now)
RMAN> RECOVER DATAFILE 4;

-- Step 4: Bring tablespace online
SQL> ALTER TABLESPACE users ONLINE;
```

!!! tip "SWITCH vs RESTORE"
    - `RESTORE` — decompresses backup set to recreate the file → **time-consuming for large files**
    - `SWITCH` — just updates Oracle's pointer to the existing image copy → **near-instant**

---

## Validate Image Copies

```sql
-- Verify copy is readable and not corrupt
RMAN> VALIDATE COPY OF DATAFILE 4;
RMAN> VALIDATE COPY OF TABLESPACE users;
RMAN> VALIDATE COPY OF DATABASE;
```
