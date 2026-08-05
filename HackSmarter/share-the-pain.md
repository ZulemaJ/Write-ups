# Share The Pain

**Category:** Active Directory

## Attack Chain

1. No credentials and no HTTP surface; NTLM theft via a writable share captures `bob.ross`'s NTLMv2 hash
2. BloodHound: `bob.ross` has `GenericAll` on `alice.wonderland`, password force-changed
3. A base64 string on Alice's desktop turns out to be just the lab flag, not credentials
4. `netstat` reveals a local MSSQL port; a Ligolo-ng tunnel gets through but authentication keeps failing
5. Switching to `chisel` and adding the missing `-windows-auth` flag fixes the MSSQL login
6. `xp_cmdshell` is enabled from the MSSQL service context, a reverse shell spawned
7. The service account holds impersonation privileges; PrintSpoofer escalates to SYSTEM

## TL;DR

With no credentials and no HTTP surface to work with, a UDP/TCP rescan turned up nothing new, so the way in came from **NTLM theft**: dropping browse-triggered lure files onto a writable SMB share and catching an NTLMv2 hash for `bob.ross` on Responder, cracked with Hashcat. BloodHound showed `bob.ross` held `GenericAll` over `alice.wonderland`, abused to force a password change and gain WinRM access. Alice's desktop held a base64-encoded string that turned out to just be the lab's user flag, not a credential, but `netstat` revealed a locally-bound MSSQL port. Pivoting to it took two attempts: a Ligolo-ng tunnel got the connection through but authentication kept failing, until switching to chisel and adding the missing `-windows-auth` flag on the client made it work. From the resulting `MSSQL` service context, impersonation and `xp_cmdshell` were both available, used to spawn a reverse shell and then escalate to SYSTEM with PrintSpoofer.

## Full Walkthrough

### Enumeration

Nmap:

![share-the-pain screenshot 1](images/share-the-pain/share-the-pain-01.png)

Common domain controller ports, no HTTP open. With no credentials, SMB and RPC are the starting points. Running `enum4linux-ng`:

![share-the-pain screenshot 2](images/share-the-pain/share-the-pain-02.png)

A few shares turn up, worth checking manually with SMBMap for actual permissions. RPC access looks denied at first glance, also worth confirming manually.

`smbmap`:

![share-the-pain screenshot 3](images/share-the-pain/share-the-pain-03.png)

Read/write access on the `Share` share.

![share-the-pain screenshot 4](images/share-the-pain/share-the-pain-04.png)

RPC is indeed inaccessible.

**SMB enumeration:** browsing the `Share` share with `impacket-smbclient`:

![share-the-pain screenshot 5](images/share-the-pain/share-the-pain-05.png)

Empty, but the write access itself is notable. If an HTTP service turned out to be linked to this share, a planted shell could be reached over the web, but the earlier scan showed no HTTP port. Worth double-checking for SNMP (UDP 161) as an alternate route to enumerate users. Running a full TCP scan plus UDP 161:

![share-the-pain screenshot 6](images/share-the-pain/share-the-pain-06.png)

Two more TCP ports appear, nothing useful. Checking UDP 161:

![share-the-pain screenshot 7](images/share-the-pain/share-the-pain-07.png)

Time to change strategy. A version/script scan for anything missed:

![share-the-pain screenshot 8](images/share-the-pain/share-the-pain-08.png)

