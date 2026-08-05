# MartiniAD — (No creds)

Legenda :

**—** = WRONG/NOT WORKING.

# ENUMERATION

NMAP :

![martiniad screenshot 1](images/martiniad/martiniad-01.png)

Trying RPCEnum :

![martiniad screenshot 2](images/martiniad/martiniad-02.png)

Trying to read SMBShares :

![martiniad screenshot 3](images/martiniad/martiniad-03.png)

Something interesting in notes maybe?

- Login in SMB via anonymous :

![martiniad screenshot 4](images/martiniad/martiniad-04.png)

Maybe checking in notes could reveal something useful

![martiniad screenshot 5](images/martiniad/martiniad-05.png)

Well, found some credentials

`mprice:*martini* `

So now I could check it if they works on crackmapexec

![martiniad screenshot 6](images/martiniad/martiniad-06.png)

It works.

So now we could try :

- RDP access

- WINRM access (even if less probable)

- SMB access to see if we can get more permissions .

Let’s try with crackmapexec --shares to check shares’ permissions

![martiniad screenshot 7](images/martiniad/martiniad-07.png)

Let’see if we can RDP :  — Failed

Let’s see WINRM : — Failed

Ok maybe now we can check some shares such as SYSVOL and enumerate RPC with rpcclient accessing with mprice to see if there are some info leaks :

![martiniad screenshot 8](images/martiniad/martiniad-08.png)

Nothing found in SYSVOL

![martiniad screenshot 9](images/martiniad/martiniad-09.png)

Nothing in NETLOGON

![martiniad screenshot 10](images/martiniad/martiniad-10.png)

With rpcclient we found users.

Interesting users for lateral movement are Athena.t0 and ATHENA_SVC (Service)

Let’s enumerate them using queryuser :

![martiniad screenshot 11](images/martiniad/martiniad-11.png)

![martiniad screenshot 12](images/martiniad/martiniad-12.png)

Nothing.

We have to path now :

1. The quickest one is to run bloodhound-ce-python with user mprice and check how to move forward

1. Try to kerberoast users from our kali.

We’ll go with the first one.

- Let’s run Bloodhound :

![martiniad screenshot 13](images/martiniad/martiniad-13.png)

Well, it’s not working because of LDAP signature…

Let’s try kerberoasting the Athena Service :

- First we generate our hosts file with nxc smb

![martiniad screenshot 14](images/martiniad/martiniad-14.png)

- Then we run impacket-getuserSPNs to try to kerberoast some users :

![martiniad screenshot 15](images/martiniad/martiniad-15.png)

Bingo. We found ATHENA_SVC hash

Now we can crack it with hashcat and try to login again

![martiniad screenshot 16](images/martiniad/martiniad-16.png)

![martiniad screenshot 17](images/martiniad/martiniad-17.png)

Cracked.  The password is :

`1dirtymartini`

Let’s check it out with crackmapexec, checking shares too

![martiniad screenshot 18](images/martiniad/martiniad-18.png)

Same permissions.

So let’s try again :

- RDP

- WINRM

- Bloodhound

RDP :

![martiniad screenshot 19](images/martiniad/martiniad-19.png)

We’re facing this error when RDP

Let’s try WINRM using Evil-Winrm ?

![martiniad screenshot 20](images/martiniad/martiniad-20.png)

Here we are.

We have now to enumerate Athena_svc.

# ATHENA_SVC ENUMERATION

First of all we check privileges with whoami /all

![martiniad screenshot 21](images/martiniad/martiniad-21.png)

We have SeMachineAccountPrivilege :

It allows the user to add workstations to the domain .

This is directly an elevation of privileges cos it allows us to perform a Resource-Based Constrained Delegation (RBCD) attacks

### (WRONG — IT DOESN’T MEAN DIRECTLY RBDC ATTACK)

The path is :

1. Creating a computer account

1. Perform delegation

1. Creating a ticket impersonating Administrator

1. Connecting via Psexec.

Before that, we could double check if Athena has GenericALL or similar on the DC01.

To do that we need to Transfer PowerView and use the cmdlet “Get-Object” filtering with “Where-Object” statement, dumping all the objects with SecurityIdentifier equal to the SID of Athena_SVC

