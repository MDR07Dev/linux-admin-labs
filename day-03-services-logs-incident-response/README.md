# Day 3 — Services, Logs & Incident Response

**Date:** 9 September 2026  
**Lab:** SkyRoute Production Simulation

## Objective

Day 3 was run as a production-support investigation focused on service health, HTTP availability, Apache logs, process evidence, and root-cause reasoning. The investigation followed a layered approach instead of restarting services or changing configuration without evidence.

## Incident Scenario

**INC-002 — SkyRoute Application Unavailable**  
**Priority:** P1  
**Impact:** Production users reported that SkyRoute was unavailable.

The investigation environment was:

- Rocky Linux 8.10 — `192.168.1.2`
- Ubuntu 24.04.3 LTS — `192.168.1.3`
- Apache HTTP Server on Rocky — TCP port 80
- SkyRoute filesystem — `/opt/skyroute`
- Apache web root — `/var/www/html`

## Investigation Workflow

The investigation progressed through these layers:

**Client reachability → TCP port → HTTP response → Apache process → application evidence → configuration → logs → historical root cause**

## 1. Network Reachability

From Ubuntu, Rocky was tested with ICMP ping.

Result: successful connectivity with no observed packet-loss problem.

**Finding:** Basic network reachability was healthy.

## 2. TCP Port Investigation

On Rocky, the listening sockets were checked with:

```bash
ss -tulnp | grep 80
```

Relevant result:

```text
tcp LISTEN 0 511 *:80 *:*
```

**Finding:** TCP port 80 was listening on all interfaces.

The firewall was not changed because there was no evidence yet that firewall filtering was the problem.

## 3. HTTP Connectivity

From Ubuntu:

```bash
curl http://192.168.1.2
```

The request successfully connected to Rocky and returned the newly created SkyRoute Operations Portal HTML.

**Finding:** TCP/HTTP connectivity and Apache content delivery were working.

## 4. Application Process Investigation

A process search was performed:

```bash
ps aux | grep sky
```

No SkyRoute-specific process was found; the only matching line was the `grep` command itself.

Apache processes were then inspected:

```bash
ps aux | grep http
```

Multiple `/usr/sbin/httpd -DFOREGROUND` processes were present, including the Apache parent process and worker processes.

**Finding:** Apache was running. No separate process named SkyRoute was present.

### Important interpretation

The absence of a process containing `sky` did **not** prove that an application backend was down. The current SkyRoute page is static HTML and can be served directly by Apache without a separate process.

## 5. SkyRoute Filesystem Investigation

The application path was inspected:

```text
/opt/skyroute/
├── bin/
├── config/
├── data/
├── logs/
└── test.txt
```

Each subdirectory contained only the `test.txt` files created during earlier access-control exercises.

Ownership and permissions were consistent with the previously established shared application structure:

```text
root:appteam
-drwxrwsr-x
```

**Finding:** `/opt/skyroute` contained the simulated application directory structure but no actual application executable, startup script, or backend artifact.

## 6. Apache Configuration Investigation

The main Apache configuration was inspected for SkyRoute references:

```bash
cat /etc/httpd/conf/httpd.conf | grep sky
```

No output was returned.

The known Apache `DocumentRoot` was:

```text
/var/www/html
```

**Finding:** Apache was not explicitly configured to reference `/opt/skyroute`. The current SkyRoute portal was therefore being served as static content from `/var/www/html`.

## 7. Apache Access Logs

The Apache access log was inspected at:

```text
/var/log/httpd/access_log
```

A historical request from Ubuntu on 8 September showed:

```text
192.168.1.3 ... "GET / HTTP/1.1" 403
```

Later requests on 9 September showed:

```text
192.168.1.3 ... "GET / HTTP/1.1" 200
192.168.1.3 ... "GET / HTTP/1.1" 200
```

Other browser clients also received HTTP 200 responses.

A `favicon.ico` request returned 404, which was treated as a normal missing favicon rather than an application outage.

**Finding:** The current HTTP endpoint was healthy and returning HTTP 200. The earlier 403 was historical.

## 8. Apache Error Log / Historical Root Cause

The Apache error log was inspected:

```text
/var/log/httpd/error_log
```

The key historical error was:

```text
AH01276: Cannot serve directory /var/www/html/: No matching DirectoryIndex (index.html) found, and server-generated directory index forbidden by Options directive
```

This corresponded to the earlier HTTP 403.

### Root Cause of the Historical 403

At the time of the failed request, Apache received `GET /` but there was no matching `index.html` under `/var/www/html/`. Because directory listing was forbidden, Apache returned **403 Forbidden**.

After `index.html` was created, the same HTTP request returned **200 OK** and served the SkyRoute Operations Portal.

The causal chain was:

```text
Client requests /
      ↓
Apache receives request
      ↓
/var/www/html/ has no index.html
      ↓
Directory listing is forbidden
      ↓
HTTP 403 Forbidden
```

After deployment:

```text
index.html created
      ↓
Apache finds DirectoryIndex
      ↓
HTTP 200 OK
      ↓
SkyRoute Operations Portal displayed
```

## 9. Application Portal Deployment

A production-like SkyRoute Operations Portal was created as a static `index.html` under Apache's existing web root.

The page includes:

- Production environment indicator
- SkyRoute Operations branding
- Application health
- Active-flight metrics
- Requests per minute
- Availability percentage
- Service health indicators
- Environment/server/release information
- Recent incident/change records
- Responsive layout for browser demonstration

The portal is intentionally static at this stage. A future exercise can introduce an actual backend process and multi-layer application architecture.

## Final Day 3 Assessment

The currently reported P1 outage could **not be reproduced** after the portal was deployed. Current evidence shows:

- Network reachability — working
- TCP port 80 — listening
- Apache — running
- HTTP connectivity from Ubuntu — working
- Current `GET /` requests — HTTP 200
- SkyRoute portal — successfully served
- `/opt/skyroute` — directory structure exists but does not contain a backend application
- Apache main configuration — no SkyRoute reference

The historical HTTP 403 was explained by the missing `index.html` in Apache's `DocumentRoot`.

No Apache restart or firewall change was required to make the static page available.

## Key Production-Support Lessons

- Do not restart a service merely because an application is reported as unavailable.
- Establish network, port, service, process, configuration, and application state in layers.
- `ping` proves basic IP reachability, not application availability.
- A listening port proves a socket is accepting connections, not that the application is healthy.
- `curl` is useful for validating the actual HTTP path and response.
- HTTP status codes are valuable incident evidence.
- Access logs show what clients requested and what HTTP status they received.
- Error logs help explain why a request failed.
- Historical errors must be separated from the current state.
- A missing process name does not automatically mean an application is down; understand the architecture first.
- Do not change firewall or service configuration without evidence that the change is required.

## Next Session — Day 4

Day 4 will use a more realistic failure scenario:

> **SkyRoute was working earlier, but is now returning an error when clients try to access it.**

The failure will be introduced deliberately and hidden from the trainee. Investigation will be ticket-driven, with minimal hints and no premature commands or fixes.

Planned workflow:

**Client symptom → hypothesis → evidence → investigation → root cause → recovery → verification → Jira documentation → RCA**
