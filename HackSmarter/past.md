# Past

# (Active Directory — No Creds - TimeRoast |
# RBCD (Resource Based Constrained Delegation Attack )
#  )

# ENUMERATION:

Nmap scan :

![past screenshot 1](images/past/past-01.png)

The output reveals common ports of a domain controller.

Trying **RPCENUMERATION**:

![past screenshot 2](images/past/past-02.png)

Access Denied.

**SMB Enumeration as Guest **

![past screenshot 3](images/past/past-03.png)

As Guest, we can read IPC$ and Share shares.

**SMB Anonymous Login** :

Using impact-smbclient :

![past screenshot 4](images/past/past-04.png)

In the /Share folder, we find AD_machines.txt which contains the machines related to the current AD.

## AD methodology NO CREDS:

Since we do not have any kind of leaked credentials, and we don’t know how to move further, we will follow the Orange-cyberdefense methodology which is specifically for AD with no Creds.

https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg

1. First of all we use **nxc smb** to **generate a host file**

![past screenshot 5](images/past/past-05.png)

And adding it to /etc/hosts

![past screenshot 6](images/past/past-06.png)

We have then tried different other methodology commands but with no success.

## TIMEROASTING :

The one that has success was **timeroast**, using **timeroast.py**

**_`Timeroast`_**_` is an Active Directory attack that exploits predictable timestamps in the Windows Time (NTP/SNTP) service to obtain password hashes of machine accounts. An attacker sends specially crafted time requests, extracts the returned cryptographic material, and attempts to crack the machine account password offline.`_

- Running timeroast.py with the output log

![past screenshot 7](images/past/past-07.png)

- We succeeded to retrieve some hashes with their corresponding RID at the beginning!

![past screenshot 8](images/past/past-08.png)

**CRACKING SNTP HASHES WITH HASHCAT **

![past screenshot 9](images/past/past-09.png)

![past screenshot 10](images/past/past-10.png)

![past screenshot 11](images/past/past-11.png)

The output reveals that we have finally cracked one of the hashed password.

## CHECKING THE RIDs :

To check which machine/user’s password we have cracked, we use nxc smb with --rid flag, which will list the RIDs of the domain.

![past screenshot 12](images/past/past-12.png)

Since the password we have cracked was associated to the **RID 1115**, we now know that **we have cracked the APPDEV01$ password**.

- Checking if we are right, using crackmapexec :

![past screenshot 13](images/past/past-13.png)

… and listing accessible shares :

![past screenshot 14](images/past/past-14.png)

- We can now login to SMB with user APPDEV01$ , to read SYSVOL share :

![past screenshot 15](images/past/past-15.png)

![past screenshot 16](images/past/past-16.png)

![past screenshot 17](images/past/past-17.png)

In the /past.local/scripts we finally found **tyler_init.cmd** which contains Tyler’s password.

# AUTHENTICATING AS “TYLER”

Trying to Auth as Tyler using crackmapexec, we face the “ACCOUNT_RESTRICTION” error.

![past screenshot 18](images/past/past-18.png)

To work around this, Kerberos authentication can be used instead.

By requesting a Ticket Granting Ticket (TGT), we authenticate via Kerberos, which does not enforce the same logon restrictions.

**REQUESTING A TGT :**

- Using impacket-getTGT

![past screenshot 19](images/past/past-19.png)

- Now that the ticket has been forged, we export it :

![past screenshot 20](images/past/past-20.png)

- And we try again to auth as Tyler but this time with Kerberos using kcache

![past screenshot 21](images/past/past-21.png)

**
# BloodHound Enumeration**

Since we have now a user account with credentials and Kerberos TGT, we can proceed to run **_Bloodhound-ce-python_** which will work from our local host.

We will use :

- -k = for kerberos

- -d = domain

- -dc = domain controller

- -ns = nameserver

- -c = Collectionmethods

- --zip = to obtain a zip file.

![past screenshot 22](images/past/past-22.png)

- Once set Bloodhound up and uploaded the .json files, inspecting it, we found that **TYLER HAS GENERIC ALL PERMISSION over the DOMAIN CONTROLLER.**

![past screenshot 23](images/past/past-23.png)

This allows us to perform a **_Resource-Based Constrained Delegation attack_**:

`add a computer, configure delegation, and impersonate a privileged user.`

![past screenshot 24](images/past/past-24.png)

# RESOURCE-BASED CONSTRAINED DELEGATION ATTACK

We could use the above exploit methodology explained in BloodHound, but this time we will use

## BloodyAD

1. **Create a Computer Account** :

![past screenshot 25](images/past/past-25.png)

1. **Add Resource Based Constrained Delegation**

Next, we configure Resource-Based Constrained Delegation, granting our machine account BLACK$ the ability to delegate to (and thus impersonate users on) the domain controller EC2AMAZ-A5O4OL8$

![past screenshot 26](images/past/past-26.png)

1. Unset the KRB5CCNAME :

![past screenshot 27](images/past/past-27.png)

1. **Requesting a Service Ticket ST**

Using :

- Impacket-getST

Request a ST for the cifs/EC2AMAZ-A5O4OL8.past.local SPN while impersonating the Administrator's account, leveraging the delegation rights of our controlled machine account BLACK$ to obtain a usable ticket as the domain admin.

![past screenshot 28](images/past/past-28.png)

1. Exporting the Ticket :

![past screenshot 29](images/past/past-29.png)

1. Trying to auth as Administrator with nxc , using kerberos Auth (-k) and kcache

![past screenshot 30](images/past/past-30.png)

We did it!

# DUMPING SECRETS AND LOGGING AS ADMIN :

Now that we have a usable ST as Admin,

We can dump secrets using impacket-secretsdump with Kerberos Authentication

![past screenshot 31](images/past/past-31.png)

![past screenshot 32](images/past/past-32.png)

- With the hashes obtained, we could log in winrm as Administrator through **evil-winrm** :

![past screenshot 33](images/past/past-33.png)

We’re in.

- To search for Ryan’s password, we checked the **History.txt** :

![past screenshot 34](images/past/past-34.png)

![past screenshot 35](images/past/past-35.png)

… and we obtained it.

# PAWNED !
