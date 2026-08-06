# AuthBy

**Category:** Windows, FTP, Web Application

## Attack Chain

1. Anonymous FTP access exposes zFTPServer account export files (`.uac`), including a backup copy of the restricted `admin` export
2. A directory traversal exploit against zFTPServer (CVE-2011-4717) is tried but doesn't yield direct access
3. FTP login as `admin` succeeds with a blank password; its home directory turns out to be the live web root itself
4. Downloaded `.htpasswd`/`.htaccess` show the web app on port 242 is protected by HTTP Basic Auth
5. The Apache `$apr1$` MD5 hash is cracked instantly with Hashcat and rockyou
6. Since the FTP `admin` account already has write access to the web root, a PHP webshell is uploaded via FTP and triggered over HTTP, no need for the cracked credentials at all
7. A standard PHP reverse shell connects but is unstable; a simple command-execution webshell is used instead, followed by a proper interactive shell via a transferred `nc.exe`
8. GodPotato and PrintSpoofer both fail; the target turns out to be an old x86 Windows Server 2008 build
9. The unpatched MS11-046 kernel exploit (`afd.sys` local privilege escalation) is compiled, transferred, and run
10. `nt authority\system` confirmed

## TL;DR

An old zFTPServer instance allowed anonymous FTP access, which leaked account export files but no directly usable credentials. The real way in was simpler: the `admin` FTP account had a blank password, and its home directory was the site's web root on port 242, itself protected only by HTTP Basic Auth. The `.htpasswd` hash pulled from that directory cracked instantly with rockyou, but the FTP access alone was already enough to plant a webshell directly into the web root and get code execution as `livda\apache`. From there, modern privilege escalation tools (GodPotato, PrintSpoofer) failed outright, because the target turned out to be a very old x86 build of Windows Server 2008 SP1. The actual path up was a matching-era kernel exploit, MS11-046, which the box was still unpatched against, yielding a SYSTEM shell.

## Full Walkthrough

### Enumeration

Nmap identifies an FTP service (zFTPServer) and a second web application on port 242.

![authby screenshot 1](images/authby/authby-01.png)

Anonymous FTP access is available. Browsing the FTP root turns up an `/accounts` directory holding zFTPServer's own account export files (`.uac`), including entries for `offsec`, `anonymous`, and `admin`.

![authby screenshot 2](images/authby/authby-02.png)

![authby screenshot 3](images/authby/authby-03.png)

The live `admin.uac` export is access-restricted, but a backup copy is reachable elsewhere on the share.

![authby screenshot 4](images/authby/authby-04.png)

![authby screenshot 5](images/authby/authby-05.png)

The web application on port 242 prompts for HTTP Basic Auth:

![authby screenshot 6](images/authby/authby-06.png)

Trying anonymous FTP again against a second port advertised by the service (3145) is denied:

![authby screenshot 7](images/authby/authby-07.png)

Running `dirsearch` against the port 242 site:

![authby screenshot 8](images/authby/authby-08.png)

### Directory Traversal Attempt

Checking the identified zFTPServer version against known vulnerabilities: zFTPServer 6.0.0.52 is affected by a remote directory traversal (CVE-2011-4717).

![authby screenshot 9](images/authby/authby-09.png)

Confirming with `searchsploit`:

![authby screenshot 10](images/authby/authby-10.png)

Running the exploit:

![authby screenshot 11](images/authby/authby-11.png)

![authby screenshot 12](images/authby/authby-12.png)

This doesn't produce direct access on its own.

### Web Root via FTP

Logging in as `offsec` fails, its home directory isn't available over FTP.

![authby screenshot 13](images/authby/authby-13.png)

Trying `admin` with a blank password instead:

![authby screenshot 14](images/authby/authby-14.png)

Access confirmed, and the directory listing reveals this account's home directory is actually `C:\wamp\www`, the live web root for the port 242 site, not a personal FTP folder. Downloading `index.php`, `.htpasswd`, and `.htaccess` from it:

![authby screenshot 15](images/authby/authby-15.png)

### Cracking the Basic Auth Hash

