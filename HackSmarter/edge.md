# Edge

**Category:** Active Directory, Kiosk Breakout

## Attack Chain

1. A writable SMB share is found; a trojanized script, ASREPRoasting, and NTLM theft are all tried against it and fail
2. The original starting credentials turn out to work directly over RDP and WinRM
3. Standard credential-hunting tools are blocked by AV
4. EdgeSnapper extracts a cleartext credential (`svc_vdi`) from Microsoft Edge's process memory
5. RDP as `svc_vdi` lands in a locked-down Kiosk-mode session
6. Downloading and opening `cmd.exe` through Edge itself breaks out of the Kiosk restriction
7. A PowerShell reverse shell is spawned; a `putty.conf` file leaks `svc_vdi_mgmt` credentials
8. `svc_vdi_mgmt` has full administrative access, confirmed over WinRM

## TL;DR

Starting from a known credential, a writable SMB share looked like an obvious path to NTLM relay or script poisoning, but three separate attempts (a trojanized scheduled script, ASREPRoasting against a preauth-disabled account, and an NTLM theft file drop) all came up empty. The actual way in was much simpler: the starting credentials worked directly over RDP and WinRM, no further exploitation needed. From that foothold, standard credential-hunting tools were blocked by AV, but **EdgeSnapper**, a tool built specifically to pull cleartext credentials out of Microsoft Edge's process memory, recovered a password for `svc_vdi`. That account landed in a locked-down Kiosk-mode RDP session, escaped by downloading and opening `cmd.exe` through the Edge browser itself. From that shell, a `putty.conf` file leaked credentials for `svc_vdi_mgmt`, which turned out to have full administrative access to the machine.

## Full Walkthrough

### Starting Credentials

```
Username: jmorris
Password: Fabricat!on2024
```

### Enumeration

Nmap:

![edge screenshot 1](images/edge/edge-01.png)

**SMB enumeration** with `enum4linux-ng`:

![edge screenshot 2](images/edge/edge-02.png)

![edge screenshot 3](images/edge/edge-03.png)

![edge screenshot 4](images/edge/edge-04.png)

No RPC connection possible. Checking with CrackMapExec instead:

![edge screenshot 5](images/edge/edge-05.png)

Write permissions turn up on the `VantaraOps` share, a possible lead toward poisoning or NTLM theft. Checking the share itself:

![edge screenshot 6](images/edge/edge-06.png)

![edge screenshot 7](images/edge/edge-07.png)

Some scripts stand out:

![edge screenshot 8](images/edge/edge-08.png)

### Attempt 1: Trojanizing a Scheduled Script

Since these look like automated scripts, modifying one to add a PowerShell one-liner reverse shell seems promising.

Downloading the file:

![edge screenshot 9](images/edge/edge-09.png)

Crafting a PowerShell payload and base64-encoding it:

![edge screenshot 10](images/edge/edge-10.png)

Appending it to the script with `powershell -enc <command>`:

![edge screenshot 11](images/edge/edge-11.png)

Starting a Penelope listener:

![edge screenshot 12](images/edge/edge-12.png)

Uploading the modified `.bat` back to the SMB share and waiting:

![edge screenshot 13](images/edge/edge-13.png)

Nothing triggers. This script isn't actually run automatically.

### Attempt 2: ASREPRoasting

Checking the other files on the share:

![edge screenshot 14](images/edge/edge-14.png)

One account has Kerberos preauth disabled, a candidate for ASREPRoasting:

![edge screenshot 15](images/edge/edge-15.png)

SNMP also looks open:

![edge screenshot 16](images/edge/edge-16.png)

Filtered, as it turns out:

![edge screenshot 17](images/edge/edge-17.png)

A list of users is still recovered from SMB:

![edge screenshot 18](images/edge/edge-18.png)

Building a user list from it:

![edge screenshot 19](images/edge/edge-19.png)

Checking other services along the way:

![edge screenshot 20](images/edge/edge-20.png)

With plenty of information gathered from SMB, the next move is ASREPRoasting against the user list, particularly the preauth-disabled `asrep_svc` account. First, updating `/etc/hosts`:

![edge screenshot 21](images/edge/edge-21.png)

This doesn't lead anywhere either.

### Attempt 3: NTLM Theft

Starting a Responder listener:

![edge screenshot 22](images/edge/edge-22.png)

