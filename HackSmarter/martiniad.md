# MartiniAD

**Category:** Active Directory, No Credentials

## Attack Chain

1. Anonymous SMB notes leak limited credentials for `mprice`
2. Kerberoasting `ATHENA_SVC` cracks its password, unlocking WinRM
3. `SeMachineAccountPrivilege` looks like a direct path to RBCD, but isn't; the attempt is correctly blocked
4. PowerView, BloodHound, Rubeus, and LaZagne all fail (a WinPEAS `GenericAll` finding turns out to be a false positive)
5. PowerShell history leaks the Administrator password directly
6. Several Mimikatz/secretsdump attempts to extract `krbtgt` fail
7. A specific `lsadump::lsa /inject` Mimikatz command extracts the `krbtgt` hash

## TL;DR

Anonymous SMB access exposed a notes file with credentials for `mprice`, valid but limited to basic access, no RDP, WinRM, or useful shares. Kerberoasting the `ATHENA_SVC` service account (found via `rpcclient`) cracked its password, unlocking WinRM access. From there, `ATHENA_SVC` turned out to hold `SeMachineAccountPrivilege`, which looked at first like a direct path to a Resource-Based Constrained Delegation attack, but wasn't: the privilege alone doesn't grant the delegation rights RBCD needs, and the attempt was correctly blocked with an insufficient-rights error. PowerView, BloodHound, Rubeus, and LaZagne all failed to surface anything further (a WinPEAS-reported `GenericAll` on the DC also turned out to be a false positive), until a PowerShell history file leaked the Administrator password directly. From Administrator, several attempts to dump credentials with different Mimikatz builds and a manual SAM/SYSTEM copy all failed, until a specific Mimikatz `lsadump::lsa` command, found by researching the errors, extracted the krbtgt hash: full domain compromise.

## Full Walkthrough

### Enumeration

Nmap:

![martiniad screenshot 1](images/martiniad/martiniad-01.png)

Trying RPC enumeration:

![martiniad screenshot 2](images/martiniad/martiniad-02.png)

Reading SMB shares:

![martiniad screenshot 3](images/martiniad/martiniad-03.png)

A "notes" reference stands out. Logging into SMB anonymously:

![martiniad screenshot 4](images/martiniad/martiniad-04.png)

Checking the notes:

![martiniad screenshot 5](images/martiniad/martiniad-05.png)

Credentials turn up:

```
mprice:*martini*
```

Testing them with CrackMapExec:

![martiniad screenshot 6](images/martiniad/martiniad-06.png)

They work. Worth trying RDP, WinRM, and a closer look at SMB share permissions from here. Checking shares:

![martiniad screenshot 7](images/martiniad/martiniad-07.png)

RDP fails, and so does WinRM.

Checking SYSVOL and NETLOGON, and enumerating RPC as `mprice` for info leaks:

![martiniad screenshot 8](images/martiniad/martiniad-08.png)

Nothing in SYSVOL.

![martiniad screenshot 9](images/martiniad/martiniad-09.png)

Nothing in NETLOGON either.

![martiniad screenshot 10](images/martiniad/martiniad-10.png)

`rpcclient` does turn up users, two of which look useful for lateral movement: `Athena.t0` and the service account `ATHENA_SVC`. Querying them:

![martiniad screenshot 11](images/martiniad/martiniad-11.png)

![martiniad screenshot 12](images/martiniad/martiniad-12.png)

Nothing further from that angle. Two paths forward: run BloodHound as `mprice`, or try kerberoasting from Kali directly. BloodHound first:

![martiniad screenshot 13](images/martiniad/martiniad-13.png)

It fails due to LDAP signing requirements.

### Kerberoasting ATHENA_SVC

Generating a hosts file with `nxc smb`:

![martiniad screenshot 14](images/martiniad/martiniad-14.png)

Running `impacket-getuserSPNs` to kerberoast:

![martiniad screenshot 15](images/martiniad/martiniad-15.png)

A hash for `ATHENA_SVC` comes back. Cracking it with Hashcat:

![martiniad screenshot 16](images/martiniad/martiniad-16.png)

![martiniad screenshot 17](images/martiniad/martiniad-17.png)

Cracked:

```
1dirtymartini
```

Checking it with CrackMapExec, including shares:

![martiniad screenshot 18](images/martiniad/martiniad-18.png)

Same permission level as before. Trying RDP, WinRM, and BloodHound again:

![martiniad screenshot 19](images/martiniad/martiniad-19.png)

RDP hits an error. WinRM via `evil-winrm`:

![martiniad screenshot 20](images/martiniad/martiniad-20.png)

That works.

### ATHENA_SVC Enumeration

Checking privileges with `whoami /all`:

![martiniad screenshot 21](images/martiniad/martiniad-21.png)

