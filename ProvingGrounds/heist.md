# Heist

**Difficulty:** Hard  
**Category:** Active Directory, Server-Side Request Forgery (SSRF)

Server-side request forgery is a web security vulnerability that allows an attacker to cause the server-side application to make requests to an unintended location. In a typical SSRF attack, the attacker might cause the server to connect to internal-only services within the organization's infrastructure, or force it to connect to arbitrary external systems, potentially leaking sensitive data such as authorization credentials.

## Attack Chain

1. A Werkzeug/Python web app on port 8080 accepts a URL parameter, a likely SSRF candidate
2. Pointing it at a local Python HTTP server confirms the app fetches and displays external content, SSRF confirmed
3. Pointing it at Responder (listening on `tun0`) instead of a real HTTP server captures an NTLMv2 hash for user `enox`, since the "URL fetch" is really an SMB/HTTP-style connection attempt Responder can intercept
4. The hash is cracked with John, and the resulting password grants WinRM access
5. Manual enumeration goes nowhere; SharpHound shows `enox` is a member of Web Admins, which can read the GMSA password of `svc_apache$`
6. The Linux GMSA-reading path fails; a compiled Windows tool (`GMSAPasswordReader.exe`) succeeds and returns `svc_apache$`'s hash, unlocking WinRM as that account
7. `svc_apache$` holds `SeRestorePrivilege`; the `Invoke-SeRestoreAbuse` PowerShell script is tested and confirmed to work
8. The script is run to spawn a SYSTEM-level reverse shell via a transferred `nc.exe`

## TL;DR

A Flask/Werkzeug app accepting a URL parameter turned out to be a straightforward SSRF: pointing it at a local Python HTTP server showed the app fetching and rendering external content directly. Pointing that same parameter at a local Responder instance instead of a real server turned the SSRF into an NTLM relay opportunity, capturing an NTLMv2 hash for user `enox`, cracked with John. The resulting credentials gave WinRM access, and while manual enumeration didn't lead anywhere, SharpHound showed `enox`'s membership in Web Admins granted rights to read the GMSA password of a service account, `svc_apache$`. The Linux tooling for that particular abuse didn't cooperate, but a precompiled Windows reader tool did, handing over `svc_apache$`'s hash and WinRM access as that account. From there, `SeRestorePrivilege` was the final piece: a public abuse script for that privilege, tested first as a dry run, spawned a SYSTEM-level reverse shell once confirmed working.

## Full Walkthrough

### Enumeration

Nmap shows the open ports of a domain controller, including port 8080:

```
Werkzeug/2.0.1 Python/3.9.0
```

![heist screenshot 1](images/heist/heist-01.png)

The app accepts a URL input, worth testing for SSRF.

### Checking for SSRF

Setting up a Python HTTP server on port 445 and pointing the app's URL field at the attacking host's IP on that port:

![heist screenshot 2](images/heist/heist-02.png)

![heist screenshot 3](images/heist/heist-03.png)

The response displays the contents served by the local HTTP server, confirming the application is vulnerable to SSRF.

### Retrieving an NTLMv2 Hash via Responder

Starting Responder:

```
sudo responder -I tun0 -vv
```

Pointing the vulnerable URL field at a closed port on the attacking machine instead of a real server:

![heist screenshot 4](images/heist/heist-04.png)

![heist screenshot 5](images/heist/heist-05.png)

Responder captures a username and NTLMv2 hash for user `enox`.

### Initial Access

Cracking the password with John:

![heist screenshot 6](images/heist/heist-06.png)

Once cracked, trying WinRM:

![heist screenshot 7](images/heist/heist-07.png)

Access confirmed.

### Lateral Movement

Manual enumeration doesn't turn up anything useful, so SharpHound is run against the target instead. It confirms lateral movement should target `svc_apache$`:

![heist screenshot 8](images/heist/heist-08.png)

`enox` is a member of the Web Admins group, which grants the ability to read the GMSA password of `svc_apache$`:

![heist screenshot 9](images/heist/heist-09.png)

Checking BloodHound's information panel on `ReadGMSAPassword` for exploitation guidance on both Linux and Windows:

![heist screenshot 10](images/heist/heist-10.png)

Trying to abuse the GMSA password from Linux doesn't work:

![heist screenshot 11](images/heist/heist-11.png)

`enox`'s Web Admins membership is confirmed correct, so the Windows-side abuse path is tried instead. Downloading a precompiled [GMSAPasswordReader.exe](https://github.com/expl0itabl3/Toolies), transferring it, and running it:

![heist screenshot 12](images/heist/heist-12.png)

The hash for `svc_apache$` comes back. Trying WinRM with it:

![heist screenshot 13](images/heist/heist-13.png)

Access confirmed.

### Privilege Escalation

Enumerating `svc_apache$`'s privileges:

![heist screenshot 14](images/heist/heist-14.png)

`SeRestorePrivilege` is present. Researching abuse techniques turns up [Invoke-SeRestoreAbuse](https://github.com/0x4D-5A/Invoke-SeRestoreAbuse), a PowerShell script built for exactly this privilege.

Downloading, transferring, and running a test invocation to confirm it works:

![heist screenshot 15](images/heist/heist-15.png)

Confirmed working. Spawning a reverse shell with `NT AUTHORITY\SYSTEM` privileges: transferring `nc.exe`, setting up a listener, and running the script with the appropriate command:

![heist screenshot 16](images/heist/heist-16.png)

![heist screenshot 17](images/heist/heist-17.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
