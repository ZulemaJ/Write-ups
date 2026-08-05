# Past

**Category:** Active Directory, No Credentials

## Attack Chain

1. Guest SMB access leaks a list of AD hosts, no direct compromise yet
2. TimeRoasting cracks the `APPDEV01$` machine account hash
3. `APPDEV01$` SMB access exposes a logon script containing Tyler's cleartext password
4. NTLM logon is restricted for Tyler; a Kerberos TGT is used to authenticate instead
5. BloodHound shows Tyler has `GenericAll` on the domain controller object
6. A Resource-Based Constrained Delegation (RBCD) attack forges a ticket impersonating Administrator
7. `secretsdump` and `evil-winrm` complete the compromise

## TL;DR

Starting with zero credentials against a domain controller, guest SMB access leaked a list of AD hosts but little else. Most of the standard no-creds AD playbook came up empty. The path in was **TimeRoasting**, which cracked a machine account hash (`APPDEV01$`). That account's SMB access exposed a logon script on SYSVOL containing a user's (`tyler`) cleartext password. NTLM logon was blocked by an account restriction, so authentication was done over **Kerberos** instead. BloodHound then revealed that `tyler` held `GenericAll` on the domain controller object, abused via an **RBCD attack** to forge a ticket as Administrator, dump secrets, and log in with `evil-winrm`.

## Full Walkthrough

### Enumeration

An Nmap scan against the target reveals the common port set of a domain controller.

![past screenshot 1](images/past/past-01.png)

RPC enumeration is attempted first but returns Access Denied.

![past screenshot 2](images/past/past-02.png)

SMB as a **guest** account fares better: the `IPC$` and `Share` shares are both readable.

![past screenshot 3](images/past/past-03.png)

Confirming the same access anonymously with `impacket-smbclient`:

![past screenshot 4](images/past/past-04.png)

Inside `/Share`, `AD_machines.txt` lists the machines belonging to this AD environment.

### Working the No-Creds AD Methodology

With no leaked credentials to work from, the next step is to follow [Orange Cyberdefense's AD methodology for environments with no creds](https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg).

First, `nxc smb` generates a hosts file from the domain:

![past screenshot 5](images/past/past-05.png)

...which gets added to `/etc/hosts`:

![past screenshot 6](images/past/past-06.png)

Most of the remaining techniques on the mindmap were worked through systematically and led nowhere on this target. The one that actually paid off was **TimeRoasting**.

### TimeRoasting

`Timeroast` is an Active Directory attack that exploits predictable timestamps in the Windows Time (NTP/SNTP) service to obtain password hashes of machine accounts. An attacker sends specially crafted time requests, extracts the returned cryptographic material, and attempts to crack the machine account password offline.

Running `timeroast.py` and logging the output:

![past screenshot 7](images/past/past-07.png)

Several hashes come back, each with its RID prefixed at the start of the line:

![past screenshot 8](images/past/past-08.png)

Cracking the SNTP hashes with Hashcat:

![past screenshot 9](images/past/past-09.png)

![past screenshot 10](images/past/past-10.png)

![past screenshot 11](images/past/past-11.png)

One of the hashes cracks successfully.

### Identifying the Cracked Account

`nxc smb --rid` lists the RIDs of the domain so the cracked hash can be matched to an account:

![past screenshot 12](images/past/past-12.png)

The cracked password maps to **RID 1115** (the `APPDEV01$` machine account). Confirming with CrackMapExec:

![past screenshot 13](images/past/past-13.png)

...and listing the shares it can access:

![past screenshot 14](images/past/past-14.png)

Logging into SMB as `APPDEV01$` to read the `SYSVOL` share:

![past screenshot 15](images/past/past-15.png)

![past screenshot 16](images/past/past-16.png)

![past screenshot 17](images/past/past-17.png)

Inside `/past.local/scripts`, `tyler_init.cmd` contains Tyler's password in plaintext.

### Authenticating as Tyler

A direct CrackMapExec auth attempt as Tyler hits an `ACCOUNT_RESTRICTION` error:

![past screenshot 18](images/past/past-18.png)

NTLM logon for this account is restricted, but Kerberos authentication isn't subject to the same restriction. Requesting a Ticket Granting Ticket (TGT) with `impacket-getTGT`:

![past screenshot 19](images/past/past-19.png)

Exporting the forged ticket:

![past screenshot 20](images/past/past-20.png)

Authenticating as Tyler again, this time over Kerberos with `KRB5CCNAME`:

![past screenshot 21](images/past/past-21.png)

### BloodHound Enumeration

With a working Kerberos-authenticated account, `bloodhound-ce-python` can now run from the attacking host:

- `-k`: Kerberos auth
- `-d`: domain
- `-dc`: domain controller
- `-ns`: nameserver
- `-c`: collection methods
- `--zip`: package the output as a zip

![past screenshot 22](images/past/past-22.png)

Loading the collected data into BloodHound reveals that **Tyler has `GenericAll` over the domain controller object.**

![past screenshot 23](images/past/past-23.png)

That permission is enough for a **Resource-Based Constrained Delegation** attack: add a computer account, configure delegation on the DC to trust it, and impersonate a privileged user through it.

![past screenshot 24](images/past/past-24.png)

### Resource-Based Constrained Delegation Attack

BloodHound's built-in guidance covers the exploit path, but this run uses **BloodyAD** instead.

Creating a computer account:

![past screenshot 25](images/past/past-25.png)

Configuring Resource-Based Constrained Delegation, granting the new machine account `BLACK$` the ability to delegate to (and thus impersonate users on) the domain controller `EC2AMAZ-A5O4OL8$`:

![past screenshot 26](images/past/past-26.png)

Unsetting `KRB5CCNAME` before requesting a fresh ticket:

![past screenshot 27](images/past/past-27.png)

Requesting a Service Ticket with `impacket-getST` for the `cifs/EC2AMAZ-A5O4OL8.past.local` SPN, impersonating the Administrator account and leveraging `BLACK$`'s delegation rights:

![past screenshot 28](images/past/past-28.png)

Exporting the ticket:

![past screenshot 29](images/past/past-29.png)

Authenticating as Administrator with `nxc`, using Kerberos (`-k`) and the exported ccache:

![past screenshot 30](images/past/past-30.png)

Domain Admin, confirmed.

### Dumping Secrets and Logging In as Admin

With a valid Administrator ticket, `impacket-secretsdump` dumps the domain secrets over Kerberos:

![past screenshot 31](images/past/past-31.png)

![past screenshot 32](images/past/past-32.png)

The dumped hashes give WinRM access as Administrator via `evil-winrm`:

![past screenshot 33](images/past/past-33.png)

Full administrative access to the DC.

One more loose end: Ryan's password, found in `History.txt`:

![past screenshot 34](images/past/past-34.png)

![past screenshot 35](images/past/past-35.png)

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
