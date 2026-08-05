# Sysco — (Username Crafting | Username-anarchy | -GPOAbuse)

# ENUMERATION

NMAP:

![sysco screenshot 1](images/sysco/sysco-01.png)

## SMBENUM :

- Enum4linux

![sysco screenshot 2](images/sysco/sysco-02.png)

- Smbclient

![sysco screenshot 3](images/sysco/sysco-03.png)

- nxc smb --users

![sysco screenshot 4](images/sysco/sysco-04.png)

## RPCENUM

- Rpcclient

![sysco screenshot 5](images/sysco/sysco-05.png)

No users.

Let’s check OrangeDefense methodology (AD Enum no creds) :

https://orange-cyberdefense.github.io/ocd-mindmaps/img/mindmap_ad_dark_classic_2025.03.excalidraw.svg

## TIMEROASTING

Going for TimeRoast :

- timeorast.py

![sysco screenshot 6](images/sysco/sysco-06.png)

![sysco screenshot 7](images/sysco/sysco-07.png)

Did we get an hash?

![sysco screenshot 8](images/sysco/sysco-08.png)

Nope.

## HTTP ENUM

I forgot that there was an 80 HTTP open :

![sysco screenshot 9](images/sysco/sysco-09.png)

![sysco screenshot 10](images/sysco/sysco-10.png)

Directories?

- Feroxbuster

![sysco screenshot 11](images/sysco/sysco-11.png)

/roundcube

/roundcube/public_html

![sysco screenshot 12](images/sysco/sysco-12.png)

![sysco screenshot 13](images/sysco/sysco-13.png)

Tried common credentials like :

Admin:admin

Root:root

Sysco:sysco

## Team Section

![sysco screenshot 14](images/sysco/sysco-14.png)

Maybe some usernames?

Let’s move forward

## ASREPROASTING?

Some ASREP-Roasting (with no creds)

https://www.thehacker.recipes/ad/movement/kerberos/roasting/asreproast

![sysco screenshot 15](images/sysco/sysco-15.png)

`impacket-GetNPUsers -request -format hashcat -outputfile ASREProastables.txt -dc-ip "10.0.27.32" "sysco.local/"`

![sysco screenshot 16](images/sysco/sysco-16.png)

Let’s update first /etc/hosts

![sysco screenshot 17](images/sysco/sysco-17.png)

When browsing for the error:

`[-] Error in searchRequest -> operationsError: 000004DC: LdapErr: DSID-0C090A58, comment: In order to perform this operation a successful bind must be completed on the connection., data 0, v4f7c`

![sysco screenshot 18](images/sysco/sysco-18.png)

# USERNAMES CRAFTING — ACCESS AS JACK

Let’s try saving the potential users we found and using that list with -usersfile

![sysco screenshot 19](images/sysco/sysco-19.png)

![sysco screenshot 20](images/sysco/sysco-20.png)

Usernames are not correct.

Can we use a tool to potentially create usernames from names?

Indeed we can :

https://github.com/urbanadventurer/username-anarchy

- Username-anarchy

Username-anarchy can craft usernames starting from names :

![sysco screenshot 21](images/sysco/sysco-21.png)

We can even use -country flag to specify which country is the server .

![sysco screenshot 22](images/sysco/sysco-22.png)

That will be something like this.

Let’s try to put the entire users lists and to output a usernames lists:

- -i = input file

![sysco screenshot 23](images/sysco/sysco-23.png)

![sysco screenshot 24](images/sysco/sysco-24.png)

Now we can re-try as-repRoast

![sysco screenshot 25](images/sysco/sysco-25.png)

![sysco screenshot 26](images/sysco/sysco-26.png)

And we got a hash!

![sysco screenshot 27](images/sysco/sysco-27.png)

![sysco screenshot 28](images/sysco/sysco-28.png)

But we also know that greg.shields and lainey.moore have UF_DONT_REQUIRE_PREAUTH set.

Let’s crack it :

![sysco screenshot 29](images/sysco/sysco-29.png)

![sysco screenshot 30](images/sysco/sysco-30.png)

We got credentials :

`jack.dowland : musicman1`

Cool, let’s see what we can do with him.

First:

## SMBENUM :

![sysco screenshot 31](images/sysco/sysco-31.png)

![sysco screenshot 32](images/sysco/sysco-32.png)

Anything useful in SYSVOL?

