---
tags: [block-corruption, validate, recover, corruption]
---

# Block Corruption

A block is Oracle's smallest I/O unit (usually 8KB). Corruption means that block's content is either unreadable (physical) or structurally invalid (logical). RMAN can detect and repair corrupted blocks without restoring the entire datafile.

---

## Types of Corruption

| Type | What Happens | Cause |
|---|---|---|
| **Physical** | Block header is damaged — Oracle cannot read it at all | Disk failure, bad write, storage bug |
| **Logical** | Block is readable but internal structure is wrong | Software bug, incomplete write |

---

## Detect Corruption — Without Creating a Backup

```sql
-- Scan every block in the database for corruption (no backup created)
RMAN> BACKUP VALIDATE DATABASE;

-- Also check for logical corruption (slower, more thorough)
RMAN> BACKUP VALIDATE CHECK LOGICAL DATABASE;

-- Validate a specific datafile by number
RMAN> VALIDATE DATAFILE 4;

-- Validate a specific datafile with logical check
RMAN> VALIDATE CHECK LOGICAL DATAFILE 4;

-- Validate a specific tablespace
RMAN> VALIDATE TABLESPACE users;

-- Validate a block range in a datafile (when you know the block# from an error)
RMAN> VALIDATE DATAFILE 4 BLOCK 100 TO 200;

-- Validate entire database (shorthand)
RMAN> VALIDATE DATABASE;
RMAN> VALIDATE DATABASE CHECK LOGICAL;
```

After validation, results go into `V$DATABASE_BLOCK_CORRUPTION`:

```sql
-- Check what VALIDATE found
-- CORRUPTION_TYPE: SOFT=logical, FRACTURED/ALL ZERO=physical
SELECT FILE#, BLOCK#, BLOCKS, CORRUPTION_TYPE
FROM V$DATABASE_BLOCK_CORRUPTION;
```

---

## Repair Corrupted Blocks

### Option A — Manual Block Recovery

Restore only the bad blocks from a valid backup — much faster than restoring the whole datafile.

```sql
-- Recover a block range in a specific datafile
-- Needs: a valid RMAN backup + archive logs to bridge the gap
RMAN> RECOVER DATAFILE 4 BLOCK 100 TO 200;

-- Recover a single block
RMAN> RECOVER DATAFILE 4 BLOCK 100;

-- Recover multiple non-contiguous blocks
RMAN> RECOVER DATAFILE 4 BLOCK 100, 250, 380;
```

### Option B — Automated Repair (RECOVER CORRUPTION LIST)

RMAN reads `V$DATABASE_BLOCK_CORRUPTION` and automatically recovers everything in it.

```sql
-- Step 1: populate V$DATABASE_BLOCK_CORRUPTION
RMAN> VALIDATE DATABASE;

-- Step 2: recover every bad block RMAN found
RMAN> RECOVER CORRUPTION LIST;
```

!!! tip "Block recovery is much faster than full datafile restore"
    Only the corrupted blocks are pulled from the backup + archive logs applied to those blocks only. The datafile stays online during recovery (unless it is SYSTEM or UNDO).

---

## Section Commands Summary

```sql
-- Detect corruption
RMAN> BACKUP VALIDATE DATABASE;
RMAN> BACKUP VALIDATE CHECK LOGICAL DATABASE;
RMAN> VALIDATE DATABASE;
RMAN> VALIDATE DATABASE CHECK LOGICAL;
RMAN> VALIDATE DATAFILE 4;
RMAN> VALIDATE CHECK LOGICAL DATAFILE 4;
RMAN> VALIDATE TABLESPACE users;
RMAN> VALIDATE DATAFILE 4 BLOCK 100 TO 200;

-- Check results
SELECT FILE#, BLOCK#, BLOCKS, CORRUPTION_TYPE FROM V$DATABASE_BLOCK_CORRUPTION;

-- Repair specific blocks
RMAN> RECOVER DATAFILE 4 BLOCK 100 TO 200;
RMAN> RECOVER DATAFILE 4 BLOCK 100;
RMAN> RECOVER DATAFILE 4 BLOCK 100, 250, 380;

-- Automated repair
RMAN> VALIDATE DATABASE;
RMAN> RECOVER CORRUPTION LIST;
```
