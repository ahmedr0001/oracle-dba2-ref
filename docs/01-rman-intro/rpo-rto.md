---
tags: [rman, rpo, rto, recovery]
---

# RPO & RTO

Two business requirements that drive every backup decision.

---

## RPO — Recovery Point Objective

> **"How much data loss is acceptable?"**

The maximum age of the data you can restore to. If your last backup was 6 hours ago and the DB crashes now — you lose 6 hours of work.

| RPO | What It Means | Solution |
|---|---|---|
| 0 seconds | Zero data loss | Oracle Data Guard (synchronous) |
| < 1 minute | Near-zero loss | Data Guard async / GoldenGate |
| < 1 hour | Lose up to 1 hour | RMAN + frequent archive log backup |
| 24 hours | Lose up to one day | Nightly RMAN full backup |

---

## RTO — Recovery Time Objective

> **"How long can the business survive without the database?"**

The maximum downtime allowed — from the moment failure occurs to the moment users can work again.

| RTO | Solution |
|---|---|
| < 30 seconds | Oracle RAC / Data Guard failover |
| < 15 minutes | Data Guard switchover |
| < 2 hours | RMAN restore |
| Days | Full restore from offsite tape |

---

## What Eats Into Your RTO

Total downtime = time to **detect** + time to **decide** + time to **restore** + time to **verify**.
Plan for all four — not just the restore.

---

## Recovery Plan Prerequisites

Before any incident, your organization must have:

| Requirement | Why |
|---|---|
| Technology license | RMAN is free; Data Guard and GoldenGate are not |
| Hardware | Enough disk/tape space for backups |
| Backup location | Local disk, tape, or cloud |
| Tested runbook | A procedure that was never tested will fail when you need it most |
