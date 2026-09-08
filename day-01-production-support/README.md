# Day 1 — Linux Production Support Session Log

**Date:** 7 September 2026  
**Lab:** SkyRoute Production Simulation

## Objective

Day 1 was run as a production-support shift: establish Linux baselines, verify connectivity, investigate a Jira ticket without premature changes, collect evidence, separate unrelated findings, and document the result.

## Environment

- Rocky Linux 8.10 — production-like application server
- Ubuntu 24.04.3 LTS — second Linux VM for connectivity/SSH testing
- Rocky IP: `192.168.1.2/24`
- Ubuntu IP: `192.168.1.3/24`
- SkyRoute path: `/opt/skyroute`
- Jira project: **SkyRoute IT Operations (SIO)**
- Jira work item: **SIO-2 — INC-001 — SkyRoute application directory access failure**

## Baseline

The baseline checks established OS/kernel, hostname, IP/network, storage, memory/swap, and service health. Rocky had no failed services. The mounted Rocky ISO was recognized as normal installation media rather than a disk problem.

## Connectivity

Both VMs were tested in both directions. Ping succeeded with 4/4 replies and 0% packet loss in both directions.

An SSH attempt reached host-key verification. The observed `Host key verification failed` result was **not** treated as proof that the SSH service itself was down.

## SkyRoute Shared Directory

`appuser` and `supportuser` were members of the shared `appteam` group. `/opt/skyroute` and its application subdirectories were configured with `root:appteam` ownership and setgid permissions (`drwxrwsr-x`), providing shared group access and group inheritance.

## SIO-2 Investigation

### Reported symptom

> `supportuser` cannot create/modify files under `/opt/skyroute`.

### Investigation rule

**Reproduce first. Collect evidence before changing the system.**

### Reproduction as `supportuser`

The user identity and group membership were verified. `supportuser` successfully entered `/opt/skyroute`, created `test.txt`, inspected it with `ls`/`stat`, and modified its contents.

### Cross-user shared access

`appuser` successfully modified the file created by `supportuser`, confirming shared group write access.

### Application subdirectories

File creation was tested in `bin`, `config`, `data`, and `logs`. All four operations returned `exit_code=0`.

## Deeper Access-Control Checks

### Path traversal

`namei -l /opt/skyroute/test.txt` confirmed the required traversal permissions through `/` and `/opt` and the expected permissions on `/opt/skyroute` and the test file.

### POSIX ACLs

`getfacl` checks showed no additional ACL entries beyond standard owner/group/other permissions.

### SELinux

SELinux contexts were checked and `sudo ausearch -m AVC -ts recent` returned no recent AVC matches. There was no evidence that SELinux denied the tested operations.

## Application / Service Investigation

No SkyRoute-specific process was running. No failed systemd units were reported. Apache `httpd` was running and port 80 was listening.

A local Apache request returned **HTTP 403 Forbidden**. The Apache error log identified the cause as `/var/www/html/` having no matching `DirectoryIndex` while directory listing was forbidden. `DocumentRoot` was `/var/www/html`.

This was documented as a **separate Apache/content configuration finding**, not the cause of SIO-2.

## Final Verification — Ubuntu to Rocky HTTP

Without rebooting Rocky or restarting Apache, Ubuntu was used to verify external HTTP connectivity:

```bash
curl http://192.168.1.2
```

The request returned the Rocky Linux Apache HTTP Server Test Page successfully.

This confirmed:

- Ubuntu could reach Rocky over the network.
- TCP/HTTP port 80 was reachable.
- Apache was serving content successfully.
- No Apache restart was required for this verification.

The successful remote test also reinforced the earlier conclusion that the Apache observation was a separate webroot/content configuration finding rather than an HTTP service outage.

## Final SIO-2 Assessment

The reported filesystem access failure was **NOT reproducible under the tested conditions**.

Evidence showed that:

- Unix permissions were working.
- Group membership was correct.
- Setgid group inheritance was working.
- POSIX ACLs did not introduce a restriction.
- No recent SELinux AVC denial was found.
- Both `supportuser` and `appuser` could create/modify files in the tested application paths.
- HTTP connectivity from Ubuntu to Rocky was verified successfully without restarting the Rocky host or Apache.

No filesystem access-control root cause was established and no corrective system change was required.

The exact original failure remains unknown. If the issue recurs, the next evidence required is the original failing operation, affected path/file, exact error message, user/account, timestamp, and application context so the incident can be reproduced under equivalent conditions.

**Closure disposition:** Not Reproducible / No Root Cause Established.

## Production-Support Mindset

The workflow practiced today was:

**Symptom → Reproduce → Evidence → Hypothesis → Investigation → Root Cause → Minimum Safe Fix → Verify → Document**

Key lessons:

- A successful test is evidence too.
- “Not reproducible” is a valid investigation result; it does not automatically mean the original report was wrong.
- Do not change permissions simply because a permission problem is suspected.
- Use logs and system evidence instead of guessing.
- Keep unrelated findings separate from the incident being investigated.
- In production, reproduce only when the action is safe and authorized; use staging/test environments for risky operations.

## Jira

**SIO-2:** INC-001 — SkyRoute application directory access failure  
**Disposition:** Not Reproducible / No Root Cause Established  
**Status:** Closed after final verification and closure documentation.