![sysco screenshot 33](images/sysco/sysco-33.png)

Nope.

WINRM? RDP?

![sysco screenshot 34](images/sysco/sysco-34.png)

Nope

RDP = nope.

## BLOODHOUND ENUM

Ok, let’s run bloodhound and see what happens :

![sysco screenshot 35](images/sysco/sysco-35.png)

Cool, let’s analyze it :

![sysco screenshot 36](images/sysco/sysco-36.png)

Jack has not Outbound Object Control.

Can we kerberoast some users?

![sysco screenshot 37](images/sysco/sysco-37.png)

Nope

# ACCESS AS LAINAY MOORE

Let’s try roundcube then :

![sysco screenshot 38](images/sysco/sysco-38.png)

And here we are.

![sysco screenshot 39](images/sysco/sysco-39.png)

We know the version.

Let’s check for un exploit?

Maybe some plugins exploit?

Navigating in the platform, we discovered an email with a config file:

![sysco screenshot 40](images/sysco/sysco-40.png)

![sysco screenshot 41](images/sysco/sysco-41.png)

We found a secret in the cfg file.

We can now crack the password and spray it, knowing the users.

Let’s crack it.

It should be md5 hash :

![sysco screenshot 42](images/sysco/sysco-42.png)

Indeed it is.

- Hashcat

![sysco screenshot 43](images/sysco/sysco-43.png)

And crack it :

![sysco screenshot 44](images/sysco/sysco-44.png)

Merde.

Let’s try with JTR:

- JohnTheRipper

![sysco screenshot 45](images/sysco/sysco-45.png)

Cool.

Let’s spray it :

![sysco screenshot 46](images/sysco/sysco-46.png)

`Lainey.moore:Chocolate1 `

Let’s win-rm if possible :

![sysco screenshot 47](images/sysco/sysco-47.png)

Done.

## ENUMERATION:

Let’s enumerate.

![sysco screenshot 48](images/sysco/sysco-48.png)

Of course /notes.txt is interesting.

![sysco screenshot 49](images/sysco/sysco-49.png)

Mhh..

There are credentials somewhere.

Internal Ports?

![sysco screenshot 50](images/sysco/sysco-50.png)

??

What if we connect to roundcube with Lainey?

![sysco screenshot 51](images/sysco/sysco-51.png)

Failed.

# ACCESS AS GREG SHIELDS

Let’s RDP to have a better view

![sysco screenshot 52](images/sysco/sysco-52.png)

![sysco screenshot 53](images/sysco/sysco-53.png)

In the **Putty HS Router Login Properties** we found credentials :

`netadmin :  5y5coSmarter2025!!!`

Who’s the netadmin?

Let’s spray password:

![sysco screenshot 54](images/sysco/sysco-54.png)

We found that the password is **Greg.shields’ password**.

# FULL SYSTEM COMPROMISE :

![sysco screenshot 55](images/sysco/sysco-55.png)

Greg Shields is member of Group Policy Creator which members have:

- WriteDACL

- GenericALL

- GenericWrite

- WriteOwner

On the Default Domain Policy

That would lead to Full Control to Domain Policy.

Let’s try it out:

![sysco screenshot 56](images/sysco/sysco-56.png)

We can use **pyGPOAbuse.py** to create a scheduled task in order to get access to Admin.

https://github.com/Hackndo/pyGPOAbuse/blob/master/README.md

![sysco screenshot 57](images/sysco/sysco-57.png)

This tool can be used when a controlled account can modify an existing GPO that applies to one or more users & computers. It will create an immediate scheduled task as SYSTEM on the remote computer for computer GPO, or as logged in user for user GPO.

In order to do that we need the GPO ID that we can find in Bloodhound :

![sysco screenshot 58](images/sysco/sysco-58.png)

![sysco screenshot 59](images/sysco/sysco-59.png)

The script will add a user John with password “H4x00r123..” as admin user.

Let’s do it :

- pygpoabuse.py

`python3 pygpoabuse.py sysco.local/greg.shields -gpo-id 31B2F340-016D-11D2-945F-00C04FB984F9`

![sysco screenshot 60](images/sysco/sysco-60.png)

We wait a little so that the task will be triggered

And check if the password is correct and the user exists :

![sysco screenshot 61](images/sysco/sysco-61.png)

Pwned!  This means that we have admin privs.

![sysco screenshot 62](images/sysco/sysco-62.png)

# PWNED
