<img width="444" height="794" alt="image" src="https://github.com/user-attachments/assets/e45375a5-dda0-4ce6-9c6f-aa217db90098" />

# Wekor 
CTF challenge involving Sqli , WordPress , vhost enumeration and recognizing internal services.


## Enumeration & Initial Access

### Port Scanning 

We start with a port scanning to identify open ports: 

<img width="433" height="146" alt="Screenshot 2026-01-28 alle 14 25 21" src="https://github.com/user-attachments/assets/c064fd60-95a7-49c4-893a-6013e42fcbfc" />

The result reveals 2 open ports: 
- 22 (SSH)
- 80 (HTTP)

### Service Version & Script Scan

Next, we run a service and default script scan to gather information about the services running on the open ports:

<img width="564" height="271" alt="Screenshot 2026-01-28 alle 14 27 48" src="https://github.com/user-attachments/assets/33d03f52-d34b-42a2-8b42-4f192029990c" />

During this scan, we identify interesting entries in `/robots.txt` which disallows 9 entries. 
Upon checking, no significant directories were found except for `/comingreallysoon`.

<img width="1017" height="129" alt="Screenshot 2026-01-28 alle 14 30 07" src="https://github.com/user-attachments/assets/a12b25ea-d8ee-4fe9-bec6-84753f8c2997" />

We then visit the `/it-next` directory and discover a live website.

### SQLi

Exploring the website, we find a form to apply coupons. 
We attempt SQL injection and immediately get syntax error messages:

<img width="1338" height="158" alt="Screenshot 2026-01-28 alle 14 40 21" src="https://github.com/user-attachments/assets/d8bcfc48-e2a7-4975-8343-ee8772a4e24d" />

We intercept the request with BurpSuite and save it to a `.txt` file. 
After identifying the vulnerable parameter, we use SQLmap with the `-r` flag:


> sqlmap -r request.req -p coupon_code --dump-all -batch


<img width="1710" height="225" alt="Screenshot 2026-01-28 alle 15 06 34" src="https://github.com/user-attachments/assets/f21ccb64-0d62-4458-9c96-98aaf70e0bd4" />


SQLmap produces extensive output, but we focus on the WordPress database, which reveals user details and hashed passwords.

<img width="1709" height="312" alt="Screenshot 2026-01-29 alle 06 54 39" src="https://github.com/user-attachments/assets/642721b6-f3b2-4fa2-8ec2-73ccd3236a0c" />

### Unhashing Passwords 

We use `John the Ripper` to crack the hashes of the WordPress users:

<img width="840" height="219" alt="Screenshot 2026-01-29 alle 08 27 17" src="https://github.com/user-attachments/assets/b024a154-f662-4025-8e34-8755c59d3371" />

After successfully cracking the passwords, we add the domain we found to `/etc/hosts` and begin navigating through the WordPress site.

<img width="1337" height="784" alt="Screenshot 2026-01-29 alle 08 28 20" src="https://github.com/user-attachments/assets/14c15ba2-4b5c-4926-9b5c-d41b9d320fde" />

We find the login page and attempt to log in using the credentials we cracked.

<img width="1710" height="678" alt="Screenshot 2026-01-29 alle 08 32 53" src="https://github.com/user-attachments/assets/1c417cd7-5bbd-40d8-8767-6bc664d862b4" />

### Accessing the WordPress Dashboard

We successfully log in with the admin account, Yura.

Next, we easily upload a web shell via the WordPress theme editor:

<img width="724" height="281" alt="Screenshot 2026-01-29 alle 08 41 01" src="https://github.com/user-attachments/assets/3cdace06-1e75-4f69-82fa-0e5d48cbde37" />

### Initial Shell Access

We set up a listener using `Netcat` on the specified port and visit a non-existent page to trigger a 404 error:

<img width="965" height="185" alt="Screenshot 2026-01-29 alle 08 54 53" src="https://github.com/user-attachments/assets/1aaca686-a778-43f8-a0ee-d3c426884c31" />

shell obtained. 

## Post-Exploitation 

### Stabilizing the Shell

> /bin/bash -i

> python3 -c 'import pty; pty.spawn ("/bin/bash")'

### User Enumeration

We perform user enumeration and find that Orka has restricted access to their home directory, and we lack the required permissions to access the user.txt file.

<img width="314" height="63" alt="Screenshot 2026-01-29 alle 08 57 44" src="https://github.com/user-attachments/assets/c5e60ddd-0cbd-4cb4-9965-3d4fa2f9f05d" />


Upon further enumeration, we discover that `Memcached` is running on the system:

