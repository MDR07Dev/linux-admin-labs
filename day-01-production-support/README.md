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

Both VMs were tested in both directions:

```bash
ping 192.168.1.3
ping 192.168.1.2
```

Both directions succeeded with 4/4 replies and 0% packet loss.

An SSH attempt reached host-key verification. The observed `Host key verification failed` result was **not** treated as proof that the SSH service itself was down.

## SkyRoute Shared Directory

```bash
sudo mkdir -p /opt/skyroute
sudo mkdir -p /opt/skyroute/{bin,config,logs,data}
```

`appuser` and `supportuser` were members of the shared `appteam` group. The application directory was configured as:

```text
drxrwsr-x. root appteam /opt/skyroute
```

More precisely, the observed mode was:

```text
drwxrwsr-x. root appteam /opt/skyroute
```

The `s` in the group position is the **setgid** bit. It causes newly created files/directories beneath the directory to inherit the `appteam` group.

## SIO-2 Investigation

### Reported symptom

> `supportuser` cannot create/modify files under `/opt/skyroute`.

### Investigation rule

**Reproduce first. Collect evidence before changing the system.**

### Reproduction as `supportuser`

```bash
su - supportuser
whoami
id
cd /opt/skyroute
touch test.txt
ls -l test.txt
stat test.txt
```

Results:

- Correct user identity confirmed.
- `supportuser` belonged to `appteam`.
- Directory access succeeded.
- File creation succeeded.
- File showed `supportuser` as owner and `appteam` as group.

Modification also succeeded:

```bash
echo "SkyRoute production test" > test.txt
cat test.txt
```

### Cross-user shared access

`appuser` was tested against the file created by `supportuser`:

```bash
su - appuser
whoami
id
cd /opt/skyroute
echo "appuser update" > test.txt
cat test.txt
```

This succeeded, confirming shared group write access.

### Application subdirectories

```bash
for dir in bin config data logs; do
  echo "=== $dir ==="
  touch "/opt/skyroute/$dir/test.txt"
  echo "exit_code=$?"
done
```

All four tests returned `exit_code=0`.

## Deeper Access-Control Checks

### Path traversal

```bash
namei -l /opt/skyroute/test.txt
```

Confirmed required traversal permissions through `/` and `/opt`.

### POSIX ACLs

```bash
getfacl /opt/skyroute
getfacl /opt/skyroute/test.txt
```

No additional ACL entries were present beyond the standard owner/group/other permissions.

### SELinux

```bash
ls -Zd /opt/skyroute
ls -Z /opt/skyroute
sudo ausearch -m AVC -ts recent
```

The observed contexts were consistent with the lab environment, and `ausearch` returned no recent AVC matches. There was no evidence that SELinux denied the tested operations.

## Application / Service Investigation

```bash
ps -ef | grep -i skyroute
systemctl --type=service --state=running
systemctl --failed
ss -lntp
```

- No SkyRoute-specific process was running; only the `grep` process appeared.
- No failed systemd units were reported.
- Apache `httpd` was running and port 80 was listening.

Apache was then checked:

```bash
sudo systemctl status httpd --no-pager
apachectl -S
curl -I http://localhost
sudo tail -n 30 /var/log/httpd/error_log
grep -R "DocumentRoot" /etc/httpd/conf /etc/httpd/conf.d
```

`curl -I http://localhost` returned **HTTP 403 Forbidden**. Apache's error log identified the cause: `/var/www/html/` had no matching `DirectoryIndex` and directory listing was forbidden. `DocumentRoot` was `/var/www/html`.

This was documented as a **separate Apache/content configuration finding**, not the cause of SIO-2.

## Final SIO-2 Assessment

The reported filesystem access failure was **NOT reproducible under the tested conditions**.

Evidence showed that:

- Unix permissions were working.
- Group membership was correct.
- Setgid group inheritance was working.
- POSIX ACLs did not introduce a restriction.
- No recent SELinux AVC denial was found.
- Both `supportuser` and `appuser` could create/modify files in the tested application paths.

No filesystem access-control root cause was established.

The Jira ticket was updated with the evidence and left **Open** rather than being closed prematurely. The correct next step, if the issue must be pursued, is to obtain the original failing operation, affected file/application component, exact error message, and approximate time of failure.

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

**SIO-2:** [INC-001 — SkyRoute application directory access failure](https://rafeeqdocs.atlassian.net/browse/SIO-2)
