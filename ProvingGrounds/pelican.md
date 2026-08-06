# Pelican

**Category:** Linux, Exhibitor for ZooKeeper

## Attack Chain

1. Nmap finds Samba, two HTTP services (8080, 8081), and CUPS 2.2; `enum4linux` and directory brute-forcing on 8080/631 turn up nothing
2. Port 8081 hosts Exhibitor for ZooKeeper v1.0
3. A known Exhibitor vulnerability (TALOS-2019-0790) allows editing the `java.env` script from the web UI
4. `java.env` is modified with a reverse shell payload, committed, and the service restarted
5. The restart triggers the payload, landing a shell as `Charles`
6. `sudo -l` shows a passwordless `gcore` entry
7. `gcore` is used (per GTFOBins) to dump the memory of a running `/usr/bin/password` process
8. `strings` against the core dump reveals a plaintext password, used to `su root`

## TL;DR

Exhibitor for ZooKeeper, exposed on port 8081, was vulnerable to a known issue (TALOS-2019-0790) that allowed the `java.env` startup script to be edited directly from the web UI. Replacing it with a reverse shell payload and restarting the service through the same UI triggered the shell on the next startup, landing as user `Charles`. From there, `sudo -l` showed passwordless access to `gcore`, a GTFOBins entry for dumping process memory to a local file. Dumping the memory of a suspiciously named `/usr/bin/password` process and filtering the resulting core file with `strings` revealed a plaintext password in memory, enough to `su` directly to root.

## Full Walkthrough

### Enumeration

Nmap:

![pelican screenshot 1](images/pelican/pelican-01.png)

![pelican screenshot 2](images/pelican/pelican-02.png)

Ports of interest: Samba (445), HTTP on 8080 and 8081, and CUPS 2.2 on 631. `enum4linux` doesn't return anything useful.

**HTTP enumeration, port 8080:**

![pelican screenshot 3](images/pelican/pelican-03.png)

Running `dirsearch` here turns up nothing. Port 631 (CUPS 2.2):

![pelican screenshot 4](images/pelican/pelican-04.png)

Also nothing from `dirsearch`, and no exploits found after researching CUPS itself. Port 8081:

![pelican screenshot 5](images/pelican/pelican-05.png)

Exhibitor for ZooKeeper v1.0, likely the entry point.

### Initial Access

Researching known vulnerabilities in Exhibitor for ZooKeeper turns up something relevant:

![pelican screenshot 6](images/pelican/pelican-06.png)

[TALOS-2019-0790](https://talosintelligence.com/vulnerability_reports/TALOS-2019-0790):

![pelican screenshot 7](images/pelican/pelican-07.png)

Checking the local Exhibitor instance:

![pelican screenshot 8](images/pelican/pelican-08.png)

The `java.env` script is enabled and editable directly from the UI. Modifying it per the vulnerability report to spawn a reverse shell:

![pelican screenshot 9](images/pelican/pelican-09.png)

Committing the change and restarting the service:

![pelican screenshot 10](images/pelican/pelican-10.png)

Checking the listener:

![pelican screenshot 11](images/pelican/pelican-11.png)

Shell obtained as `Charles`.

### Privilege Escalation

Checking `sudo` permissions:

![pelican screenshot 12](images/pelican/pelican-12.png)

A passwordless `gcore` entry. Checking GTFOBins:

![pelican screenshot 13](images/pelican/pelican-13.png)

`gcore` can dump the memory of a running process to a local file, which can then be filtered with `strings` for anything sensitive left in memory. Checking running processes for a promising target:

![pelican screenshot 14](images/pelican/pelican-14.png)

`/usr/bin/password` stands out. Running `sudo gcore $PID` against it:

![pelican screenshot 15](images/pelican/pelican-15.png)

A `core.490` file is produced. Filtering it with `strings`:

![pelican screenshot 16](images/pelican/pelican-16.png)

![pelican screenshot 17](images/pelican/pelican-17.png)

A password turns up. Using it to `su root`:

![pelican screenshot 18](images/pelican/pelican-18.png)

Full system compromise.

---

*Educational write-up from an authorized lab environment (OffSec Proving Grounds). Not a guide for unauthorized access.*
