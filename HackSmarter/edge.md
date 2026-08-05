**
# Edge — (EdgeSnapper — svc_vdi) **

Creds :

`Username: jmorris`

`Password: Fabricat!on2024`

# ENUMERATION :

NMAP :

![edge screenshot 1](images/edge/edge-01.png)

## SMB ENUM

- Enum4linux-ng

![edge screenshot 2](images/edge/edge-02.png)

![edge screenshot 3](images/edge/edge-03.png)

![edge screenshot 4](images/edge/edge-04.png)

No RCP connection possible.

- Crackmapexec

![edge screenshot 5](images/edge/edge-05.png)

Write permissions in VantaraOps.

This could mean : POISONING — NTLM Theft

Let’s check first the share

![edge screenshot 6](images/edge/edge-06.png)

![edge screenshot 7](images/edge/edge-07.png)

Let’s check scripts

![edge screenshot 8](images/edge/edge-08.png)

Since they are automated scripts, maybe we could just modify the script adding a powershell one-liner reverse shell?

1. Download the file

![edge screenshot 9](images/edge/edge-09.png)

1. Craft a powershell script and code it in base64

![edge screenshot 10](images/edge/edge-10.png)

1. Add it to the script with : “powershell -enc <command>”

![edge screenshot 11](images/edge/edge-11.png)

1. Run Penelope

![edge screenshot 12](images/edge/edge-12.png)

1. Put the malicious .bat in the SMB share

![edge screenshot 13](images/edge/edge-13.png)

And wait …

If it’s triggered, we can obtained a reverse shell.

Otherwise, we’re gonna go for pollution

No output. Nothing is triggered.

Let’s check the other files in the share :

![edge screenshot 14](images/edge/edge-14.png)

Preauth disabled!

This could lead to ASREPRoasting

![edge screenshot 15](images/edge/edge-15.png)

SNMP seems to be open

Let’s check it :

![edge screenshot 16](images/edge/edge-16.png)

Filtered.

![edge screenshot 17](images/edge/edge-17.png)

Here we have a list of users.

Let’s create a list.

![edge screenshot 18](images/edge/edge-18.png)

And other services

![edge screenshot 19](images/edge/edge-19.png)

Do we have Windows Defender running?

We have pretty much information from SMB.

What we can try now before pollution is asreproasting with the users file we got, especially asrep_svc

First we update the /etc/hosts

![edge screenshot 20](images/edge/edge-20.png)

![edge screenshot 21](images/edge/edge-21.png)

Well, seems not to be the right path

Let’s proceed to NTLM_THEFT

1. Start a responder

![edge screenshot 22](images/edge/edge-22.png)

1. Clone the repository

https://github.com/Greenwolf/ntlm_theft

![edge screenshot 23](images/edge/edge-23.png)

1. Run ntlm_theft.py

![edge screenshot 24](images/edge/edge-24.png)

1. Put the files into the smb

![edge screenshot 25](images/edge/edge-25.png)

1. Wait for responder

1.
![edge screenshot 26](images/edge/edge-26.png)

No output.

That’s not the path.

Can we simply WINRM or RDP with credentials we have?

![edge screenshot 27](images/edge/edge-27.png)

Indeed we can wtf.

A lot of time lost whatthefuck

Btw, let’s enumerate:

![edge screenshot 28](images/edge/edge-28.png)

We can see that there is Microsoft Edge.lnk

Can we get profiles from it?

Following Browser Artifacts from Hacktricks, we can try to get Edge Profiles:

https://hacktricks.wiki/en/generic-methodologies-and-resources/basic-forensic-methodology/specific-software-file-type-tricks/browser-artifacts.html

![edge screenshot 29](images/edge/edge-29.png)

![edge screenshot 30](images/edge/edge-30.png)

![edge screenshot 31](images/edge/edge-31.png)

![edge screenshot 32](images/edge/edge-32.png)

Nothing in /Temp

