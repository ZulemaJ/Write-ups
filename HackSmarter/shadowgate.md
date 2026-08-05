# ShadowGate

**Category:** Active Directory, No Credentials

## Attack Chain

1. SMB/RPC enumeration lists 12 users but leaks nothing further; TimeRoast cracks no hash
2. ASREPRoasting with the enumerated user list cracks `jtrueblood`'s password
3. BloodHound: `jtrueblood` has `GenericWrite` on `bbrown`, abused via a targeted Kerberoast
4. `bbrown` belongs to an ADCS-reader group; Certipy flags the CA as vulnerable to ESC8
5. Certipy relay plus PetitPotam coercion captures the domain controller's own NTLM authentication
6. A certificate is obtained for the DC's machine account, used to recover its NTLM hash
7. DCSync with the DC hash dumps every credential in the domain

## TL;DR

Starting with no credentials, SMB and RPC enumeration listed 12 users but leaked nothing further, and an initial TimeRoast attempt cracked no hash. Falling back to ASREPRoasting with the enumerated user list worked, cracking a password for `jtrueblood`. BloodHound showed `jtrueblood` held `GenericWrite` over `bbrown`, abused with a targeted Kerberoast to recover `bbrown`'s (weak) password. `bbrown` belonged to an ADCS-reader group, and Certipy flagged the CA as vulnerable to **ESC8**: an NTLM relay attack against the CA's HTTP enrollment endpoint. Relaying a PetitPotam-coerced authentication from the domain controller itself yielded a certificate for the DC's machine account, which was used to obtain its NTLM hash and run a DCSync, extracting every credential in the domain.

## Full Walkthrough

### Enumeration

Nmap:

![shadowgate screenshot 1](images/shadowgate/shadowgate-01.png)

**SMB enumeration** with `enum4linux-ng`:

![shadowgate screenshot 2](images/shadowgate/shadowgate-02.png)

12 users discovered, saved into a list for later. No passwords yet.

![shadowgate screenshot 3](images/shadowgate/shadowgate-03.png)

**RPC enumeration:** querying every user with `rpcclient`'s `queryuser <user-rid>` leaks nothing.

![shadowgate screenshot 4](images/shadowgate/shadowgate-04.png)

No shares either. Checking whether UDP 161 (SNMP) is open with a full port scan:

![shadowgate screenshot 5](images/shadowgate/shadowgate-05.png)

Filtered.

![shadowgate screenshot 6](images/shadowgate/shadowgate-06.png)

**HTTP enumeration:** port 80 wasn't flagged in the initial scan but shows up in the full one, an IIS instance:

![shadowgate screenshot 7](images/shadowgate/shadowgate-07.png)

Searching for directories with Feroxbuster across several wordlists turns up nothing:

![shadowgate screenshot 8](images/shadowgate/shadowgate-08.png)

### Following the No-Creds AD Methodology

