# Stellar

**Category:** Active Directory

## Attack Chain

1. Anonymous FTP exposes a PDF with a default password, still valid on `junior.analyst`
2. `junior.analyst` has `WriteOwner` on a group holding `ForceChangePassword` over `ops.controller`
3. Ownership is taken, group membership added, `ops.controller`'s password reset
4. Kerberoasting, ASREPRoasting, AV tampering, scheduled tasks, and PowerView are all blocked
5. A Firefox profile pulled from `ops.controller`'s desktop decrypts to reveal `astro.researcher`'s credentials
6. `astro.researcher` has `WriteDACL` on `eng.payload`, abused to reset its password
7. `eng.payload` can read a GMSA password and holds DCSync rights on the domain controller
8. DCSync dumps the Administrator hash

## TL;DR

Anonymous FTP access exposed a user guide PDF listing a default password, still unchanged on `junior.analyst`. That account had `WriteOwner` on a group whose members held `ForceChangePassword` over `ops.controller`, abused (after correcting the DACL abuse chain: taking ownership before granting write-members rights) to add `junior.analyst` to the group and reset `ops.controller`'s password. From `ops.controller`, several standard privesc angles (Kerberoasting, ASREPRoasting, AV tampering, scheduled tasks, PowerView) were all blocked, but a Firefox profile pulled from the desktop decrypted to reveal credentials for `astro.researcher`. That account had `WriteDACL` over `eng.payload`, abused to grant full control and reset its password. `eng.payload` in turn could read a GMSA password directly and, more importantly, held DCSync rights on the domain controller, used with `impacket-secretsdump` to dump the Administrator hash and complete the compromise.

## Full Walkthrough

### Enumeration

Nmap:

![stellar screenshot 1](images/stellar/stellar-01.png)

Standard port set. **FTP enumeration:** checking for anonymous access:

![stellar screenshot 2](images/stellar/stellar-02.png)

Allowed. Checking the contents:

![stellar screenshot 3](images/stellar/stellar-03.png)

A "Docs" directory looks interesting:

![stellar screenshot 4](images/stellar/stellar-04.png)

Downloading the files for later review. Opening one PDF immediately fails:

![stellar screenshot 5](images/stellar/stellar-05.png)

Setting it aside for now and continuing enumeration.

`enum4linux`:

![stellar screenshot 6](images/stellar/stellar-06.png)

**HTTP enumeration**, port 80:

![stellar screenshot 7](images/stellar/stellar-07.png)

Nothing there. Directory brute-forcing with Feroxbuster:

![stellar screenshot 8](images/stellar/stellar-08.png)

![stellar screenshot 9](images/stellar/stellar-09.png)

Also empty. **SMB enumeration:**

![stellar screenshot 10](images/stellar/stellar-10.png)

Nothing found here either. The FTP documents look like the most promising lead, but the PDF keeps reporting a "file damaged" error.

**Back to FTP:** opening the file with a different reader, `mupdf`, works:

![stellar screenshot 11](images/stellar/stellar-11.png)

The user guide contains a default password, `Galaxy123!`, with a note that all users must change it after first login. Worth checking whether `junior.analyst` still has it:

![stellar screenshot 12](images/stellar/stellar-12.png)

They do. Checking accessible shares with CrackMapExec:

![stellar screenshot 13](images/stellar/stellar-13.png)

SYSVOL is always worth a look. Updating `/etc/hosts` first:

![stellar screenshot 14](images/stellar/stellar-14.png)

Logging into SMB:

![stellar screenshot 15](images/stellar/stellar-15.png)

![stellar screenshot 16](images/stellar/stellar-16.png)

Nothing interesting inside. **RPC enumeration** to gather users:

![stellar screenshot 17](images/stellar/stellar-17.png)

Trying WinRM and RDP with `junior.analyst`'s credentials:

![stellar screenshot 18](images/stellar/stellar-18.png)

