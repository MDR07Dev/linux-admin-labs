# Lab 01: User & Group Management

## Objective
Create users and groups and configure role-based sudoers access.

## Environment
- OS: Rocky Linux 8.10
- Platform: VMware

## Users
- harry
- ron

## Groups
- asc_opsteam
- asc_opsteamplus

## Commands Used
- useradd -m
- groupadd
- usermod
- visudo -f /etc/sudoers.d/project.access -- %groupname ALL=(ALL) /path/to/tool command,

## Verification

- user harry was able access only to "systemctl restart httpd" and "journalctl -u httpd" ,
- verified by "su harry" and tried access both authorized and unauthorized entries. Resulted as expected.

- user ron was able to access what harry was able to do along with able to install/update packages, start/stop httpd service.
- verified by "su ron" and tried access both authorized and unauthorized entries. Resulted as expected.


- #1 asc_opsteam ; User in the group: harry ; harry being junior Ops Eng, has access to only restart the service(httpd) and read its logs(journalctl).
  Being the member of asc_opsteam, harry cannot install packages, start/stop services.
- #2 asc_opsteamplus ; User in the group: ron ; ron being senior
 Ops Eng have access to what junior ops eng plus start, restart, stop the service(httpd) along with install and update packages(dnf install/update),
 and access to its config file. being the member of asc_opsteamplus,
 ron cannot reboot the system along with cannot access to anything outside /etc/httpd/conf/.

## Problems Encountered

- user harry or ron were unable to perform any of the permitted commands.

## Solution & what I learned

- Figured out the any file inside the "/etc/sudoers.d/ " should not be named with "." eg; project.access,
- rather it can be name with "_", eg; project_access. And that resolved the issue.

## Production Relevance

- Before setting up the infra for the project, setting up the permissions for the users.
- 

- 
  
