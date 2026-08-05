# Stellar — (AV — Firefox Data — DCSYNC)

# ENUMERATION :

NMAP:

![stellar screenshot 1](images/stellar/stellar-01.png)

Ports as usual.

## FTP ENUM

Let’s check FTP if anon is allowed:

![stellar screenshot 2](images/stellar/stellar-02.png)

It is.

What’s inside?

![stellar screenshot 3](images/stellar/stellar-03.png)

Docs seem to be interesting.

Let’s check it out:

![stellar screenshot 4](images/stellar/stellar-04.png)

We’r gonna download them and then check them.

When opening the pdf :

![stellar screenshot 5](images/stellar/stellar-05.png)

Let’s keep it aside for now and continue enumerating.

## ENUM4LINUX :

![stellar screenshot 6](images/stellar/stellar-06.png)

## HTTP ENUM:

PORT 80:

![stellar screenshot 7](images/stellar/stellar-07.png)

Nothing wow.

Any dirs?

- Feroxbuster :

![stellar screenshot 8](images/stellar/stellar-08.png)

![stellar screenshot 9](images/stellar/stellar-09.png)

Well, nothing.

## SMB ENUM :

![stellar screenshot 10](images/stellar/stellar-10.png)

Can’t find nothing for now.

I think that the only way should be through some documents in ftp.

But it dumps always an error like “file damaged”

## FTP ENUM (AGAIN):

Let’s use another tool to open the pdf :

- Mupdf

It works :

![stellar screenshot 11](images/stellar/stellar-11.png)

In the userGuide we found a default password.

`Galaxy123!`

All users are required to change their password immediately after their first login.

What if junior.analyst has not changed the password yet?

Let’s check if junior.analyst has changed its password:

![stellar screenshot 12](images/stellar/stellar-12.png)

And indeed he hasn’t.

Let’s see which shares we can access w crackmapexec

![stellar screenshot 13](images/stellar/stellar-13.png)

SYSVOL always interesting

Let’s check it , but first let’s update /etc/hosts

![stellar screenshot 14](images/stellar/stellar-14.png)

Login to SMB :

![stellar screenshot 15](images/stellar/stellar-15.png)

![stellar screenshot 16](images/stellar/stellar-16.png)

Nothing interesting inside.

## RPCENUM :

Let’s gather users through rpcclient

![stellar screenshot 17](images/stellar/stellar-17.png)

Cool

Let’s now try to WINRM and RDP

WINRM = Failed

![stellar screenshot 18](images/stellar/stellar-18.png)

RDP = Failed

Ok, we’r gonna work on our kali.

# INITIAL ACCESS AS OPS.CONTROLLER :

## BLOODHOUND ENUM :

Let’s try to run bloodhound remotely first w/ :

- bloodhound-ce-python

![stellar screenshot 19](images/stellar/stellar-19.png)

And now let’s see if we have some misconfigurations :

![stellar screenshot 20](images/stellar/stellar-20.png)

And we have write-owner on Stellarops-control !

What we can do with it?

Checking in Bloodhound “Linux Abuse” :

![stellar screenshot 21](images/stellar/stellar-21.png)

We can add members to the group

Let’s inspect first this group :

![stellar screenshot 22](images/stellar/stellar-22.png)

Stellarops memebrs have ForceChange Password on OPS.CONTROLLER.

Let’s add junior.analyst to the group so that we can change the password of OPS.CONTROLLER and move laterally

Following the instructions :

`net rpc group addmem "TargetGroup" "TargetUser" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"`

![stellar screenshot 23](images/stellar/stellar-23.png)

So maybe we have to change the permissions first

`dacledit.py -action 'write' -rights 'WriteMembers' -principal 'controlledUser' -target-dn 'groupDistinguishedName' 'domain'/'controlledUser':'password'`

![stellar screenshot 24](images/stellar/stellar-24.png)

Well, maybe it is a matter of Distinguished name.

Let’s put the entire one :

![stellar screenshot 25](images/stellar/stellar-25.png)

![stellar screenshot 26](images/stellar/stellar-26.png)

Why??

Checking better on Bloodhound:

We first need to change the membership with :

`owneredit.py -action write -owner 'attacker' -target 'victim' 'DOMAIN'/'USER':'PASSWORD'`

![stellar screenshot 27](images/stellar/stellar-27.png)

## I put new-owner flag instead of -owner , cos “owner” is no longer supported

And again :

![stellar screenshot 28](images/stellar/stellar-28.png)

And now we can add him to the group :

![stellar screenshot 29](images/stellar/stellar-29.png)

No output == success.

Let’s now Change the password of OPS.CONTROLLER

![stellar screenshot 30](images/stellar/stellar-30.png)

![stellar screenshot 31](images/stellar/stellar-31.png)

And check :

![stellar screenshot 32](images/stellar/stellar-32.png)

Yes!

Let’s go ahead

Ops.controller has no outbound object control.

Let’s try to WINRM:

![stellar screenshot 33](images/stellar/stellar-33.png)

Failed

RDP?  == failed

![stellar screenshot 34](images/stellar/stellar-34.png)

Weird, cos he is member of Remote Management.

Let’s try again winrm

Changing -i with ip address (WHATTHEFUCK Chloe)

