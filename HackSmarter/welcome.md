# Welcome

**Category:** Active Directory Certificate Services (AD CS)  
**Techniques:** PDF password cracking, RPC user enumeration, password spraying, BloodHound ACL abuse (GenericAll), ESC1 certificate template abuse

## TL;DR

Starting from one known credential, SMB enumeration led to a password-protected PDF cracked with `pdf2john` and John, revealing a default password pattern for new users. RPC enumeration plus a password spray against that pattern turned up valid credentials for `a.harris`. From there, a BloodHound-mapped `GenericAll` chain was walked twice: `a.harris` (via HR group membership) reset `i.park`'s password, and `i.park` (via Helpdesk group membership) reset both `svc_ca` and `svc_web`. Neither service account allowed WinRM or RDP, but `svc_ca` pointed toward AD CS. Running Certipy against the CA turned up an **ESC1** misconfiguration: a low-privileged account could request a certificate impersonating any identity, including Administrator. Requesting that certificate and authenticating with it yielded the Administrator hash and full domain compromise.

## Full Walkthrough

### Starting Credentials

```
e.hills:Il0vemyj0b2025!
```

### Enumeration

Nmap:

![welcome screenshot 1](images/welcome/welcome-01.png)

### SMB Enumeration

![welcome screenshot 2](images/welcome/welcome-02.png)

A "Human Resources" share stands out. First, updating `/etc/hosts`:

![welcome screenshot 3](images/welcome/welcome-03.png)

![welcome screenshot 4](images/welcome/welcome-04.png)

Logging into SMB with `impacket-smbclient`:

![welcome screenshot 5](images/welcome/welcome-05.png)

### Initial Access as a.harris

Attempting to open `StartGuide.pdf`:

![welcome screenshot 6](images/welcome/welcome-06.png)

The known password doesn't open it, so the hash is extracted with `pdf2john` and cracked:

![welcome screenshot 7](images/welcome/welcome-07.png)

![welcome screenshot 8](images/welcome/welcome-08.png)

![welcome screenshot 9](images/welcome/welcome-09.png)

The cracked password turns out to be a default assigned to new users, a good candidate for spraying.

**RPC user enumeration:**

![welcome screenshot 10](images/welcome/welcome-10.png)

**Password spray:** building a user list and spraying the default password with CrackMapExec:

![welcome screenshot 11](images/welcome/welcome-11.png)

Valid credentials come back:

```
a.harris:Welcome2025!@
```

Testing WinRM with them:

![welcome screenshot 12](images/welcome/welcome-12.png)

Initial access confirmed as `a.harris`.

### Access as i.park

Running `bloodhound-ce-python` to map lateral movement options:

![welcome screenshot 13](images/welcome/welcome-13.png)

![welcome screenshot 14](images/welcome/welcome-14.png)

`a.harris` is a member of the HR group, which holds `GenericAll` over `i.park`. Abusing it by resetting the password:

![welcome screenshot 15](images/welcome/welcome-15.png)

```
net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"
```

![welcome screenshot 16](images/welcome/welcome-16.png)

A password policy blocks the first attempt. Using a more complex password instead:

![welcome screenshot 17](images/welcome/welcome-17.png)

Confirming the change:

![welcome screenshot 18](images/welcome/welcome-18.png)

`i.park`'s password is successfully reset.

### Access as svc_ca and svc_web

Checking what `i.park` can reach:

![welcome screenshot 19](images/welcome/welcome-19.png)

`i.park` is a member of the Helpdesk group, which can reset the password of both service accounts. Abusing both the same way:

```
net rpc password "TargetUser" "newP@ssword2022" -U "DOMAIN"/"ControlledUser"%"Password" -S "DomainController"
```

![welcome screenshot 20](images/welcome/welcome-20.png)

Neither account allows WinRM or RDP. The name `svc_ca` is the more interesting lead: it points at Certificate Services rather than a general-purpose account.

### Access as Administrator

With that in mind, the next step is checking the CA itself for vulnerable certificate templates, using [Certipy](https://github.com/ly4k/Certipy):

![welcome screenshot 21](images/welcome/welcome-21.png)

Following the [installation guide](https://github.com/ly4k/Certipy/wiki/04-%E2%80%90-Installation):

![welcome screenshot 22](images/welcome/welcome-22.png)

Running `certipy find -vulnerable` to surface only vulnerable templates:

![welcome screenshot 23](images/welcome/welcome-23.png)

The resulting JSON report:

![welcome screenshot 24](images/welcome/welcome-24.png)

Two findings stand out: **ESC1** and **ESC17**.

**ESC1** is an AD CS misconfiguration where a low-privileged user can request a certificate for any identity, including Domain Administrator. This lets an attacker authenticate as that privileged user without ever knowing their password, leading to privilege escalation and potentially full domain compromise.

**ESC17** is an AD CS misconfiguration where a certificate template allows a low-privileged user to request a Server Authentication certificate for an arbitrary internal hostname, via a user-controlled SAN. Combined with the ability to manipulate internal DNS records, this can let an attacker impersonate trusted HTTPS services (such as WSUS) and achieve SYSTEM-level code execution on clients.

The JSON report notes ESC17 may need extra prerequisites here, so ESC1 is the path taken.

**Exploiting ESC1:** the plan is to request a certificate on behalf of any specified account, per [Certipy's privilege escalation guide](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation):

![welcome screenshot 25](images/welcome/welcome-25.png)

First, adding `WELCOME-CA` to `/etc/hosts` to match the certificate:

![welcome screenshot 26](images/welcome/welcome-26.png)

![welcome screenshot 27](images/welcome/welcome-27.png)

Requesting the certificate with `certipy req`:

![welcome screenshot 28](images/welcome/welcome-28.png)

A certificate for Administrator comes back. Authenticating with it using `certipy auth -pfx` against the saved `administrator.pfx`:

![welcome screenshot 29](images/welcome/welcome-29.png)

The Administrator hash is recovered. Confirming access over WinRM:

![welcome screenshot 30](images/welcome/welcome-30.png)

Domain Admin, confirmed.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