Both fail. Continuing the attack from Kali instead of expecting direct shell access.

### Initial Access as ops.controller

**BloodHound enumeration**, run remotely with `bloodhound-ce-python`:

![stellar screenshot 19](images/stellar/stellar-19.png)

Checking for misconfigurations:

![stellar screenshot 20](images/stellar/stellar-20.png)

`junior.analyst` has `WriteOwner` on the `Stellarops-control` group. Checking BloodHound's Linux Abuse guidance for this edge:

![stellar screenshot 21](images/stellar/stellar-21.png)

`WriteOwner` allows adding members to the group. Inspecting the group first:

![stellar screenshot 22](images/stellar/stellar-22.png)

Members of `Stellarops-control` hold `ForceChangePassword` over `ops.controller`. The plan: add `junior.analyst` to the group, then reset `ops.controller`'s password.

Following the standard instructions to add a member directly:

```
net rpc group addmem "TargetGroup" "TargetUser" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"
```

![stellar screenshot 23](images/stellar/stellar-23.png)

That alone isn't enough, the permissions need changing first:

```
dacledit.py -action 'write' -rights 'WriteMembers' -principal 'controlledUser' -target-dn 'groupDistinguishedName' 'domain'/'controlledUser':'password'
```

![stellar screenshot 24](images/stellar/stellar-24.png)

Still no luck, possibly a distinguished name issue. Trying the fully qualified DN:

![stellar screenshot 25](images/stellar/stellar-25.png)

![stellar screenshot 26](images/stellar/stellar-26.png)

Still not working. Re-reading BloodHound more carefully clarifies the actual order of operations: ownership needs to change before write-members rights can be granted:

```
owneredit.py -action write -owner 'attacker' -target 'victim' 'DOMAIN'/'USER':'PASSWORD'
```

![stellar screenshot 27](images/stellar/stellar-27.png)

(Note: the correct flag is `-new-owner`, not `-owner`, which is no longer supported.)

Retrying with the correct flag:

![stellar screenshot 28](images/stellar/stellar-28.png)

With ownership sorted, adding `junior.analyst` to the group:

![stellar screenshot 29](images/stellar/stellar-29.png)

No output, which means success here. Changing `ops.controller`'s password:

![stellar screenshot 30](images/stellar/stellar-30.png)

![stellar screenshot 31](images/stellar/stellar-31.png)

Confirming:

![stellar screenshot 32](images/stellar/stellar-32.png)

Password changed successfully. `ops.controller` has no outbound object control of its own. Trying WinRM:

![stellar screenshot 33](images/stellar/stellar-33.png)

Fails, as does RDP:

![stellar screenshot 34](images/stellar/stellar-34.png)

Unexpected, since the account is a member of Remote Management. Retrying WinRM after realizing the target flag needs the IP address rather than the hostname:

![stellar screenshot 35](images/stellar/stellar-35.png)

That was the issue. Access confirmed.

### Enumeration as ops.controller

Checking PowerShell history:

![stellar screenshot 36](images/stellar/stellar-36.png)

Empty. Checking BloodHound for kerberoastable users:

![stellar screenshot 37](images/stellar/stellar-37.png)

Trying to kerberoast with Rubeus:

![stellar screenshot 38](images/stellar/stellar-38.png)

Possibly blocked by AV. Trying from Kali instead:

![stellar screenshot 39](images/stellar/stellar-39.png)

Also unsuccessful. ASREPRoasting:

![stellar screenshot 40](images/stellar/stellar-40.png)

Running WinPEAS:

![stellar screenshot 41](images/stellar/stellar-41.png)

Blocked, as expected given the AV presence. Listing processes:

![stellar screenshot 42](images/stellar/stellar-42.png)

Microsoft Defender is running. Checking whether it can be stopped:

![stellar screenshot 43](images/stellar/stellar-43.png)

No permissions for that. Listing scheduled tasks:

![stellar screenshot 44](images/stellar/stellar-44.png)

