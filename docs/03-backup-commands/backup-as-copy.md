---
tags: [backup, image-copy, backup-as-copy]
---

# Backup As Copy (Image Copies)

An image copy is a block-for-block duplicate of an Oracle file — identical to the original, readable as a normal OS file. Unlike a backup set (RMAN's compressed proprietary format), an image copy can be used **directly as a datafile** without a restore step.

| | Backup Set | Image Copy |
|---|---|---|
| Format | RMAN proprietary | Exact duplicate of original |
| Size | Smaller (can compress) | Same as original |
| Restore step needed? | Yes | No — just switch |
| Recovery speed | Slower | Faster |

---

## Create Image Copies

```sql
-- Copy entire database
RMAN> BACKUP AS COPY DATABASE;

-- To a specific directory (%b = original filename)
RMAN> BACKUP AS COPY DATABASE FORMAT '/u01/image_copies/%b';

-- Copy a specific tablespace
RMAN> BACKUP AS COPY TABLESPACE users;

-- Copy a datafile by number
RMAN> BACKUP AS COPY DATAFILE 4 FORMAT '/u01/copies/df4.dbf';

-- Copy a datafile by path (skip checksum validation for speed)
RMAN> BACKUP AS COPY NOCHECKSUM
      DATAFILE '/u01/app/oradata/ORCL/users01.dbf'
      FORMAT '/u01/copies/users01.dbf';

-- Copy control file
RMAN> BACKUP AS COPY CURRENT CONTROLFILE FORMAT '/u01/copies/cf.bkp';

-- Copy archive logs
RMAN> BACKUP AS COPY ARCHIVELOG ALL FORMAT '/u01/copies/arch_%e_%s.arc';
```

---

## DB_FILE_NAME_CONVERT — Redirect to a Different Directory

Change the output path while keeping the original filenames:

```sql
-- All files under /u01/oradata/ORCL/ are copied to /u01/copies/
RMAN> BACKUP AS COPY
      DB_FILE_NAME_CONVERT ('/u01/oradata/ORCL', '/u01/copies')
      TABLESPACE users;
-- /u01/oradata/ORCL/users01.dbf → /u01/copies/users01.dbf

-- Multiple path mappings
RMAN> BACKUP AS COPY
      DB_FILE_NAME_CONVERT (
        '/u01/oradata/ORCL',  '/u01/copies',
        '/u01/oradata/ORCL2', '/u01/copies2'
      )
      DATABASE;
```

---

## List Image Copies

```sql
RMAN> LIST COPY;                      -- all image copies
RMAN> LIST COPY OF DATABASE;          -- datafile copies only
RMAN> LIST COPY OF TABLESPACE users;  -- copies of a specific tablespace
RMAN> LIST COPY OF DATAFILE 4;        -- copies of one datafile
RMAN> LIST COPY OF CONTROLFILE;       -- control file copies
```

---

## Instant Recovery with SWITCH

SWITCH tells Oracle to use the image copy as the live datafile — no restore needed.

```sql
-- Scenario: users01.dbf is lost/corrupted
-- You have a copy at /u01/copies/users01.dbf

-- Step 1: take the tablespace offline so Oracle stops trying to use the bad file
SQL> ALTER TABLESPACE users OFFLINE IMMEDIATE;

-- Step 2: switch Oracle's pointer from the bad file to the image copy
-- (no data is moved — Oracle just updates its internal record)
RMAN> SWITCH DATAFILE '/u01/app/oradata/ORCL/users01.dbf' TO COPY;
-- Or by file number:
RMAN> SWITCH DATAFILE 4 TO COPY;

-- Step 3: apply archive logs from backup time to now (make it current)
RMAN> RECOVER DATAFILE 4;

-- Step 4: bring the tablespace back online
SQL> ALTER TABLESPACE users ONLINE;
```

---

## Validate Image Copies

```sql
-- Check that the copy file is readable and not corrupted
RMAN> VALIDATE COPY OF DATAFILE 4;
RMAN> VALIDATE COPY OF TABLESPACE users;
RMAN> VALIDATE COPY OF DATABASE;
```

---

## Section Commands Summary

```sql
-- Create copies
RMAN> BACKUP AS COPY DATABASE;
RMAN> BACKUP AS COPY DATABASE FORMAT '/path/%b';
RMAN> BACKUP AS COPY TABLESPACE users;
RMAN> BACKUP AS COPY DATAFILE 4 FORMAT '/path/df4.dbf';
RMAN> BACKUP AS COPY NOCHECKSUM DATAFILE '/path/file.dbf' FORMAT '/path/copy.dbf';
RMAN> BACKUP AS COPY CURRENT CONTROLFILE FORMAT '/path/cf.bkp';
RMAN> BACKUP AS COPY ARCHIVELOG ALL FORMAT '/path/arch_%e_%s.arc';

-- DB_FILE_NAME_CONVERT (redirect path)
RMAN> BACKUP AS COPY
      DB_FILE_NAME_CONVERT ('/u01/oradata/ORCL','/u01/copies')
      TABLESPACE users;

-- List
RMAN> LIST COPY;
RMAN> LIST COPY OF DATABASE;
RMAN> LIST COPY OF TABLESPACE users;
RMAN> LIST COPY OF DATAFILE 4;
RMAN> LIST COPY OF CONTROLFILE;

-- Instant recovery
SQL> ALTER TABLESPACE users OFFLINE IMMEDIATE;
RMAN> SWITCH DATAFILE 4 TO COPY;
RMAN> RECOVER DATAFILE 4;
SQL> ALTER TABLESPACE users ONLINE;

-- Validate
RMAN> VALIDATE COPY OF DATAFILE 4;
RMAN> VALIDATE COPY OF TABLESPACE users;
RMAN> VALIDATE COPY OF DATABASE;
```
