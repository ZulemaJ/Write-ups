# ShadowGate — (AD NoCreds: TimeRoast - ASRepRoast - ESC8 )

# ENUMERATION :

Nmap :

![shadowgate screenshot 1](images/shadowgate/shadowgate-01.png)

## SMB Enumeration
##  w/ :

- Enum4linux-ng

![shadowgate screenshot 2](images/shadowgate/shadowgate-02.png)

We discovered 12 users.

![shadowgate screenshot 3](images/shadowgate/shadowgate-03.png)

We can now write a list and put it aside.

We still have no passwords.

## RPCENUM

Let’s try to query every user with rpcclient to see if there are info leaks, using :

- queryuser <user-rid>

Nothing.

![shadowgate screenshot 4](images/shadowgate/shadowgate-04.png)

No shares.

Are there other ports?

Is port 161 UDP open?

Let’s run a full scan :

![shadowgate screenshot 5](images/shadowgate/shadowgate-05.png)

Port 161 filtered.

![shadowgate screenshot 6](images/shadowgate/shadowgate-06.png)

## HTTP ENUM

It’s weird because port 80, in the initial scan, was not detected, whereas now we know that it’s open thanks to the full scan.

Navigating to it, we find IIS Service.

![shadowgate screenshot 7](images/shadowgate/shadowgate-07.png)

Let’s search for directories using :

- Feroxbuster

![shadowgate screenshot 8](images/shadowgate/shadowgate-08.png)

I used different directories but found nothing.

## AD ENUM ORANGE-CYBERDEFENSE (Credentialsless)

Let’s follow the Orange-cyberdefense roadmap for AD enum with no credentials:

https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg

- Let’s start again with SMB :

![shadowgate screenshot 9](images/shadowgate/shadowgate-09.png)

![shadowgate screenshot 10](images/shadowgate/shadowgate-10.png)

Nothing.

Maybe we can try TimeRoast?

What is TimeRoast?

**`Timeroast`**` is an Active Directory attack that abuses the Windows Time Service (NTP) to obtain encrypted password-related data from a domain controller. The captured data can then be cracked offline to recover weak machine account passwords.`

But first let’s update the /etc/hosts

![shadowgate screenshot 11](images/shadowgate/shadowgate-11.png)

Time to TIMEROAST

`timeroast.py <dc_ip> -o <output_log>`

![shadowgate screenshot 12](images/shadowgate/shadowgate-12.png)

![shadowgate screenshot 13](images/shadowgate/shadowgate-13.png)

Awesome, we got a hash.

Let’s try to crack it with hashcat:

![shadowgate screenshot 14](images/shadowgate/shadowgate-14.png)

We remove the “1000:” at the beginning of the hash, and we crack it :

![shadowgate screenshot 15](images/shadowgate/shadowgate-15.png)

Unsuccessful.

# ACCESS AS JTRUEBLOOD

Maybe we can ASREPRoast some users with no authentication?

I found a cool methodology for asreproasting with no credentials:

https://www.thehacker.recipes/ad/movement/kerberos/roasting/asreproast#practice

1. Using users list dynamically queried with an LDAP anonymous bind

`GetNPUsers.py -request -format hashcat -outputfile ASREProastables.txt -dc-ip "$DC_IP" "$DOMAIN/"`

![shadowgate screenshot 16](images/shadowgate/shadowgate-16.png)

1. With a provided users file :

`GetNPUsers.py -usersfile users.txt -request -format hashcat -outputfile ASREProastables.txt -dc-ip "$DC_IP "$DOMAIN/"`

![shadowgate screenshot 17](images/shadowgate/shadowgate-17.png)

And we got jtrueblood hash.

![shadowgate screenshot 18](images/shadowgate/shadowgate-18.png)

Let’ try to crack it now

![shadowgate screenshot 19](images/shadowgate/shadowgate-19.png)

![shadowgate screenshot 20](images/shadowgate/shadowgate-20.png)

`Jtrueblood : blood_brothers `

Let’s check shares with crackmapexec now :

![shadowgate screenshot 21](images/shadowgate/shadowgate-21.png)

We keep it in mind.

Let’s try first to RDP or WINRM :

![shadowgate screenshot 22](images/shadowgate/shadowgate-22.png)

Winrm Failed

RDP Failed.

## SMBENUM

Let’s check SYSVOL and CertEnroll shares.

![shadowgate screenshot 23](images/shadowgate/shadowgate-23.png)

![shadowgate screenshot 24](images/shadowgate/shadowgate-24.png)

![shadowgate screenshot 25](images/shadowgate/shadowgate-25.png)

![shadowgate screenshot 26](images/shadowgate/shadowgate-26.png)

