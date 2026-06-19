---
tags: [scenarios, recovery]
---

# 🚨 Recovery Scenarios

Real-world recovery situations from the lectures. Each scenario includes the problem, assumptions, exact commands, and expected outcome.

| Scenario | Situation |
|---|---|
| [Scenario 1 — Recover Dropped Tables](scenario1-dropped-tables.md) | Incomplete recovery by time, sequence, and SCN |
| [Scenario 2 — Datafile Loss](scenario2-datafile-loss.md) | NOARCHIVELOG complete, NOARCHIVELOG incomplete, ARCHIVELOG complete |
| [Scenario 3 — User Tablespace Loss](scenario3-tablespace-loss.md) | DB stays open, offline-restore-recover-online |
| [Scenario 4 — Recover by Image Copy](scenario4-image-copy.md) | SWITCH DATABASE TO COPY — no restore needed |
| [Scenario 5 — Recover to New Location](scenario5-new-location.md) | SET NEWNAME, move datafiles to a different disk |
| [Scenario 6 — Redo Log Group Loss](scenario6-redo-log.md) | Inactive/Active/Current group loss |