Let’s enumerate C:/

![edge screenshot 33](images/edge/edge-33.png)

What is kiosk?

![edge screenshot 34](images/edge/edge-34.png)

![edge screenshot 35](images/edge/edge-35.png)

We got a process

![edge screenshot 36](images/edge/edge-36.png)

We actually have a lot of users.

Let’s try to run Invoke-AllChecks with powerUp.ps1 and winpeas.

Even rubeus if needed.

![edge screenshot 37](images/edge/edge-37.png)

We have an antivirus, we can’t do anything.

No powershell history.

Internal Ports?

![edge screenshot 38](images/edge/edge-38.png)

No internal ports.

Can we bloodhound?

![edge screenshot 39](images/edge/edge-39.png)

We can’t

Processes?

![edge screenshot 40](images/edge/edge-40.png)

We find Edge Running.

Since the Labs’s name is Edge, I suppose it is something related to edge.

I was not able to retrieve credentials or profiles from edge.

Is there any tool capable of this?

# ACCESS AS SVC_VDI

Indeed, there is :

- EdgeSnapper

https://github.com/Dragkob/EdgeSnapper

`EdgeSnapper is a security research toolkit focused on analyzing cleartext credential persistence within Microsoft Edge process memory.`

![edge screenshot 41](images/edge/edge-41.png)

Let’s try Path Beta, knowing that there is an AV running.

`// To compile on Linux:`

`x86_64-w64-mingw32-g++ edgeSnapper.cpp -o edgeSnapper.exe -static -static-libgcc -static-libstdc++ -ldbghelp -lpsapi`

![edge screenshot 42](images/edge/edge-42.png)

We transfer it and we run it :

![edge screenshot 43](images/edge/edge-43.png)

Cool!  We got password for:

`svc_vdi : V@ntara`

How to move forward?

Let’s try to RDP

![edge screenshot 44](images/edge/edge-44.png)

Successful.

Seems to be in Kiosk mode.

Accessing with svc_vdi creds :

![edge screenshot 45](images/edge/edge-45.png)

What is it?

![edge screenshot 46](images/edge/edge-46.png)

We can see that agriffin and tpham logged in a while ago.

And version :

![edge screenshot 47](images/edge/edge-47.png)

![edge screenshot 48](images/edge/edge-48.png)

There is a site with a released version.

Trying to click it :

![edge screenshot 49](images/edge/edge-49.png)

We can see that there is cmd.exe downloaded.

Can we open it from Edge?

![edge screenshot 50](images/edge/edge-50.png)

![edge screenshot 51](images/edge/edge-51.png)

Opening the file::

![edge screenshot 52](images/edge/edge-52.png)

We got cmd.exe as svc_vdi

# FULL SYSTEM COMPROMISE

![edge screenshot 53](images/edge/edge-53.png)

Not interactive.

So maybe from here we could craft a powershell one-liner, evoking a reverse shell?

![edge screenshot 54](images/edge/edge-54.png)

![edge screenshot 55](images/edge/edge-55.png)

![edge screenshot 56](images/edge/edge-56.png)

![edge screenshot 57](images/edge/edge-57.png)

Can we transfer ncat?

![edge screenshot 58](images/edge/edge-58.png)

![edge screenshot 59](images/edge/edge-59.png)

No output.

It’s blocked.

Can we list trees?

`Tree /f `

![edge screenshot 60](images/edge/edge-60.png)

There is a putty.conf in /Documents.

Let’s try to read it

![edge screenshot 61](images/edge/edge-61.png)

We got an username and a password

`svc_vdi_mgmt : 56tyghbn%^TYGHBN`

Let’s check :

![edge screenshot 62](images/edge/edge-62.png)

Pwned!

svc_vdi_mgmt has root access on the machine.

Let’s win-rm

![edge screenshot 63](images/edge/edge-63.png)

![edge screenshot 64](images/edge/edge-64.png)

# PWNED
