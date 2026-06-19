---
tags: [home]
hide:
  - toc
---

<div class="hero-banner">
<h1>🛡️ Oracle DBA2 Reference</h1>
<p>Workshop 2 — RMAN, Backup Strategy, Recovery, and Advanced Administration</p>
</div>

<div class="card-grid">

<a class="card" href="01-rman-intro/">
<div class="card-icon">⚙️</div>
<h3>1 · RMAN Fundamentals</h3>
<p>Why backup, failure types, RPO/RTO, RMAN architecture, connections, and all 8 RMAN configurations</p>
</a>

<a class="card" href="02-backup-types/">
<div class="card-icon">💾</div>
<h3>2 · Storage & Redo</h3>
<p>Fast Recovery Area (FRA), Multiplexing control files & redo logs, Archive Log mode</p>
</a>

<a class="card" href="03-backup-commands/">
<div class="card-icon">📦</div>
<h3>3 · Backup Commands</h3>
<p>Full/Incremental, Whole/Partial, Control files, Archived logs, Image copies, Tags</p>
</a>

<a class="card" href="04-backup-types/">
<div class="card-icon">🔄</div>
<h3>4 · Consistent vs Inconsistent</h3>
<p>Cold/Hot backup modes, ARCHIVELOG vs NOARCHIVELOG matrix, when each is valid</p>
</a>

<a class="card" href="05-reporting-maintenance/">
<div class="card-icon">📊</div>
<h3>5 · Reporting & Maintenance</h3>
<p>Automated backups, Block Corruption Tracking (BCT), Dynamic views, LIST, REPORT, Crosscheck & Delete</p>
</a>

<a class="card" href="06-restore-recover/">
<div class="card-icon">♻️</div>
<h3>6 · Restore & Recovery</h3>
<p>Restore vs Recover, Complete & Incomplete Recovery, Redo Log Recovery, Restore Points</p>
</a>

<a class="card" href="07-scenarios/">
<div class="card-icon">🎯</div>
<h3>7 · Real-World Scenarios</h3>
<p>Dropped tables, datafile loss, tablespace loss, image copies, relocating files, redo log recovery</p>
</a>

<a class="card" href="quick-reference/">
<div class="card-icon">⚡</div>
<h3>Quick Reference</h3>
<p>Every RMAN command on one page — copy and go</p>
</a>

</div>

---

## How to Use This Site

- **Top tabs** → navigate between major topics
- **Search** (`S` or `/`) → find any command instantly
- **Copy button** → hover any code block to copy it
- `--` and `#` comments in code explain what each line does

!!! tip "Start Here — Connect to RMAN"
    ```bash
    # Local connection (no catalog)
    rman target /

    # Local with log file
    rman target / log=/tmp/rman.log APPEND

    # With Recovery Catalog
    rman target / catalog rmanadmin/pass@CatDB

    # Execute a script file
    rman target / cmdfile=/path/to/backup.rman
    ```

!!! info "DBA2 builds on DBA1"
    This site assumes you are familiar with Oracle architecture basics (startup/shutdown, tablespaces, users). If you need a refresher, see the [DBA1 Reference](https://ahmedr0001.github.io/oracle-dba-ref/).
