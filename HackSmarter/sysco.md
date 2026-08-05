# Sysco

**Category:** Active Directory, No Credentials

## Attack Chain

1. TimeRoasting and common webmail credentials fail; ASREPRoasting with guessed usernames hits an LDAP bind error
2. `username-anarchy` generates realistic usernames from names found on the site
3. ASREPRoasting with the generated list cracks `jack.dowland`'s hash
4. `jack.dowland` has no useful BloodHound edges but does have Roundcube webmail access
5. A config file in the mailbox leaks an MD5 secret, cracked and sprayed to land `lainey.moore`
6. RDP as Lainey finds a saved PuTTY session with router credentials, reused by `greg.shields`
7. `greg.shields` belongs to Group Policy Creator Owners, full control of the Default Domain Policy
8. `pyGPOAbuse` pushes a scheduled task that creates a new local administrator

## TL;DR

Starting with no credentials, several early leads went nowhere: TimeRoasting returned no hash, common credentials against a discovered Roundcube webmail instance failed, and an ASREPRoasting attempt with guessed usernames hit an LDAP bind error. Generating realistic usernames from names found on the site with `username-anarchy` finally produced a crackable AS-REP hash, yielding credentials for `jack.dowland`. That account had no useful BloodHound edges and no Roundcube access, but its mailbox contained a config file with an MD5 secret, cracked and sprayed to land `lainey.moore`. RDP as Lainey surfaced a saved PuTTY session with router credentials that turned out to be reused by `greg.shields`. Greg belonged to a group with full control over the Default Domain Policy, abused with `pyGPOAbuse` to push a scheduled task that created a new local administrator, completing the compromise.

## Full Walkthrough

### Enumeration

Nmap:

![sysco screenshot 1](images/sysco/sysco-01.png)

**SMB enumeration** with `enum4linux`:

![sysco screenshot 2](images/sysco/sysco-02.png)

`smbclient`:

![sysco screenshot 3](images/sysco/sysco-03.png)

`nxc smb --users`:

![sysco screenshot 4](images/sysco/sysco-04.png)

**RPC enumeration** with `rpcclient`:

![sysco screenshot 5](images/sysco/sysco-05.png)

