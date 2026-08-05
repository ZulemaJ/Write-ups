# Lumon — (CVE-2025-24054 — AD | not completed )

![lumon screenshot 1](images/lumon/lumon-01.png)

![lumon screenshot 2](images/lumon/lumon-02.png)

![lumon screenshot 3](images/lumon/lumon-03.png)

![lumon screenshot 4](images/lumon/lumon-04.png)

![lumon screenshot 5](images/lumon/lumon-05.png)

![lumon screenshot 6](images/lumon/lumon-06.png)

![lumon screenshot 7](images/lumon/lumon-07.png)

![lumon screenshot 8](images/lumon/lumon-08.png)

RDP Failed

Winrm Failed

![lumon screenshot 9](images/lumon/lumon-09.png)

![lumon screenshot 10](images/lumon/lumon-10.png)

Wow, we got a lot of users

But we got a service too : IntranetSvc

Let’s query all the users first, with “query user” in rpcclient,

then we’ll try to kerberoast.

No leaks of info found in rpcclient querying users.

Let’s go ahead and try to kerberoast

![lumon screenshot 11](images/lumon/lumon-11.png)

So time to bloodhound

Let’s run bloodhound-ce-python

bloodhound-ce-python -u hellyr -p H3lenaR\!2025  -d lumons.hacksmarter -dc DC01.lumons.hacksmarter -ns 10.0.23.182 -c All --zip

![lumon screenshot 12](images/lumon/lumon-12.png)

![lumon screenshot 13](images/lumon/lumon-13.png)

Hellyr has no Outbound object Control.

Let’s try password spray now that we have a list of users.

![lumon screenshot 14](images/lumon/lumon-14.png)

Since it’s H3lenaR hellyr, can we try H3lenaE!2025 for hellye ? 

![lumon screenshot 15](images/lumon/lumon-15.png)

Ok, that was a stupid move.

Let’s go ahead.

What about asreproasting?

![lumon screenshot 16](images/lumon/lumon-16.png)

Ahh, what is the path?

I think that should be SMB

![lumon screenshot 17](images/lumon/lumon-17.png)

Nothing in NETLOGON.

Reviewing again SYSVOL:

![lumon screenshot 18](images/lumon/lumon-18.png)

What is it?

We found also Registry.pol readable.

![lumon screenshot 19](images/lumon/lumon-19.png)

Can we parse the file with a tool?

We can try regpol.py :

https://github.com/jtpereyda/regpol

![lumon screenshot 20](images/lumon/lumon-20.png)

![lumon screenshot 21](images/lumon/lumon-21.png)

Nothing particularly interesting.

Rechecking Bloodhound we can see that hellyr is part of the microdata refinement group.

![lumon screenshot 22](images/lumon/lumon-22.png)

On the other side, hellye is part of Domain Admins Group.

![lumon screenshot 23](images/lumon/lumon-23.png)

INTRANET ENUMERATION

Let’s Enumerate the intranet to see what’s going on overthere.

![lumon screenshot 24](images/lumon/lumon-24.png)

Again, let’s check smb

![lumon screenshot 25](images/lumon/lumon-25.png)

![lumon screenshot 26](images/lumon/lumon-26.png)

Cool! That’s way more interesting.

We have write permissions.

That is SMB Poisoning direct.

![lumon screenshot 27](images/lumon/lumon-27.png)

![lumon screenshot 28](images/lumon/lumon-28.png)

![lumon screenshot 29](images/lumon/lumon-29.png)

![lumon screenshot 30](images/lumon/lumon-30.png)

Let’s check with exiftool just in case

![lumon screenshot 31](images/lumon/lumon-31.png)

Ok let’s try to login in the dashboard

https://intranet.lumons.hacksmarter/login

As stated in the pdf :

`On the intranet sign-in page, enter your domain username in the format:`

`your-username or LUMONS\your-username or your-username@lumons.hacksmarter.`

![lumon screenshot 32](images/lumon/lumon-32.png)

![lumon screenshot 33](images/lumon/lumon-33.png)

What?

![lumon screenshot 34](images/lumon/lumon-34.png)

![lumon screenshot 35](images/lumon/lumon-35.png)

Apparently we can do nothing from here.

Mh, let’s try dirsearch?

![lumon screenshot 36](images/lumon/lumon-36.png)

![lumon screenshot 37](images/lumon/lumon-37.png)

![lumon screenshot 38](images/lumon/lumon-38.png)

So what’s the point?

Let’s change wordlists to see if we can discovery more :

Meanwhile in the PDF :

![lumon screenshot 39](images/lumon/lumon-39.png)

Seoul Annex?

Let’s go ahead trying to poison SMB

We’re gonna use :

https://github.com/Greenwolf/ntlm_theft

1. Clone

![lumon screenshot 40](images/lumon/lumon-40.png)

1. Run a responder

![lumon screenshot 41](images/lumon/lumon-41.png)

1. Run the ntlm_theft.py

![lumon screenshot 42](images/lumon/lumon-42.png)

1. Replace the “Lumons Intranet.url” with a malicious one

