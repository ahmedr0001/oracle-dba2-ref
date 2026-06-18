---
tags: [rman, rpo, rto, recovery]
---

# RPO & RTO

Before choosing a backup strategy, you must understand two critical business requirements. These are not technical concepts — they are business decisions that drive every technical choice.

---

## RPO — Recovery Point Objective

> **"How much data loss is acceptable?"**

RPO defines the **maximum age** of data you can restore from backup.
It is measured in time: seconds, minutes, hours, or days.

```
Time line:
──────────────────────────────────────────────────────►
       Last backup                    Failure
           │                             │
           └─────────── RPO ─────────────┘
                    (data loss window)
```

| RPO | Meaning | Typical Solution |
|---|---|---|
| **0 seconds** | Zero data loss — every transaction must survive | Oracle Data Guard (synchronous) |
| **< 1 minute** | Near-zero loss | Data Guard (async) / GoldenGate |
| **< 1 hour** | Loss of up to 1 hour of work | RMAN + archive logs every 30 min |
| **24 hours** | Loss of up to one full day | Nightly RMAN backup only |

---

## RTO — Recovery Time Objective

> **"How long can the business survive without the database?"**

RTO defines the **maximum downtime** allowed after a failure — from the moment the incident is detected to the moment the database is serving users again.

```
Time line:
──────────────────────────────────────────────────────►
       Failure           Detection           DB Online
          │                 │                    │
          └────────── RTO ──────────────────────┘
                    (total acceptable downtime)
```

| RTO | What It Means | Typical Solution |
|---|---|---|
| **< 30 seconds** | Users barely notice | Oracle RAC, Data Guard failover |
| **< 15 minutes** | Short disruption acceptable | Data Guard switchover |
| **< 2 hours** | Moderate disruption | RMAN restore (small/medium DB) |
| **Days** | Disaster scenario | Full restore from tape/offsite |

---

## Downtime Factors

RTO is not just "restore time." The clock starts at failure and stops when users can work again:

```
Total RTO = Identification time
          + Recovery plan review
          + Actual restore/recovery time
          + Verification time
          + Application restart time
```

| Factor | Description |
|---|---|
| **Identification** | How fast does the team know there's a problem? Monitoring/alerting matters here |
| **Recovery plan** | Is there a runbook? Or does the DBA have to figure it out under pressure? |
| **Recovery time** | Actual RMAN restore duration — depends on backup size and I/O speed |
| **Verification** | Confirming data integrity after recovery before allowing connections |

---

## Recovery Plan Prerequisites

A recovery plan needs more than just RMAN commands. Your organization needs:

| Requirement | Why It Matters |
|---|---|
| **Technology license** | RMAN is free; Oracle Active Data Guard, GoldenGate require licenses |
| **Hardware** | Enough disk/tape space for backups + restore target |
| **Backup location** | Local disk vs tape vs cloud — each has different RTO implications |
| **Network connectivity** | Remote backups need fast, reliable links for acceptable restore speed |

!!! tip "Document your recovery plan *before* you need it"
    Write and test your recovery procedures when the database is healthy. A DBA debugging a failed restore script during a production outage is a very bad situation.
