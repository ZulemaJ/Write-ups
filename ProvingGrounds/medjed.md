# Medjed

**Category:** Windows, Barracuda Web Server, WebDAV

## Attack Chain

1. A full Nmap scan finds a Barracuda web server on port 8000; its Configuration Wizard allows setting up a fresh Administrator account with no prior authentication
2. The admin panel exposes a Web-File-Server, which turns out to be a WebDAV endpoint with access to the entire `C:\`
3. `davtest` shows only `.html` and `.txt` uploads actually execute
4. Enumeration finds an `htdocs` directory tied to a hosted website, an anonymous FTP server (nothing useful), and two more HTTP services (3303, a login page; 45332, a Quiz App)
5. Cross-referencing the `htdocs` subdirectories against the two web apps identifies the Quiz App as the one backed by the writable directory
6. A webshell and `nc.exe` are uploaded via `cadaver` through WebDAV into that directory, then triggered over HTTP for code execution and a reverse shell
7. A public BarracudaDrive 6.5 privilege escalation exploit (insecure folder permissions, EDB-ID 48789) is identified; it requires the current account to hold shutdown privileges
8. An `AddAdmin.c` payload from the exploit is compiled, transferred, and placed to run after a reboot, creating a new administrator account
9. After reboot, `Invoke-RunasCs.ps1` spawns a reverse shell under the new admin context
10. A documented fallback (replacing `bd.exe` with an `msfvenom` payload and forcing a reboot) is noted as an alternative if the primary path fails

## TL;DR

A Barracuda web server exposed its own setup wizard without prior authentication, letting a fresh Administrator account be created directly through the UI. That admin panel included a Web-File-Server, actually a WebDAV endpoint with access to the entire C: drive, though `davtest` showed only `.html`/`.txt` files would actually execute when placed there. Cross-referencing the writable `htdocs` directories against the site's other exposed web apps (a login page and a Quiz App) identified which directory backed which app, and a webshell uploaded through WebDAV into the Quiz App's folder gave code execution, upgraded to a reverse shell with a transferred `nc.exe`. Privilege escalation used a public BarracudaDrive 6.5 exploit for insecure folder permissions: a small compiled C payload was placed to create a new admin account on the next reboot, which was then triggered (the exploit needs shutdown privileges on the current account), and a follow-up `Invoke-RunasCs.ps1` run spawned a shell under the new administrative context.

## Full Walkthrough

### Enumeration

A full Nmap scan reveals:

![medjed screenshot 1](images/medjed/medjed-01.png)

Checking further ports, a Barracuda server is running on port 8000:

![medjed screenshot 2](images/medjed/medjed-02.png)

Its Configuration Wizard allows setting up an Administrator account directly:

![medjed screenshot 3](images/medjed/medjed-03.png)

Setting one:

![medjed screenshot 4](images/medjed/medjed-04.png)

Proceeding:

![medjed screenshot 5](images/medjed/medjed-05.png)

A "Web-File-Server" option is visible. Checking it:

![medjed screenshot 6](images/medjed/medjed-06.png)

It's a WebDAV endpoint:

![medjed screenshot 7](images/medjed/medjed-07.png)

Access extends to the entire `C:\` of the system. Checking what file types can actually be uploaded and executed with `davtest`:

![medjed screenshot 8](images/medjed/medjed-08.png)

Only `.html` and `.txt` files execute. Digging further turns up an `htdocs` directory belonging to a hosted website:

![medjed screenshot 9](images/medjed/medjed-09.png)

Continuing enumeration to work out the right path to initial access: an FTP server allows anonymous login,

![medjed screenshot 10](images/medjed/medjed-10.png)

but nothing particularly useful is inside. HTTP on port 3303:

![medjed screenshot 11](images/medjed/medjed-11.png)

Just a login page for now. HTTP on port 45332:

![medjed screenshot 12](images/medjed/medjed-12.png)

A Quiz App. Cross-referencing the directories found under `/htdocs` against these web apps identifies the Quiz App as the one actually backed by the writable directory found via WebDAV, and its directories are accessible.

The plan: upload a shell into that `/htdocs` directory and trigger it over HTTP.

### Initial Access

Using `cadaver` to interact with the Barracuda WebDAV file server: navigating into the target directory and uploading both a webshell and `nc.exe`:

![medjed screenshot 13](images/medjed/medjed-13.png)

Visiting `192.168.122.127:45332/shell.php`:

![medjed screenshot 14](images/medjed/medjed-14.png)

Initial access confirmed. Executing the already-uploaded `nc.exe` to connect back:

![medjed screenshot 15](images/medjed/medjed-15.png)

![medjed screenshot 16](images/medjed/medjed-16.png)

Reverse shell obtained.

### Privilege Escalation

Researching BarracudaDrive 6.5-specific exploits turns up a privilege escalation issue tied to insecure folder permissions: [EDB-ID 48789](https://www.exploit-db.com/exploits/48789).

![medjed screenshot 17](images/medjed/medjed-17.png)

Working through it step by step:

![medjed screenshot 18](images/medjed/medjed-18.png)

![medjed screenshot 19](images/medjed/medjed-19.png)

The exploit requires the current account to hold shutdown privileges:

![medjed screenshot 20](images/medjed/medjed-20.png)

![medjed screenshot 21](images/medjed/medjed-21.png)

Creating the `AddAdmin.c` payload from the exploit:

![medjed screenshot 22](images/medjed/medjed-22.png)

Two missing `#include` statements had to be added at the top for it to compile correctly. Transferring the compiled binary:

![medjed screenshot 23](images/medjed/medjed-23.png)

Confirming the target admin account name doesn't already exist:

![medjed screenshot 24](images/medjed/medjed-24.png)

Rebooting the system to trigger the exploit:

![medjed screenshot 25](images/medjed/medjed-25.png)

Reconnecting via the reverse shell and checking the current user:

![medjed screenshot 26](images/medjed/medjed-26.png)

Running `Invoke-RunasCs.ps1` to spawn a reverse shell under the new administrative context:

![medjed screenshot 27](images/medjed/medjed-27.png)

![medjed screenshot 28](images/medjed/medjed-28.png)

![medjed screenshot 29](images/medjed/medjed-29.png)

Full system compromise.

### Documented Fallback

If this path doesn't work, an alternative approach using the same insecure-permissions issue: creating an `msfvenom` payload named `bd.exe`,

![medjed screenshot 30](images/medjed/medjed-30.png)

transferring it to replace the original `bd.exe`, and forcing a reboot with `shutdown -r`:

![medjed screenshot 31](images/medjed/medjed-31.png)

With a listener running to catch the resulting callback:

![medjed screenshot 32](images/medjed/medjed-32.png)

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
