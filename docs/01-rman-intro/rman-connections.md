---
tags: [rman, connections, login]
---

# RMAN Connections

## Connection Syntax

All RMAN connections are made from the **Linux shell** as the `oracle` OS user. RMAN connects to the Target DB via SYSDBA privileges.

---

## Target Only (No Catalog)

```bash
# Simplest — local OS authentication, no catalog
rman target /

# Equivalent explicit form
rman target / nocatalog

# With password (needed for remote or password-file auth)
rman target sys/oracle

# With explicit SID
export ORACLE_SID=ORCL
rman target /
```

---

## Target + Recovery Catalog

```bash
# Connect to both target and catalog in one command
rman target / catalog rmanadmin/pass@CatDB

# With explicit target credentials
rman target sys/oracle@ORCL catalog rmanadmin/pass@CatDB
```

---

## Logging & Script Execution

```bash
# Write all RMAN output to a log file (useful for scheduled jobs)
rman target / log=/u01/rman_logs/backup_$(date +%Y%m%d).log

# Append to existing log (don't overwrite)
rman target / log=/u01/rman_logs/rman.log APPEND

# Execute a command file (script) from OS
rman target / cmdfile=/u01/scripts/full_backup.rman

# Execute a script file from inside RMAN prompt
RMAN> @/u01/scripts/full_backup.rman

# Combine: catalog + log
rman target / catalog rmanadmin/pass@CatDB log=/tmp/rman.log APPEND
```

!!! tip "Always log in production"
    Scheduled RMAN jobs (via `cron`) should always write to a log file. Without a log, you have no record of what happened during the backup.

---

## Inside the RMAN Prompt

```bash
# These commands are run at the RMAN> prompt after connecting

# Show current configuration
RMAN> SHOW ALL;

# Check what's backed up
RMAN> LIST BACKUP SUMMARY;

# Run multiple commands in a block
RMAN> RUN {
  BACKUP DATABASE;
  BACKUP ARCHIVELOG ALL DELETE ALL INPUT;
}

# Exit RMAN
RMAN> EXIT;
RMAN> QUIT;
```

---

## Example: Scheduled Backup Script

A typical production backup script:

```bash
#!/bin/bash
# /u01/scripts/nightly_backup.sh
# Run via cron: 0 2 * * * /u01/scripts/nightly_backup.sh

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

## Connecting to Auxiliary Database

```bash
# Auxiliary is used for DUPLICATE DATABASE (cloning)
# You connect to it separately during the duplicate operation
rman target sys/oracle@ORCL auxiliary sys/oracle@AUXDB

# RMAN then runs:
RMAN> DUPLICATE TARGET DATABASE TO AUXDB;
```
