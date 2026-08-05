# Lumon

**Category:** Active Directory  
**Status:** Not completed, this lab is left as an open investigation rather than a full compromise

## Attack Chain

1. Initial SMB/AD enumeration with `hellyr`'s credentials: a large user list, no direct leaks
2. A guessed password spray and ASREPRoasting both fail
3. A writable SMB share on the intranet app is found; three NTLM poisoning attempts fail
4. CVE-2025-24054 (`.library-ms` NTLM leak) coerces and captures `harmonyc`'s NTLMv2 hash
5. `harmonyc` (Administration group) gains Admin Panel access on the intranet site
6. `ADMIN$`/`C$` browsing looks promising, but an `ntlmrelayx` relay attempt produces no output
7. Investigation left open, mid-attempt

## TL;DR

Starting from a known credential (`hellyr`), initial SMB and AD enumeration turned up a large user list plus a service account, `IntranetSvc`, but no direct outbound object control, no successful password spray, and no ASREPRoastable accounts. A Group Policy `Registry.pol` file parsed cleanly but revealed nothing useful. A separate intranet web application had a writable SMB share, an obvious lead, but the credentials that worked there didn't map to Windows access anywhere (RDP, WinRM, and SMB relay attempts all failed against both hosts, at times because of the lab's own networking hiccups). The actual breakthrough came from **CVE-2025-24054**, an NTLM hash leak triggered by a `.library-ms` file: dropping a crafted file onto the share and coercing a user via Responder captured and cracked an NTLMv2 hash for `harmonyc`, a member of the Administration group, and got Admin Panel access on the intranet site. From there, browsing `ADMIN$` and `C$` looked promising, but a follow-up NTLM relay attempt with `impacket-ntlmrelayx` produced no output. The investigation ends there, mid-attempt.

## Full Walkthrough

### Initial Enumeration

Starting from known credentials for `hellyr`, initial scanning and SMB enumeration map out the domain controller and its shares:

![lumon screenshot 1](images/lumon/lumon-01.png)

![lumon screenshot 2](images/lumon/lumon-02.png)

![lumon screenshot 3](images/lumon/lumon-03.png)

![lumon screenshot 4](images/lumon/lumon-04.png)

![lumon screenshot 5](images/lumon/lumon-05.png)

![lumon screenshot 6](images/lumon/lumon-06.png)

![lumon screenshot 7](images/lumon/lumon-07.png)

![lumon screenshot 8](images/lumon/lumon-08.png)

RDP and WinRM both fail with these credentials.

![lumon screenshot 9](images/lumon/lumon-09.png)

![lumon screenshot 10](images/lumon/lumon-10.png)

A large number of users comes back, along with a service account, `IntranetSvc`. Querying all users with `query user` in `rpcclient` before trying to kerberoast doesn't leak anything useful.

Kerberoasting:

![lumon screenshot 11](images/lumon/lumon-11.png)

Running `bloodhound-ce-python`:

```
bloodhound-ce-python -u hellyr -p H3lenaR!2025 -d lumons.hacksmarter -dc DC01.lumons.hacksmarter -ns 10.0.23.182 -c All --zip
```

![lumon screenshot 12](images/lumon/lumon-12.png)

![lumon screenshot 13](images/lumon/lumon-13.png)

`hellyr` has no outbound object control. With a list of users in hand, a password spray is worth trying. Since the pattern for `hellyr` is `H3lenaR!2025`, a guess at `H3lenaE!2025` for a similarly-named user `hellye`:

![lumon screenshot 14](images/lumon/lumon-14.png)

![lumon screenshot 15](images/lumon/lumon-15.png)

That guess doesn't pan out. ASREPRoasting is worth a shot next:

![lumon screenshot 16](images/lumon/lumon-16.png)

Nothing usable comes back either. Circling back to SMB:

![lumon screenshot 17](images/lumon/lumon-17.png)

Nothing in NETLOGON. Reviewing SYSVOL again:

![lumon screenshot 18](images/lumon/lumon-18.png)