![lumon screenshot 43](images/lumon/lumon-43.png)

![lumon screenshot 44](images/lumon/lumon-44.png)

No output.

What’s the fucking path?

RDP= failed

WINRM = failed

Well, at this time I have to check the next step in a walkthrough, cos I really don’t know what to do.

It’s talking about something that I don’t know, related to a CVE-2025-24054 :

a Windows vulnerability that allows to leak NTLM  hashes by tricking users into interacting with a .library-ms file.

Windows automatically attempts NTLM authentication when accessing remote resources, often without clear user warning. This implicit trust in network locations allows to coerce the system into sending credential material.

https://research.checkpoint.com/2025/cve-2025-24054-ntlm-exploit-in-the-wild/

And the POC:

https://github.com/helidem/CVE-2025-24054_CVE-2025-24071-PoC

![lumon screenshot 45](images/lumon/lumon-45.png)

![lumon screenshot 46](images/lumon/lumon-46.png)

Ok now once Responder is launched, let’s put the file into SMB

![lumon screenshot 47](images/lumon/lumon-47.png)

And we got output !

![lumon screenshot 48](images/lumon/lumon-48.png)

Harmonyc NTLMv2 hash.

Let’s crack it with hashcat

![lumon screenshot 49](images/lumon/lumon-49.png)

![lumon screenshot 50](images/lumon/lumon-50.png)

![lumon screenshot 51](images/lumon/lumon-51.png)

And finally cracked.

Now we can crackmapexec on both Internal and DC01 to see what we have

![lumon screenshot 52](images/lumon/lumon-52.png)

![lumon screenshot 53](images/lumon/lumon-53.png)

Weird. Why?

Let’s check in Bloodhound what we have

![lumon screenshot 54](images/lumon/lumon-54.png)

No outbound OC

Let’s try to RDP and WINRM

WINRM on Internal :: failed

WINRM on DC01 :: failed

RDP on Internal :: Failed

WINRM on DC01 :: Failed

What’s wrong?

![lumon screenshot 55](images/lumon/lumon-55.png)

In the dashboard credentials are correct.

So something is not working

Maybe it’s just bugging.

Let’s restart the machine

Ok maybe  it is a problem with NetworkManager, cos I changed the /etc/hosts and I had to restart it cos it was bugging

![lumon screenshot 56](images/lumon/lumon-56.png)

Let’s try again

![lumon screenshot 57](images/lumon/lumon-57.png)

![lumon screenshot 58](images/lumon/lumon-58.png)

That’s weird

Let’s try with SMBMap

![lumon screenshot 59](images/lumon/lumon-59.png)

![lumon screenshot 60](images/lumon/lumon-60.png)

I can’t understand

![lumon screenshot 61](images/lumon/lumon-61.png)

Let’s try to list everything manually

![lumon screenshot 62](images/lumon/lumon-62.png)

Not interesting.

Don’t know.

![lumon screenshot 63](images/lumon/lumon-63.png)

We can see in bloodhound that harmonyc is member of Administration.

Rechecking the Intranet panel we can see that now we are in the Admin Panel. Weird.

![lumon screenshot 64](images/lumon/lumon-64.png)

![lumon screenshot 65](images/lumon/lumon-65.png)

![lumon screenshot 66](images/lumon/lumon-66.png)

Maybe harmonyc account is locked, and we can try to unlock it

![lumon screenshot 67](images/lumon/lumon-67.png)

![lumon screenshot 68](images/lumon/lumon-68.png)

Nope.

In the Admin Panel:

![lumon screenshot 69](images/lumon/lumon-69.png)

Can we browse ?

![lumon screenshot 70](images/lumon/lumon-70.png)

Browsing \\10.0.27.145\ADMIN$

![lumon screenshot 71](images/lumon/lumon-71.png)

![lumon screenshot 72](images/lumon/lumon-72.png)

Interesting Panther, let’s go ahead

\\10.0.27.145\ADMIN$\Panther

![lumon screenshot 73](images/lumon/lumon-73.png)

![lumon screenshot 74](images/lumon/lumon-74.png)

Nope.

Let’s try C$

\\10.0.27.145\C$\

![lumon screenshot 75](images/lumon/lumon-75.png)

![lumon screenshot 76](images/lumon/lumon-76.png)

Maybe from here we can do an smb relay attack with impacket-ntlmrelayx

Let’s try it out.

- First , we need a powershell one liner encoded in base 64

![lumon screenshot 77](images/lumon/lumon-77.png)

- Then we have to run impacket-ntlmrelayx with the following flags :

- --no-http-server

- -smb2support

- -t = target

- -c = powershell encoded for reverse shell

![lumon screenshot 78](images/lumon/lumon-78.png)

- Then we launch Penelope on port 4444

![lumon screenshot 79](images/lumon/lumon-79.png)

- And finally we list a share in the Admin Panel

![lumon screenshot 80](images/lumon/lumon-80.png)

But no output……

WTF

This is not working.

Maybe just calling a share with responding running, we could have an NTLM

![lumon screenshot 81](images/lumon/lumon-81.png)
