---
tags: [rman, architecture, catalog, channels]
---

# RMAN Architecture

## The Big Picture

RMAN is not a separate database — it is an Oracle client application that connects to databases and orchestrates backup/recovery operations on your behalf.

```
┌──────────────────────────────────────────────────────────┐
│                    RMAN Architecture                      │
│                                                          │
│   ┌──────────┐      ┌─────────────┐    ┌─────────────┐  │
│   │  RMAN    │─────►│  Target DB  │    │ Catalog DB  │  │
│   │ (client) │      │  (ORCL)     │◄───│  (CatDB)    │  │
│   └──────────┘      └──────┬──────┘    └─────────────┘  │
│                            │                             │
│                     ┌──────▼──────┐                      │
│                     │  Channel(s) │                      │
│                     │  (I/O proc) │                      │
│                     └──────┬──────┘                      │
│                            │                             │
│                     ┌──────▼──────┐                      │
│                     │  Backup     │                      │
│                     │  Location   │                      │
│                     │ (Disk/Tape) │                      │
│                     └─────────────┘                      │
└──────────────────────────────────────────────────────────┘
```

---

## Key Components

### 1. Target Database
The database you are backing up or recovering. RMAN connects to it as a privileged user (SYSDBA).

### 2. RMAN Repository (Metadata Store)
RMAN must store metadata about every backup it takes — what files, when, where on disk. This metadata lives in **one of two places**:

| Option | Where | Pros | Cons |
|---|---|---|---|
| **Control File** (default) | Inside the target DB's control file | No extra setup | Limited history (`CONTROL_FILE_RECORD_KEEP_TIME` default 7 days) |
| **Recovery Catalog** | Separate database (CatDB) | Unlimited history, supports multiple targets, scripting | Requires extra DB to maintain |

!!! info "Best Practice"
    For production use, always set up a **Recovery Catalog**. The control file is acceptable only for small/dev environments. Oracle recommends not setting `CONTROL_FILE_RECORD_KEEP_TIME` above 10 days — if you need longer retention, use a catalog.

### 3. Channels
A channel is an I/O stream — a server process on the Target DB that reads/writes backup data. More channels = more parallelism = faster backups (up to the number of physical disks).

```
Channel → reads datafiles → writes to disk/tape
```

### 4. Backup Output Types

=== "Backup Set (default)"
    - RMAN's proprietary format — multiple datafiles compressed into backup **pieces**
    - Can use compression and encryption
    - Cannot be used directly — must be restored first
    - One backup set can span multiple backup pieces
    ```
    Datafiles 1,2,3,4,5
         │
         ▼
    ┌─────────────┐  ┌─────────────┐
    │ Backup Piece│  │ Backup Piece│   ← backup set
    │  (DBF 1-3) │  │  (DBF 4-5) │
    └─────────────┘  └─────────────┘
    ```

=== "Image Copy"
    - Exact block-for-block copy of a datafile — same as `cp` but RMAN-tracked
    - Can be used immediately as a datafile without restore step (much faster recovery)
    - Takes same space as original file — no compression
    ```
    users01.dbf  ──COPY──►  /backup/users01.dbf (identical)
    ```

### 5. Auxiliary Database
A separate instance used for:
- Duplicating (cloning) the target database
- Restoring a backup under a new name
- Used with `DUPLICATE DATABASE` command

---

## What RMAN Can & Cannot Back Up

| ✅ Can Back Up | ❌ Cannot Back Up |
|---|---|
| Data files | Temp files (no user data) |
| Control files | Online redo log files (use archiving instead) |
| Server parameter file (SPFILE) | Password file |
| Archived redo logs | Network config files (`listener.ora`, `tnsnames.ora`) |

!!! warning "Online redo logs are never backed up directly"
    RMAN backs up **archived** redo logs. Online redo logs are backed up indirectly — by switching and archiving them before backup.