A readable `Registry.pol` file turns up:

![lumon screenshot 19](images/lumon/lumon-19.png)

Parsing it with [regpol](https://github.com/jtpereyda/regpol):

![lumon screenshot 20](images/lumon/lumon-20.png)

![lumon screenshot 21](images/lumon/lumon-21.png)

Nothing particularly interesting in the output. Rechecking BloodHound shows `hellyr` is part of the "microdata refinement" group:

![lumon screenshot 22](images/lumon/lumon-22.png)

Meanwhile `hellye`, the earlier guessed account, turns out to be part of Domain Admins:

![lumon screenshot 23](images/lumon/lumon-23.png)

### Intranet Enumeration

Enumerating the intranet application:

![lumon screenshot 24](images/lumon/lumon-24.png)

Checking SMB again:

![lumon screenshot 25](images/lumon/lumon-25.png)

![lumon screenshot 26](images/lumon/lumon-26.png)

This is more promising: write permissions on a share, a direct opening for SMB poisoning.

![lumon screenshot 27](images/lumon/lumon-27.png)

![lumon screenshot 28](images/lumon/lumon-28.png)

![lumon screenshot 29](images/lumon/lumon-29.png)

![lumon screenshot 30](images/lumon/lumon-30.png)

Checking file metadata with `exiftool` just in case:

![lumon screenshot 31](images/lumon/lumon-31.png)

Trying to log into the intranet dashboard at `https://intranet.lumons.hacksmarter/login`. A PDF found earlier states the sign-in format: `your-username`, `LUMONS\your-username`, or `your-username@lumons.hacksmarter`.

![lumon screenshot 32](images/lumon/lumon-32.png)

![lumon screenshot 33](images/lumon/lumon-33.png)

![lumon screenshot 34](images/lumon/lumon-34.png)

![lumon screenshot 35](images/lumon/lumon-35.png)

Nothing usable comes of this angle yet. Trying `dirsearch`:

![lumon screenshot 36](images/lumon/lumon-36.png)

![lumon screenshot 37](images/lumon/lumon-37.png)

![lumon screenshot 38](images/lumon/lumon-38.png)

Switching wordlists to see if more paths surface. Meanwhile, the PDF mentions a "Seoul Annex", worth remembering.

![lumon screenshot 39](images/lumon/lumon-39.png)

### NTLM Poisoning Attempt

Trying [ntlm_theft](https://github.com/Greenwolf/ntlm_theft) against the writable share. Cloning it:

![lumon screenshot 40](images/lumon/lumon-40.png)

Running a Responder listener:

![lumon screenshot 41](images/lumon/lumon-41.png)

Running `ntlm_theft.py`:

![lumon screenshot 42](images/lumon/lumon-42.png)

Replacing "Lumons Intranet.url" with a malicious version:

![lumon screenshot 43](images/lumon/lumon-43.png)

![lumon screenshot 44](images/lumon/lumon-44.png)

No output. RDP and WinRM both fail again on this attempt.

### CVE-2025-24054: NTLM Hash Leak via .library-ms

Research into the next step points to **CVE-2025-24054**, a Windows vulnerability that leaks NTLM hashes by tricking a user into interacting with a `.library-ms` file. Windows automatically attempts NTLM authentication when accessing remote resources, often without a clear warning to the user, and this implicit trust in network locations can be coerced into sending credential material. Background and a proof of concept:

- [CVE-2025-24054 NTLM exploit in the wild (Check Point Research)](https://research.checkpoint.com/2025/cve-2025-24054-ntlm-exploit-in-the-wild/)
- [CVE-2025-24054 / CVE-2025-24071 PoC](https://github.com/helidem/CVE-2025-24054_CVE-2025-24071-PoC)

![lumon screenshot 45](images/lumon/lumon-45.png)

![lumon screenshot 46](images/lumon/lumon-46.png)

With Responder running, the crafted file is placed on the SMB share:

![lumon screenshot 47](images/lumon/lumon-47.png)

This time there's output:

![lumon screenshot 48](images/lumon/lumon-48.png)

An NTLMv2 hash for `harmonyc`. Cracking it with Hashcat:

![lumon screenshot 49](images/lumon/lumon-49.png)

![lumon screenshot 50](images/lumon/lumon-50.png)

![lumon screenshot 51](images/lumon/lumon-51.png)

Cracked. Checking the credentials against both Internal and DC01 with CrackMapExec:

![lumon screenshot 52](images/lumon/lumon-52.png)

![lumon screenshot 53](images/lumon/lumon-53.png)

The result looks inconsistent at first glance. Checking BloodHound for `harmonyc`:

![lumon screenshot 54](images/lumon/lumon-54.png)

No outbound object control there either. RDP and WinRM against both Internal and DC01 all fail:

![lumon screenshot 55](images/lumon/lumon-55.png)

The dashboard confirms the credentials are correct, so something else is off, possibly environmental. Restarting the attacking machine to rule out a stale network state (the `/etc/hosts` change earlier had required a NetworkManager restart):

![lumon screenshot 56](images/lumon/lumon-56.png)

Retrying:

![lumon screenshot 57](images/lumon/lumon-57.png)

![lumon screenshot 58](images/lumon/lumon-58.png)

Still inconsistent. Trying `smbmap` for a different view:

![lumon screenshot 59](images/lumon/lumon-59.png)

![lumon screenshot 60](images/lumon/lumon-60.png)

![lumon screenshot 61](images/lumon/lumon-61.png)

Listing everything manually instead:

![lumon screenshot 62](images/lumon/lumon-62.png)

Nothing especially useful there.

![lumon screenshot 63](images/lumon/lumon-63.png)

BloodHound confirms `harmonyc` is a member of the Administration group. Rechecking the intranet panel now shows Admin Panel access.

![lumon screenshot 64](images/lumon/lumon-64.png)

![lumon screenshot 65](images/lumon/lumon-65.png)

![lumon screenshot 66](images/lumon/lumon-66.png)

The account may be locked; trying to unlock it:

![lumon screenshot 67](images/lumon/lumon-67.png)

![lumon screenshot 68](images/lumon/lumon-68.png)

No change. Continuing in the Admin Panel:

![lumon screenshot 69](images/lumon/lumon-69.png)

Browsing from there:

![lumon screenshot 70](images/lumon/lumon-70.png)

`\\10.0.27.145\ADMIN$`:

![lumon screenshot 71](images/lumon/lumon-71.png)

![lumon screenshot 72](images/lumon/lumon-72.png)

A `Panther` directory looks worth following:

`\\10.0.27.145\ADMIN$\Panther`

![lumon screenshot 73](images/lumon/lumon-73.png)

![lumon screenshot 74](images/lumon/lumon-74.png)

Nothing there. Trying `C$` instead:

`\\10.0.27.145\C$\`

![lumon screenshot 75](images/lumon/lumon-75.png)

![lumon screenshot 76](images/lumon/lumon-76.png)

### NTLM Relay Attempt

From this position, an SMB relay attack with `impacket-ntlmrelayx` looks like a reasonable next step.

First, a base64-encoded PowerShell one-liner:

![lumon screenshot 77](images/lumon/lumon-77.png)

Running `impacket-ntlmrelayx` with `--no-http-server`, `-smb2support`, `-t` for the target, and `-c` for the encoded reverse shell command:

![lumon screenshot 78](images/lumon/lumon-78.png)

Starting Penelope on port 4444:

![lumon screenshot 79](images/lumon/lumon-79.png)

Listing a share in the Admin Panel to trigger the relay:

![lumon screenshot 80](images/lumon/lumon-80.png)

No output. The relay attempt doesn't work as expected here; simply calling a share while Responder is running might still capture an NTLM hash on a future pass.

![lumon screenshot 81](images/lumon/lumon-81.png)

This lab is left here as an open investigation. The NTLM relay path and the "Seoul Annex" lead from the PDF remain unexplored.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
