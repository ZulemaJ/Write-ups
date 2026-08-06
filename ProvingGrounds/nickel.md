# Nickel

**Category:** Windows, REST API Abuse

## Attack Chain

1. A DevOps dashboard on port 8089 and an "invalid token" API on port 33333 are found; direct requests to the dashboard's IP-bound endpoints fail
2. Intercepting a request in Burp reveals it must be sent as `POST` (not `GET`) with a `Content-Length` header, following the app's own source-visible request patterns
3. The `list-running-procs` endpoint leaks a plaintext-looking password tied to user `ariah` for SSH
4. SSH, CrackMapExec, and FTP all reject the credential as-is; HashID identifies it as base64-encoded, and decoding it produces a working SSH password
5. An `Infrastructure.pdf` found in the FTP directory is password-protected; `pdf2john` and John crack it
6. The PDF documents an internal API endpoint on port 80, not exposed externally but reachable via SSH local port forwarding
7. The forwarded dev-API rejects a guessed parameter format; adding a `?` before the parameter, per the PDF's documented syntax, unlocks command execution as `NT AUTHORITY\SYSTEM`
8. `nc.exe` is transferred via SSH and triggered through the API (its command URL-encoded via CyberChef) for a SYSTEM reverse shell

## TL;DR

A DevOps dashboard exposed a REST API that only worked when called with the exact HTTP method, headers, and IP it expected, discovered by reading the page's own source and confirming the details in Burp. One of its endpoints, listing running processes, leaked what looked like a plaintext SSH password for user `ariah`, but every login attempt failed until HashID revealed it was actually base64-encoded; decoding it fixed the login. From there, a password-protected `Infrastructure.pdf` on the FTP share was cracked with `pdf2john` and John, and documented an internal-only API on port 80. SSH local port forwarding exposed that API locally, and once the exact parameter syntax it expected (a leading `?`) was matched, it granted command execution running as `NT AUTHORITY\SYSTEM` directly, no further privilege escalation needed, just a reverse shell to capture it interactively.

## Full Walkthrough

### Enumeration

Nmap:

![nickel screenshot 1](images/nickel/nickel-01.png)

![nickel screenshot 2](images/nickel/nickel-02.png)

![nickel screenshot 3](images/nickel/nickel-03.png)

Ports of interest: 445, 21, 5040, 8089, 33333.

**SMB enumeration:**

![nickel screenshot 4](images/nickel/nickel-04.png)

![nickel screenshot 5](images/nickel/nickel-05.png)

Not much beyond the OS and domain name (`nickel`).

**HTTP enumeration:** port 8089 hosts a DevOps dashboard.

![nickel screenshot 6](images/nickel/nickel-06.png)

Running some commands through it:

![nickel screenshot 7](images/nickel/nickel-07.png)

It fails to connect to an IP address that isn't the one actually assigned to this box. Checking port 33333:

![nickel screenshot 8](images/nickel/nickel-08.png)

"Invalid token." Inspecting the source of the port 8089 page shows the various `GET`-based requests it makes internally:

![nickel screenshot 9](images/nickel/nickel-09.png)

Trying one directly with `curl`:

![nickel screenshot 10](images/nickel/nickel-10.png)

Can't reach that IP. Substituting the actual target IP instead:

![nickel screenshot 11](images/nickel/nickel-11.png)

"Method GET is not allowed."

### Working Out the API's Requirements

Intercepting the request in Burp:

![nickel screenshot 12](images/nickel/nickel-12.png)

Changing the method to `POST` and the `Host` header to the target IP:

![nickel screenshot 13](images/nickel/nickel-13.png)

Still not quite right. Trying `curl -X POST` instead:

![nickel screenshot 14](images/nickel/nickel-14.png)

A `411 Length Required` error. Research points to a missing `Content-Length` header:

![nickel screenshot 15](images/nickel/nickel-15.png)

Adding it explicitly:

![nickel screenshot 16](images/nickel/nickel-16.png)

"Not implemented" for this particular request. Trying a different documented request, `list-running-procs`:

![nickel screenshot 17](images/nickel/nickel-17.png)

![nickel screenshot 18](images/nickel/nickel-18.png)

A list of running processes comes back, and one entry stands out:

![nickel screenshot 19](images/nickel/nickel-19.png)

A password for user `ariah`, apparently meant for SSH.

### Initial Access

Trying SSH with `ariah`'s credentials:

![nickel screenshot 20](images/nickel/nickel-20.png)

Fails. Checking with CrackMapExec:

![nickel screenshot 21](images/nickel/nickel-21.png)

Logon failure. Trying FTP:

![nickel screenshot 22](images/nickel/nickel-22.png)

Also fails. Worth checking whether the password is actually encoded. Running it through HashID:

![nickel screenshot 23](images/nickel/nickel-23.png)

Base64-encoded. Decoding it:

![nickel screenshot 24](images/nickel/nickel-24.png)

Trying SSH again with the decoded password:

![nickel screenshot 25](images/nickel/nickel-25.png)

Access confirmed.

### Enumeration

Browsing directories, an `Infrastructure.pdf` turns up in `/ftp`:

![nickel screenshot 26](images/nickel/nickel-26.png)

Copying it over via a local `impacket-smbserver` share. Opening it prompts for a password:

![nickel screenshot 27](images/nickel/nickel-27.png)

The already-known credential doesn't unlock it.

**Extracting and cracking the PDF password:** extracting the hash with `pdf2john`:

![nickel screenshot 28](images/nickel/nickel-28.png)

Cracking it with John:

![nickel screenshot 29](images/nickel/nickel-29.png)

Password recovered. Opening the PDF:

![nickel screenshot 30](images/nickel/nickel-30.png)

It documents several reachable endpoints, the first apparently running on port 80. The earlier scan showed no port 80 exposed externally, worth checking whether it's bound locally instead. Checking with `netstat -ano`:

![nickel screenshot 31](images/nickel/nickel-31.png)

Port 80 is indeed listening locally. Forwarding it via SSH local port forwarding:

![nickel screenshot 32](images/nickel/nickel-32.png)

Accessing it from Kali:

![nickel screenshot 33](images/nickel/nickel-33.png)

A dev-API, worth pushing further into. Trying to add an `/id` parameter:

![nickel screenshot 34](images/nickel/nickel-34.png)

"Incorrect parameter," meaning something specific is expected. Per the PDF, the format should be `http://nickel/?...`. Trying with a leading `?` before the parameter:

![nickel screenshot 35](images/nickel/nickel-35.png)

That works, and the response confirms execution as `NT AUTHORITY\SYSTEM`.

### Privilege Escalation

With command execution already running as SYSTEM, the remaining step is turning it into an interactive shell. Transferring `nc.exe` via SSH to `C:\Temp`, then triggering the endpoint to spawn a reverse shell with SYSTEM privileges.

The command needs URL-encoding to work as a query parameter. Using CyberChef:

![nickel screenshot 36](images/nickel/nickel-36.png)

![nickel screenshot 37](images/nickel/nickel-37.png)

Sending the encoded command triggers `nc.exe` from `/Temp`. With a listener running:

![nickel screenshot 38](images/nickel/nickel-38.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