Reading `.htaccess`: the site requires Basic Auth against a local `.htpasswd` file.

![authby screenshot 16](images/authby/authby-16.png)

Reading `.htpasswd`: an Apache `$apr1$` MD5 hash for user `offsec`.

![authby screenshot 17](images/authby/authby-17.png)

Confirming the hash format:

![authby screenshot 18](images/authby/authby-18.png)

Identifying the matching Hashcat mode:

![authby screenshot 19](images/authby/authby-19.png)

Saving the hash and cracking it with Hashcat (mode 1600) against rockyou:

![authby screenshot 20](images/authby/authby-20.png)

![authby screenshot 21](images/authby/authby-21.png)

Cracked in about a second:

```
offsec : elite
```

Logging into the Basic Auth-protected site to confirm:

![authby screenshot 22](images/authby/authby-22.png)

### Getting a Shell

Since the FTP `admin` account already has write access to the live web root, a webshell can simply be uploaded there directly, no need for the Basic Auth credentials at all. Copying a standard PHP reverse shell:

![authby screenshot 23](images/authby/authby-23.png)

Uploading it via FTP:

![authby screenshot 24](images/authby/authby-24.png)

Triggering it from the browser gets a connection, but the shell process terminates almost immediately, too unstable to be useful:

![authby screenshot 25](images/authby/authby-25.png)

Checking the listener: the connection lands, but the very first command attempted (`uname`) fails, confirming the target is Windows and that this particular script isn't well suited to it.

![authby screenshot 26](images/authby/authby-26.png)

Switching to a simple command-execution webshell instead, more reliable for one-off Windows commands:

![authby screenshot 27](images/authby/authby-27.png)

Uploading it and confirming code execution:

![authby screenshot 28](images/authby/authby-28.png)

```
livda\apache
```

For a proper interactive shell, transferring `nc.exe` via FTP:

![authby screenshot 29](images/authby/authby-29.png)

Triggering it through the webshell with a listener running:

![authby screenshot 30](images/authby/authby-30.png)

An interactive shell lands as `livda\apache`.

![authby screenshot 31](images/authby/authby-31.png)

### Privilege Escalation Attempts

Checking privileges:

![authby screenshot 32](images/authby/authby-32.png)

`SeImpersonatePrivilege` is enabled. Checking the .NET Framework version, since GodPotato needs a build matching it:

![authby screenshot 33](images/authby/authby-33.png)

.NET Framework 2.0 is installed. Transferring the matching GodPotato build:

![authby screenshot 34](images/authby/authby-34.png)

Running it:

![authby screenshot 35](images/authby/authby-35.png)

Fails with "No combase module found." Trying PrintSpoofer64 next:

![authby screenshot 36](images/authby/authby-36.png)

Wrong architecture. Checking system info:

![authby screenshot 37](images/authby/authby-37.png)

The target is an x86 build of Windows Server 2008 SP1 (build 6001), running on VMware. Trying PrintSpoofer32 instead:

![authby screenshot 38](images/authby/authby-38.png)

This doesn't work either. Both Potato-family techniques rely on RPCSS/Print Spooler behavior that isn't present on a system this old, a different approach is needed.

### Privilege Escalation: MS11-046

Given the system's age, a matching-era kernel exploit is worth checking. Searching Exploit-DB for MS11-046, a local privilege escalation in `afd.sys` (CVE-2011-1249):

![authby screenshot 39](images/authby/authby-39.png)

It fits: it targets exactly this range of Windows versions (Vista through Windows 7/Server 2008), requires only low-privilege local access, and needs the target unpatched for KB2503665.

![authby screenshot 40](images/authby/authby-40.png)

Confirming the affected patch list:

![authby screenshot 41](images/authby/authby-41.png)

Confirming this build predates the corresponding hotfix:

![authby screenshot 42](images/authby/authby-42.png)

Transferring the compiled exploit via FTP:

![authby screenshot 43](images/authby/authby-43.png)

Running it:

![authby screenshot 44](images/authby/authby-44.png)

### Full System Compromise

![authby screenshot 45](images/authby/authby-45.png)

```
nt authority\system
```

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
