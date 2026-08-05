<img width="1280" height="640" alt="image" src="https://github.com/user-attachments/assets/a7f3533e-94ab-466e-9ead-dde48f77cb5b" />

# The Server From Hell
Face a server that feels as if it was configured and deployed by Satan himself. 
Can you escalate to root?


## Enumeration & Initial Acces

### Port Scan

We perform a port scan to enumerate open ports and quickly realize that we’re dealing with a honeypot.

<img width="695" height="865" alt="Screenshot 2026-02-01 alle 16 19 56" src="https://github.com/user-attachments/assets/0f02d86b-a36a-4baa-b2ad-0b6274b51a38" />


We then start from the hint provided in the room:

> *Start at port 1337 and enumerate your way. Good luck.*


So we connect to port 1337 using `netcat`:

<img width="438" height="130" alt="Screenshot 2026-02-01 alle 16 21 44" src="https://github.com/user-attachments/assets/7b9b0bb8-af5c-48fb-824f-33716c7908bb" />

The service greets us with the following message:

> *To begin, find the trollface
Legend says he's hiding in the first 100 ports
Try printing the banners from the ports*

Interesting. 


### Banner Enumeration 

We decide to enumerate the banners on the first 100 ports using Nmap:


> nmap -sV --script banner -p 1-100 10.49.181.191 | grep "banner:"


<img width="641" height="896" alt="Screenshot 2026-02-01 alle 16 24 20" src="https://github.com/user-attachments/assets/6e0d69a5-b3f6-4a12-84b6-a0e0c212bab9" />


And there it is.

We successfully find the trollface, and right in the middle of it we can read:


> *go to port 12345*


Let's follow the hint. 


### NFS Enumeration

<img width="1407" height="329" alt="Screenshot 2026-02-01 alle 16 25 45" src="https://github.com/user-attachments/assets/5f88e0df-786a-4842-a48a-19c1ed31adc5" />

Connecting to port 12345, we receive another message:


> *NFS shares are cool, especially when they are misconfigured. It's on the standard port, no need for another scan*


At this point, it’s clear that we’re dealing with NFS.

We enumerate available NFS shares using:

> `showmount -e <IPADDR>`


<img width="265" height="83" alt="Screenshot 2026-02-01 alle 16 27 24" src="https://github.com/user-attachments/assets/b4459f5e-0c39-41ee-8e42-40634510199b" />


We discover that `/home/nfs` is mountable.


### Mounting the NFS Share

1. We create a local mount point

> `mkdir /tmp/nfs`

4. Mount the share:

> `sudo mount -t nfs <IPADDR>:/home/nfs /tmp/nfs`
   

Inside the mounted directory, we find a password-protected `backup.zip` file:


<img width="460" height="140" alt="Screenshot 2026-02-01 alle 16 29 58" src="https://github.com/user-attachments/assets/81722284-1ec1-49ed-b300-5b9156e57da4" />



### Cracking the ZIP Password

We extract the hash using `zip2john`. 

<img width="1255" height="201" alt="Screenshot 2026-02-01 alle 16 30 57" src="https://github.com/user-attachments/assets/44ae97fe-6050-4f96-a1c7-a0b4d0941557" />


Then we crack it with `John the Ripper`:

<img width="842" height="184" alt="Screenshot 2026-02-01 alle 16 31 42" src="https://github.com/user-attachments/assets/d88dacf6-0adc-4e85-93ca-f56d2b74c89a" />


Success.

We unzip the archive and find:

	•	the first flag
	•	a private SSH key
	•	a public SSH key
	•	a hint.txt file

  
We copy the private key locally and fix its permissions since we’ll need it:

> chmod 600 id_rsa


The `hint.txt` contains only: 

> *2500-4500*


### SSH Port Discovery

Trying to connect via SSH on port 22 results in:

> *Connection is reset by the peer* 

So SSH is clearly running on a non-standard port, most likely between 2500 and 4500.


That’s a large range, so we write a quick brute-force script to find the correct SSH port using the private key:

````
seq 2500 4500 | xargs -P 20 -I {} bash -c "ssh -o ConnectTimeout=2 -o BatchMode=yes -o StrictHostKeyChecking=no -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedAlgorithms=+ssh-rsa -i id_rsa -p {} hades@<IPADDR> exit 2>&1 | grep -iE 'Permission denied|Welcome' && echo '[+] PORT FOUND: {}'"

````

Eventually, we get a hit:

> `[+] PORT FOUND: 3333`



### SSH Access

We connect to SSH on port 3333:

<img width="977" height="854" alt="Screenshot 2026-02-01 alle 16 40 45" src="https://github.com/user-attachments/assets/282fe28d-2800-4a5c-a2b8-29b082581656" />

> (Note: I had to clean the id_rsa file because some spaces were added while pasting it.)



We’re in — but instead of a Bash shell, we land inside a **Ruby jail**.

That means standard Linux commands won’t work directly.


### Escaping the Ruby Jail

Escaping is actually very simple.

Ruby allows command execution using backticks `command`.

For example:

<img width="644" height="57" alt="Screenshot 2026-02-01 alle 16 45 06" src="https://github.com/user-attachments/assets/d6bcadb2-a454-450e-bf1a-260ec183f5e9" />


We can now try to spawn a proper shell: 

<img width="329" height="53" alt="Screenshot 2026-02-01 alle 16 45 56" src="https://github.com/user-attachments/assets/b163ec16-d39a-4c46-b017-a7f330bbe381" />

And voilà — we’re finally in a Bash shell.


At this point, we can easily retrieve the user flag.



## Post-Exploitation 

### Privilege Escalation 

The goal is now to read root.txt.

We enumerate Linux capabilities using:

> `getcap -r / 2>/dev/null`

We notice something very interesting:

`/bin/tar = cap_dac_read_search+ep` 

This capability allows tar to bypass file permission checks, meaning we can read any file on the system, including root-owned files.

<img width="329" height="61" alt="Screenshot 2026-02-01 alle 16 47 42" src="https://github.com/user-attachments/assets/f1c0a6c7-ae72-4664-911f-99b892c24d0a" />


### Reading root.txt

The easiest way is to directly read the file from `/root` using tar:

> `tar -cf - /root/root.txt | tar -xf - -O`


And finally...

<img width="460" height="58" alt="Screenshot 2026-02-01 alle 16 50 13" src="https://github.com/user-attachments/assets/23cdb635-ed69-49db-a7ed-d73d3edff63e" />


We successfully retrieve `root.txt`.



