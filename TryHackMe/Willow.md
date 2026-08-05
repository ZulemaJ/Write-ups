<img width="400" height="400" alt="e7e6f48a6ae9cb49f91e2934b14b5a34 (1)" src="https://github.com/user-attachments/assets/31e8815d-ed55-41a8-a0c8-9cfffbccb419" />

# Willow 
What lies under the Willow Tree?

## Enumeration & Initial Access 

### Port Scanning 

We begin with a port scan to identify exposed services: 

<img width="621" height="230" alt="Screenshot 2026-02-01 alle 08 44 30" src="https://github.com/user-attachments/assets/7a3b198e-4868-4b26-a8d2-33abb004604d" />

Scan output reveals the following open ports: 
- 22 SSH 
- 80 HTTP 
- 111 rpcbind
- 2049 NFS


### Web Enumeration (Port 80) 

Visiting port 80, we are presented with an endless hexadecimal string: 

<img width="1706" height="997" alt="Screenshot 2026-02-01 alle 08 51 40" src="https://github.com/user-attachments/assets/c2954323-2d83-43a3-bf87-ee8f42145739" />


At first glance, this appears to be encoded data. We copy the content and attempt to decode it using CyberChef:

<img width="1370" height="681" alt="Screenshot 2026-02-01 alle 08 53 03" src="https://github.com/user-attachments/assets/ef8f9cee-2c5c-4a68-a8cf-e2a2abaf1bbe" />

While decoding confirms the data is meaningful, it becomes clear that a decryption key is missing.


At this point, we pivot back to further enumeration.


### Service Enumeration

We run an Nmap scan with default scripts and version detection on the identified ports.

<img width="808" height="788" alt="Screenshot 2026-02-01 alle 08 56 11" src="https://github.com/user-attachments/assets/c3ead005-81e2-4c08-a5d4-dd8753f9c7e1" />

Nothing immediately exploitable appears from this scan, so we move on to investigate NFS.


### NFS Enumeration

Using `showmount` we discover an exposed NFS share: 

<img width="276" height="83" alt="Screenshot 2026-02-01 alle 08 56 59" src="https://github.com/user-attachments/assets/36b07cf2-7c7a-44b2-9ba4-4cc91e2f462a" />


 `/var/failsafe` can be mounted without authentication.


### Mounting the NFS Share:

1. We first create a local mount point: 

> mkdir /tmp/nfs


2. We then mount the share:

<img width="522" height="57" alt="Screenshot 2026-02-01 alle 09 04 23" src="https://github.com/user-attachments/assets/c74ff36d-a41e-4bc7-85f2-1726f5714519" />


<img width="304" height="146" alt="Screenshot 2026-02-01 alle 09 04 40" src="https://github.com/user-attachments/assets/d0c9cdfc-2f10-4929-b3e0-15ef31b82d04" />


Once mounted, we discover both RSA public and private key pairs. 


### Custom RSA Decryption:

The discovered data suggests a **custom RSA entryption** where:
- the ciphertext is stored as decimal blocks
- each block must be decrypted individually using: 

> m = c^d \bmod n


To reconstruct the original private key, we write a custom Python script:

> nano decrypt_rsa.py



```python

cipher_blocks = [#numbers separated by comma here]
d, n = 61527, 37627

open("id_rsa", "wb").write(
    b"".join(pow(c, d, n).to_bytes(2, "big") for c in cipher_blocks)
)

```


We make the script executable and run it:

> chmod + x decrypt_rsa.py

> python3 decrypt_rsa.py

The script reconstructs the original binary data and generates an `id_rsa` file in the current directory.


### SSH Access

As shown below, this file is a valid RSA private key:

<img width="609" height="532" alt="Screenshot 2026-02-01 alle 09 39 09" src="https://github.com/user-attachments/assets/8bef788d-9779-453c-ac66-953227f5f2e5" />


Attempting SSH authentication with the key prompts for a passphrase. : 

<img width="672" height="105" alt="Screenshot 2026-02-01 alle 10 11 00" src="https://github.com/user-attachments/assets/2d20270f-c789-498f-ab2e-fa8987f611ad" />



