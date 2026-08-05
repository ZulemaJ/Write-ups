<img width="300" height="300" alt="image" src="https://github.com/user-attachments/assets/2a06df12-2c3c-46f1-b9b0-e9b77ac9d599" />

# EMPLINE 
Are you good enough to apply for this job? 

## Enumeration & Initial Access 

### Port Scanning

We start with a standard Nmap scan to identify open ports on the target:

<img width="428" height="159" alt="Screenshot 2026-01-28 alle 12 09 11" src="https://github.com/user-attachments/assets/3f6aaa2c-09ba-4483-ba8b-07912eeb56f6" />

From the output, we can observe the following open ports:
- 22 SSH
- 80 HTTP
- 3306 Mysql


### Service Version & Script Scan 

Next, we run a service version scan with Nmap’s default scripts to further investigate the services running on the open ports:

<img width="1258" height="375" alt="Screenshot 2026-01-28 alle 12 09 36" src="https://github.com/user-attachments/assets/90b6f2f6-2d93-4b57-9fc2-642732fcebd5" />

Upon reviewing the results, we don’t find any immediate vulnerabilities in the service versions.

### Web Enumeration 

Checking port 80 reveals a basic website with no noticeable vulnerabilities. 
We proceed by running dirb and dirsearch to find hidden directories: 

<img width="482" height="405" alt="Screenshot 2026-01-28 alle 12 30 10" src="https://github.com/user-attachments/assets/b04fd3dd-30f6-4b8e-bd7c-7d826b0ded7b" />

Despite trying different wordlists, we don’t uncover any useful information.


#### VHOST Enumeration

We then attempt a VHOST bruteforce to see if there are any additional subdomains hidden:

<img width="937" height="309" alt="Screenshot 2026-01-28 alle 12 32 42" src="https://github.com/user-attachments/assets/ae77a70c-47a5-40f3-978f-44350ceafa3f" />


This approach reveals the subdomain:

- job.empline.thm 

Visiting this subdomain brings us to a login page running OpenCATS v0.9.4.


<img width="301" height="279" alt="Screenshot 2026-01-28 alle 12 36 42" src="https://github.com/user-attachments/assets/4718a709-9de5-4d7f-b216-a67a35e1139b" />


A quick search on searchsploit reveals a Remote Code Execution (RCE) vulnerability via Unrestricted File Upload for this version:


<img width="440" height="97" alt="Screenshot 2026-01-28 alle 12 37 56" src="https://github.com/user-attachments/assets/8ab7ee9f-0c00-4a10-b2c5-2b9a02b853e2" />


### Exploiting OpenCATS 0.9.4

This vulnerability allows us to exploit an Unrestricted File Upload. 
By prepending GIF87a to a PHP web shell, we bypass the basic file-type validation and upload the shell as a “resume” linked to an active job ID. 
This file is stored in a publicly accessible directory, which gives us the ability to execute arbitrary system commands.

After successfully uploading the malicious file, we gain a shell:

<img width="687" height="284" alt="Screenshot 2026-01-28 alle 12 47 20" src="https://github.com/user-attachments/assets/4c3e2c5b-6f70-403c-ab8e-3d11096a2ae6" />


## Post-Exploitation

### Lateral Movement:

Exploring the system further, we check the /etc/passwd file and find an interesting user: **george**.


We then search through the /var/www/opencats directory and discover a config.php file containing credentials for the OpenCATS portal and the MySQL database.

<img width="537" height="469" alt="Screenshot 2026-01-28 alle 12 56 02" src="https://github.com/user-attachments/assets/3248ae9a-0632-42bd-893e-7d69f2cebf03" />


We proceed by connecting to the MySQL database:

<img width="489" height="134" alt="Screenshot 2026-01-28 alle 13 24 06" src="https://github.com/user-attachments/assets/00712807-9722-450f-8247-45f08136f5be" />


After some digging, we find encrypted credentials in the user tables (MD5 hash):


<img width="483" height="112" alt="Screenshot 2026-01-28 alle 13 25 51" src="https://github.com/user-attachments/assets/3f8ea550-22d7-466e-ba50-55cc10ee285b" />



We use John the Ripper to decrypt the hash:

<img width="609" height="146" alt="Screenshot 2026-01-28 alle 13 31 12" src="https://github.com/user-attachments/assets/0de1d459-721f-43b8-9956-1c6fc6ca0e82" />


With the decrypted password, we attempt to log in via SSH and gain access to the system.

<img width="237" height="28" alt="Screenshot 2026-01-28 alle 13 31 54" src="https://github.com/user-attachments/assets/52a26978-2081-42f1-bfdd-da06e161c369" />



We now have access to the george user’s home directory and the first flag.


### Privilege Escalation: 

Now, we focus on escalating our privileges. 
Running sudo -l reveals that the george user cannot run sudo commands on the empline system.


Next, we search for misconfigured Linux capabilities. Using the following command:

getcap / -r 2>/dev/null

<img width="314" height="40" alt="Screenshot 2026-01-28 alle 13 50 57" src="https://github.com/user-attachments/assets/8f260ec3-6c2e-475d-9e94-0bd0ce41a97a" />

We identify a dangerous capability on the Ruby binary:

/usr/local/bin/ruby = cap_chown+ep

The cap_chown capability allows a process to change the ownership of any file on the system, posing a major security risk.

#### Hijacking 

The goal is to use Ruby to take ownership of the /etc/passwd file, which stores user account information. 
Once we own the file, we can modify it to add a new root-level user.

##### - Step 1: Change File Ownership
We use a Ruby one-liner to change the owner of /etc/passwd to our current UID (george).

<img width="594" height="14" alt="Screenshot 2026-01-28 alle 13 56 29" src="https://github.com/user-attachments/assets/41d18278-0d31-4db5-81de-a4d7fe1c8698" />


#### - Step 2: Verify the Hijack 
By running ls -l /etc/passwd, we can confirm that the file owner has changed from root to george.

<img width="348" height="26" alt="Screenshot 2026-01-28 alle 13 56 57" src="https://github.com/user-attachments/assets/23344fa5-8f6d-4275-a75c-e52c6d182547" />


#### - Step 3: Generate a Password Hash 
We need a valid password hash to insert into the file. Using openssl, we generate an MD5-based hash for the password password123.

<img width="365" height="26" alt="Screenshot 2026-01-28 alle 13 57 26" src="https://github.com/user-attachments/assets/7c8ce5ab-208e-4218-bb5e-951cbedeaab8" />


#### - Step 4: Injecting a New Root User 
Since we now own /etc/passwd, we can append a new entry. 
We create a user named zulema and, crucially, assign them UID 0 and GID 0, which are the identifiers for the root account.

<img width="625" height="15" alt="Screenshot 2026-01-28 alle 13 58 02" src="https://github.com/user-attachments/assets/3a99f9b7-60f3-40b7-b3a7-d00fa54e0153" />

#### Obtaining the Root Shell
Finally, we use the su (substitute user) command to switch to our newly created account.

<img width="177" height="25" alt="Screenshot 2026-01-28 alle 13 58 38" src="https://github.com/user-attachments/assets/81085233-d5bf-44f2-a9f3-abae597d55a5" />


After entering the password password123, we successfully escalated our privileges to root.

<img width="188" height="67" alt="Screenshot 2026-01-28 alle 13 59 05" src="https://github.com/user-attachments/assets/3c9512a3-1d2d-43ae-a32d-a6573b89d6e4" />


We can now get the last flag in /root directory. 
















