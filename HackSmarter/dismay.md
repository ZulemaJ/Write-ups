# Dismay

**Category:** Active Directory  
**Techniques:** Loot from a foothold, archive password cracking, credential leak in a document, BloodHound ACL abuse chain (ForceChangePassword, GenericAll)

## TL;DR

Starting from an already-obtained low-privilege credential (`xiao.ge`), RDP access to the first host led to a Recycle Bin full of deleted files, including a password-protected `Confidential.7z`. One recovered document (an invoice) contained a set of staging credentials that turned out to be a dead end. The real breakthrough came from cracking the 7z archive with John and rockyou: inside was a fake "Penetration Test Report" whose appendix leaked a plaintext password for `guy.rookie`. That single credential kicked off a BloodHound ACL abuse chain: `guy.rookie` had `ForceChangePassword` over `jena.yamazaki`, who in turn had `GenericAll` over `mike.silver`. Both were compromised in sequence with `net rpc password`, ultimately gaining valid credentials against both domain controllers (DC1 and DC2). The notes end with a second BloodHound sweep against DC1, before any further escalation.

## Full Walkthrough

### Initial Foothold and Loot

Starting point is SMB access as `xiao.ge` against the first target:

![dismay screenshot 1](images/dismay/dismay-01.png)

An Nmap scan confirms a standard Windows host with IIS, SMB, RDP, and WinRM exposed:

![dismay screenshot 2](images/dismay/dismay-02.png)

Port 80 serves the default IIS landing page, nothing custom hosted:

![dismay screenshot 3](images/dismay/dismay-03.png)

RDP access as `xiao.ge`:

![dismay screenshot 4](images/dismay/dismay-04.png)

The Recycle Bin holds several deleted files worth reviewing, including `Confidential.7z`, an invoice draft, a meeting notes archive, and a system audit log:

![dismay screenshot 5](images/dismay/dismay-05.png)

### A Dead End and a Real Lead

`Confidential.7z` is password protected. A first extraction attempt fails:

![dismay screenshot 6](images/dismay/dismay-06.png)

![dismay screenshot 7](images/dismay/dismay-07.png)

Rather than guess further, the hash is extracted with `7z2john` and handed to John the Ripper against rockyou:

![dismay screenshot 8](images/dismay/dismay-08.png)

![dismay screenshot 9](images/dismay/dismay-09.png)

While that runs in the background, one of the other recovered files, `Invoice_Draft_2026_Q2.pdf`, is reviewed. It contains an internal note with a set of temporary staging credentials (`staging_admin` / `Spring_2026_Temp!`) and a database key:

![dismay screenshot 10](images/dismay/dismay-10.png)

This lead doesn't pan out. It's the cracked archive that delivers: the John run against `Confidential.7z` succeeds.

![dismay screenshot 11](images/dismay/dismay-11.png)

Inside is a document styled as a "Confidential: Penetration Test Report" for a fictional internal engagement:

![dismay screenshot 12](images/dismay/dismay-12.png)

Its appendix, meant to illustrate an accidental credential leak, does exactly that: a plaintext password for the account `guy.rookie`.

![dismay screenshot 13](images/dismay/dismay-13.png)

![dismay screenshot 14](images/dismay/dismay-14.png)

### Enumeration DC02

Testing the leaked credential against the domain controller at `10.1.45.17` confirms it's valid, and share enumeration turns up a `CertEnroll` share, a hint that AD Certificate Services is in play:

![dismay screenshot 15](images/dismay/dismay-15.png)

A full port scan of this host shows the expected AD DC footprint (Kerberos, LDAP, Global Catalog, ADCS-related ports):

![dismay screenshot 16](images/dismay/dismay-16.png)

`rpcclient` enumerates the domain users:

![dismay screenshot 17](images/dismay/dismay-17.png)

A quick spray confirms the leaked password belongs to `guy.rookie` only. No password reuse across other accounts:

![dismay screenshot 18](images/dismay/dismay-18.png)

Generating a hosts file for the domain:

![dismay screenshot 19](images/dismay/dismay-19.png)

### BloodHound ACL Abuse Chain

Running `bloodhound-ce-python` as `guy.rookie` against DC2:

![dismay screenshot 20](images/dismay/dismay-20.png)

The collected data shows `guy.rookie` holds `ForceChangePassword` over `jena.yamazaki`:

![dismay screenshot 21](images/dismay/dismay-21.png)

BloodHound's built-in abuse guidance for this edge points to Samba's `net rpc password` for a one-line credential reset:

![dismay screenshot 22](images/dismay/dismay-22.png)

Resetting `jena.yamazaki`'s password as `guy.rookie`:

![dismay screenshot 23](images/dismay/dismay-23.png)

The new credentials work:

![dismay screenshot 24](images/dismay/dismay-24.png)

A second look at the graph shows the chain continues: `jena.yamazaki` holds `GenericAll` over `mike.silver`.

![dismay screenshot 25](images/dismay/dismay-25.png)

Repeating the same technique to reset `mike.silver`'s password:

![dismay screenshot 26](images/dismay/dismay-26.png)

Confirmed, with the same share layout (and the same `CertEnroll` hint) as before:

![dismay screenshot 27](images/dismay/dismay-27.png)

Both `mike.silver` and `jena.yamazaki` also authenticate successfully against the second domain controller, DC1 (`10.1.188.85`), the same domain accounts being valid across both DCs as expected:

![dismay screenshot 28](images/dismay/dismay-28.png)

A fresh BloodHound collection is kicked off against DC1 as `mike.silver` to keep mapping the domain from this new vantage point. The notes end here, mid-chain, with `CertEnroll`/ADCS still an open thread to pull on.

![dismay screenshot 29](images/dismay/dismay-29.png)

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
