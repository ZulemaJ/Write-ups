# Welcome (CA ESC1 — Certipy)

Credentials obtained :

`e.hills:Il0vemyj0b2025!`

# ENUMERATION

NMAP:

![welcome screenshot 1](images/welcome/welcome-01.png)

## SMB ENUMERATION

![welcome screenshot 2](images/welcome/welcome-02.png)

Human Resources seems interesting

Let’s update the /etc/hosts first

![welcome screenshot 3](images/welcome/welcome-03.png)

![welcome screenshot 4](images/welcome/welcome-04.png)

And now login to smb :

- Impacket-smbclient

![welcome screenshot 5](images/welcome/welcome-05.png)

# INITIAL ACCESS AS A.HARRIS

Trying to open the StartGuide.pdf :

![welcome screenshot 6](images/welcome/welcome-06.png)

We are gonna try the password we already know otherwise we’r gonna use :

- pdf2john  to extract the password and crack it

Since the password didn’t work :

- Pdf2john

![welcome screenshot 7](images/welcome/welcome-07.png)

- Crack it with JTR

![welcome screenshot 8](images/welcome/welcome-08.png)

![welcome screenshot 9](images/welcome/welcome-09.png)

So we know that there is a default password for new users.

## RPC USERS ENUM

We can try now to enumerate users with rpcclient and try password spraying

![welcome screenshot 10](images/welcome/welcome-10.png)

## PASSWORD SPRAY

We create a list of users and we spray passwords with crackmapexec :

![welcome screenshot 11](images/welcome/welcome-11.png)

We found creds :

`a.harris:Welcome2025!@`

Let’s see what we can do with him.

Let’s try to WINRM :

![welcome screenshot 12](images/welcome/welcome-12.png)

We got initial access as a.harris.

# ACCESS AS I.PARK

Let’s run bloodhound to see what we can do and how we can move laterally :

- Bloodhound-ce-python

![welcome screenshot 13](images/welcome/welcome-13.png)

![welcome screenshot 14](images/welcome/welcome-14.png)

We can see that A.HARRIS is member of HR which members have GenericALL on I.PARK

Let’s Abuse it changing password:

![welcome screenshot 15](images/welcome/welcome-15.png)

`net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"`

![welcome screenshot 16](images/welcome/welcome-16.png)

We have a password policy set.

Let’s make it more complex

![welcome screenshot 17](images/welcome/welcome-17.png)

And check :

![welcome screenshot 18](images/welcome/welcome-18.png)

Cool.

We changed successfully i.park’s password.

# ACCESS AS SVC_CA & SVC_WEB

What we can do with i.park?

![welcome screenshot 19](images/welcome/welcome-19.png)

I.PARK member of HELPDESK which members can change password to both svc.

Let’s abuse both

`net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"`

![welcome screenshot 20](images/welcome/welcome-20.png)

With them we cannot WIRM or RDP

How to escalate as a root?

What is svc_ca?

Well at this point I had to check a hint in the walkthrough cos I didn’t know what to do with it.

# ACCESS AS ADMINISTRATOR

Apparently it is a matter of Certificates Service.

Now that we have access as svc_ca we try to identify vulnerable certificate templates using:

- Certipy

https://github.com/ly4k/Certipy

![welcome screenshot 21](images/welcome/welcome-21.png)

Let’s follow the installation:

https://github.com/ly4k/Certipy/wiki/04-%E2%80%90-Installation

Once installed:

![welcome screenshot 22](images/welcome/welcome-22.png)

To look for specific modules’ help, we run :

- Certipy <module> -h

We’re gonna run “Certipy find” with -vulnerable flag, in order to discover just vulnerable certificates

![welcome screenshot 23](images/welcome/welcome-23.png)

In the json :

![welcome screenshot 24](images/welcome/welcome-24.png)

## But what is
## ESC1
##  and
## ESC17
## ?

**ESC1** is an **Active Directory Certificate Services (AD CS)** misconfiguration where a low-privileged user can request a certificate for **any identity** (e.g., Domain Administrator).

This allows the attacker to authenticate as that privileged user without knowing their password, leading to **privilege escalation and potentially full domain compromise**.

**ESC17** is an **AD CS misconfiguration** where a certificate template allows a low-privileged user to request a **Server Authentication** certificate for an arbitrary internal hostname (via a user-controlled SAN). When combined with the ability to manipulate internal DNS records, an attacker can impersonate trusted HTTPS services (such as **WSUS**) and achieve **SYSTEM-level code execution** on clients.

As stated in the .json report, this may require other prerequisites.

We’re definitely gonna go for ESC1

## ESC1 EXPLOITATION

In order to exploit it, we need to request for a certificate on behalf of any specified account

According to :

https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation

![welcome screenshot 25](images/welcome/welcome-25.png)

We’re gonna use:

- certipy req

First, we’re gonna add WELCOME-CA to /etc/hosts according to our certificate :

![welcome screenshot 26](images/welcome/welcome-26.png)

![welcome screenshot 27](images/welcome/welcome-27.png)

Then request the certificate with :

- certipy req

![welcome screenshot 28](images/welcome/welcome-28.png)

We got a certificate for administrator.

How to use it now?

To authenticate as administrator, we need to use :

- Certipy auth -pfx (the private key saved as administrator.pfx)

![welcome screenshot 29](images/welcome/welcome-29.png)

We finally got the hash of administrator.

Let’s check it authenticating with WINRM :

![welcome screenshot 30](images/welcome/welcome-30.png)

# PWNED