`ATHENA_SVC` holds `SeMachineAccountPrivilege`, which allows adding workstations to the domain. At first glance this looks like a direct path to a Resource-Based Constrained Delegation (RBCD) attack, the usual route being: create a computer account, configure delegation, forge a ticket impersonating Administrator, and connect via `psexec`. That assumption turns out to be wrong, the privilege alone doesn't grant RBCD rights on its own, and confirming that costs a few steps.

Before attempting it, checking whether `ATHENA_SVC` already has `GenericAll` or similar rights on DC01 is worth ruling out first. Transferring PowerView and filtering AD objects by `ATHENA_SVC`'s SID with `Get-Object` / `Where-Object`:

![martiniad screenshot 22](images/martiniad/martiniad-22.png)

Nothing found. Running BloodHound to double-check:

![martiniad screenshot 23](images/martiniad/martiniad-23.png)

SharpHound runs, but BloodHound returns no useful output either. Trying RBCD anyway: requesting a TGT for `ATHENA_SVC` with `impacket-getTGT`:

![martiniad screenshot 24](images/martiniad/martiniad-24.png)

Exporting it:

![martiniad screenshot 25](images/martiniad/martiniad-25.png)

Creating a computer account with `impacket-addcomputer`:

![martiniad screenshot 26](images/martiniad/martiniad-26.png)

Configuring Resource-Based Constrained Delegation with BloodyAD, granting the new machine account `BLACK$` delegation rights to (and thus the ability to impersonate users on) the domain controller `DC01`:

![martiniad screenshot 27](images/martiniad/martiniad-27.png)

Insufficient access rights, confirming the earlier assumption was wrong: `SeMachineAccountPrivilege` alone isn't enough for RBCD here. Time to change approach.

### Further Enumeration

Trying `Rubeus.exe` and `LaZagne.exe` (LaZagne fails outright on insufficient privileges):

![martiniad screenshot 28](images/martiniad/martiniad-28.png)

![martiniad screenshot 29](images/martiniad/martiniad-29.png)

![martiniad screenshot 30](images/martiniad/martiniad-30.png)

Nothing useful. Enumerating the machine further:

![martiniad screenshot 31](images/martiniad/martiniad-31.png)

`inetpub` contains `DeviceHealthAttestation`, which includes a DLL, a possible DLL hijack candidate. Running WinPEAS and PowerUp's `Invoke-AllChecks` for a fuller sweep:

![martiniad screenshot 32](images/martiniad/martiniad-32.png)

PowerUp only surfaces a path that isn't useful here, since the identity reference is `Athena_svc` itself. WinPEAS:

![martiniad screenshot 33](images/martiniad/martiniad-33.png)

![martiniad screenshot 34](images/martiniad/martiniad-34.png)

Its output suggests `Athena_SVC` has `GenericAll` on Administrator and DC01, but this echoes the same false lead already ruled out earlier. Checking PowerShell history instead:

![martiniad screenshot 35](images/martiniad/martiniad-35.png)

The Administrator password is sitting right there.

### Access as Administrator

With Administrator credentials in hand, the plan is RDP followed by Mimikatz to extract the krbtgt hash:

![martiniad screenshot 36](images/martiniad/martiniad-36.png)

The password has expired, so a new one is set:

![martiniad screenshot 37](images/martiniad/martiniad-37.png)

![martiniad screenshot 38](images/martiniad/martiniad-38.png)

Running Mimikatz produces an error: `ERROR kuhl_m_sekurlsa_acquireLSA ; Logon list`. Research points to this being version-related, worth trying a different Mimikatz build or falling back to LaZagne.

Neither works, even after switching Mimikatz versions. Copying `SAM` and `SYSTEM` locally to dump them with `impacket-secretsdump` is the next attempt:

![martiniad screenshot 39](images/martiniad/martiniad-39.png)

![martiniad screenshot 40](images/martiniad/martiniad-40.png)

![martiniad screenshot 41](images/martiniad/martiniad-41.png)

![martiniad screenshot 42](images/martiniad/martiniad-42.png)

![martiniad screenshot 43](images/martiniad/martiniad-43.png)

![martiniad screenshot 44](images/martiniad/martiniad-44.png)

![martiniad screenshot 45](images/martiniad/martiniad-45.png)

![martiniad screenshot 46](images/martiniad/martiniad-46.png)

This doesn't work either. Research into [Golden Ticket attacks and how to dump the krbtgt hash](https://netwrix.com/en/resources/blog/complete-domain-compromise-with-golden-tickets/) points to a different Mimikatz command:

```
lsadump::lsa /inject /name:krbtgt
```

This immediately dumps the krbtgt hash.

![martiniad screenshot 47](images/martiniad/martiniad-47.png)

Full domain compromise.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
