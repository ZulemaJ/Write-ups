# BankSmarter — (SNMP — Socket — PATH Hijack)

# ENUMERATION

NMAP:

![banksmarter screenshot 1](images/banksmarter/banksmarter-01.png)

Well, it’s obvious that I have to search for other ports

Full scan:

![banksmarter screenshot 2](images/banksmarter/banksmarter-02.png)

Mh, UDP? 161?

![banksmarter screenshot 3](images/banksmarter/banksmarter-03.png)

Of course :P

Let’s enumerate snmp running -sVC to see what’s going on

![banksmarter screenshot 4](images/banksmarter/banksmarter-04.png)

# INITIAL ACCESS AS LAYNE

Let’s run snmpwalk with the discovered cred “public”

![banksmarter screenshot 5](images/banksmarter/banksmarter-05.png)

Are these credentials?

Let’s try to SSH with creds :

`Layne.Stanley:5t6^jahTRjab `

![banksmarter screenshot 6](images/banksmarter/banksmarter-06.png)

And indeed they are.

Let’s enumerate a bit to see what we can do to escalate

![banksmarter screenshot 7](images/banksmarter/banksmarter-07.png)

There is a bankSmarter_backup.sh

Interesting way of privesc.

![banksmarter screenshot 8](images/banksmarter/banksmarter-08.png)

It runs under the privs of Scott.weiland.

We cannot write in for now.

Let’s check sudo :

![banksmarter screenshot 9](images/banksmarter/banksmarter-09.png)

No sudo for Layne.

/home:

![banksmarter screenshot 10](images/banksmarter/banksmarter-10.png)

So we got 2 other interesting users.

Internal ports:

![banksmarter screenshot 11](images/banksmarter/banksmarter-11.png)

Nothing

Capabilities?

![banksmarter screenshot 12](images/banksmarter/banksmarter-12.png)

SUIDs?

![banksmarter screenshot 13](images/banksmarter/banksmarter-13.png)

That’s interesting.

But what is the way of exploiting it? Let’s take it in mind for now

Cronatab?

![banksmarter screenshot 14](images/banksmarter/banksmarter-14.png)

/tmp?

![banksmarter screenshot 15](images/banksmarter/banksmarter-15.png)

Interesting folder.

What is it?

Let’s check inside:

![banksmarter screenshot 16](images/banksmarter/banksmarter-16.png)

Interesting..

Let’s check the backup script :

![banksmarter screenshot 17](images/banksmarter/banksmarter-17.png)

The directory /tmp/bank_exports is defined

![banksmarter screenshot 18](images/banksmarter/banksmarter-18.png)

If there is already the dir, it will not be created, otherwise it will

Let’s try to run it

![banksmarter screenshot 19](images/banksmarter/banksmarter-19.png)

We can’t

Can we mv it?

![banksmarter screenshot 20](images/banksmarter/banksmarter-20.png)

Indeed we can modify the name!

# ACCESS AS SCOTT

Let’s see if we put a malicious payload instead.

![banksmarter screenshot 21](images/banksmarter/banksmarter-21.png)

And privs

![banksmarter screenshot 22](images/banksmarter/banksmarter-22.png)

And we setup a listener on port 1234 w/ Penelope

Waiting…

![banksmarter screenshot 23](images/banksmarter/banksmarter-23.png)

And here we are as scott.weiland

Enumeration again.

![banksmarter screenshot 24](images/banksmarter/banksmarter-24.png)

In /opt we find some other interesting binaries and scripts.

Let’s check it.

![banksmarter screenshot 25](images/banksmarter/banksmarter-25.png)

The script run as Ronnie

![banksmarter screenshot 26](images/banksmarter/banksmarter-26.png)

pty_server.py too

`This pty_server.py script implements a local `**`UNIX domain socket server`**` that provides an interactive `**`Bash shell`**` to any client that successfully connects to the socket.`

`When started, the script performs the following actions:`

`Creates the socket directory (if it does not already exist).`

`Removes any existing socket file at the configured path.`

`Creates and binds a UNIX socket at:  /opt/bank/sockets/live.sock`

`Changes the ownership of the socket to:`

**`Owner:`**` ronnie.stone`

**`Group:`**` bank-team`

`Sets the socket permissions to `**`770`**`, allowing only the owner and members of the bank-team group to connect.`

`Once a client connects, the server uses Python’s pty.fork() function to create a new pseudo-terminal (PTY). The child process immediately executes:`

`/bin/bash`

`As a result, any authorized user capable of connecting to the UNIX socket obtains an interactive Bash shell running with the `**`same privileges as the user executing pty_server.py`**`. `

![banksmarter screenshot 27](images/banksmarter/banksmarter-27.png)

![banksmarter screenshot 28](images/banksmarter/banksmarter-28.png)

# ACCESS AS RONNIE

We can check who’s running pty_server with :

`Ps aux | grep pty_server `

And then try to connect to the socket with socat:

`socat -,raw,echo=0 UNIX-CONNECT:/opt/bank/sockets/live.sock`

![banksmarter screenshot 29](images/banksmarter/banksmarter-29.png)

And we’re Ronnie.stone

Enumeration again

![banksmarter screenshot 30](images/banksmarter/banksmarter-30.png)

What is now the final step?

Crontab?

![banksmarter screenshot 31](images/banksmarter/banksmarter-31.png)

Let’s check maybe with linenum.sh

Well I’m stucked

![banksmarter screenshot 32](images/banksmarter/banksmarter-32.png)

This is always dumping…

I think that’s the way

The group is bankers.

Let’s check a bit more about bankers :

![banksmarter screenshot 33](images/banksmarter/banksmarter-33.png)

![banksmarter screenshot 34](images/banksmarter/banksmarter-34.png)

Let’s see what is inside /usr/local/bin

![banksmarter screenshot 35](images/banksmarter/banksmarter-35.png)

Interesting.

What if I run it?

![banksmarter screenshot 36](images/banksmarter/banksmarter-36.png)

What’s in the .py?

![banksmarter screenshot 37](images/banksmarter/banksmarter-37.png)

#!/usr/bin/env python3 is specifying which Interpreter should be used to run the script.

The script runs as **root user** and uses /usr/bin/env python3, and **you can control the ****$PATH** that gets searched **before the real python3**, you might trick the system into executing **your malicious binary** instead of the real Python.

# ACCESS AS ROOT

Let’s create our own binary python3 and add it to the Path.

1. Creating a fake python3 binary

`echo -e '#!/bin/bash\n/bin/bash -p' > python3`

![banksmarter screenshot 38](images/banksmarter/banksmarter-38.png)

Add privs

![banksmarter screenshot 39](images/banksmarter/banksmarter-39.png)

1. Add it to the path

`PATH=/tmp:$PATH`

1.
![banksmarter screenshot 40](images/banksmarter/banksmarter-40.png)

![banksmarter screenshot 41](images/banksmarter/banksmarter-41.png)

Now if we run the ./bank_backupd , It will look for the Interpreter from the PATH and we have added it.

![banksmarter screenshot 42](images/banksmarter/banksmarter-42.png)

# PWNED
