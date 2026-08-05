# Share The Pain — (NTLM Theft — xp_cmdshell — Impersonate)

# ENUMERATION :

- Nmap :

![share-the-pain screenshot 1](images/share-the-pain/share-the-pain-01.png)

Common DC ports.

We do not have an HTTP open.

Since we do not have any credentials, we should check :

- SMB

- RPC

We first run Enum4Linux-ng to see if there’s something juicy

![share-the-pain screenshot 2](images/share-the-pain/share-the-pain-02.png)

- Enum4linux dumped some shares that we will check manually with SMBMap to check permissions.

- RPC access is apparently denied.  But we will check it manually btw.

- SMBMap :

![share-the-pain screenshot 3](images/share-the-pain/share-the-pain-03.png)

We have **R/W permissions** just in “Share”

![share-the-pain screenshot 4](images/share-the-pain/share-the-pain-04.png)

As stated before, no access to RPC.

## SMBENUM :

- Let’s access “Share” share, to see if we can find something inside.  We’ll use as usual impacket-smbclient

![share-the-pain screenshot 5](images/share-the-pain/share-the-pain-05.png)

Nothing inside.

What’s particularly interesting is the fact that we have write permissions.

Maybe, if there is some HTTP port open, it could be linked to the share, meaning that if we can put a shell on it, we could open it via HTTP ?

From our scan, no HTTP open.

Or maybe, is there 161 UDP open?  From this, we could enumerate users.

- Let’s scan all TCP ports w/ NMAP and UDP 161.

![share-the-pain screenshot 6](images/share-the-pain/share-the-pain-06.png)

I found 2 more TCP ports, but nothing interesting.

- Let’s check now 161 UDP :

![share-the-pain screenshot 7](images/share-the-pain/share-the-pain-07.png)

Ok we have to change strategy.

Let’s see if we can find something running Script Scan w Nmap :

![share-the-pain screenshot 8](images/share-the-pain/share-the-pain-08.png)

Nothing.

So we have no clues.

I remember I pwned a machine in Offsec which was pretty the same situation.

What I used there was

**NTLM Theft. **

That’s the article where I Got it :

https://github.com/Greenwolf/ntlm_theft

`It can be used for phishing when either the target allows smb traffic outside their network, or if you are already inside the internal network.`

Let’s try it .

# NTLM THEFT — GAINING BOB CREDS.

So according to the repository,

- we clone it first :

![share-the-pain screenshot 9](images/share-the-pain/share-the-pain-09.png)

- We install xlswriter with pipx, which is needed.

![share-the-pain screenshot 10](images/share-the-pain/share-the-pain-10.png)

- We run a responder in another terminal

![share-the-pain screenshot 11](images/share-the-pain/share-the-pain-11.png)

- We run the script ntlm_theft.py to generate the files. ( -s is the attacker ip)

![share-the-pain screenshot 12](images/share-the-pain/share-the-pain-12.png)

We can see that it generates many kinds of files.  We’r gonna use the ones with “BROWSE TO FOLDER” statement, which do not require opening the file.

It is enough that Windows visualizes the folder.

- Since I didn’t know which one to put, I uploaded the two :

- lure.lnk

- desktop.ini

![share-the-pain screenshot 13](images/share-the-pain/share-the-pain-13.png)

- And on our responder :

![share-the-pain screenshot 14](images/share-the-pain/share-the-pain-14.png)

## We got the NTLMv2 of user bob.ross !

We need first to crack it using Hashcat :

![share-the-pain screenshot 15](images/share-the-pain/share-the-pain-15.png)

![share-the-pain screenshot 16](images/share-the-pain/share-the-pain-16.png)

The password is :

`137Password123!@#`

Now that we have credentials, we can check :

- Permissions on shares with crackmapexec

- RDP access

- Winrm access

- RPCEnum

## SMBENUM

- Shares permissions with Crackmapexec:

![share-the-pain screenshot 17](images/share-the-pain/share-the-pain-17.png)

We keep in mind that we have READ permissions on SYSVOL.

- Checking SYSVOL :

![share-the-pain screenshot 18](images/share-the-pain/share-the-pain-18.png)

Nothing.

- RDP : failed

