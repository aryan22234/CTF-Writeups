# 🔐 TryHackMe — DarkMatter

> **Difficulty:** Easy | **OS:** N/A | **Category:** Cryptography / RSA / Ransomware Analysis
> **Author:** Aryan Barkhordar

---

## Table of Contents

- [Overview](#overview)
- [Reconnaissance](#reconnaissance)
  * [Identifying the RSA Parameters](#identifying-the-rsa-parameters)
  * [RSA Background](#rsa-background)
- [Exploitation](#exploitation)
  * [Factoring the RSA Modulus — FactorDB](#factoring-the-rsa-modulus--factordb)
  * [Calculating Euler's Totient](#calculating-eulers-totient)
  * [Recovering the Private Exponent](#recovering-the-private-exponent)
  * [Decrypting the Ransomware-Encrypted File](#decrypting-the-ransomware-encrypted-file)
- [Post-Exploitation](#post-exploitation)
  * [Flag Retrieval](#flag-retrieval)
- [Flags](#flags)
- [Key Takeaways](#key-takeaways)
- [Summary](#summary)

---

## Overview

**DarkMatter** is a TryHackMe room centred on **RSA cryptanalysis** in the context of a ransomware investigation. The challenge presents a scenario where a file has been encrypted with RSA, and the public key parameters are known. The objective is to recover the RSA private key and use it to decrypt the encrypted file. The attack is made possible by a critically weak choice of prime factors — the RSA modulus `n` can be factored using **FactorDB**, a public database of known factorisations. With `p` and `q` recovered, the private exponent `d` is derived mathematically using Euler's totient and the modular multiplicative inverse, allowing the encrypted file to be decrypted and the flag retrieved.

---

## Reconnaissance

### Identifying the RSA Parameters

The RSA public key parameters were obtained as part of the investigation:

```
n = 340282366920938460843936948965011886881
e = 65537
```

These values form the **RSA public key**:

| Parameter | Role |
|---|---|
| `n` | RSA modulus — the product of two large prime numbers `p` and `q` |
| `e` | Public exponent — used during RSA encryption; `65537` is a standard value |

> **Key Observation:** The security of RSA depends entirely on the difficulty of factoring `n`. If `n` can be decomposed into its prime factors `p` and `q`, the private key can be mathematically reconstructed. Normally this is computationally infeasible — but only when `p` and `q` are sufficiently large and randomly chosen.

---

### RSA Background

RSA is an **asymmetric encryption algorithm** — it uses a mathematically linked key pair:

- The **public key** `(n, e)` is used to encrypt data.
- The **private key** `(n, d)` is used to decrypt it.

The private exponent `d` is derived from the prime factors of `n`. If those factors are known, `d` can be calculated directly. This is the attack vector exploited in DarkMatter.

---

## Exploitation

### Factoring the RSA Modulus — FactorDB

[FactorDB](http://factordb.com) is a public database that stores factorisations of integers. It was queried with the value of `n` to determine whether its prime factors were already known:

```
n = 340282366920938460843936948965011886881
```

FactorDB returned the two prime factors immediately:

```
p = 18446744073709551533
q = 18446744073709551557
```

> **Key Finding:** The modulus was composed of two 64-bit primes — far smaller than the 1024-bit minimum recommended for RSA security. Numbers of this size are trivially factorable, and FactorDB had already catalogued them. This is a **catastrophically weak** key by modern standards.

The factorisation can be verified:

```python
p = 18446744073709551533
q = 18446744073709551557
print(p * q)
# 340282366920938460843936948965011886881 ✓
```

---

### Calculating Euler's Totient

With both prime factors recovered, **Euler's totient function** φ(n) was calculated. For RSA with two distinct prime factors, the formula is:

$$\phi(n) = (p - 1)(q - 1)$$

```python
p = 18446744073709551533
q = 18446744073709551557

phi = (p - 1) * (q - 1)
print("Phi:", phi)
```

```
Phi: 340282366920938460807043460817592783792
```

> **Key Concept:** Euler's totient φ(n) gives the count of integers less than `n` that are coprime to `n`. In RSA, it is used to define the mathematical relationship between the public and private exponents.

---

### Recovering the Private Exponent

The **private exponent** `d` is the modular multiplicative inverse of the public exponent `e` with respect to φ(n):

$$d = e^{-1} \mod \phi(n)$$

This means `d` satisfies:

$$e \cdot d \equiv 1 \pmod{\phi(n)}$$

Python's built-in `pow()` function computes this directly:

```python
e = 65537
d = pow(e, -1, phi)
print("Private exponent:", d)
```

```
Private exponent: 196442361873243903843228745541797845217
```

> **Key Observation:** Python's `pow(e, -1, phi)` computes the modular inverse natively using the extended Euclidean algorithm — no external libraries required.

The RSA private key was now fully reconstructed:

```
n = 340282366920938460843936948965011886881
d = 196442361873243903843228745541797845217
```

---

### Decrypting the Ransomware-Encrypted File

With the private key parameters `(n, d)` recovered, they were used to decrypt the ransomware-encrypted file. RSA decryption applies the private exponent to the ciphertext:

$$m = c^d \mod n$$

The decryption operation was applied to the encrypted file, restoring the original plaintext content and recovering access to the previously locked document.

---

## Post-Exploitation

### Flag Retrieval

The decrypted output was `student_grades.docx`. Opening the document revealed the flag embedded within its contents, completing the DarkMatter room.

---

## Flags

| Flag        | Value           |
| ----------- | --------------- |
| 🚩 **Flag** | `THM{REDACTED}` |

---

## Key Takeaways

| Concept | What Was Demonstrated |
|---|---|
| **RSA** | Asymmetric cryptographic algorithm based on the difficulty of factoring large numbers |
| **FactorDB** | Public database for identifying known prime factorisations of integers |
| **Euler's Totient** | φ(n) = (p−1)(q−1) — required to compute the private exponent |
| **Modular Inverse** | `d = pow(e, -1, phi)` — recovers the RSA private exponent from known factors |
| **RSA Private Key** | Reconstructed from `n` and `d` — sufficient to decrypt any data encrypted with the corresponding public key |
| **Ransomware Analysis** | Cryptographic weaknesses in key generation can allow full recovery of encrypted files |

**The core lesson from DarkMatter:** RSA encryption is only as secure as the underlying key material. The algorithm itself is mathematically sound — but if the modulus `n` can be factored, the private key can be fully recovered, and all encrypted data is compromised. Weak prime selection, reuse of known primes, or small key sizes are all conditions that make this attack viable. A minimum of 2048-bit RSA keys is recommended for modern deployments, with 4096-bit preferred for sensitive use cases.

---

## Summary

| Stage                    | Technique                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------ |
| **Recon**                | RSA public key parameters `(n, e)` obtained from challenge                          |
| **Factorisation**        | FactorDB query on `n` → `p` and `q` recovered immediately                           |
| **Totient Calculation**  | `phi = (p-1) * (q-1)` computed in Python                                            |
| **Key Recovery**         | `d = pow(e, -1, phi)` → RSA private exponent derived                                |
| **Decryption**           | Private key `(n, d)` used to decrypt ransomware-encrypted file                       |
| **Flag**                 | Flag retrieved from decrypted `student_grades.docx`                                  |

---

*Writeup by Aryan Barkhordar*
