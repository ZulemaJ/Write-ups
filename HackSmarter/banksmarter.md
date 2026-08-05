# BankSmarter

**Category:** Linux Privilege Escalation  
**Techniques:** SNMP credential leak, race condition/rename abuse, UNIX domain socket abuse, PATH hijacking

## TL;DR

An SNMP community string (`public`) leaked SSH credentials for `Layne.Stanley`. From there, a root-owned backup script that created a directory only if it didn't already exist was abused by pre-creating (moving) that directory to a location Layne could write to, planting a malicious payload that ran as `scott.weiland` when the script fired. Scott's home directory led to a UNIX domain socket server (`pty_server.py`) that handed out a Bash shell to anyone in the `bank-team` group, connected to with `socat` to become `ronnie.stone`. Ronnie's group membership (`bankers`) gave access to a root-run script that resolved its Python interpreter through `$PATH` rather than an absolute path, a classic PATH hijack: dropping a fake `python3` on the `PATH` before the real one gave a root shell.

## Full Walkthrough

### Enumeration

Nmap:

![banksmarter screenshot 1](images/banksmarter/banksmarter-01.png)

The default scan doesn't show much, so a full port scan follows:

![banksmarter screenshot 2](images/banksmarter/banksmarter-02.png)

UDP 161 stands out:

![banksmarter screenshot 3](images/banksmarter/banksmarter-03.png)

SNMP. Enumerating it with `-sVC`:

![banksmarter screenshot 4](images/banksmarter/banksmarter-04.png)

### Initial Access as Layne

Running `snmpwalk` with the discovered community string `public`:

![banksmarter screenshot 5](images/banksmarter/banksmarter-05.png)

The output looks like credentials. Trying them over SSH:

```
Layne.Stanley:5t6^jahTRjab
```

![banksmarter screenshot 6](images/banksmarter/banksmarter-06.png)

They work.

### Enumerating for Privilege Escalation

![banksmarter screenshot 7](images/banksmarter/banksmarter-07.png)

A script, `bankSmarter_backup.sh`, looks like a promising privesc path:

![banksmarter screenshot 8](images/banksmarter/banksmarter-08.png)

It runs under `scott.weiland`'s privileges, but isn't writable yet. Checking `sudo` rights:

![banksmarter screenshot 9](images/banksmarter/banksmarter-09.png)

Nothing for Layne. `/home` turns up two other users worth keeping in mind:

![banksmarter screenshot 10](images/banksmarter/banksmarter-10.png)

Internal ports:

![banksmarter screenshot 11](images/banksmarter/banksmarter-11.png)

Nothing there. Capabilities:

![banksmarter screenshot 12](images/banksmarter/banksmarter-12.png)

SUID binaries:

![banksmarter screenshot 13](images/banksmarter/banksmarter-13.png)

One entry looks interesting, though the exploitation path isn't obvious yet, worth keeping in mind. Crontab:

![banksmarter screenshot 14](images/banksmarter/banksmarter-14.png)

`/tmp`:

![banksmarter screenshot 15](images/banksmarter/banksmarter-15.png)

An interesting folder shows up. Checking inside:

![banksmarter screenshot 16](images/banksmarter/banksmarter-16.png)

Back to the backup script for a closer read:

![banksmarter screenshot 17](images/banksmarter/banksmarter-17.png)

It defines a working directory, `/tmp/bank_exports`:

![banksmarter screenshot 18](images/banksmarter/banksmarter-18.png)

If that directory already exists, the script won't recreate it. Trying to run it directly:

![banksmarter screenshot 19](images/banksmarter/banksmarter-19.png)

No permission. Can it be renamed instead?

![banksmarter screenshot 20](images/banksmarter/banksmarter-20.png)

Yes, the directory can be moved/renamed.

### Access as Scott

With the expected directory out of the way, a malicious payload is dropped in its place:

![banksmarter screenshot 21](images/banksmarter/banksmarter-21.png)

Checking its permissions:

![banksmarter screenshot 22](images/banksmarter/banksmarter-22.png)

Setting up a listener on port 1234 with Penelope and waiting for the scheduled run:

![banksmarter screenshot 23](images/banksmarter/banksmarter-23.png)

The callback lands as `scott.weiland`.

Enumerating again from this new position:

![banksmarter screenshot 24](images/banksmarter/banksmarter-24.png)

`/opt` holds more interesting binaries and scripts, worth a closer look:

![banksmarter screenshot 25](images/banksmarter/banksmarter-25.png)

One script runs as `ronnie.stone`:

![banksmarter screenshot 26](images/banksmarter/banksmarter-26.png)

Another file, `pty_server.py`, is more directly useful:

![banksmarter screenshot 27](images/banksmarter/banksmarter-27.png)

![banksmarter screenshot 28](images/banksmarter/banksmarter-28.png)

This script implements a local UNIX domain socket server that hands out an interactive Bash shell to any client that connects. On startup, it creates the socket directory if needed, removes any existing socket file, binds a UNIX socket at `/opt/bank/sockets/live.sock`, sets its owner to `ronnie.stone` and group to `bank-team`, and sets permissions to `770`, meaning only the owner and members of `bank-team` can connect. Once a client connects, the server uses Python's `pty.fork()` to spawn `/bin/bash`, so any authorized user who can reach the socket gets a shell running with the same privileges as whoever launched `pty_server.py`.

### Access as Ronnie

Checking who's running `pty_server.py`:

```
ps aux | grep pty_server
```

Then connecting to the socket with `socat`:

```
socat -,raw,echo=0 UNIX-CONNECT:/opt/bank/sockets/live.sock
```

![banksmarter screenshot 29](images/banksmarter/banksmarter-29.png)

The shell comes back as `ronnie.stone`.

Enumerating again:

![banksmarter screenshot 30](images/banksmarter/banksmarter-30.png)

Crontab:

![banksmarter screenshot 31](images/banksmarter/banksmarter-31.png)

Running `linenum.sh` for a broader sweep:

![banksmarter screenshot 32](images/banksmarter/banksmarter-32.png)

The output keeps going, but one detail stands out: Ronnie's group is `bankers`, worth digging into further.

![banksmarter screenshot 33](images/banksmarter/banksmarter-33.png)

![banksmarter screenshot 34](images/banksmarter/banksmarter-34.png)

Checking `/usr/local/bin`:

![banksmarter screenshot 35](images/banksmarter/banksmarter-35.png)

Something worth running directly:

![banksmarter screenshot 36](images/banksmarter/banksmarter-36.png)

Reading the script itself:

![banksmarter screenshot 37](images/banksmarter/banksmarter-37.png)

The shebang, `#!/usr/bin/env python3`, resolves its interpreter through `$PATH` rather than a hardcoded path. Since the script runs as root and `env` searches `$PATH` before falling back to the real `python3`, a malicious binary placed earlier on the `PATH` will run instead.

### Access as Root

Creating a fake `python3` binary:

```
echo -e '#!/bin/bash\n/bin/bash -p' > python3
```

![banksmarter screenshot 38](images/banksmarter/banksmarter-38.png)

Making it executable:

![banksmarter screenshot 39](images/banksmarter/banksmarter-39.png)

Prepending its directory to `$PATH`:

```
PATH=/tmp:$PATH
```

![banksmarter screenshot 40](images/banksmarter/banksmarter-40.png)

![banksmarter screenshot 41](images/banksmarter/banksmarter-41.png)

Running the script now resolves `python3` from the hijacked `$PATH` instead of the real interpreter:

![banksmarter screenshot 42](images/banksmarter/banksmarter-42.png)

Root shell obtained.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