### Cracking the RSA Passphrase

We extract the hash from the private key using `ssh2john`:

> python3 /usr/share/john/ssh2john.py id_rsa > private_hash.txt



Then crack it using `John the Ripper`:

<img width="742" height="213" alt="Screenshot 2026-02-01 alle 10 12 46" src="https://github.com/user-attachments/assets/ac022b7a-c5b7-47c9-8e05-86aa8ba28e4a" />

The passphrase is successfully recovered.


### SSH Login

Using the recovered passphrase, we authenticate successfully as willow via SSH:

<img width="700" height="389" alt="Screenshot 2026-02-01 alle 10 13 18" src="https://github.com/user-attachments/assets/3dd589fd-f4a0-40bd-a012-032eea8994bc" />


## Post-Exploitation 

### User Flag Discovery

In Willow's directory, we find an interesting file: `user.jpg`. 

We transfer the file to our local machine and inspect it, revealing the user flag:

<img width="1710" height="190" alt="Screenshot 2026-02-01 alle 10 23 09" src="https://github.com/user-attachments/assets/6070aff8-ab8c-48f3-8c56-762040b27494" />


<img width="976" height="241" alt="Screenshot 2026-02-01 alle 10 23 26" src="https://github.com/user-attachments/assets/0d7abdbc-2a62-4981-902f-0726e0e7f79d" />


### Privilege Escalation

We check available sudo permissions running `sudo -l` 

<img width="930" height="98" alt="Screenshot 2026-02-01 alle 10 47 43" src="https://github.com/user-attachments/assets/9962591e-5ff2-4ab1-822c-2a32cc6f5bbd" />

This indicates we can mount any device under /dev as root, without a password.


#### Device Enumeration 

Listing `/dev/` we notice an unsual entry: 

<img width="1677" height="129" alt="Screenshot 2026-02-01 alle 10 48 10" src="https://github.com/user-attachments/assets/aeac1893-7cfb-4134-a2c8-5206455fb237" />

`hidden_backup`


#### Mounting the Hidden Backup 

1. We first create a mount point:

   > mkdir /tmp/hidden_backup
   

2. Then we mount the device:

   > sudo /bin/mount /dev/hidden_backup /tmp/hidden_backup
   

We check in the mounted directory: 

<img width="149" height="26" alt="Screenshot 2026-02-01 alle 10 52 41" src="https://github.com/user-attachments/assets/8b39e7ce-5d32-4ff0-8d34-92870b5560fc" />



we find a file named `creds.txt`:

<img width="212" height="37" alt="Screenshot 2026-02-01 alle 10 53 33" src="https://github.com/user-attachments/assets/a99bf6d3-29c0-427a-b9b9-db2c1e3fe611" />


This file contains valid root and willow credentials.


#### Root Access

Using the discovered credentials, we successfully authenticate as root.
However, checking for the root flag reveals a surprise.

Inside `/root/root.txt`, a message indicates that the root flag was already given in the past.

<img width="745" height="53" alt="Screenshot 2026-02-01 alle 10 54 42" src="https://github.com/user-attachments/assets/efed5e55-1373-45a3-beed-1296ae61ccbd" />

This strongly suggests the flag is hidden elsewhere.


#### Steganography Analysis

Given the previous discovery of `user.jpg`, we suspect hidden data.

Using `steghide`, we analyze the image:

<img width="489" height="192" alt="Screenshot 2026-02-01 alle 10 56 07" src="https://github.com/user-attachments/assets/8ee84858-1203-44fc-a076-67b78438510c" />


Using the root passphrase, we can now confirm that `root.txt` hidden in the file exists. 


#### Extracting the Root Flag

We extract the embedded file running `steghide extract`:

<img width="338" height="88" alt="Screenshot 2026-02-01 alle 10 57 14" src="https://github.com/user-attachments/assets/3fe57ea6-d3af-45fa-8c49-936347c0fc63" />




The extracted `root.txt` contains the final root flag.

<img width="335" height="61" alt="Screenshot 2026-02-01 alle 10 57 27" src="https://github.com/user-attachments/assets/ea92da24-7e3e-4f15-bce1-05ae98810d98" />