Falling back to [Orange Cyberdefense's AD enumeration roadmap for no credentials](https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg), starting again with SMB:

![shadowgate screenshot 9](images/shadowgate/shadowgate-09.png)

![shadowgate screenshot 10](images/shadowgate/shadowgate-10.png)

Still nothing. Next on the list: TimeRoast, which abuses the Windows Time Service (NTP) to pull encrypted password-related data from a domain controller, crackable offline to recover weak machine account passwords. Updating `/etc/hosts` first:

![shadowgate screenshot 11](images/shadowgate/shadowgate-11.png)

Running it:

```
timeroast.py <dc_ip> -o <output_log>
```

![shadowgate screenshot 12](images/shadowgate/shadowgate-12.png)

![shadowgate screenshot 13](images/shadowgate/shadowgate-13.png)

A hash comes back. Cracking it with Hashcat, after stripping the leading `1000:` prefix:

![shadowgate screenshot 14](images/shadowgate/shadowgate-14.png)

![shadowgate screenshot 15](images/shadowgate/shadowgate-15.png)

No luck this time.

### Access as jtrueblood

ASREPRoasting without authentication is next, following [this methodology](https://www.thehacker.recipes/ad/movement/kerberos/roasting/asreproast#practice).

Dynamically querying users via anonymous LDAP bind:

```
GetNPUsers.py -request -format hashcat -outputfile ASREProastables.txt -dc-ip "$DC_IP" "$DOMAIN/"
```

![shadowgate screenshot 16](images/shadowgate/shadowgate-16.png)

With the earlier user list supplied directly:

```
GetNPUsers.py -usersfile users.txt -request -format hashcat -outputfile ASREProastables.txt -dc-ip "$DC_IP" "$DOMAIN/"
```

![shadowgate screenshot 17](images/shadowgate/shadowgate-17.png)

A hash for `jtrueblood` comes back.

![shadowgate screenshot 18](images/shadowgate/shadowgate-18.png)

Cracking it:

![shadowgate screenshot 19](images/shadowgate/shadowgate-19.png)

![shadowgate screenshot 20](images/shadowgate/shadowgate-20.png)

```
jtrueblood : blood_brothers
```

Checking shares with CrackMapExec:

![shadowgate screenshot 21](images/shadowgate/shadowgate-21.png)

Nothing actionable yet, kept in mind. RDP and WinRM:

![shadowgate screenshot 22](images/shadowgate/shadowgate-22.png)

Both fail.

**SMB enumeration:** checking SYSVOL and CertEnroll shares:

![shadowgate screenshot 23](images/shadowgate/shadowgate-23.png)

![shadowgate screenshot 24](images/shadowgate/shadowgate-24.png)

![shadowgate screenshot 25](images/shadowgate/shadowgate-25.png)

![shadowgate screenshot 26](images/shadowgate/shadowgate-26.png)

![shadowgate screenshot 27](images/shadowgate/shadowgate-27.png)

Nothing useful for now.

### Access as bbrown

**BloodHound enumeration:**

![shadowgate screenshot 28](images/shadowgate/shadowgate-28.png)

Analyzing the results:

![shadowgate screenshot 29](images/shadowgate/shadowgate-29.png)

`jtrueblood` holds `GenericWrite` over `bbrown`. BloodHound's Linux Abuse guidance for this edge:

![shadowgate screenshot 30](images/shadowgate/shadowgate-30.png)

`GenericWrite` here enables a targeted Kerberoast:

```
targetedKerberoast.py -v -d 'domain.local' -u 'controlledUser' -p 'ItsPassword'
```

![shadowgate screenshot 31](images/shadowgate/shadowgate-31.png)

A hash for `bbrown` comes back. Cracking it:

![shadowgate screenshot 32](images/shadowgate/shadowgate-32.png)

```
bbrown : 12345678
```

![shadowgate screenshot 33](images/shadowgate/shadowgate-33.png)

`bbrown` has no outbound object control, and both WinRM and RDP fail. BloodHound does show `bbrown` is a member of `ADCS-READER`:

![shadowgate screenshot 34](images/shadowgate/shadowgate-34.png)

Worth running Certipy to check for certificate template vulnerabilities:

```
certipy-ad find -u bbrown -p 12345678 -dc-host DC01.shadow.gate -dc-ip 10.1.63.6 -vulnerable
```

![shadowgate screenshot 35](images/shadowgate/shadowgate-35.png)

![shadowgate screenshot 36](images/shadowgate/shadowgate-36.png)

The finding is **ESC8**: an AD CS privilege escalation vulnerability that exploits unencrypted, HTTP-based certificate web enrollment endpoints. By coercing a privileged account, such as a domain controller, to authenticate to an attacker-controlled machine, the attacker relays that NTLM authentication to the CA and obtains a highly privileged certificate. A detailed exploitation path is documented in [Certipy's wiki](https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation).

![shadowgate screenshot 37](images/shadowgate/shadowgate-37.png)

Checking `/certsrv`:

![shadowgate screenshot 38](images/shadowgate/shadowgate-38.png)

It requires authentication. Accessing it as `bbrown`:

![shadowgate screenshot 39](images/shadowgate/shadowgate-39.png)

Rather than stop here, the plan is to follow the full ESC8 relay chain.

### Full System Compromise

![shadowgate screenshot 40](images/shadowgate/shadowgate-40.png)

**Step 1: start the Certipy NTLM relay.** Certipy runs in relay mode on the attacking machine, targeting the CA's vulnerable web enrollment endpoint:

```
certipy relay -target 'https://10.0.0.50' -template 'DomainController'
```

![shadowgate screenshot 41](images/shadowgate/shadowgate-41.png)

Certipy is now listening for incoming SMB connections, a common way to capture NTLM auth via coercion, ready to relay to the specified HTTP target.

**Step 2: coerce authentication from a privileged account.** A separate tool forces the target, in this case the domain controller itself, to attempt NTLM authentication against the attacker's machine, where Certipy's relay is listening. Following [a PetitPotam PoC writeup](https://medium.com/r3d-buck3t/domain-takeover-with-petitpotam-exploit-3900f89b38f7):

![shadowgate screenshot 42](images/shadowgate/shadowgate-42.png)

![shadowgate screenshot 43](images/shadowgate/shadowgate-43.png)

The first attempt fails. Debugging shows the `-subject` flag was missing, needed here to specify `DC01` as the identity to authenticate:

```
-subject "CN=DC01"
```

![shadowgate screenshot 44](images/shadowgate/shadowgate-44.png)

This time it succeeds.

**Step 3: authenticate using the obtained certificate.** Using the `.pfx` file obtained via the relay:

```
certipy auth -pfx 'dc.pfx' -dc-ip '10.0.0.100'
```

![shadowgate screenshot 45](images/shadowgate/shadowgate-45.png)

A hash comes back for the DC's machine account.

### DCSync

With the `DC01$` hash in hand, a DCSync becomes possible with `impacket-secretsdump`. DCSync reads the password data of any requested user by impersonating the domain controller's own replication rights.

![shadowgate screenshot 46](images/shadowgate/shadowgate-46.png)

Full domain compromise.

---

*Educational write-up from an authorized lab environment (HackSmarter). Not a guide for unauthorized access.*
