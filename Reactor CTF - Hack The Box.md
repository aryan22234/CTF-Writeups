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
<img width="1105" height="578" alt="image" src="https://github.com/user-attachments/assets/f0bc5503-bb0b-4bf2-a22e-81403f7a2b33" />


---

### Web Enumeration

Navigating to `http://<TARGET>:3000` presented a web frontend identifying itself as:

```
REACTORWATCH CORE MONITORING SYSTEM v3.2.1
```

The application appeared to be a front-end dashboard for a SCADA (Supervisory Control and Data Acquisition) system. The UI was largely *non-functional* — most controls and panels were static or non-interactive.

<!-- 📸 IMAGE: ReactorWatch web interface on port 3000 -->
<img width="1890" height="913" alt="image" src="https://github.com/user-attachments/assets/479b6938-04c4-4dd7-9bc9-101b9809eba1" />


**Directory Bruteforcing with Gobuster:**

```bash
gobuster dir -u http://<TARGET>:3000 -w /usr/share/wordlists/dirb/common.txt
```

```
/cgi-bin   (Status: 403)
```

<!-- 📸 IMAGE: Gobuster scan output -->
<img width="670" height="341" alt="image" src="https://github.com/user-attachments/assets/8d0a0580-4c85-4d99-857d-20b4602b51b6" />



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
<img width="1093" height="733" alt="image" src="https://github.com/user-attachments/assets/7d2b7a6a-19b0-4c56-96a6-12031aab6063" />


<!-- 📸 IMAGE: RCE confirmed — command output on the server -->

<img width="1225" height="748" alt="image" src="https://github.com/user-attachments/assets/31c22f4d-cccc-4ebb-8658-2dbfe57386e5" />

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
<img width="1225" height="786" alt="image" src="https://github.com/user-attachments/assets/1a0426db-025f-407d-88a5-bd514b617982" />

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
<img width="752" height="170" alt="image" src="https://github.com/user-attachments/assets/6460381e-a2dc-4c15-8c49-0cf457fe9aee" />


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
<img width="436" height="290" alt="image" src="https://github.com/user-attachments/assets/f0f3f35d-1ce3-49fa-a1bb-6978b8330f17" />


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
<img width="968" height="52" alt="image" src="https://github.com/user-attachments/assets/bf04e0d4-c30d-45d8-8361-e8ea49bff8b2" />


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
<img width="572" height="32" alt="image" src="https://github.com/user-attachments/assets/2f2940e4-9e6e-49c5-8fcc-80cb9eec7754" />


<!-- 📸 IMAGE: process.getuid() returning 0 confirming root -->
<img width="944" height="116" alt="image" src="https://github.com/user-attachments/assets/3cd3bfa9-e769-4795-8878-4f30cb2a4dcc" />



With root level code execution available via the inspector, the **root flag** was read directly from the filesystem when privileges were successfully escalated

<!-- 📸 IMAGE: Root flag captured -->
<img width="661" height="310" alt="image" src="https://github.com/user-attachments/assets/370bec3b-4a63-4168-84c5-031754afb7db" />


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
