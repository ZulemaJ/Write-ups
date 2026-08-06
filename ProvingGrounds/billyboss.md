# BillyBoss

**Category:** Windows, Sonatype Nexus Repository Manager

## Attack Chain

1. HTTP enumeration finds a BaGet NuGet server on port 80 (no clear lead) and a Sonatype Nexus Repository Manager 3.21.0-5 instance on port 8081
2. Default Nexus credentials fail, as does directory brute-forcing
3. The custom credential pair `nexus:nexus` succeeds
4. A public exploit for this Nexus version is identified and adapted with the target's URL, command, and credentials
5. `nc.exe` is transferred to the target through the exploit, then triggered for a reverse shell as `Nathan`
6. `Nathan` holds `SeImpersonatePrivilege`; PrintSpoofer, SigmaPotato, and SweetPotato all fail
7. GodPotato, matched to the installed .NET Framework version (v4), succeeds and spawns `nc.exe` back as SYSTEM

## TL;DR

A Sonatype Nexus Repository Manager instance on port 8081 stood out immediately as the likely entry point, but neither default Nexus credentials nor directory brute-forcing got anywhere. The credential pair that worked, `nexus:nexus`, was a guess based on the product name itself. Nexus 3.21.0-5 has a known public exploit, adapted here to first transfer `nc.exe` to the target and then trigger a reverse shell, landing as user `Nathan`. Nathan held `SeImpersonatePrivilege`, but three of the usual Potato-family tools (PrintSpoofer, SigmaPotato, SweetPotato) all failed against this target. GodPotato, matched to the specific .NET Framework version installed (v4), worked, spawning a SYSTEM-level `nc.exe` callback.

## Full Walkthrough

### Enumeration

Nmap:

![billyboss screenshot 1](images/billyboss/billyboss-01.png)

Version/script scan:

![billyboss screenshot 2](images/billyboss/billyboss-02.png)

Ports of interest: FTP, 80, 445 (SMB), 8081. SMB enumeration brings no clues, and FTP doesn't allow anonymous access.

**HTTP enumeration, port 80:**

![billyboss screenshot 3](images/billyboss/billyboss-03.png)

Running BaGet, a lightweight NuGet and symbol server. Checking the "Upload" section:

![billyboss screenshot 4](images/billyboss/billyboss-04.png)

An API endpoint is referenced. Checking it:

![billyboss screenshot 5](images/billyboss/billyboss-05.png)

Nothing particularly interesting here.

**Port 8081:**

![billyboss screenshot 6](images/billyboss/billyboss-06.png)

Sonatype Nexus Repository Manager 3.21.0-5, a strong candidate for the entry point.

### Gaining Access to Nexus

Trying to sign in or find accessible directories, starting with default credentials:

![billyboss screenshot 7](images/billyboss/billyboss-07.png)

![billyboss screenshot 8](images/billyboss/billyboss-08.png)

![billyboss screenshot 9](images/billyboss/billyboss-09.png)

None of the default credential pairs work. Trying directory brute-forcing instead:

![billyboss screenshot 10](images/billyboss/billyboss-10.png)

Nothing there either. After a while spent going in circles, a simpler guess pays off: since the product itself is Nexus, why not try `nexus:nexus`?

![billyboss screenshot 11](images/billyboss/billyboss-11.png)

```
nexus:nexus
```

![billyboss screenshot 12](images/billyboss/billyboss-12.png)

Access confirmed.

![billyboss screenshot 13](images/billyboss/billyboss-13.png)

### Initial Access

With authenticated access to the Sonatype server, the next goal is a full interactive reverse shell. Searching for a matching exploit:

![billyboss screenshot 14](images/billyboss/billyboss-14.png)

Reviewing what the exploit does:

![billyboss screenshot 15](images/billyboss/billyboss-15.png)

A good match. Adjusting its parameters (target URL, command, username, password):

![billyboss screenshot 16](images/billyboss/billyboss-16.png)

Using it first to transfer `nc.exe` onto the server:

![billyboss screenshot 17](images/billyboss/billyboss-17.png)

Transfer confirmed. Reconfiguring the exploit's parameters to spawn a reverse shell instead:

![billyboss screenshot 18](images/billyboss/billyboss-18.png)

Launching it:

![billyboss screenshot 19](images/billyboss/billyboss-19.png)

Checking the listener:

![billyboss screenshot 20](images/billyboss/billyboss-20.png)

Shell obtained as user `Nathan`.

### Privilege Escalation

Checking Nathan's privileges:

![billyboss screenshot 21](images/billyboss/billyboss-21.png)

`SeImpersonatePrivilege` is present. Trying PrintSpoofer, SigmaPotato, and SweetPotato in turn, none of them succeed against this target. Researching further turns up a Potato variant not tried yet: GodPotato.

![billyboss screenshot 22](images/billyboss/billyboss-22.png)

GodPotato ships different releases matched to the .NET Framework version installed on the target, so the right build needs to be identified first. Finding guidance on how to check it:

![billyboss screenshot 23](images/billyboss/billyboss-23.png)

Checking the .NET version:

![billyboss screenshot 24](images/billyboss/billyboss-24.png)

Version 4 is installed. Downloading the matching GodPotato build, transferring it, and running it with no flags to see the usage menu:

![billyboss screenshot 25](images/billyboss/billyboss-25.png)

With the syntax confirmed, targeting the already-transferred `nc.exe` on `/Users/Nathan/Desktop` to connect back as SYSTEM:

![billyboss screenshot 26](images/billyboss/billyboss-26.png)

Checking the listener:

![billyboss screenshot 27](images/billyboss/billyboss-27.png)

![billyboss screenshot 28](images/billyboss/billyboss-28.png)

`whoami` doesn't return visible output in this shell, but SYSTEM-level access is confirmed by successfully listing the Administrator's directories:

![billyboss screenshot 29](images/billyboss/billyboss-29.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
