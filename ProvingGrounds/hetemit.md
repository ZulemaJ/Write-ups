# Hetemit

**Category:** Linux, Python Code Execution, Systemd Abuse

## Attack Chain

1. A registration page (port 18000) requires an invite code; a separate API on port 50000 exposes `/generate` and `/verify` endpoints
2. `/generate` takes an email and returns a code that turns out to be the SHA-512 hash of that email
3. That code registers a working invite, but the resulting web app doesn't yield anything further (upload/edit attempts fail)
4. The `/verify` endpoint turns out to execute Python code directly; the `os` module is confirmed usable inside it
5. A reachability check, a `/etc/passwd` read, and a transferred `nc` binary confirm a working code execution primitive
6. `nc` is run from `/tmp` to spawn a reverse shell, landing as `cmeeks`
7. `sudo -l` shows access to `reboot`; a writable systemd service file (`pythonapp.service`) is found in `/etc`
8. The service file is rewritten to run a reverse shell as root on startup, then the system is rebooted (via the permitted `sudo reboot`) to trigger it

## TL;DR

An invite-only registration flow was bypassed by discovering that the site's own `/generate` API endpoint computed invite codes as the SHA-512 hash of a submitted email, letting an account be created directly. The web app itself led nowhere further, but a second endpoint on the same API, `/verify`, turned out to execute Python code server-side, and the `os` module was usable inside it. That was enough to confirm outbound connectivity, read `/etc/passwd`, transfer a static `nc` binary, and spawn a reverse shell, landing as user `cmeeks`. From there, `sudo -l` allowed a passwordless `reboot`, and a writable systemd unit file, `pythonapp.service`, was found sitting in `/etc/systemd/system`. Rewriting that unit to launch a root-level reverse shell on startup, then triggering a reboot through the permitted `sudo` command, completed the compromise on the next boot.

## Full Walkthrough

### Initial Enumeration

Nmap:

![hetemit screenshot 1](images/hetemit/hetemit-01.png)

Checking port 18000:

![hetemit screenshot 2](images/hetemit/hetemit-02.png)

Trying to register:

![hetemit screenshot 3](images/hetemit/hetemit-03.png)

An invite code is required.

![hetemit screenshot 4](images/hetemit/hetemit-04.png)

On port 50000, an API exposes `/generate` and `/verify`:

![hetemit screenshot 5](images/hetemit/hetemit-05.png)

Checking `/generate`:

![hetemit screenshot 6](images/hetemit/hetemit-06.png)

It requires an email address. Submitting one via `curl`:

![hetemit screenshot 7](images/hetemit/hetemit-07.png)

A code comes back, which turns out to be the SHA-512 hash of the email submitted. Using it to register:

![hetemit screenshot 8](images/hetemit/hetemit-08.png)

User created. Logging in:

![hetemit screenshot 9](images/hetemit/hetemit-09.png)

Nothing particularly interesting inside the app itself. Trying to edit the user profile and upload a Ruby shell disguised as an image doesn't work either. Time to look elsewhere.

The `/verify` endpoint turns out to be executing code directly:

![hetemit screenshot 10](images/hetemit/hetemit-10.png)

Knowing it's Python-based, trying the `os` module:

![hetemit screenshot 11](images/hetemit/hetemit-11.png)

### Initial Access

With `os` module access confirmed, the next step is turning this into a reverse shell. Working from [this Python `os` module cheat sheet](https://medium.com/@gupta.surender.1990/mastering-pythons-os-module-the-only-cheat-sheet-you-ll-ever-need-with-real-world-examples-0ab100052406).

Checking the current directory:

![hetemit screenshot 12](images/hetemit/hetemit-12.png)

Checking reachability to the attacking host:

![hetemit screenshot 13](images/hetemit/hetemit-13.png)

![hetemit screenshot 14](images/hetemit/hetemit-14.png)

Checking that `/etc/passwd` exists and is readable:

![hetemit screenshot 15](images/hetemit/hetemit-15.png)

Transferring a static `nc` binary:

![hetemit screenshot 16](images/hetemit/hetemit-16.png)

![hetemit screenshot 17](images/hetemit/hetemit-17.png)

Listing `/tmp` to confirm it landed:

![hetemit screenshot 18](images/hetemit/hetemit-18.png)

Changing into `/tmp`:

![hetemit screenshot 19](images/hetemit/hetemit-19.png)

Running `nc` to connect back:

![hetemit screenshot 20](images/hetemit/hetemit-20.png)

![hetemit screenshot 21](images/hetemit/hetemit-21.png)

Initial access confirmed as `cmeeks`.

### Enumeration

Checking `sudo` permissions:

![hetemit screenshot 22](images/hetemit/hetemit-22.png)

A passwordless `reboot` entry stands out. Checking for writable files in `/etc`:

![hetemit screenshot 23](images/hetemit/hetemit-23.png)

Checking the file:

![hetemit screenshot 24](images/hetemit/hetemit-24.png)

A writable systemd service file, the privilege escalation vector: modify it, then reboot to trigger it.

### Privilege Escalation

Overwriting the service definition:

```
cat <<'EOT'> /etc/systemd/system/pythonapp.service
```

(closing the heredoc with `EOT`)

![hetemit screenshot 25](images/hetemit/hetemit-25.png)

`ExecStart` and `User` are changed to run a reverse shell as root on startup. Rebooting via the permitted `sudo`:

![hetemit screenshot 26](images/hetemit/hetemit-26.png)

Setting up a listener to catch the callback on boot:

![hetemit screenshot 27](images/hetemit/hetemit-27.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