![shadowgate screenshot 27](images/shadowgate/shadowgate-27.png)

Nothing useful for now.

# ACCESS AS BBROWN :

## BLOODHOUND ENUM

Time to Bloodhound

- Bloodhound-ce-python

![shadowgate screenshot 28](images/shadowgate/shadowgate-28.png)

Now let’s analyze it

![shadowgate screenshot 29](images/shadowgate/shadowgate-29.png)

Bingo.

JtrueBlood has GenericWrite on BBrown.

- Linux Abuse:

![shadowgate screenshot 30](images/shadowgate/shadowgate-30.png)

To abuse it, we can try to kerberoast bbrown, using :

- targetedKerberoast.py

`targetedKerberoast.py -v -d 'domain.local' -u 'controlledUser' -p 'ItsPassword'`

![shadowgate screenshot 31](images/shadowgate/shadowgate-31.png)

And in fact, we got bbrown hash.

Let’s crack it :

![shadowgate screenshot 32](images/shadowgate/shadowgate-32.png)

`bbrown : 12345678`

![shadowgate screenshot 33](images/shadowgate/shadowgate-33.png)

bbrown has no outbound object control.

Let’s try to WINRM and RDP  == both failed

We can see that BBrown is member of ADCS-READER

![shadowgate screenshot 34](images/shadowgate/shadowgate-34.png)

So maybe we can run certipy to discover Certs vulns :

- Certipy-ad

`certipy-ad find -u bbrown -p 12345678 -dc-host DC01.shadow.gate -dc-ip 10.1.63.6 -vulnerable`

![shadowgate screenshot 35](images/shadowgate/shadowgate-35.png)

![shadowgate screenshot 36](images/shadowgate/shadowgate-36.png)

What does it mean?

Let’s browse it :

`ESC8 is an Active Directory Certificate Services (AD CS) privilege escalation vulnerability. It exploits unencrypted, HTTP-based certificate web enrollment endpoints. By coercing a privileged account (like a Domain Controller) to authenticate to an attacker-controlled machine, attackers relay that NTLM authentication to the CA to obtain a highly privileged certificate.`

We found a cool way of exploiting it :

https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation

![shadowgate screenshot 37](images/shadowgate/shadowgate-37.png)

Checking /certsrv :

![shadowgate screenshot 38](images/shadowgate/shadowgate-38.png)

It requests authentication.

Accessing as bbrown:

![shadowgate screenshot 39](images/shadowgate/shadowgate-39.png)

But let’s move ahead following the exploit :

# FULL SYSTEM COMPROMISE

![shadowgate screenshot 40](images/shadowgate/shadowgate-40.png)

**Step 1:**

**Start the Certipy NTLM relay.** 

The attacker starts Certipy in relay mode on a machine they control, configuring it to target the CA's vulnerable web enrollment endpoint.

`certipy relay -target 'https://10.0.0.50' -template 'DomainController'`

![shadowgate screenshot 41](images/shadowgate/shadowgate-41.png)

Certipy is now listening for incoming SMB connections (a common way to capture NTLM auth via coercion or PetitPotam) and is ready to relay to the specified HTTP target.

**Step 2: **

**Coerce authentication from a privileged account to the Certipy relay.** The attacker uses a separate tool (e.g., PetitPotam, Coercer) to force the target (e.g., a Domain Controller DC.CORP.LOCAL or a privileged user Administrator) to attempt an NTLM authentication against the attacker's machine where Certipy's relay is listening.

I found a nice article on PetitPotam PoC :

https://medium.com/r3d-buck3t/domain-takeover-with-petitpotam-exploit-3900f89b38f7

![shadowgate screenshot 42](images/shadowgate/shadowgate-42.png)

![shadowgate screenshot 43](images/shadowgate/shadowgate-43.png)

Something’s wrong .

After debugging, I’ve figured out that -subject flag was missing.

In this case is the DC01 that we want to authenticate.

- -subject “CN=DC01"

![shadowgate screenshot 44](images/shadowgate/shadowgate-44.png)

Succeeded.

**Step 3: **

**Authenticate using the obtained certificate.** 

We now use the .pfx file obtained via relay to authenticate.

`certipy auth -pfx 'dc.pfx' -dc-ip '10.0.0.100'`

![shadowgate screenshot 45](images/shadowgate/shadowgate-45.png)

We got a Hash.

## DCSYNC

Since we have DC01$ hash, we can now run DCSync using:

- impacket-secretsdump

DSYNC allows us to read passwords data of any requested user, impersonating the Domain Controller.

![shadowgate screenshot 46](images/shadowgate/shadowgate-46.png)

# PWNED