<img width="760" height="722" alt="Screenshot 2026-01-29 alle 09 12 47" src="https://github.com/user-attachments/assets/cee16d75-57ba-4480-81f1-3fd0c912ccb8" />

### Memcached Enumeration

We don't know a lot about it. 
We google it and we figure it out how it works. 
We now know that Memcached is a general-purpose distributed memory-caching system.

Let's see on which port is running: 

<img width="1002" height="70" alt="Screenshot 2026-01-29 alle 09 15 13" src="https://github.com/user-attachments/assets/ed25deb4-83a5-41de-9862-e34cc6f14fa8" />

We identify that Memcached is running on port 11211 on localhost. 

Using Telnet, we connect to it and proceed to dump cached data:


<img width="729" height="137" alt="Screenshot 2026-01-29 alle 09 16 09" src="https://github.com/user-attachments/assets/8f211e75-40fc-42f2-a70e-1d781b0534e1" />


An interesting article about Memcached can be found [here](https://www.hackingarticles.in/penetration-testing-on-memcached-server/)


<img width="341" height="275" alt="Screenshot 2026-01-29 alle 09 17 11" src="https://github.com/user-attachments/assets/8378b384-38b7-47b0-a96b-1f3dba2259b7" />

After dumping the cache, we recover the Orka password.

### Accessing Orka Account

We attempt to switch to the Orka user:

> su Orka 

<img width="539" height="88" alt="Screenshot 2026-01-29 alle 09 18 18" src="https://github.com/user-attachments/assets/f0c36ff5-4064-43e3-8285-e6d2a88434cd" />

With the password obtained, we gain access to Orka’s home directory and read the user.txt flag.

## Privilege Escalation

### Sudo Privileges 

Running the `sudo -l` command reveals that the Orka user can execute `/home/Orka/Desktop/bitcoin` with root privileges.

<img width="823" height="155" alt="Screenshot 2026-01-29 alle 09 19 41" src="https://github.com/user-attachments/assets/99ebb9ae-a41f-4be1-9454-38fee52e59eb" />


### Investigating the Bitcoin File

In the Desktop directory, we find two files: Bitcoin and transfer.py. 
We don't have writing permissions on both of them.

We attempt to run the Bitcoin file but it requires a password, which is different from the one we used to log into the Orka account.

<img width="488" height="87" alt="Screenshot 2026-01-29 alle 10 14 47" src="https://github.com/user-attachments/assets/385c6ad1-2490-4a0d-826b-c82c40464cf6" />


We try to run strings on Bitcoin binary, but nothing interesting, except for the fact that it calls transfer.py

<img width="518" height="616" alt="Screenshot 2026-01-29 alle 10 16 17" src="https://github.com/user-attachments/assets/7b3a92d3-9156-4008-8d98-b17c04945da2" />


We need to find the password accepted by Bitcoin in order to run it.

If we take a closer look to sudo -l :

> secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

We can see that the bitcoin binary is not using an absolute path for the python executable. 
This means that we can create our own python binary and let it be executed by Bitcoin if we find a writeable path.

We then check if we have writing permissions on one of the secure path: 

<img width="509" height="202" alt="Screenshot 2026-01-29 alle 10 19 55" src="https://github.com/user-attachments/assets/34465113-5728-49ac-a205-4d32ce35302f" />

/usr/sbin seems to be writeable.

We create our own Python binary: 

<img width="576" height="100" alt="Screenshot 2026-01-29 alle 10 23 44" src="https://github.com/user-attachments/assets/ae8afcfe-da18-4769-ac74-6c0df8799c87" />

And we give it execution permissions: 

<img width="380" height="21" alt="Screenshot 2026-01-29 alle 10 24 31" src="https://github.com/user-attachments/assets/5946756a-a6eb-4c29-82bb-d4211f53781c" />

#### Time to find Bitcoin password: 

We transfer the Bitcoin binary to our local system using a Python HTTP server and we open it with `Ghidra`:

<img width="1393" height="639" alt="Screenshot 2026-01-29 alle 10 25 29" src="https://github.com/user-attachments/assets/5b984611-b844-4289-ae34-52a69b94e6b3" />

After analyzing the binary with Ghidra, we successfully extract the password required to execute the Bitcoin binary.

### Root Shell

With the password in hand, we run the Bitcoin binary with the extracted password:

<img width="563" height="261" alt="Screenshot 2026-01-29 alle 10 27 19" src="https://github.com/user-attachments/assets/b606db73-46c5-443f-8996-81e2c54feed1" />

And voilà.

This grants us root access, and we can now read the root.txt file.