![martiniad screenshot 22](images/martiniad/martiniad-22.png)

Apparently nothing has been found.

Let’s run bloodhound to see if we can double check it

![martiniad screenshot 23](images/martiniad/martiniad-23.png)

After having transferred sharp hound and invoked Bloodhound, no output received.

Hmm, let’s try to see if we can perform RBCD anyhow :

We need first to request a Ticket for Athena_SVC, using : impacket-getTGT

![martiniad screenshot 24](images/martiniad/martiniad-24.png)

Now we have to export it :

![martiniad screenshot 25](images/martiniad/martiniad-25.png)

Now:

1. Create a Computer Account using impacket-addcomputer

![martiniad screenshot 26](images/martiniad/martiniad-26.png)

1. **Add Resource Based Constrained Delegation**

Next, we configure Resource-Based Constrained Delegation, granting our machine account BLACK$ the ability to delegate to (and thus impersonate users on) the domain controller DC01

We’re using BloodyAD

![martiniad screenshot 27](images/martiniad/martiniad-27.png)

So that’s not the way.

Insufficient access rights.

That means that the rights are not enough.

We have to change the way of Privesc.

What about running Rubeus.exe and Lazagne.exe? (Not enough privs for Lazagne)

![martiniad screenshot 28](images/martiniad/martiniad-28.png)

![martiniad screenshot 29](images/martiniad/martiniad-29.png)

![martiniad screenshot 30](images/martiniad/martiniad-30.png)

Nothing.

Let’s enumerate more the machine

![martiniad screenshot 31](images/martiniad/martiniad-31.png)

In inetpub we found DeviceHealthAttestation which contains a dll.

Maybe we could ddl hijack?

Let’s run :

- winpeas to fully enumerate

- Powerup Invoke-AllChecks to see if we can Abuse something.

PowerUp Invoke-AllChecks :

![martiniad screenshot 32](images/martiniad/martiniad-32.png)

It dumped just this path that it’s not interesting to us, since the IdentityRefernce is Athena_svc.

Let’s run WINPEAS :

![martiniad screenshot 33](images/martiniad/martiniad-33.png)

![martiniad screenshot 34](images/martiniad/martiniad-34.png)

From the output it seems that Athena_SVC has GenericALL on Administrator and DC01 , but we have seen that it potentially is a false positive.

Let’s check the **_Powershell History_** , maybe we could find something interesting :

![martiniad screenshot 35](images/martiniad/martiniad-35.png)

Yep, we got admin password.

Now we can connect via RDP, running mimikatz and extracting krbgt hash.

![martiniad screenshot 36](images/martiniad/martiniad-36.png)

Password expired.  Let’s set a new one

![martiniad screenshot 37](images/martiniad/martiniad-37.png)

![martiniad screenshot 38](images/martiniad/martiniad-38.png)

Running mimikatz I got an error : ERROR kuhl_m_sekurlsa_acquireLSA ; Logon list

Browsing on the net, I found that this error could be related to the version of mimikatz.

I could try to search for another version or trying to run Lazagne first.

Let’s see.

Even changing version is not working.

Maybe we can just copy SAM and SYSTEM and dumping it with impacket-secretsdump on local machine

![martiniad screenshot 39](images/martiniad/martiniad-39.png)

![martiniad screenshot 40](images/martiniad/martiniad-40.png)

![martiniad screenshot 41](images/martiniad/martiniad-41.png)

![martiniad screenshot 42](images/martiniad/martiniad-42.png)

![martiniad screenshot 43](images/martiniad/martiniad-43.png)

![martiniad screenshot 44](images/martiniad/martiniad-44.png)

![martiniad screenshot 45](images/martiniad/martiniad-45.png)

![martiniad screenshot 46](images/martiniad/martiniad-46.png)

**NOT WORKING !**

So I’m lost.

Browsing on the net I found this article on Golden Tickets, which talks about how to dump krbtgt hash.

It took me to **another command with mimikatz**.

https://netwrix.com/en/resources/blog/complete-domain-compromise-with-golden-tickets/

`Lsadump::lsa /inject /name:krbtgt `

This dumped immediately the krbtgt hash.

![martiniad screenshot 47](images/martiniad/martiniad-47.png)
