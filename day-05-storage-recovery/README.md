# Day 5 — Storage & Recovery

## Incident
**INC-004 — /var Disk Utilization Increasing** (P2)

## Investigation
- `/var` filesystem: 4.0G total, ~2.4G used, ~60% utilization.
- `/var/cache` was the dominant consumer (~2.0G).
- `/var/cache/PackageKit` was ~1.8G.
- PackageKit RPM payloads were ~1.61G: BaseOS ~1013M and AppStream ~597M.
- Cached RPMs were observed with timestamps dating back to 2026-06-24.
- DNF cache was ~197M and `dnf clean all` did not remove the PackageKit cache.
- Multiple historical PackageKit update transactions were present across August and September.
- Sample cached RPM versions matched newer versions currently available from BaseOS, so the cache was not classified wholesale as unusable/orphaned.
- `GetTransactionList` returned `ao 0`, confirming no active PackageKit transactions at investigation time.
- `packagekitd` was running and held repository metadata files open; no evidence showed the RPM payloads themselves were actively open.

## RCA
Accumulated PackageKit-downloaded RPM payloads are the primary contributor to the elevated `/var` utilization. The investigation established retention and accumulation, but did not prove a deeper PackageKit configuration defect.

## Status
**Investigation complete; remediation deferred to next session.**

## Next session
1. Record before-state.
2. Controlled PackageKit stop.
3. Targeted cleanup of retained cache.
4. Restart PackageKit.
5. Verify `/var`, PackageKit, and DNF/package operations.
6. Document remediation and close INC-004.
