# 🛡️ Security-Labs

Personal pentesting lab — write-ups, methodology notes, and proof-of-concept work across CTF platforms and self-hosted environments. Every machine documented here was rooted manually, no automated exploit chains.

Focus: enumeration discipline, privilege escalation paths, and understanding *why* a vulnerability exists, not just that it does.

---

## Labs & Write-ups

| Machine                                                   | Platform   | Difficulty  | Key Techniques                                                 | Status |
| :-------------------------------------------------------- | :--------- | :---------- | :------------------------------------------------------------- | :----- |
| [Fowsniff](./TryHackMe/Fowsniff/)                         | TryHackMe  | Easy/Medium | OSINT, POP3 bruteforce, Python reverse shell                   | ✅      |
| [Common Linux PrivEsc](./TryHackMe/Common-Linux-PrivEsc/) | TryHackMe  | Easy        | SUID, sudo abuse, cron jobs, PATH hijacking, /etc/passwd write | ✅      |
| [Basic Pentesting](./TryHackMe/Basic-Pentesting/)         | TryHackMe  | Easy        | SMB enumeration, SSH bruteforce, RSA key cracking              | ✅      |
| [NodeClimb](./DockerLabs/NodeClimb/)                      | DockerLabs | Easy        | Anonymous FTP, zip2john, sudo node GTFOBins                    | ✅      |
| [Vacaciones](./DockerLabs/Vacaciones/)                    | DockerLabs | Very Easy   | SSH bruteforce, user pivoting, sudo ruby GTFOBins              | ✅      |
---

## Toolset

| Category | Tools |
| :------- | :---- |
| Recon & Enumeration | Nmap, Gobuster, enum4linux, smbclient |
| Exploitation | Metasploit, Netcat, Python scripting |
| Credential Attacks | Hydra, John the Ripper, rockyou / SecLists |
| Active Directory | BloodHound (learning), Kerberos abuse, GPO analysis |
| Environment | Fedora, Distrobox (Kali), OpenVPN |

---

## Current Focus

Working through [TryHackMe](https://tryhackme.com) learning paths while building toward **CompTIA Security+**. Next milestone: HackTheBox after cert.

Longer-term target: Jr. Pentester or SOC Analyst role, with a preference for offensive work.

---

## Related Repos

| Repo | Description |
| :--- | :---------- |
| [Active Directory Home Lab](https://github.com/GrooveRoot/active-directory-home-lab.git) | AD deployment from scratch — GPO hardening, Kerberos, PrivEsc paths |

---

## Notes on Methodology

Write-ups here follow a consistent structure: recon → enumeration → foothold → post-exploitation → lessons learned. The goal isn't just to document *what* worked — it's to explain *why* the attack surface existed and what a defender would need to fix.

Notes are also kept locally in Obsidian for faster iteration during active labs.
