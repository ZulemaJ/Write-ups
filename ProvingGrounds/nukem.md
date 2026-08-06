# Nukem

**Category:** Linux, WordPress, DOSBox Privilege Escalation

## Attack Chain

1. Ports found: 80 (WordPress), 13000, and Samba forwarded off its non-standard port (36445)
2. SMB allows file uploads but nothing else useful; WPScan on the WordPress site finds a plugin with a potential RCE/file upload vulnerability
3. A first exploit attempt fails; a second, modified with a Bash reverse shell one-liner, succeeds on port 80 (port 4444 doesn't connect)
4. Initial shell lands as the low-privilege `http` user
5. A SUID `dosbox` binary is found; escalating with it requires a GUI, so a stable session is needed first
6. WordPress's own `wp-config` file (readable under `/srv/http`) leaks credentials for user `commander`, who has SSH access
7. A local VNC service (port 5901) is discovered via `netstat` and forwarded over SSH
8. From the VNC GUI session, the DOSBox SUID exploit runs a mount/redirect trick to append a root-equivalent user to `/etc/passwd`
9. `su - superroot` with the exploit's hardcoded password completes the compromise

## TL;DR

A vulnerable WordPress plugin, found via WPScan, gave a foothold as the low-privilege `http` user after a couple of exploit variants were tried (the first failed outright, and even the working one needed the callback moved from port 4444 to port 80 to connect). From there, `http`'s access to WordPress's own config file leaked SSH credentials for `commander`. A SUID `dosbox` binary looked like the privilege escalation path, but DOSBox's escalation trick needs a graphical session to work, so the next step was finding one: a locally-bound VNC service, forwarded over the existing SSH session and accessed with `commander`'s own password. From inside that GUI session, the DOSBox exploit used its mount/redirect capabilities to append a new root-equivalent user directly into `/etc/passwd`, switched into with `su`.

## Full Walkthrough

### Initial Enumeration

![nukem screenshot 1](images/nukem/nukem-01.png)

Port 80, port 13000, and Samba on the non-standard port 36445.

**Forwarding the port** to reach SMB more conveniently with standard tools:

![nukem screenshot 2](images/nukem/nukem-02.png)

![nukem screenshot 3](images/nukem/nukem-03.png)

![nukem screenshot 4](images/nukem/nukem-04.png)

![nukem screenshot 5](images/nukem/nukem-05.png)

![nukem screenshot 6](images/nukem/nukem-06.png)

Files can be uploaded to the share, but nothing else useful turns up here for now. Moving on to enumerating the WordPress site with WPScan:

![nukem screenshot 7](images/nukem/nukem-07.png)

![nukem screenshot 8](images/nukem/nukem-08.png)

A plugin is identified that could be vulnerable to RCE or file upload.

### Initial Access

Checking it further:

![nukem screenshot 9](images/nukem/nukem-09.png)

![nukem screenshot 10](images/nukem/nukem-10.png)

Trying a first exploit:

![nukem screenshot 11](images/nukem/nukem-11.png)

Doesn't work. Trying alternatives:

![nukem screenshot 12](images/nukem/nukem-12.png)

![nukem screenshot 13](images/nukem/nukem-13.png)

Modifying the exploit to include a Bash reverse shell one-liner with the attacking IP:

![nukem screenshot 14](images/nukem/nukem-14.png)

![nukem screenshot 15](images/nukem/nukem-15.png)

Port 4444 doesn't connect. Trying port 80 instead:

![nukem screenshot 16](images/nukem/nukem-16.png)

Shell obtained.

### Enumeration and Lateral Movement

![nukem screenshot 17](images/nukem/nukem-17.png)

Access as `http`, with very limited privileges. After a fair amount of enumeration, a SUID `dosbox` binary stands out among the SUID list:

![nukem screenshot 18](images/nukem/nukem-18.png)

Researching DOSBox privilege escalation confirms it: SUID DOSBox allows reading or writing arbitrary files, but the escalation technique needs a graphical session to work. The reference exploit: [DOSBox Privilege Escalation (B00t2R00t)](https://github.com/H3llKa1ser/B00t2R00t/blob/main/Privilege%20Escalation/Linux/Dosbox%20Privilege%20Escalation.md).

To get a stable, GUI-capable session, checking the WordPress install itself for reusable credentials. Since this is a WordPress site, `/srv/http` is worth inspecting, and `wp-config` there contains credentials:

![nukem screenshot 19](images/nukem/nukem-19.png)

Trying SSH with user `commander`:

![nukem screenshot 20](images/nukem/nukem-20.png)

Access confirmed. Running `netstat` from this session shows port 5901 (VNC) bound locally:

![nukem screenshot 21](images/nukem/nukem-21.png)

Forwarding it over SSH:

![nukem screenshot 22](images/nukem/nukem-22.png)

Port 5901 is now reachable at `127.0.0.1:5901` locally.

**Accessing VNC:**

![nukem screenshot 23](images/nukem/nukem-23.png)

`commander`'s password works for the VNC session too.

![nukem screenshot 24](images/nukem/nukem-24.png)

With a graphical session available, the DOSBox exploit can now run.

### Privilege Escalation

Following the [DOSBox privilege escalation technique](https://github.com/H3llKa1ser/B00t2R00t/blob/main/Privilege%20Escalation/Linux/Dosbox%20Privilege%20Escalation.md).

From the VNC session, opening a terminal and running the first stage:

![nukem screenshot 25](images/nukem/nukem-25.png)

This creates a file containing a one-liner meant to be written into `/etc/passwd`. Launching DOSBox, which opens its GUI:

![nukem screenshot 26](images/nukem/nukem-26.png)

Inside the DOSBox GUI, running the commands that overwrite `/etc/passwd` using its mount/redirect capabilities:

![nukem screenshot 27](images/nukem/nukem-27.png)

Checking whether the new user, `superroot`, was successfully appended:

![nukem screenshot 28](images/nukem/nukem-28.png)

It was. Switching to it with the exploit's hardcoded password:

```
su - superroot
```

password: `toor`

![nukem screenshot 29](images/nukem/nukem-29.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
