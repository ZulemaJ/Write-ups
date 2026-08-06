# Jacko

**Category:** Windows, H2 Database, DLL Hijacking

## Attack Chain

1. A full port scan finds an H2 Database Console on port 8082 alongside a generic H2 landing page on port 80
2. A public JNDI-based unauthenticated RCE writeup for H2 Console is tried but doesn't work as documented
3. Searchsploit turns up a working alternative: an SQL-query-based exploit that writes a native library, loads it, and evaluates arbitrary system commands
4. Following the exploit gets RCE as user `Tony`
5. `nc.exe` is transferred (blocked by AV until the output path is changed to `Tony`'s own folder) for an interactive reverse shell
6. `PATH` is missing entirely in the shell and has to be set manually before basic commands like `whoami` work
7. Manual enumeration finds "PaperStream IP" software, version 1.42, with a known DLL hijacking privilege escalation exploit
8. A malicious DLL is built with `msfvenom`, and the PowerShell-based exploit script is run (requiring PowerShell 7 to be transferred first, since it's absent on the target)
9. The first payload attempt fails; reverting the machine and switching the payload's callback port succeeds

## TL;DR

An H2 Database Console exposed on port 8082 was the clear target, but the well-documented JNDI-based RCE technique for it didn't work against this instance. A different, SQL-query-based exploit found via searchsploit did: it writes a native library through H2's SQL interface, loads it, and evaluates arbitrary commands, yielding RCE as user `Tony`. Getting a proper interactive shell needed some troubleshooting: AV blocked `nc.exe` until it was written into Tony's own folder, and once connected, `PATH` wasn't set at all, breaking even `whoami` until fixed manually. Manual folder enumeration then turned up "PaperStream IP" version 1.42, vulnerable to a known DLL hijacking privilege escalation. Building a malicious DLL with `msfvenom` and running the matching PowerShell exploit (after transferring PowerShell 7 itself, since the target had none installed) got a SYSTEM callback, though only after reverting the machine and switching the payload's listening port on a second attempt.

## Full Walkthrough

### Enumeration

Nmap:

![jacko screenshot 1](images/jacko/jacko-01.png)

A full port scan reveals more:

![jacko screenshot 2](images/jacko/jacko-02.png)

Ports of interest: 80, 5040, 7680, 8082, 9092, and 445 (SMB). SMB enumeration gives no hints. Service/script scan on the most promising ports:

![jacko screenshot 3](images/jacko/jacko-03.png)

Ports 80 and 8082 stand out as the main focus.

**HTTP enumeration, port 80:**

![jacko screenshot 4](images/jacko/jacko-04.png)

An H2 Database Engine landing page. Port 8082:

![jacko screenshot 5](images/jacko/jacko-05.png)

The H2 Database Console, allowing SQL queries once connected. Port 9092:

![jacko screenshot 6](images/jacko/jacko-06.png)

Nothing there.

### Focus on H2 Database Console

Researching known H2 Console vulnerabilities turns up a [JNDI-based unauthenticated RCE writeup](https://jfrog.com/blog/the-jndi-strikes-back-unauthenticated-rce-in-h2-database-console/):

![jacko screenshot 7](images/jacko/jacko-07.png)

The technique is fairly involved, dealing with specific Java driver class and JDBC URL parameters. Following it as closely as possible, modifying the driver class and JDBC URL, doesn't produce results here. It does confirm the underlying issue class though: a JNDI-related vulnerability in the H2 Console.

Checking `searchsploit` for other options:

![jacko screenshot 8](images/jacko/jacko-08.png)

A promising alternative. Reviewing it:

![jacko screenshot 9](images/jacko/jacko-09.png)

This exploit runs a sequence of SQL queries that write a native library, load it, and evaluate a script with embedded system commands, a different mechanism from the JNDI approach, and one that looks directly usable here.

### Initial Access

Following the exploit step by step.

Writing the native library:

![jacko screenshot 10](images/jacko/jacko-10.png)

No errors. Loading the library:

![jacko screenshot 11](images/jacko/jacko-11.png)

No errors. Evaluating the script with a `whoami` test command:

![jacko screenshot 12](images/jacko/jacko-12.png)

RCE confirmed, running as user `Tony`.

![jacko screenshot 13](images/jacko/jacko-13.png)

Tony holds `SeImpersonatePrivilege`, worth remembering for later. The next step is transferring `nc.exe` to get an interactive reverse shell.

Transferring it with `curl`:

![jacko screenshot 14](images/jacko/jacko-14.png)

![jacko screenshot 15](images/jacko/jacko-15.png)

Despite an error message, the transfer appears to complete, but running `nc.exe` from the working directory doesn't connect back, likely AV blocking execution from that location. Changing the output path to `/Users/Tony/nc.exe` instead:

![jacko screenshot 16](images/jacko/jacko-16.png)

![jacko screenshot 17](images/jacko/jacko-17.png)

This works. Running it to spawn a shell:

![jacko screenshot 18](images/jacko/jacko-18.png)

![jacko screenshot 19](images/jacko/jacko-19.png)

Connection established.

### Enumeration

Running `whoami`:

![jacko screenshot 20](images/jacko/jacko-20.png)

Not recognized as a command. Research points to a missing `PATH` environment variable. Setting it manually:

```
set PATH=%SystemRoot%\system32;%SystemRoot%;
```

![jacko screenshot 21](images/jacko/jacko-21.png)

![jacko screenshot 22](images/jacko/jacko-22.png)

That resolves it. Inspecting directories on the host turns up "PaperStream IP", worth a closer look:

![jacko screenshot 23](images/jacko/jacko-23.png)

Reading `READMEENU.RTF` for the version:

![jacko screenshot 24](images/jacko/jacko-24.png)

PaperStream IP driver 1.42.

### Checking Exploits for PaperStream

Learning a bit about what PaperStream is:

![jacko screenshot 25](images/jacko/jacko-25.png)

Searching for a matching exploit:

![jacko screenshot 26](images/jacko/jacko-26.png)

A privilege escalation exploit, exactly what's needed.

![jacko screenshot 27](images/jacko/jacko-27.png)

The exploit targets a DLL hijacking vulnerability, delivered as a PowerShell (`.ps1`) script.

### Privilege Escalation

Following the exploit's steps.

Creating the malicious payload with `msfvenom -dll`:

![jacko screenshot 28](images/jacko/jacko-28.png)

Updating the exploit script to point at the payload's path on the target, `/Users/Tony/Desktop`:

![jacko screenshot 29](images/jacko/jacko-29.png)

Transferring the `.dll`:

![jacko screenshot 30](images/jacko/jacko-30.png)

Transferring `exploit.ps1`:

![jacko screenshot 31](images/jacko/jacko-31.png)

At this point, PowerShell turns out to be entirely absent on the remote system. Downloading the PowerShell 7 zip release and transferring it over:

![jacko screenshot 32](images/jacko/jacko-32.png)

Unzipping it:

![jacko screenshot 33](images/jacko/jacko-33.png)

Running PowerShell via `pwsh.exe` (the modern entry point, since `powershell.exe` is deprecated in version 7):

![jacko screenshot 34](images/jacko/jacko-34.png)

Running the exploit:

![jacko screenshot 35](images/jacko/jacko-35.png)

The first attempt doesn't produce a callback. Reverting the machine and repeating the same steps with the payload's listening port changed to 8082 resolves it.

Checking the listener:

![jacko screenshot 36](images/jacko/jacko-36.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
