
![boilerCTF](https://github.com/user-attachments/assets/3c9e20a1-1d83-472d-a64e-83935ede5c90)

# Boiler CTF - Intermediate level CTF 


## Task 1: 

### Port Scanning

- First, we run an Nmap scan to identify open ports on the target machine: 


<img width="533" height="230" alt="Nmap-scan" src="https://github.com/user-attachments/assets/a5ac8676-af80-473c-b3d5-faa689f48263" />


From the scan results, we can see that the following ports are open:
	•	21 – FTP
	•	80 – HTTP
	•	10000 – HTTP (Webmin)
	•	55007 – SSH
  

### FTP Enumeration

- Next, we attempt an anonymous login on the FTP service:

<img width="252" height="126" alt="Screenshot 2026-01-28 alle 09 45 52" src="https://github.com/user-attachments/assets/6da9b897-f1dd-43dd-8ba6-e68618c28eb1" />

Anonymous login is allowed. 

Listing the contents of the current directory reveals a file.
- The file extension of this file is the answer to Question 1.


<img width="429" height="183" alt="Screenshot 2026-01-28 alle 09 48 21" src="https://github.com/user-attachments/assets/7c1a261c-613a-456a-8c33-f8bc61147e24" />


### Service Version & Script Scan

- We then run a version and default script scan (-sV -sC) to identify the services and their versions:


<img width="581" height="468" alt="Screenshot 2026-01-28 alle 09 57 47" src="https://github.com/user-attachments/assets/85b2e651-b7bb-4011-8f0c-fee3f6b65819" />

From the scan output:

- Questions 2 and 3 can be answered directly by identifying the services and versions detected.
- Question 5 can be answered by Googling the service running on port 10000 (Webmin) and checking for known CVEs.


### Web Enumeration

- To discover hidden directories, and answer to question 6, we run a directory brute-force scan:
(You can use whatever tool of your choice)


<img width="631" height="197" alt="Screenshot 2026-01-28 alle 10 12 15" src="https://github.com/user-attachments/assets/d996650a-8b85-44cd-8022-6ee22529233c" />


Among the discovered directories, we find /joomla/.
We continue enumerating directories recursively inside it.

After checking multiple directories, the folder /joomla/_test/ stands out.
Inside it, we notice an unusual application:
  **sar2html**

  
<img width="804" height="431" alt="Screenshot 2026-01-28 alle 10 35 12" src="https://github.com/user-attachments/assets/297f9f03-82d6-4a3b-807d-0bbbe1342947" />


### sar2html Exploitation (RCE)

sar2html is a script used to convert system activity reports (sar) into HTML.
If exposed to the web and outdated, it is known to be vulnerable to Remote Command Execution (RCE).

After searching online, we find a public exploit on Exploit-DB.


<img width="954" height="346" alt="Screenshot 2026-01-28 alle 10 41 16" src="https://github.com/user-attachments/assets/5386b437-1fe8-4800-80cb-c2ffeeec6e8c" />


The vulnerable parameter is "plot".

After clicking on NEW, we modify the URL:

 
http://IPADDR/joomla/_test/index.php?plot=;ls 


This confirms command execution on the server.

The output of ls gives us the answer to Question 7.


## TASK 2:

### Credential Discovery

Using the RCE, we inspect files discovered during enumeration:


<img width="765" height="347" alt="Screenshot 2026-01-28 alle 10 47 34" src="https://github.com/user-attachments/assets/cb0a39ea-de3a-46dd-b998-3fb9eca2c6bb" />


Inside one of the files, we find credentials.


### SSH Access

Using the discovered credentials, we connect via SSH on the non-standard port:

<img width="252" height="15" alt="Screenshot 2026-01-28 alle 10 57 53" src="https://github.com/user-attachments/assets/7434d72c-8ac0-44f9-883a-f181e241d9cc" />

Login is successful. 


### Lateral Movement

Inside the home directory, we find a file containing another user’s password. (Question 8) 

Switching user: 

 su stoner
 
After entering the password found in the file, access is granted.

The user flag (user.txt) is located in this user’s home directory (Question 9).


### Privilege Escalation: 

#### Enumeration

Running sudo -l does not provide useful information.

We then search for SUID binaries:

find / -perm -u=s 2>/dev/null 

<img width="344" height="289" alt="Screenshot 2026-01-28 alle 11 23 48" src="https://github.com/user-attachments/assets/2dfe9080-3897-4583-bb19-e01f2f695bd8" />


Among the results, /usr/bin/find has the SUID bit set.

This is the answer to Question 10.


#### Privilege Escalation via SUID find

Using GTFOBins, we exploit find to spawn a root shell:

<img width="349" height="86" alt="Screenshot 2026-01-28 alle 11 25 12" src="https://github.com/user-attachments/assets/cbbabef4-1e2f-4d96-8565-bb8120e09333" />

We now have a root shell and we can read the root flag. 



