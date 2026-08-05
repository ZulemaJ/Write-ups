<img width="400" height="400" alt="af878fdc94fd054dd34b05b7977a6c09 (1)" src="https://github.com/user-attachments/assets/c919e78c-5429-46cb-a525-53cb27b39934" />

# CMSpit 

This is a machine that allows you to practise web app hacking and privilege escalation using recent vulnerabilities.

You've identified that the CMS installed on the web server has several vulnerabilities that allow attackers to enumerate users and change account passwords.
Your mission is to exploit these vulnerabilities and compromise the web server.

## Enumeration & Initial Access

#### Task 1: 

##### 1. What is the name of the Content Management System (CMS) installed on the server?

### Port Scanning 

Port scanning to identify open ports: 

<img width="616" height="191" alt="Screenshot 2026-01-30 alle 10 42 35" src="https://github.com/user-attachments/assets/b8c64d93-f06a-4377-8d49-e3eb2cc161ab" />

The scan output reveals 2 open ports: 
- 22 SSH
- 80 HTTP

Visiting port 80 is enough to answer the first question: 

<img width="652" height="619" alt="Screenshot 2026-01-30 alle 10 44 30" src="https://github.com/user-attachments/assets/02d63004-7ca5-4d59-8630-64bbe2287109" />


##### 2. What is the version of the Content Management System (CMS) installed on the server?

We can easily answer this question checking the source page: 

<img width="787" height="68" alt="Screenshot 2026-01-30 alle 10 45 32" src="https://github.com/user-attachments/assets/93ab9830-6d4a-459f-a87f-1e98de6de6cd" />


##### 3. What is the path that allow user enumeration?

We run `searchsploit` to check whether exploits are available for the detected CMS version.

<img width="1301" height="82" alt="Screenshot 2026-01-30 alle 10 55 01" src="https://github.com/user-attachments/assets/158fe83d-a68e-4e38-9c8c-db8a1ab7f81e" />

After inspecting the exploit code, we can easily answer question 3.



##### 4. How many users can you identify when you reproduce the user enumeration attack?

We download the exploit and execute it:

<img width="575" height="147" alt="Screenshot 2026-01-30 alle 10 57 52" src="https://github.com/user-attachments/assets/9a02e50b-1dfb-4a03-b00b-1d550d9efdcd" />

Question 4 is now answered. 


##### 5. What is the path that allows you to change user account passwords?

By reading the exploit code, we can easily find the answer to this question. 


##### 6. Compromise the Content Management System (CMS). What is Skidy's email.

During the exploitation process, requesting information about one of the enumerated users provides the answer to question 6:

<img width="1148" height="716" alt="Screenshot 2026-01-30 alle 11 01 13" src="https://github.com/user-attachments/assets/f820fe94-c593-4626-acab-66b37fd04731" />


##### 7. What is the web flag?

Now that we can retrieve any user’s password, we proceed to obtain the admin credentials.

In a real pentest scenario, changing a user’s password is not recommended, as it would be highly suspicious.
However, in this CTF context, we reset the admin password simply to save time.

After logging into the CMS with the new credentials, we search for webflag.php:


<img width="983" height="841" alt="Screenshot 2026-01-30 alle 11 17 48" src="https://github.com/user-attachments/assets/075229a4-586e-4917-b92e-609fa5ae7867" />



##### 8. Compromise the machine and enumerate collections in the document database installed in the server. What is the flag in the database?

Time to upload a shell.

We search for a directory that allows file uploads. 
The assets directory is suitable for this purpose.

We upload a php-reverse-shell.php file:


<img width="983" height="543" alt="Screenshot 2026-01-30 alle 11 08 28" src="https://github.com/user-attachments/assets/ce23d203-d558-4d75-85fa-3c00e1021a0e" />


Once uploaded, we set up a listener on our machine: 

> nc -nvlp <port>

We click on `Edit asset`, then `Direct link to asset`

<img width="636" height="409" alt="Screenshot 2026-01-30 alle 11 09 41" src="https://github.com/user-attachments/assets/220a9ce1-1566-4e45-9487-11340c3d56ce" />


We successfully got a shell on our listener:


<img width="944" height="290" alt="Screenshot 2026-01-30 alle 11 11 01" src="https://github.com/user-attachments/assets/58cb2515-b5dc-4362-80c6-65e0726f2f54" />


### Shell Stabilization

We stabilize the shell:

> /bin/bash -i

> python3 -c 'import pty; pty.spawn("/bin/bash")'


We discover that MongoDB is running on localhost:

<img width="1270" height="105" alt="Screenshot 2026-01-30 alle 11 24 37" src="https://github.com/user-attachments/assets/a36e00e1-2f8a-434a-a47f-c44387e9b4a8" />


We just type `mongo` and we'll be connected :

<img width="945" height="205" alt="Screenshot 2026-01-30 alle 11 28 38" src="https://github.com/user-attachments/assets/8a3ef7be-2840-4432-8ec3-03472fcb5fe2" />


We navigate through the MongoDB databases and collections until we find the answer to question 8:

<img width="325" height="242" alt="Screenshot 2026-01-30 alle 11 32 48" src="https://github.com/user-attachments/assets/03245d5e-fe9e-47c7-a4ac-184e295fd882" />


We also discover additional sensitive information that may be useful later:

<img width="678" height="173" alt="Screenshot 2026-01-30 alle 11 53 58" src="https://github.com/user-attachments/assets/57e5033f-ae5b-4304-a866-6d6a31dab1f1" />


### Post Exploitation & Privilege Escalation

##### 9. What is the user.txt flag?


Time to elevate privs! 

We already know that user.txt is located in `/home/stux` but we do not have permissions to read it. 

Since we previously found stux’s password in the database, we switch to that user:

> su stux


We navigate to /home/stux and easily retrieve the user.txt flag.


##### 10. What is the CVE number for the vulnerability affecting the binary assigned to the system user? Answer format: CVE-0000-0000

We simply run: 

> sudo -l

Then we Google the binary that can be executed with root privileges to identify the corresponding CVE.


##### 11. What is the utility used to create the PoC file?
##### 12. Escalate your privileges. What is the flag in root.txt?

To aswer these 2 final questions, we follow the PoC described in the [following article](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/sudo/exiftool/) 

We successfully escalate privileges and obtain root access:

<img width="409" height="68" alt="Screenshot 2026-01-30 alle 12 06 03" src="https://github.com/user-attachments/assets/98fb30d0-93ab-4c7a-9246-c12d0636bbed" />


#### Final Thoughts

This was a very easy CTF, and I would not consider it a medium-difficulty machine.