Not permitted either.

![stellar screenshot 45](images/stellar/stellar-45.png)

A more promising thread: the `SatLink Service` account holds `DCSync`, `GetChanges`, `GetChangesInFilteredSet`, and `GetChangesAll` on `DC01`, clearly the eventual path to full compromise. BloodHound flags it as kerberoastable, but the earlier attempts found nothing.

Checking other tools for kerberoastable users, starting with [ADenum](https://github.com/SecuProject/ADenum):

![stellar screenshot 46](images/stellar/stellar-46.png)

![stellar screenshot 47](images/stellar/stellar-47.png)

No entries found. Trying PowerView:

![stellar screenshot 48](images/stellar/stellar-48.png)

Blocked as well. This particular thread (kerberoasting `SatLink Service` directly) isn't panning out, time to change direction and enumerate more broadly.

![stellar screenshot 49](images/stellar/stellar-49.png)

A Firefox setup is visible in `/ops.controller/Desktop`, worth investigating for stored credentials. Following [HackTricks' browser artifacts guide](https://hacktricks.wiki/en/generic-methodologies-and-resources/basic-forensic-methodology/specific-software-file-type-tricks/browser-artifacts.html):

![stellar screenshot 50](images/stellar/stellar-50.png)

![stellar screenshot 51](images/stellar/stellar-51.png)

Downloading the entire profile directory with `download*` from `evil-winrm`:

![stellar screenshot 52](images/stellar/stellar-52.png)

Once downloaded, it can be examined properly on Kali. HackTricks also documents what kind of information typically lives in a Firefox profile:

![stellar screenshot 53](images/stellar/stellar-53.png)

...including a way to decrypt the master password with [firefox_decrypt](https://github.com/unode/firefox_decrypt):

![stellar screenshot 54](images/stellar/stellar-54.png)

### Decrypting the Master Password

Running it:

![stellar screenshot 55](images/stellar/stellar-55.png)

Credentials recovered:

```
astro.researcher:Cosmos@42
```

Checking outbound object control for this account in BloodHound:

![stellar screenshot 56](images/stellar/stellar-56.png)

`WriteDACL` on `eng.payload`. Checking the available options:

![stellar screenshot 57](images/stellar/stellar-57.png)

Either changing the password directly (after assigning the right permissions) or kerberoasting. Going with the password change.

**Assigning rights:**

```
dacledit.py -action 'write' -rights 'FullControl' -principal 'controlledUser' -target 'targetUser' 'domain'/'controlledUser':'password'
```

![stellar screenshot 58](images/stellar/stellar-58.png)

**Changing the password:**

```
net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"
```

![stellar screenshot 59](images/stellar/stellar-59.png)

Confirming:

![stellar screenshot 60](images/stellar/stellar-60.png)

Done. Checking `eng.payload`'s own outbound object control:

![stellar screenshot 61](images/stellar/stellar-61.png)

**ReadGMSAPassword** is available:

![stellar screenshot 62](images/stellar/stellar-62.png)

Reading it with BloodyAD, following [this methodology](https://www.thehacker.recipes/ad/movement/dacl/readgmsapassword):

![stellar screenshot 63](images/stellar/stellar-63.png)

![stellar screenshot 64](images/stellar/stellar-64.png)

Both the base64-encoded and NT-format passwords come back:

![stellar screenshot 65](images/stellar/stellar-65.png)

No need to crack anything further for now, the real prize is `eng.payload`'s DCSync rights, noted earlier.

### Access as Administrator

With DCSync rights confirmed, the domain controller's replication can be impersonated to request any user's password directly with `impacket-secretsdump`. Since there's no cleartext password for `eng.payload`, the `-hashes` flag is used instead (NTLM format):

![stellar screenshot 66](images/stellar/stellar-66.png)

The Administrator hash comes back.

![stellar screenshot 67](images/stellar/stellar-67.png)

Full domain compromise.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