Still nothing. With the writable share as the only real lead, this looks like a good fit for **NTLM theft**, a technique used previously on a similar OffSec-style box. Reference: [ntlm_theft](https://github.com/Greenwolf/ntlm_theft), which notes it works for phishing scenarios where SMB traffic is allowed out of the target network, or when already inside it.

### NTLM Theft: Gaining Bob's Credentials

Cloning the repository:

![share-the-pain screenshot 9](images/share-the-pain/share-the-pain-09.png)

Installing `xlsxwriter` with `pipx`, a dependency:

![share-the-pain screenshot 10](images/share-the-pain/share-the-pain-10.png)

Starting Responder in another terminal:

![share-the-pain screenshot 11](images/share-the-pain/share-the-pain-11.png)

Running `ntlm_theft.py` to generate the lure files (`-s` is the attacker's IP):

![share-the-pain screenshot 12](images/share-the-pain/share-the-pain-12.png)

Several file types come out of this. The "BROWSE TO FOLDER" variants are the most convenient, since they don't require the target to open anything, just browsing to the folder is enough. To cover both bases, both `lure.lnk` and `desktop.ini` are uploaded to the share:

![share-the-pain screenshot 13](images/share-the-pain/share-the-pain-13.png)

Checking Responder:

![share-the-pain screenshot 14](images/share-the-pain/share-the-pain-14.png)

An NTLMv2 hash for `bob.ross` comes through. Cracking it with Hashcat:

![share-the-pain screenshot 15](images/share-the-pain/share-the-pain-15.png)

![share-the-pain screenshot 16](images/share-the-pain/share-the-pain-16.png)

```
137Password123!@#
```

With credentials in hand, the next checks are share permissions, RDP, WinRM, and RPC.

**SMB enumeration:** share permissions with CrackMapExec:

![share-the-pain screenshot 17](images/share-the-pain/share-the-pain-17.png)

Read access on SYSVOL, worth noting. Checking it:

![share-the-pain screenshot 18](images/share-the-pain/share-the-pain-18.png)

Nothing there. RDP fails, WinRM fails. **RPC enumeration:**

![share-the-pain screenshot 19](images/share-the-pain/share-the-pain-19.png)

A user list comes back. Querying each with `queryuser` for possible info leaks:

![share-the-pain screenshot 20](images/share-the-pain/share-the-pain-20.png)

No leaks, but the user list is saved for later. With a credentialed account but no direct access, the next moves are BloodHound and kerberoasting.

**Kerberoasting** with `impacket-getUserSPNs`:

![share-the-pain screenshot 21](images/share-the-pain/share-the-pain-21.png)

Nothing comes of it.

**BloodHound**, running `bloodhound-ce-python`:

![share-the-pain screenshot 22](images/share-the-pain/share-the-pain-22.png)

Analyzing `bob.ross`'s outbound object control:

![share-the-pain screenshot 23](images/share-the-pain/share-the-pain-23.png)

`bob.ross` holds `GenericAll` over `alice.wonderland`, enough to change her password directly. BloodHound's Linux Abuse tab shows the commands needed:

![share-the-pain screenshot 24](images/share-the-pain/share-the-pain-24.png)

### Force Change Password

Generating a hosts file with `nxc smb` and adding it to `/etc/hosts`:

![share-the-pain screenshot 25](images/share-the-pain/share-the-pain-25.png)

![share-the-pain screenshot 26](images/share-the-pain/share-the-pain-26.png)

Changing the password with `net rpc password`:

![share-the-pain screenshot 27](images/share-the-pain/share-the-pain-27.png)

No output, which means success.

### Access as alice.wonderland

Rechecking SMB permissions to confirm the password change, then RDP and WinRM:

![share-the-pain screenshot 28](images/share-the-pain/share-the-pain-28.png)

Password change confirmed. RDP fails, but WinRM via `evil-winrm` works:

![share-the-pain screenshot 29](images/share-the-pain/share-the-pain-29.png)

Access confirmed. Marking both users as owned in BloodHound to check for a shortest path from owned objects doesn't turn up anything, `alice` has no outbound object control of her own.

### Alice Enumeration

Checking privileges with `whoami /all`:

![share-the-pain screenshot 30](images/share-the-pain/share-the-pain-30.png)

Nothing interesting. Checking directories:

![share-the-pain screenshot 31](images/share-the-pain/share-the-pain-31.png)

A `SQL2019` reference looks promising:

![share-the-pain screenshot 32](images/share-the-pain/share-the-pain-32.png)

Access denied. Checking PowerShell history:

![share-the-pain screenshot 33](images/share-the-pain/share-the-pain-33.png)

No PowerShell folder even exists on manual inspection. Checking the Desktop:

![share-the-pain screenshot 34](images/share-the-pain/share-the-pain-34.png)

A base64-encoded string turns up, possibly credentials. Decoding it:

![share-the-pain screenshot 35](images/share-the-pain/share-the-pain-35.png)

Testing whether it's `tyler.ramsey`'s password:

![share-the-pain screenshot 36](images/share-the-pain/share-the-pain-36.png)

It isn't. It turns out to just be the lab's user flag, not a credential. Running WinPEAS turns up nothing interesting either.

Trying PowerUp's `Invoke-AllChecks` for possible hijacking opportunities:

![share-the-pain screenshot 37](images/share-the-pain/share-the-pain-37.png)

Nothing there either. Worth checking for internal ports that might have been missed:

```
netstat -ant -p tcp
```

![share-the-pain screenshot 38](images/share-the-pain/share-the-pain-38.png)

MSSQL, port 1433, bound locally. The plan is to pivot into it with a tunnel and reach it from Kali.

### Port Forwarding

Starting the Ligolo-ng proxy on Kali:

![share-the-pain screenshot 39](images/share-the-pain/share-the-pain-39.png)

Downloading and transferring the Ligolo agent to the target, then connecting with `-ignore-cert`:

![share-the-pain screenshot 40](images/share-the-pain/share-the-pain-40.png)

![share-the-pain screenshot 41](images/share-the-pain/share-the-pain-41.png)

Forwarding port 1433:

![share-the-pain screenshot 42](images/share-the-pain/share-the-pain-42.png)

![share-the-pain screenshot 43](images/share-the-pain/share-the-pain-43.png)

Checking connectivity with `nc`:

![share-the-pain screenshot 44](images/share-the-pain/share-the-pain-44.png)

Attempting to log in:

![share-the-pain screenshot 45](images/share-the-pain/share-the-pain-45.png)

![share-the-pain screenshot 46](images/share-the-pain/share-the-pain-46.png)

Login fails. Without SQL-specific credentials or a config file to lean on, and `tyler_ramsey` as an untested lateral move, this angle stalls for a while. Switching the forwarding tool looks worth trying.

### Accessing MSSQL

Trying chisel instead. Downloading and transferring it:

![share-the-pain screenshot 47](images/share-the-pain/share-the-pain-47.png)

Starting the chisel server on Kali:

![share-the-pain screenshot 48](images/share-the-pain/share-the-pain-48.png)

Running the chisel client on the Windows target:

![share-the-pain screenshot 49](images/share-the-pain/share-the-pain-49.png)

![share-the-pain screenshot 50](images/share-the-pain/share-the-pain-50.png)

![share-the-pain screenshot 51](images/share-the-pain/share-the-pain-51.png)

Trying Kerberos authentication for the SQL connection instead, requesting a ticket:

![share-the-pain screenshot 52](images/share-the-pain/share-the-pain-52.png)

![share-the-pain screenshot 53](images/share-the-pain/share-the-pain-53.png)

![share-the-pain screenshot 54](images/share-the-pain/share-the-pain-54.png)

The actual fix turns out to be much simpler: the client just needed the `-windows-auth` flag.

![share-the-pain screenshot 55](images/share-the-pain/share-the-pain-55.png)

That was the missing piece the whole time. Access confirmed.

### Access as service\MSSQL

Enumerating the database to see whether a shell is reachable from here:

![share-the-pain screenshot 56](images/share-the-pain/share-the-pain-56.png)

Nothing immediately. Checking for impersonation options:

![share-the-pain screenshot 57](images/share-the-pain/share-the-pain-57.png)

Checking whether `xp_cmdshell` can be enabled:

![share-the-pain screenshot 58](images/share-the-pain/share-the-pain-58.png)

![share-the-pain screenshot 59](images/share-the-pain/share-the-pain-59.png)

It works. Spawning a reverse shell with a PowerShell one-liner: encoding it in base64,

![share-the-pain screenshot 60](images/share-the-pain/share-the-pain-60.png)

setting up a Penelope listener,

![share-the-pain screenshot 61](images/share-the-pain/share-the-pain-61.png)

and running the payload through `xp_cmdshell`:

![share-the-pain screenshot 62](images/share-the-pain/share-the-pain-62.png)

![share-the-pain screenshot 63](images/share-the-pain/share-the-pain-63.png)

Shell access confirmed.

### Privilege Escalation

Checking the service account's privileges:

![share-the-pain screenshot 64](images/share-the-pain/share-the-pain-64.png)

Impersonation privileges are present, exploitable with a Potato-family tool or PrintSpoofer. Using PrintSpoofer, after transferring netcat:

![share-the-pain screenshot 65](images/share-the-pain/share-the-pain-65.png)

![share-the-pain screenshot 66](images/share-the-pain/share-the-pain-66.png)

SYSTEM shell obtained.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
