---
tags: [rman, connections, login]
---

# RMAN Connections

Run all RMAN commands from the Linux shell as the `oracle` OS user.

---

## Connect Syntax

```bash
# Local connection — uses OS authentication (no password needed)
rman target /

# Explicit: local with no catalog
rman target / nocatalog

# With password (for remote connections)
rman target sys/oracle

# Set SID first if needed
export ORACLE_SID=ORCL
rman target /

# Connect to target + catalog at the same time
rman target / catalog rmanadmin/pass@CatDB

# Explicit target credentials + catalog
rman target sys/oracle@ORCL catalog rmanadmin/pass@CatDB
```

---

## Logging and Script Files

```bash
# Write RMAN output to a log file
rman target / log=/u01/rman_logs/backup_$(date +%Y%m%d).log

# Append to existing log (don't overwrite previous runs)
rman target / log=/u01/rman_logs/rman.log APPEND

# Run a pre-written .rman script from the OS shell
rman target / cmdfile=/u01/scripts/full_backup.rman

# Run a script from inside the RMAN prompt
RMAN> @/u01/scripts/full_backup.rman

# Catalog + log combined
rman target / catalog rmanadmin/pass@CatDB log=/tmp/rman.log APPEND
```

---

## Basic Commands Inside RMAN Prompt

```sql
RMAN> SHOW ALL;              -- view all CONFIGURE settings
RMAN> LIST BACKUP SUMMARY;  -- view all recorded backups
RMAN> REPORT SCHEMA;        -- view all datafiles RMAN tracks

-- Run multiple commands as one job
RMAN> RUN {
  BACKUP DATABASE;
  BACKUP ARCHIVELOG ALL DELETE ALL INPUT;
}

RMAN> EXIT;
```

---

## Scheduled Backup Script

```bash
#!/bin/bash
# cron: 0 2 * * * /u01/scripts/nightly_backup.sh

export ORACLE_SID=ORCL
export ORACLE_HOME=/u01/app/oracle/product/19.3.0/dbhome_1
export PATH=$ORACLE_HOME/bin:$PATH

LOGFILE=/u01/rman_logs/backup_$(date +%Y%m%d_%H%M).log

rman target / log=$LOGFILE APPEND << 'EOF'
RUN {
  BACKUP AS COMPRESSED BACKUPSET DATABASE PLUS ARCHIVELOG;
  DELETE NOPROMPT OBSOLETE;
}
EXIT;
EOF
```

---

## Section Commands Summary

```bash
rman target /                                   # local, no catalog
rman target / nocatalog                         # explicit no catalog
rman target sys/oracle                          # with password
rman target / catalog rmanadmin/pass@CatDB      # with catalog
rman target / log=/tmp/rman.log APPEND          # with log file
rman target / cmdfile=/path/script.rman         # run a script
RMAN> @/path/script.rman                        # run script from prompt
RMAN> SHOW ALL;                                 # view all config
RMAN> LIST BACKUP SUMMARY;                      # view backups
RMAN> EXIT;                                     # quit
```
