<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/2bfd782e-e880-4ce1-947c-38d62277902c" />

# Revenge 

You've been hired by Billy Joel to get revenge on Ducky Inc...the company that fired him. 
Can you break into the server and complete your mission?

##### Billy Joel has sent you a message regarding your mission.  
Download it, read it and continue on.

<img width="882" height="256" alt="Screenshot 2026-02-02 alle 09 26 48" src="https://github.com/user-attachments/assets/04068889-e661-48a2-b822-b1f8bec0a152" />

This is revenge! 
You've been hired by Billy Joel to break into and deface the Rubber Ducky Inc. webpage. 
He was fired for probably good reasons but who cares, you're just here for the money. 

Can you fulfill your end of the bargain?


## Enumeration & Initial Access

### Port Scanning 

We enumerate open ports and services: 

<img width="796" height="328" alt="Screenshot 2026-02-02 alle 10 46 56" src="https://github.com/user-attachments/assets/a122add7-6502-4479-bbf3-4c786376f01c" />

The scan reveals the following open ports:

- 22 SSH
- 80 HHTP


### Web Enumeration 

While running `dirsearch`, we take a look to the website manually.

Something immediately catches our eye:

<img width="1038" height="146" alt="Screenshot 2026-02-02 alle 10 49 43" src="https://github.com/user-attachments/assets/09e887de-3837-4561-a741-79ef58e14b3b" />

In *Products* category, we notice a potential IDOR vector.
The application exposes a direct reference to an internal object, in this case, the product ID in the URL.

Since the parameter looks injectable, we decide to use `sqlmap`: 

> sqlmap -u http://<IPADDR>/products/1 --current-db


<img width="772" height="301" alt="Screenshot 2026-02-02 alle 11 11 41" src="https://github.com/user-attachments/assets/57759450-ce81-4244-9fd9-66d3b6678e3f" />


From the output, we discover that the database of interest is called `duckyinc`.

We re-run `sqlmap` to enumerate the tables:


> sqlmap -u http://<IPADDR>/products/1 -D duckyinc --tables


<img width="589" height="293" alt="Screenshot 2026-02-02 alle 11 13 17" src="https://github.com/user-attachments/assets/6d62d823-9675-42fe-853f-dff9b1a3cc46" />


Now, let’s dump the relevant data:


> sqlmap -u http://<IPADDR>/products/1 -D duckyinc --tables user --dump


Somewhere in this output, we find `Flag1`:


<img width="231" height="179" alt="Screenshot 2026-02-02 alle 11 17 57" src="https://github.com/user-attachments/assets/0bd65198-9e7b-4beb-8631-bd098233bdfb" />



<img width="975" height="175" alt="Screenshot 2026-02-02 alle 11 15 27" src="https://github.com/user-attachments/assets/61aeffaf-9a12-44d9-a27d-b6917656da40" />


And there it is, along with password hashes.


### Password Cracking

We crack the extracted hashes using `John the Ripper`:

<img width="857" height="191" alt="Screenshot 2026-02-02 alle 11 16 25" src="https://github.com/user-attachments/assets/0af16381-3a3c-4f94-94b4-325ab4dead71" />


The password is successfully recovered.



### SSH Access 

Now it’s time to access the server.

We connect via SSH using the `server-admin` credentials we just obtained.


<img width="739" height="534" alt="Screenshot 2026-02-02 alle 11 17 11" src="https://github.com/user-attachments/assets/aedb6884-034d-47b2-897a-5d79baa83351" />



Once logged in, we immediately find `Flag2.txt`.



### Privilege Escalation 

Next step: privilege escalation.

We check sudo permissions: 

> `sudo -l`

<img width="1704" height="122" alt="Screenshot 2026-02-02 alle 11 19 50" src="https://github.com/user-attachments/assets/64b184ac-efb6-4185-95bc-f744cb5f1b47" />


Interesting output, the user can run the following commands as root:

````
/bin/systemctl start duckyinc.service
/bin/systemctl enable duckyinc.service
/bin/systemctl restart duckyinc.service
/bin/systemctl daemon-reload
sudoedit /etc/systemd/system/duckyinc.service
````

The key point here is `sudoedit` on `/etc/systemd/system/duckyinc.service`.


### Abusing systemd service

We inspect the service file:

> sudoedit /etc/systemd/system/duckyinc.service


That's what we find: 


<img width="1130" height="395" alt="Screenshot 2026-02-02 alle 10 08 18" src="https://github.com/user-attachments/assets/9beb4b9b-4e1e-43ef-8cbf-a2b72daf3430" />


Inside the editor, we modify the `ExecStart` directive in the `[Service]` section,  to execute a command as root:

We replace it with something like: 


````
[Service]
User=root
ExecStart=/bin/bash -c "chmod +s /bin/bash"

````


To apply the changes, we notify systemd and restart the service:


> sudo /bin/systemctl daemon-reload


> sudo /bin/systemctl restart duckyinc.service


Once the service restarts, we spawn a privileged shell:

> `/bin/bash -p`

And we now have root privileges.


<img width="690" height="102" alt="Screenshot 2026-02-02 alle 11 27 10" src="https://github.com/user-attachments/assets/69c4bc0c-be6c-42b9-a0ff-8e74db7458b7" />


### Final Step — Defacement

At this point, searching `Flag3.txt` won't work. 

Why?
Because the task explicitly requires defacing the front page of the website.


So we move to:

> `/var/www/duckyinc/templates/index.html` 

We overwrite the file as requested.

After defacing the page, the final flag appears in the `/root` directory.


<img width="272" height="67" alt="Screenshot 2026-02-02 alle 11 30 10" src="https://github.com/user-attachments/assets/ae9c6c6c-8212-4f8a-9561-11713984a514" />



Mission completed.
