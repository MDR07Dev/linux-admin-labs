PURPOSE: 
- To create directory named /asc/project/ and set permissions for users, groups for the files that is/are to be created within this directory. This directory is a shared folder created to collaborate the works of both tiers.

Group/ownership model: 
 - A new group created as asc_opsshared.
 - ron, harry are the members of this group.
 - purpose of this group is to share few common permissions to both ron and harry as they are being on different level.
 - Reusing their usual groups may end up permission messes.
 - __Commands used: **mkdir**_

SetGID Explanation:
 - This directory has been set with SetGID to have it's own group ownership and inherit group ownership for all the files that are to be created inside this directory.
 - _Commands used: **chmod g+s**_

Base Permissions:
 - Base permission has been set as below,
   owner  - 7
   group  - 7
   others - 0
 - _Commands used: **chmod 770**_

ACL exception:
 - ACL permission of r-- set to alice, because the regular permissions are limited with 3 buckets. No provision to add one extra user specific without ACL tool.
 - _Commands used: **setfacl**_

File/directory path: /asc/project/
