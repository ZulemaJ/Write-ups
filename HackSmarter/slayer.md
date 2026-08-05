# Slayer

**Category:** Active Directory  
**Techniques:** RDP access, host enumeration, credential dumping attempts

## TL;DR

Starting from known credentials (`tyler.ramsey`), RDP access to the target was straightforward. From there, several post-exploitation avenues were tried: hunting for interesting files and a possible database, checking running processes for a credential dump, attempting `edgesnapper.exe`, and enumerating RPC and SMB with `enum4linux-ng`. None of them turned up a clear escalation path on this pass. This walkthrough documents the enumeration performed and the dead ends hit; it is left as a work in progress rather than a completed compromise.

## Full Walkthrough

### Initial Access

Credentials were already available for this target:

```
tyler.ramsey:P@ssw0rd!
```

![slayer screenshot 1](images/slayer/slayer-01.png)

![slayer screenshot 2](images/slayer/slayer-02.png)

![slayer screenshot 3](images/slayer/slayer-03.png)

Connecting over RDP:

![slayer screenshot 4](images/slayer/slayer-04.png)

![slayer screenshot 5](images/slayer/slayer-05.png)

![slayer screenshot 6](images/slayer/slayer-06.png)

Nothing immediately interesting on first look around the desktop.

![slayer screenshot 7](images/slayer/slayer-07.png)

### Filesystem and Process Enumeration

One folder stands out and is worth a closer look:

![slayer screenshot 8](images/slayer/slayer-08.png)

![slayer screenshot 9](images/slayer/slayer-09.png)

Checking whether a database is present, and reviewing running processes for anything that could be dumped for credentials:

![slayer screenshot 10](images/slayer/slayer-10.png)

![slayer screenshot 11](images/slayer/slayer-11.png)

Microsoft Defender is active on the host, which limits what can be done here without triggering detection.

### Further Enumeration Attempts

Trying to run `edgesnapper.exe` doesn't lead anywhere conclusive:

![slayer screenshot 12](images/slayer/slayer-12.png)

Enumerating RPC for a possible information leak:

![slayer screenshot 13](images/slayer/slayer-13.png)

No usable output. Trying `enum4linux-ng` as a final pass:

![slayer screenshot 14](images/slayer/slayer-14.png)

Also inconclusive. This lab is left here as an open enumeration exercise rather than a full compromise.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