- WINRM : failed

- RPC Enum

![share-the-pain screenshot 19](images/share-the-pain/share-the-pain-19.png)

We got users.

- We could query users with “queryuser”  to see if there are info leaks :

![share-the-pain screenshot 20](images/share-the-pain/share-the-pain-20.png)

No leaks.

We write down users btw.

We’re gonna need them later.

What should we do now?   We got a user and its credentials. But no access.

We could try to :

- Run bloodhound-ce-python

- Kerberoast some users

## KERBEROASTING

Let’s start with kerberoasting.

We’re gonna use : impacket-getUserSPNs

![share-the-pain screenshot 21](images/share-the-pain/share-the-pain-21.png)

Nothing.

## BLOODHOUND

- Running Bloodhound-ce-python :

![share-the-pain screenshot 22](images/share-the-pain/share-the-pain-22.png)

We got the zip file.

- Let’s analyze it on Bloodhound so that we can first check what kind of OUTBOUND OBJECT CONTROLS are assigned to bob.ross

![share-the-pain screenshot 23](images/share-the-pain/share-the-pain-23.png)

Bingo.

We can see that bob.ross has “**GenericALL**” on Alice.Wonderland.

## With this ACL set, we can change Alice’s password directly from our system.

Reading the Linux Abuse on Bloodhound we can see the commands to run :

![share-the-pain screenshot 24](images/share-the-pain/share-the-pain-24.png)

# FORCE CHANGE PASSWORD

1. We first need to generate and add the hosts file to the /etc/hosts file.

- Generate it with nxc smb :

![share-the-pain screenshot 25](images/share-the-pain/share-the-pain-25.png)

- Add it to /etc/hosts

![share-the-pain screenshot 26](images/share-the-pain/share-the-pain-26.png)

1. Now try to change the password using “net rpc password”

![share-the-pain screenshot 27](images/share-the-pain/share-the-pain-27.png)

No output means “Success”.

# ACCESS AS ALICE.WONDERLAND

Now we start enum again, trying :

- Check SMB permissions and if password has successfully changed with crackmapexec

- RDP

- WINRM

- Crackmapexec

![share-the-pain screenshot 28](images/share-the-pain/share-the-pain-28.png)

Password has successfully changed.

- RDP = failed

- WINRM  w/ evil-winrm

![share-the-pain screenshot 29](images/share-the-pain/share-the-pain-29.png)

We’re in.

We now have to Enumerate Alice.wonderland to see how we’r gonna elevate our privs.

But first, we can maybe check on Bloodhound marking the users as owned to see if there is a shortest path from owned objects.

Nothing from here.

Alice seems to have NO OUTBOUND OBJECTS CONTROL

# ALICE ENUMERATION :

- Let’s enumerate Alice and DC01 from win-rm whoami /all

![share-the-pain screenshot 30](images/share-the-pain/share-the-pain-30.png)

No interesting privileges

- Checking dirs :

![share-the-pain screenshot 31](images/share-the-pain/share-the-pain-31.png)

Something interesting in SQL2019??

![share-the-pain screenshot 32](images/share-the-pain/share-the-pain-32.png)

Denied.

- Something interesting in Powershell history?

-
![share-the-pain screenshot 33](images/share-the-pain/share-the-pain-33.png)

Even checking manually there is no Powershell folder apparently.

- Let’s check the Desktop to see if we can find something :

![share-the-pain screenshot 34](images/share-the-pain/share-the-pain-34.png)

We found something encoded in base64.

Credentials??

Let’s decode it :

![share-the-pain screenshot 35](images/share-the-pain/share-the-pain-35.png)

Is it a password?

- Let’s check if it’s tyler.ramsey password :

![share-the-pain screenshot 36](images/share-the-pain/share-the-pain-36.png)

Nope.

So it’s just the **FUCKING** user flag for the lab.

- Let’s run winpeas : Nothing interesting found.

## What is the path?

- I could try to run Invoke-AllChecks to see if there is some possibilities of hijacking? (PowerUp.ps1)

![share-the-pain screenshot 37](images/share-the-pain/share-the-pain-37.png)

Nothing.

## Maybe there is some internal port that I could access?

