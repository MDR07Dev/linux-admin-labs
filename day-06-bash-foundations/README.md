# Day 6 — Bash Foundations

## Incident
**INC-005 — Bash disk utilization calculation failure**

## Symptom
A health-check script reported 0% utilization despite `/var` being about 60% used.

## Root cause
Bash integer arithmetic evaluated `60 / 100` as `0` before multiplying by 100.

Incorrect pattern:
```bash
percentage=$((used / total * 100))
```

Correct pattern:
```bash
percentage=$((used * 100 / total))
```

## Validation
The corrected calculation returned `60%`.

## Independent implementation
The user then built a multi-filesystem health-check without assistance for `/`, `/var`, and `/home`, using:
- `for` loop
- command substitution
- `df`
- `awk`
- `tr`
- `if/else`
- integer `-ge` comparison

Example output:
```text
/: OK - 45%
/var: OK - 60%
/home: OK - 4%
```

## Automation discussion
The health check was designed for daily cron execution at 06:00 with output appended to `/var/log/filesys_healthchk.log`. Least-privilege considerations were discussed: ACL is preferable to broad sudo access when only log-file write access is required.

## Status
INC-005 investigation and fix demonstrated. Cron implementation remains for a later session.
