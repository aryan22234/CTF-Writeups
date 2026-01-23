1. Initial Enumeration

A full TCP port scan revealed only a single open port:

- TCP 22 (SSH)

<img width="759" height="223" alt="image" src="https://github.com/user-attachments/assets/ab49d2e8-5d97-4126-94f2-35983b674e8b" />

Given the lack of TCP services, a UDP scan was performed, which revealed:

- UDP 500 – ISAKMP/IKE

<img width="722" height="252" alt="image" src="https://github.com/user-attachments/assets/1d846226-a42b-44d0-abba-5505cf8f2de3" />

2. ISAKMP Enumeration

To enumerate the IKE service, I used:

**"_ike-scan -A [Machine IP]_"**

This returned useful information, including the presence of a user named ike.

<img width="1099" height="144" alt="image" src="https://github.com/user-attachments/assets/018f8517-d6d9-4d35-9516-e7193a5c14e3" />

To extract the PSK hash, I ran:

_**"sudo ike-scan -A --id=ike@expressway.htb -Ppresharedkey.txt [Machine IP]"**_

<img width="1093" height="177" alt="image" src="https://github.com/user-attachments/assets/c4a3372b-02ff-44b4-8a94-1a202569a96e" />

3. Cracking the PSK

Using a dictionary attack with RockYou:

**_"sudo psk-crack -d /usr/share/wordlists/rockyou.txt presharedkey.txt"_**

The key was successfully cracked.

<img width="737" height="107" alt="image" src="https://github.com/user-attachments/assets/bf3141e0-172b-4c8a-8c7d-bba4873f5be9" />

4. User Access

Using the recovered credentials, I logged in via SSH as ike.

I was able to retrieve the user flag.

<img width="260" height="32" alt="image" src="https://github.com/user-attachments/assets/8cf85e98-7e7a-4997-8049-1ead4b604bc6" />

5. Privilege Escalation

Standard privilege escalation enumeration was performed:

- sudo -l
- SUID binary checks
- Manual filesystem review

None of these were revealed anything interesting.

6. Automated Enumeration

I executed linPEAS to identify possible escalation vectors.

<img width="785" height="60" alt="image" src="https://github.com/user-attachments/assets/e9c21d76-2089-4248-90fb-888bbefac5dd" />

7. Vulnerability Identification

The sudo version installed on the system was vulnerable to CVE-2025-32463

8. Exploitation

A public proof-of-concept exploit for CVE-2025-32463 was downloaded, transferred to the target machine, and executed.

<img width="665" height="339" alt="image" src="https://github.com/user-attachments/assets/9495f3b6-1fd3-4703-8c71-3ad4a7928b28" />

9. Root Access

The exploit resulted in a root shell. I was then able to retrieve the root flag.

<img width="294" height="31" alt="image" src="https://github.com/user-attachments/assets/aa7039a9-1749-47ff-8357-2b96c38f7a47" />