Let’s try to list again all TCP ports, probably I missed something:

- use : netstat -ant -p tcp

![share-the-pain screenshot 38](images/share-the-pain/share-the-pain-38.png)

That’s the **MSSQL port 1433**.

What we should do now is to pivot the port with ligolo, then try to access it from our kali

# PORT FORWARDING

- Start ligolo proxy on kali :

![share-the-pain screenshot 39](images/share-the-pain/share-the-pain-39.png)

- Download the ligolo-agent.exe for windows and transfer it to the target

Once transferred :

- Connect with -ignore-cert flag

![share-the-pain screenshot 40](images/share-the-pain/share-the-pain-40.png)

![share-the-pain screenshot 41](images/share-the-pain/share-the-pain-41.png)

- Now we can forward the port 1433

![share-the-pain screenshot 42](images/share-the-pain/share-the-pain-42.png)

![share-the-pain screenshot 43](images/share-the-pain/share-the-pain-43.png)

- Checking with NC :

![share-the-pain screenshot 44](images/share-the-pain/share-the-pain-44.png)

Now we can try to login :

![share-the-pain screenshot 45](images/share-the-pain/share-the-pain-45.png)

![share-the-pain screenshot 46](images/share-the-pain/share-the-pain-46.png)

## Of course putain

I got stucked as fuck

What’s next?

I suppose I have to move laterally to tyler_ramsey?

But how?

I need creds for SQL but no config files…

At this point I really don’t know what to do

## Maybe changing tool for forwarding?

# ACCESS MSSQL :

Let’s try with chisel :

- After having downloaded and transferred it :

![share-the-pain screenshot 47](images/share-the-pain/share-the-pain-47.png)

- On kali, start chisel server

![share-the-pain screenshot 48](images/share-the-pain/share-the-pain-48.png)

- On windows, chisel client :

![share-the-pain screenshot 49](images/share-the-pain/share-the-pain-49.png)

![share-the-pain screenshot 50](images/share-the-pain/share-the-pain-50.png)

![share-the-pain screenshot 51](images/share-the-pain/share-the-pain-51.png)

## Maybe we should try to request a ticket and enter with kerberos auth?

![share-the-pain screenshot 52](images/share-the-pain/share-the-pain-52.png)

![share-the-pain screenshot 53](images/share-the-pain/share-the-pain-53.png)

![share-the-pain screenshot 54](images/share-the-pain/share-the-pain-54.png)

## So maybe it’s just a matter of -windows-auth flag !

![share-the-pain screenshot 55](images/share-the-pain/share-the-pain-55.png)

Fuck!!!!!!

Why am I so stupid?

We’re in.

# ACCESS AS SERVICE\MSSQL

-  enumerate the db and see if we can gain a shell

![share-the-pain screenshot 56](images/share-the-pain/share-the-pain-56.png)

Nothing

- Can we impersonate?

![share-the-pain screenshot 57](images/share-the-pain/share-the-pain-57.png)

- Can we enable xp_cmdshell?

![share-the-pain screenshot 58](images/share-the-pain/share-the-pain-58.png)

![share-the-pain screenshot 59](images/share-the-pain/share-the-pain-59.png)

It works.

## We have now to spawn a reverse shell with a powershell one-liner :

- Encode the Powershell one-liner in base64

![share-the-pain screenshot 60](images/share-the-pain/share-the-pain-60.png)

- Set up a listener w/ Penelope

![share-the-pain screenshot 61](images/share-the-pain/share-the-pain-61.png)

- Run the payload with xp_cmdshell

![share-the-pain screenshot 62](images/share-the-pain/share-the-pain-62.png)

![share-the-pain screenshot 63](images/share-the-pain/share-the-pain-63.png)

We’re in again.

# PRIVILEGE ESCALATION

- Enumerating privs of the service :

![share-the-pain screenshot 64](images/share-the-pain/share-the-pain-64.png)

Let’s impersonate privs with Potatos or PrintSpoofer :

- With Printspoofer (after having transferred netcat) :

![share-the-pain screenshot 65](images/share-the-pain/share-the-pain-65.png)

![share-the-pain screenshot 66](images/share-the-pain/share-the-pain-66.png)

# PAWNED
