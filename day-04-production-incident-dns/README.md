# Day 4 — Blind Production Incident: DNS / Hostname Resolution

**Date:** 10 September 2026  
**Lab:** SkyRoute Production Simulation  
**Incident:** INC-003 / SIO-4 — SkyRoute production application unavailable

## Objective

Investigate a production P1 application-availability report without being told what component was deliberately broken. The session emphasized evidence-driven troubleshooting, hypothesis formation, controlled remediation, verification, and incident documentation.

## Incident Scenario

**SIO-4 — INC-003 — SkyRoute production application unavailable**  
**Priority:** P1  
**Impact:** Multiple users reported that SkyRoute was unavailable.

Environment:

- Rocky Linux 8.10 — `192.168.1.2`
- Ubuntu 24.04.3 LTS — `192.168.1.3`
- Apache HTTP Server on Rocky — TCP port 80
- SkyRoute portal — static HTML served by Apache

## Investigation Approach

The investigation was deliberately trainee-led. The workflow was:

**Client symptom → network validation → HTTP validation → service/process validation → logs → application architecture → hostname resolution → DNS verification → controlled remediation → verification**

No service restart or unrelated configuration change was performed during diagnosis.

## 1. Client-to-Server Reachability

From Ubuntu, Rocky was tested using ICMP.

Result: 100% packet success.

**Finding:** Basic network connectivity between the client and Rocky was healthy.

## 2. Direct HTTP Validation

The SkyRoute endpoint was tested directly using the Rocky IP and port 80.

```text
http://192.168.1.2:80
```

Result: HTTP request succeeded and returned the SkyRoute HTML page.

**Finding:** The server was reachable and the web endpoint was responding successfully when addressed directly by IP.

## 3. Apache Service and Process Validation

Apache was checked with `systemctl status httpd`.

Result:

- `httpd.service` was `active (running)`.
- Apache had been running for approximately two days.
- Apache parent and worker processes were present.

`ps aux | grep http` also showed the Apache parent and worker processes.

**Finding:** Apache was not currently down.

## 4. Apache Logs

The Apache error log contained a historical 403 event from 8 September:

```text
AH01276: Cannot serve directory /var/www/html/: No matching DirectoryIndex (index.html) found, and server-generated directory index forbidden by Options directive
```

The access log showed:

- 8 September: Ubuntu request returned `403`.
- 9 September: multiple clients returned `200`.
- Ubuntu requests returned `200`.
- Some historical client requests returned `408`, but the same client also had successful `200` requests.

**Finding:** The historical 403 was not the current P1 failure. Current direct HTTP access was healthy.

## 5. Application Architecture Check

A process search:

```text
ps aux | grep sky
```

returned only the `grep` command.

The `/opt/skyroute` directory contained the simulated application structure but no actual backend process. Apache's main configuration also contained no active `skyroute` hostname reference.

The current portal architecture was therefore confirmed as static HTML served by Apache from `/var/www/html`.

## 6. Hostname Investigation

Because direct IP access worked while users reported an application outage, hostname-based access was investigated.

The client failed to resolve `skyroute` and returned:

```text
Temporary failure in name resolution
```

Ubuntu's resolver configuration showed:

```text
127.0.0.53
```

as the local systemd-resolved stub, with upstream DNS:

```text
192.168.1.1
```

A direct DNS query was then made to the upstream server:

```text
dig @192.168.1.1 skyroute
```

The result was:

```text
status: NXDOMAIN
ANSWER: 0
SERVER: 192.168.1.1#53(192.168.1.1)
```

**Finding:** The upstream DNS server was reachable but had no DNS record for `skyroute`.

## 7. `/etc/hosts` Check

The Ubuntu `/etc/hosts` file was inspected. There was no `skyroute` mapping.

No change was made until DNS failure had been established.

## 8. Root Cause

The immediate availability failure was hostname resolution:

```text
Client requests skyroute
        ↓
Ubuntu resolver
        ↓
192.168.1.1 DNS
        ↓
NXDOMAIN
        ↓
Hostname cannot resolve
        ↓
User cannot reach SkyRoute by hostname
```

Direct IP access continued to work:

```text
192.168.1.2:80
        ↓
Apache
        ↓
SkyRoute HTML
        ↓
HTTP 200
```

The historical Apache 403 was unrelated to the current P1 and had already been resolved on Day 3 by providing `index.html`.

## 9. Controlled Remediation

A client-side `/etc/hosts` mapping was added on Ubuntu for the lab:

```text
skyroute → 192.168.1.2
```

This was treated as a **workaround**, not as a repair of the upstream DNS infrastructure.

## 10. Verification

Hostname-based access was retested after the mapping was added.

Result: **passed**. `http://skyroute` successfully returned the SkyRoute HTML content.

Final verification chain:

```text
skyroute
   ↓
/etc/hosts
   ↓
192.168.1.2
   ↓
Apache :80
   ↓
SkyRoute HTML
   ↓
SUCCESS
```

## Final Incident Disposition

**Status:** Resolved in the lab using a client-side workaround.

**Root cause:** Missing hostname resolution for `skyroute`; upstream DNS server `192.168.1.1` returned `NXDOMAIN`.

**Resolution:** Added `skyroute → 192.168.1.2` to the affected Ubuntu client's `/etc/hosts` and verified successful hostname-based access.

**Permanent fix recommendation:** Create and manage the appropriate DNS record through the organization's DNS/change-management process rather than relying on per-client `/etc/hosts` entries.

## Production-Support Lessons

- A successful direct-IP test does not prove hostname-based access works.
- `NXDOMAIN` means the DNS server responded but reports that the queried name does not exist.
- `127.0.0.53` on Ubuntu is a local systemd-resolved stub, not necessarily the upstream DNS server.
- `resolvectl status` can reveal the upstream DNS server.
- `dig @server hostname` isolates the upstream DNS query and provides stronger evidence.
- Do not modify `/etc/hosts` merely because a hostname is missing; first establish whether DNS is the intended source of truth.
- A client-side hosts entry is a workaround unless the environment explicitly uses hosts files as its name-resolution mechanism.
- Historical log errors must be separated from current incident evidence.
- Never restart a healthy service simply because an application is reported as unavailable.
- The correct troubleshooting pattern is **hypothesis → evidence → conclusion → controlled remediation → verification**.

## Interview-Relevant Exposure

This incident also introduced practical reasoning around DNS as an application-support dependency. The same investigation pattern will later extend to cloud load balancers, container networking, Kubernetes Services, CI/CD deployment endpoints, and monitoring alerts.