![stellar screenshot 35](images/stellar/stellar-35.png)

We’re in :)

# ENUMERATION AGAIN :

Let’s check powershell history

![stellar screenshot 36](images/stellar/stellar-36.png)

No history?

Checking Kerberoastable users on Bloodhound, we found :

![stellar screenshot 37](images/stellar/stellar-37.png)

Let’s try to kerberoast it with Rubeus :

![stellar screenshot 38](images/stellar/stellar-38.png)

Possible AV

Let’s try from our kali :

![stellar screenshot 39](images/stellar/stellar-39.png)

That’s weird

And asreproast?

![stellar screenshot 40](images/stellar/stellar-40.png)

Let’s run winpeas

![stellar screenshot 41](images/stellar/stellar-41.png)

Of course we can’t.

Listing processes:

![stellar screenshot 42](images/stellar/stellar-42.png)

We can see that Microsoft Defender is running.

Can we stop it?

![stellar screenshot 43](images/stellar/stellar-43.png)

We do not have permissions.

Listing Scheduled Tasks :

![stellar screenshot 44](images/stellar/stellar-44.png)

Not permitted.

![stellar screenshot 45](images/stellar/stellar-45.png)

SatLink Service has:

- DCSync

- GetChanges

- GetChangesInFilteredSet

- GetChangesALL on the DC01.

This should be our way of privesc!

But how?

On Bloodhound it says that it is kerberoastable, but we found nothing.

## Let’s maybe use other tools?

Searching on Hacktricks I found some other ways of checking Kerberoastable users:

- One way is using ADEnum : https://github.com/SecuProject/ADenum

After having installed dependencies and run it:

![stellar screenshot 46](images/stellar/stellar-46.png)

![stellar screenshot 47](images/stellar/stellar-47.png)

No Entry found.

I wanted to check with Powerview, but :

![stellar screenshot 48](images/stellar/stellar-48.png)

Blocked.

I need to change direction.

I’m in a rabbit hole.

Let’s enumerate a bit more again :

![stellar screenshot 49](images/stellar/stellar-49.png)

We can see that there is a Firefox setup in /ops.controller/Desktop.

They’re using Firefox then.

What if we could retrieve profiles info?

Following the Hacktricks guide on Browser Artifacts :

https://hacktricks.wiki/en/generic-methodologies-and-resources/basic-forensic-methodology/specific-software-file-type-tricks/browser-artifacts.html

We found :

![stellar screenshot 50](images/stellar/stellar-50.png)

And :

![stellar screenshot 51](images/stellar/stellar-51.png)

We’re gonna download the entire directory with “download*” (being in evil-winrm)

![stellar screenshot 52](images/stellar/stellar-52.png)

Once everything is downloaded, we can check better on our kali.

What’s inside?

Hacktricks also describes the kind of information that we could find inside :

![stellar screenshot 53](images/stellar/stellar-53.png)

It also talks about a way of decrypting the master password using :

https://github.com/unode/firefox_decrypt

![stellar screenshot 54](images/stellar/stellar-54.png)

## DECRYPTING MASTER PASSWORD:

Let’s run it :

![stellar screenshot 55](images/stellar/stellar-55.png)

And we found creds!

`astro.researcher:Cosmos@42`

On Bloodhound, let’s check Outbound Object Control:

![stellar screenshot 56](images/stellar/stellar-56.png)

And we have WriteDacl on ENG.PAYLOAD.

What can we do with it?

![stellar screenshot 57](images/stellar/stellar-57.png)

We can either Change Password (after having assigned privs), or Kerberoast it.

Let’s change password:

1. ASSIGNING RIGHTS :

`dacledit.py -action 'write' -rights 'FullControl' -principal 'controlledUser' -target 'targetUser' 'domain'/'controlledUser':'password'`

![stellar screenshot 58](images/stellar/stellar-58.png)

1. CHANGING PASSWORD:

`net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"`

![stellar screenshot 59](images/stellar/stellar-59.png)

1. Check :

![stellar screenshot 60](images/stellar/stellar-60.png)

And we’ve done it.

Now that we owned eng.payload, let’s see its Outbound Object Control :

![stellar screenshot 61](images/stellar/stellar-61.png)

## ReadGMSAPassword.

![stellar screenshot 62](images/stellar/stellar-62.png)

Let’s try to read the password:

We’re gonna use BloodyAD to try to read the password :

https://www.thehacker.recipes/ad/movement/dacl/readgmsapassword

![stellar screenshot 63](images/stellar/stellar-63.png)

![stellar screenshot 64](images/stellar/stellar-64.png)

And we got the base64 encoded and NT password.

Check it :

![stellar screenshot 65](images/stellar/stellar-65.png)

We don’t need explicitly to crack it for now.

# ACCESS AS ADMINISTRATOR :

As we’ve seen before that we can DCSync, meaning that we can impersonate the Domain Controller requesting any user’s password.

Let’s ask for it using:

- Impacket-secretsdumps

Since we do not have the cleartext password, we’re gonna use the flag “-hashes” (NTLM format)

![stellar screenshot 66](images/stellar/stellar-66.png)

And we got Administrator Hash :)

![stellar screenshot 67](images/stellar/stellar-67.png)

# PWNED