Cloning [ntlm_theft](https://github.com/Greenwolf/ntlm_theft):

![edge screenshot 23](images/edge/edge-23.png)

Running it to generate bait files:

![edge screenshot 24](images/edge/edge-24.png)

Dropping the generated files onto the SMB share:

![edge screenshot 25](images/edge/edge-25.png)

Waiting on Responder:

![edge screenshot 26](images/edge/edge-26.png)

No output. Not the right path either.

### The Simple Path: Direct Access

After three failed exploitation attempts on the share, a much simpler question: do the original credentials just work over WinRM or RDP directly?

![edge screenshot 27](images/edge/edge-27.png)

They do. All three share-based attempts turn out to have been unnecessary detours.

Enumerating from this foothold:

![edge screenshot 28](images/edge/edge-28.png)

A `Microsoft Edge.lnk` shortcut stands out. Following [HackTricks' browser artifacts methodology](https://hacktricks.wiki/en/generic-methodologies-and-resources/basic-forensic-methodology/specific-software-file-type-tricks/browser-artifacts.html) to look for Edge profile data:

![edge screenshot 29](images/edge/edge-29.png)

![edge screenshot 30](images/edge/edge-30.png)

![edge screenshot 31](images/edge/edge-31.png)

![edge screenshot 32](images/edge/edge-32.png)

Nothing in `/Temp`. Enumerating `C:\` instead:

![edge screenshot 33](images/edge/edge-33.png)

A `kiosk` directory raises questions:

![edge screenshot 34](images/edge/edge-34.png)

![edge screenshot 35](images/edge/edge-35.png)

A related process turns up:

![edge screenshot 36](images/edge/edge-36.png)

The host also has quite a few local users. Running `Invoke-AllChecks` from PowerUp, WinPEAS, and considering Rubeus:

![edge screenshot 37](images/edge/edge-37.png)

Antivirus blocks all of them. No PowerShell history to lean on either. Internal ports:

![edge screenshot 38](images/edge/edge-38.png)

Nothing there. BloodHound collection:

![edge screenshot 39](images/edge/edge-39.png)

Blocked as well. Checking running processes:

![edge screenshot 40](images/edge/edge-40.png)

Microsoft Edge is running, fitting given the lab's name. Standard credential and profile extraction from Edge hasn't worked so far, so the question becomes whether a purpose-built tool exists for this.

### Access as svc_vdi

It does: [EdgeSnapper](https://github.com/Dragkob/EdgeSnapper), a security research toolkit focused on analyzing cleartext credential persistence within Microsoft Edge's process memory.

![edge screenshot 41](images/edge/edge-41.png)

With AV active, the Path Beta build is used, compiled on Linux with:

```
x86_64-w64-mingw32-g++ edgeSnapper.cpp -o edgeSnapper.exe -static -static-libgcc -static-libstdc++ -ldbghelp -lpsapi
```

![edge screenshot 42](images/edge/edge-42.png)

Transferring and running it:

![edge screenshot 43](images/edge/edge-43.png)

A credential comes back:

```
svc_vdi : V@ntara
```

Trying RDP with it:

![edge screenshot 44](images/edge/edge-44.png)

Successful, landing in what looks like Kiosk mode. Logging in with the `svc_vdi` credentials:

![edge screenshot 45](images/edge/edge-45.png)

![edge screenshot 46](images/edge/edge-46.png)

Two other users, `agriffin` and `tpham`, show recent logins. Checking the version info:

![edge screenshot 47](images/edge/edge-47.png)

![edge screenshot 48](images/edge/edge-48.png)

A site referencing a released version is visible. Clicking through:

![edge screenshot 49](images/edge/edge-49.png)

A `cmd.exe` download shows up. Opening it directly from Edge:

![edge screenshot 50](images/edge/edge-50.png)

![edge screenshot 51](images/edge/edge-51.png)

![edge screenshot 52](images/edge/edge-52.png)

`cmd.exe` runs as `svc_vdi`, breaking out of the Kiosk restriction.

### Full System Compromise

![edge screenshot 53](images/edge/edge-53.png)

The shell isn't interactive. Crafting a PowerShell one-liner reverse shell instead:

![edge screenshot 54](images/edge/edge-54.png)

![edge screenshot 55](images/edge/edge-55.png)

![edge screenshot 56](images/edge/edge-56.png)

![edge screenshot 57](images/edge/edge-57.png)

Trying to transfer `ncat`:

![edge screenshot 58](images/edge/edge-58.png)

![edge screenshot 59](images/edge/edge-59.png)

Blocked, no output. Listing the filesystem tree instead:

```
tree /f
```

![edge screenshot 60](images/edge/edge-60.png)

A `putty.conf` file turns up in `/Documents`. Reading it:

![edge screenshot 61](images/edge/edge-61.png)

It leaks a username and password:

```
svc_vdi_mgmt : 56tyghbn%^TYGHBN
```

Checking the account:

![edge screenshot 62](images/edge/edge-62.png)

`svc_vdi_mgmt` has full administrative access to the machine. Connecting over WinRM:

![edge screenshot 63](images/edge/edge-63.png)

![edge screenshot 64](images/edge/edge-64.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