No users come back. Falling back to [Orange Cyberdefense's no-creds AD methodology](https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg).

### TimeRoasting Attempt

Trying `timeroast.py`:

![sysco screenshot 6](images/sysco/sysco-06.png)

![sysco screenshot 7](images/sysco/sysco-07.png)

Checking for a hash:

![sysco screenshot 8](images/sysco/sysco-08.png)

Nothing comes back.

### HTTP Enumeration

Port 80 was open and worth circling back to:

![sysco screenshot 9](images/sysco/sysco-09.png)

![sysco screenshot 10](images/sysco/sysco-10.png)

Directory brute-forcing with Feroxbuster:

![sysco screenshot 11](images/sysco/sysco-11.png)

`/roundcube` and `/roundcube/public_html` turn up:

![sysco screenshot 12](images/sysco/sysco-12.png)

![sysco screenshot 13](images/sysco/sysco-13.png)

Common credentials (`admin:admin`, `root:root`, `sysco:sysco`) all fail. A "Team" section on the site is worth a look for potential usernames:

![sysco screenshot 14](images/sysco/sysco-14.png)

### First ASREPRoasting Attempt

Trying [ASREPRoasting with no credentials](https://www.thehacker.recipes/ad/movement/kerberos/roasting/asreproast):

![sysco screenshot 15](images/sysco/sysco-15.png)

```
impacket-GetNPUsers -request -format hashcat -outputfile ASREProastables.txt -dc-ip "10.0.27.32" "sysco.local/"
```

![sysco screenshot 16](images/sysco/sysco-16.png)

Updating `/etc/hosts` first:

![sysco screenshot 17](images/sysco/sysco-17.png)

The request fails with an LDAP bind error:

```
[-] Error in searchRequest -> operationsError: 000004DC: LdapErr: DSID-0C090A58, comment: In order to perform this operation a successful bind must be completed on the connection., data 0, v4f7c
```

![sysco screenshot 18](images/sysco/sysco-18.png)

### Username Crafting: Access as Jack

Trying the potential usernames spotted on the site with `-usersfile`:

![sysco screenshot 19](images/sysco/sysco-19.png)

![sysco screenshot 20](images/sysco/sysco-20.png)

Wrong format. Rather than keep guessing, [username-anarchy](https://github.com/urbanadventurer/username-anarchy) can generate realistic usernames from real names:

![sysco screenshot 21](images/sysco/sysco-21.png)

The `-country` flag lets it match the naming convention for a given locale:

![sysco screenshot 22](images/sysco/sysco-22.png)

Running the full list of names through it with `-i` to produce a proper username list:

![sysco screenshot 23](images/sysco/sysco-23.png)

![sysco screenshot 24](images/sysco/sysco-24.png)

Retrying ASREPRoasting with the generated list:

![sysco screenshot 25](images/sysco/sysco-25.png)

![sysco screenshot 26](images/sysco/sysco-26.png)

A hash comes back this time.

![sysco screenshot 27](images/sysco/sysco-27.png)

![sysco screenshot 28](images/sysco/sysco-28.png)

Two other accounts, `greg.shields` and `lainey.moore`, also turn out to have `UF_DONT_REQUIRE_PREAUTH` set, worth remembering for later. Cracking the hash in hand:

![sysco screenshot 29](images/sysco/sysco-29.png)

![sysco screenshot 30](images/sysco/sysco-30.png)

Credentials recovered:

```
jack.dowland : musicman1
```

**SMB enumeration** as Jack:

![sysco screenshot 31](images/sysco/sysco-31.png)

![sysco screenshot 32](images/sysco/sysco-32.png)

Checking SYSVOL for anything useful:

![sysco screenshot 33](images/sysco/sysco-33.png)

Nothing there. WinRM and RDP:

![sysco screenshot 34](images/sysco/sysco-34.png)

Both closed off to Jack.

**BloodHound enumeration:**

![sysco screenshot 35](images/sysco/sysco-35.png)

![sysco screenshot 36](images/sysco/sysco-36.png)

Jack has no outbound object control. Checking for kerberoastable accounts:

![sysco screenshot 37](images/sysco/sysco-37.png)

None available either.

### Access as Lainey Moore

Trying Roundcube with Jack's credentials:

![sysco screenshot 38](images/sysco/sysco-38.png)

That works.

![sysco screenshot 39](images/sysco/sysco-39.png)

The Roundcube version is now known, worth checking for a matching exploit or vulnerable plugin. Browsing the mailbox turns up an email with a config file attached:

![sysco screenshot 40](images/sysco/sysco-40.png)

![sysco screenshot 41](images/sysco/sysco-41.png)

The config file contains a secret, crackable and then sprayable across the known users. It looks like an MD5 hash:

![sysco screenshot 42](images/sysco/sysco-42.png)

It is. Trying Hashcat first:

![sysco screenshot 43](images/sysco/sysco-43.png)

![sysco screenshot 44](images/sysco/sysco-44.png)

No luck there. John the Ripper does the job instead:

![sysco screenshot 45](images/sysco/sysco-45.png)

Spraying the cracked password:

![sysco screenshot 46](images/sysco/sysco-46.png)

```
lainey.moore : Chocolate1
```

WinRM:

![sysco screenshot 47](images/sysco/sysco-47.png)

Access confirmed.

**Enumeration** as Lainey:

![sysco screenshot 48](images/sysco/sysco-48.png)

A `/notes.txt` file is worth checking:

![sysco screenshot 49](images/sysco/sysco-49.png)

It references credentials existing somewhere, without giving them directly. Internal ports:

![sysco screenshot 50](images/sysco/sysco-50.png)

Nothing conclusive. Trying Roundcube with Lainey's credentials:

![sysco screenshot 51](images/sysco/sysco-51.png)

That fails.

### Access as Greg Shields

RDP instead, for a better view of the filesystem:

![sysco screenshot 52](images/sysco/sysco-52.png)

![sysco screenshot 53](images/sysco/sysco-53.png)

Saved **PuTTY HS Router Login Properties** contain credentials:

```
netadmin : 5y5coSmarter2025!!!
```

Spraying this password to see who else it belongs to:

![sysco screenshot 54](images/sysco/sysco-54.png)

It matches `greg.shields`' password.

### Full System Compromise

![sysco screenshot 55](images/sysco/sysco-55.png)

Greg Shields belongs to the Group Policy Creator group, whose members hold `WriteDACL`, `GenericAll`, `GenericWrite`, and `WriteOwner` on the Default Domain Policy, full control over it.

![sysco screenshot 56](images/sysco/sysco-56.png)

[pyGPOAbuse](https://github.com/Hackndo/pyGPOAbuse/blob/master/README.md) can turn that control into a scheduled task that creates an admin user:

![sysco screenshot 57](images/sysco/sysco-57.png)

This tool works when a controlled account can modify a GPO applied to users or computers: it creates an immediate scheduled task, running as SYSTEM for a computer GPO or as the logged-in user for a user GPO. The GPO ID needed for this is found in BloodHound:

![sysco screenshot 58](images/sysco/sysco-58.png)

![sysco screenshot 59](images/sysco/sysco-59.png)

The script adds a user, `John`, with password `H4x00r123..`, as an admin user:

```
python3 pygpoabuse.py sysco.local/greg.shields -gpo-id 31B2F340-016D-11D2-945F-00C04FB984F9
```

![sysco screenshot 60](images/sysco/sysco-60.png)

After waiting for the scheduled task to trigger, checking whether the new user exists and the password works:

![sysco screenshot 61](images/sysco/sysco-61.png)

It does, admin privileges confirmed.

![sysco screenshot 62](images/sysco/sysco-62.png)

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
